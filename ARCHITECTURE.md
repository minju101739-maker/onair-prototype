# ON AIR — 시스템 아키텍처 (데이터 소싱 · DB · API)

> 문제의식: 프로토타입(index.html)의 일정·참석자·회의실은 전부 하드코딩이다. "내 일정은 어디서 오는가", "어떤 시간이 ON AIR/피함/비어 있음인가", "회의 데이터는 어디에 어떻게 쌓이는가"에 대한 구조 정의가 없으면 플랫폼이 아니라 화면 목업에 머문다. 이 문서는 **일정의 상태등 재정의 → 데이터 소싱 파이프라인 → DB 스키마 → API → 데이터 자산화**를 정의한다.

- 문서 버전: v0.3
- 작성일: 2026-07-04 · 개정: 2026-07-09
- 상위 문서: [CONCEPT.md](./CONCEPT.md) · 관련: [INTEGRATIONS.md](./INTEGRATIONS.md)(**연동 소스 전수조사** — 소스별 실현 가능성·프라이버시 등급 판정), [DATA-SOURCING.md](./DATA-SOURCING.md)(소스별 신호 정의), [CONTEXT-MODEL.md](./CONTEXT-MODEL.md)(프라이버시 경계), [FLOW.md](./FLOW.md)(회의 수명주기)
- 구현체 반영: index.html — 일정 출처(src) 모델 · ON AIR/피함/비어 있음 상태 표시 · 내 일정 추가 · 캘린더 연동 상태 표시 · 내 기준/오늘 상태

---

## 1. 일정의 재정의 — 일정 ⊇ 회의

지금까지의 설계는 "내 일정 = 내 회의"를 암묵 전제했다. 실제 개인의 캘린더는 회의보다 넓다:

| kind | 예 | ON AIR에서의 역할 |
|---|---|---|
| `meeting` | ON AIR에서 조율·확정된 회의 | 회의 축 데이터(Meeting)와 연결 — 목적·역할·기록 보유. 화면에서는 ON AIR 점등 |
| `external` | 외부 캘린더에서 동기화된 일정 (고객 미팅, 병원 예약…) | 제목은 개인 축 — 조율 엔진에는 **바쁨**으로만 반영 |
| `personal` | 직접 추가한 개인 일정 (점심 약속 등) | 위와 동일 |
| `focus` | 집중 시간 블록 | 바쁨 반영 + 내 기준("오전 집중 보호")의 근거 데이터 |

**원칙: 개인의 일정 전체가 조율의 입력이고, 그중 회의만 ON AIR의 산출물이다.** 회의가 아닌 일정은 존재(ON AIR/바쁨)만 조율에 기여하고 내용은 개인 경계 안에 남는다 (CONCEPT P1).

### 1.1 시간 상태 프로젝션

`events`, `free_busy`, `preference_rules`, `today_status`는 화면에서 아래 상태로 합쳐진다.

| 상태 | 데이터 조건 | 조율 강도 |
|---|---|---|
| `on_air` | 연동 업무 일정, ON AIR 확정 회의, hard 개인 일정 | 하드 차단 |
| `avoid` | 내 기준 avoid, 오늘 상태, soft 개인 일정, 근무시간 밖 | 소프트 후순위 |
| `free` | 위 조건이 없는 빈 시간 | 기본 후보 |

## 2. 데이터 소싱 파이프라인 — 내 일정은 세 곳에서 온다

```
 ① 캘린더 동기화 (자동)                 ② ON AIR 확정 (자체 생성)      ③ 직접 추가 (수동)
 Google Calendar API                    조율 플로우 Phase 3 확정        POST /v1/events
 Microsoft Graph API                            │                            │
        │ OAuth2 연동                           │                            │
        ▼                                       ▼                            ▼
 ┌─ Sync Worker ──────────────┐        ┌─ Meeting Service ─┐        ┌─ Event Service ─┐
 │ 초기 전체 동기화            │        │ Meeting 생성       │        │ Event 생성       │
 │ → syncToken/deltaLink 저장 │        │ + Event(meeting)  │        │ (kind 선택)      │
 │ → watch 채널(webhook) 등록  │        │ + 초대장 발송      │        └────────┬────────┘
 │ → 푸시 수신 시 증분 동기화   │        │ + ①로 역기록(쓰기) │                 │
 └──────────┬─────────────────┘        └────────┬──────────┘                 │
            ▼                                   ▼                            ▼
        ┌──────────────────── events (통합 일정 저장소) ────────────────────────┐
        │  정규화: {source, externalId, kind, title, startsAt, endsAt, ...}    │
        └──────────────────────────────┬───────────────────────────────────────┘
                                       ▼ 파생(개인 정보 제거)
                          free_busy 프로젝션 (userId × slot → busy)
                                       ▼
                        조율 엔진은 이 프로젝션만 읽는다 — 동료의 events에 접근 불가
```

### 2.1 캘린더 연동 (①) 구체 설계

> 아래는 MVP 2종(Google·Microsoft)의 동기화 파이프라인이다. **어떤 소스가 연동 가능한지의 전수조사**(Apple/CalDAV/ICS/NAVER WORKS/Kakao 포함 11종, 구조적·정책적 프라이버시 등급 판정과 로드맵)는 [INTEGRATIONS.md](./INTEGRATIONS.md) 참고. 신규 소스는 INTEGRATIONS §5의 어댑터 패턴(`getFreeBusy(user, range) → BusyInterval[]`)으로 이 파이프라인에 추가된다.

| 항목 | Google Workspace | Microsoft 365 |
|---|---|---|
| 인증 | OAuth2 `calendar.events` (읽기·쓰기) 또는 `calendar.freebusy`(최소 스코프 모드) | Graph `Calendars.ReadWrite` |
| 초기 동기화 | `events.list` 페이지네이션 → `nextSyncToken` 저장 | `calendarView/delta` → `deltaLink` 저장 |
| 변경 감지 | `events.watch` 푸시 채널 (만료 7일 → 자동 재등록) | Graph subscriptions (만료 3일 → 자동 재등록) |
| 증분 동기화 | webhook 수신 → `syncToken`으로 delta만 조회 | webhook 수신 → `deltaLink`로 delta만 조회 |
| 폴백 | 채널 유실 대비 6시간 주기 reconcile | 동일 |
| 쓰기(역방향) | ON AIR 확정 회의를 사용자 캘린더에 이벤트로 생성, `extendedProperties.onairMeetingId`로 연결 | `singleValueExtendedProperties` 사용 |

- **최소 스코프 모드**: 조직 정책상 이벤트 내용 접근이 불가하면 free/busy 스코프만으로 연동 — 이 경우 `events.title=null`, 캘린더에는 바쁨 블록만 표시. 제품은 두 모드 모두에서 동작해야 한다.
- 반복 일정(RRULE)은 발생 인스턴스로 전개해 free_busy에 반영하되 원본 규칙을 보존한다.
- 충돌 규칙: 같은 `externalId`의 수정은 최신 `updatedAt` 승자. ON AIR가 만든 이벤트를 외부에서 삭제하면 → 해당 회의에 "캘린더에서 제거됨" 플래그, 주최자에게 알림.

## 3. DB 스키마 — 개인 축 / 회의 축 / 자원 축

프라이버시 모델(회의 축 = 투명, 개인 축 = 비열람)이 **스키마 수준에서** 강제되도록 축을 물리적으로 분리한다.

### 3.1 개인 축 (본인만 접근, RLS: `user_id = current_user`)

```sql
users               (id, org_id, name, dept, team, rank, duty, email)
calendar_connections(id, user_id, provider, scope_mode,        -- events | freebusy
                     sync_token, webhook_channel_id, webhook_expires_at,
                     last_sync_at, status)                     -- active | error | revoked
events              (id, user_id, source,                      -- google | ms | onair | manual
                     external_id, kind,                        -- meeting | external | personal | focus
                     meeting_id NULL,                          -- kind=meeting → meetings FK
                     title, starts_at, ends_at, all_day, location,
                     rrule NULL, visibility,                   -- busy_only(기본) | title_shared
                     created_at, updated_at, deleted_at)
preference_rules    (id, user_id, kind,                        -- avoid | prefer
                     anchor, weight, scope, derived_from)      -- manual | inferred | today_status
```

> 제품 표기는 `내 기준`으로 통일한다. `preference_rules`는 내부 호환을 위한 테이블명이며, `scope=today` 또는 `derived_from=today_status`가 컨디션·업무량 같은 **오늘 상태**를 표현한다.

### 3.2 파생 계층 (동료가 읽는 것은 이것뿐)

```sql
free_busy           (user_id, slot_start, slot_end, busy)      -- events에서 파생된 프로젝션.
                                                               -- 제목·출처·종류 없음. 조율 엔진의 유일한 입력.
```

### 3.3 회의 축 (참석자 전원 공개)

```sql
meetings            (id, org_id, title, status,                -- draft | coordinating | reported
                                                               -- | confirmed | closed | cancelled
                     origin, purposes[], source_ref,           -- 계기 · 목적(복수) · 출처(기록 링크)
                     format, duration_min, decision_mode, scribe_id,
                     confirmed_start NULL, room_id NULL, video_link NULL,
                     created_by, created_at)
meeting_participants(meeting_id, user_id, role,                -- decide | info | exec | fyi
                     grade,                                    -- req | opt | fyi (role에서 파생)
                     rsvp)                                     -- pending | accepted | declined
agendas             (id, meeting_id, ord, title, status,       -- open | done | carry
                     carried_from_agenda_id NULL)              -- 이월 사슬 → 반복 이월 감지
candidate_slots     (meeting_id, starts_at, room_id NULL,
                     coverage, req_coverage, penalty, grade)   -- 조율 중에만 존재(만료 시 삭제)
votes               (meeting_id, slot_key, voter_hash,         -- voter_hash = HMAC(user, meeting)
                     created_at)                               -- 중복 방지용. 역산 불가 → 익명 보장
meeting_records     (meeting_id, agenda_id, notes[], decision, status)
action_items        (id, meeting_id, agenda_id, owner_id, due, text, done)
feedbacks           (meeting_id, bucket,                       -- yes | short | async
                     created_at)                               -- user_id 없음 — 집계만 저장(N≥3 공개)
```

`meeting_participants`는 초대받은 사람만이 아니라 **주최자를 포함한 전체 참석자 명단의 단일 원천**이다. 구성원·초대·캘린더·회의 기록은 이름 문자열을 따로 저장하지 않고 모두 `users.id`를 참조한다. 주최자·응답자·정리자·액션 담당자는 `(meeting_id, user_id)` 복합 FK로 같은 회의 참석자임을 보장한다. 구체 제약은 [DB-SCHEMA.md](./DB-SCHEMA.md) v1.1을 따른다.

### 3.4 자원 축

```sql
rooms               (id, branch, floor, name, capacity, equipment[])
room_bookings       (room_id, meeting_id NULL, starts_at, ends_at, source)
```

**스키마가 곧 프라이버시 정책이다**: `votes`에 user_id가 없고, `feedbacks`에 개인 식별자가 없고, 동료 조회 경로에는 `free_busy`밖에 없다. 정책 위반이 코드 실수로도 불가능한 구조.

## 4. API 설계

| 영역 | 엔드포인트 | 설명 |
|---|---|---|
| 일정 | `GET /v1/me/schedule?from&to` | 통합 일정 (events + 확정 회의 병합) — 메인 캘린더의 데이터 소스 |
| | `POST /v1/events` | **내 일정 추가** (kind: personal/focus/external) |
| | `PATCH /v1/events/{id}` · `DELETE` | 수정·삭제 (source=onair인 회의 이벤트는 회의 취소 플로우로 리다이렉트) |
| 가용성 | `GET /v1/users/{id}/freebusy?from&to` | **동료에 대해 호출 가능한 유일한 일정 API** |
| 연동 | `POST /v1/calendar/connections` | OAuth 시작 (provider, scope_mode) |
| | `DELETE /v1/calendar/connections/{id}` | 연동 해제 → 해당 source events 소프트 삭제 |
| | `POST /v1/webhooks/{provider}` | 푸시 수신 → 증분 동기화 트리거 |
| 회의 | `POST /v1/meetings` | 발의 (Phase 0–1 페이로드: 계기·목적·안건·참석자 역할·형식) |
| | `POST /v1/meetings/{id}/candidates` | 후보 슬롯 계산 (서버: free_busy ∩ 회의실 ∩ 내 기준) |
| | `POST /v1/meetings/{id}/votes` | 익명 투표 (voter_hash 서버 생성) |
| | `POST /v1/meetings/{id}/report` | 상신 (수직 모드) |
| | `POST /v1/meetings/{id}/confirm` | 확정 → 회의실 예약 + 참석자 캘린더 역기록 + 초대장 |
| | `POST /v1/meetings/{id}/close` | 종결 카드 → meeting_records 적재 + FYI 배포 |
| 기록 | `GET /v1/records?participant=me&carry=true` | 기록 저장소 (후속 회의 생성의 진입점) |
| 인사이트 | `GET /v1/insights/meetings?team&period` | §5의 집계 지표 (개인 식별 불가 집계만) |

## 5. 회의 데이터의 자산화 — 쌓이는 데이터가 다음 회의를 줄인다

회의 수명주기의 모든 단계가 이벤트 스트림으로 적재된다 (`meeting.initiated`, `gate.diverted_async`, `vote.cast`, `meeting.confirmed`, `meeting.closed`, `agenda.carried`, `feedback.submitted`). 이 스트림에서 파생되는 지표:

| 지표 | 소스 | 환류 지점 |
|---|---|---|
| 인당 주간 회의 시간 | events(kind=meeting) 집계 | 발의 게이트: "이번 주 팀 회의 총량이 이미 N시간입니다" |
| 비동기 전환율 | gate.diverted_async / meeting.initiated | CONCEPT 성공 지표 G4 |
| 안건 이월 반복률 | agendas.carried_from 사슬 길이 | 3회 이월 → "회의로 풀 수 없는 안건" 개입 (FLOW §8.3) |
| 회의 필요도 | feedbacks 버킷 분포 | 정기 회의 재검증: "최근 3회 '비동기로 충분' 60%" |
| 성립 실패율 | 필참 declined / confirmed | 역할 남발(전원 필참) 조직 습관 감지 |
| 회의실 점유 효율 | room_bookings vs 실사용 | 자원 축 최적화 |

모든 지표는 **집계 수준**(팀·기간)으로만 노출 — 개인별 드릴다운은 제공하지 않는다 (CONCEPT N3: 감시 도구가 아니다).

## 6. 프로토타입(index.html) ↔ 아키텍처 매핑

| 프로토타입 | 대응 API/테이블 | 상태 |
|---|---|---|
| `MY_EVENTS[].src` (google/onair/manual) | `events.source` | 반영됨 — 일정 상세에 출처 표시 |
| 내 일정 추가 모달 | `POST /v1/events` | 반영됨 — 추가 즉시 free_busy 재계산(`rebuildBusy`) |
| 사이드바 캘린더 연동 상태 | `calendar_connections` | 반영됨 — 연동/미연동·마지막 동기화 표시 |
| 주간 캘린더 | `GET /v1/me/schedule` | 시뮬레이션 |
| 가용성 매트릭스 | `GET /v1/users/{id}/freebusy` | 시뮬레이션 (CONTACTS.busy) |
| 위저드 확정 | `POST .../confirm` (회의실 예약 + 역기록) | 시뮬레이션 (ROOMS_DATA.bk push) |
| 종결 카드 → 기록 | `POST .../close` → meeting_records | 시뮬레이션 (RECORDS unshift) |

## 7. MVP 우선순위

1. **P0**: events + free_busy 프로젝션 + Google 연동(읽기) + `GET /me/schedule` + `POST /events` — *일정이 진짜 데이터가 되는 최소선*
2. **P0**: meetings/participants/agendas + candidates 계산 + confirm(역기록 쓰기)
3. **P1**: votes(익명)·report(상신)·records·feedbacks
4. **P1**: Microsoft 연동, 최소 스코프(freebusy) 모드
5. **P2**: 인사이트 집계, 이월 개입, 정기 회의 재검증

## 8. 열린 질문

- **Q-A1.** free_busy 프로젝션의 슬롯 해상도 — 30분 고정인가, 조직 설정인가.
- **Q-A2.** 연동 해제 시 이미 확정된 회의의 캘린더 이벤트 처리 (남긴다/지운다).
- **Q-A3.** voter_hash의 키 관리 — 회의 종료 후 키 폐기로 완전 익명화할지.
- **Q-A4.** 멀티 캘린더(개인 Gmail + 회사 Workspace) 연결 허용 범위와 바쁨 합산 규칙.
