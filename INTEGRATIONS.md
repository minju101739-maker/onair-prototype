# 캘린더 연동 종합 & 실현 가능성 판단

> 이 시스템의 캘린더 연동은 일반 연동과 판단 기준이 다르다. 핵심 제약([CONCEPT P1·P2](./CONCEPT.md))은
> **"free/busy(바쁨/한가)만 읽고, 이벤트 제목·참석자·사유 같은 상세는 구조적으로 못 보게 한다"** 이다.
> 따라서 질문은 "연동 가능한가?"가 아니라 **"프라이버시 경계를 지키며 연동 가능한가?"** 이다.

- 문서 버전: v0.1
- 작성일: 2026-07-04 (사실 확인: 공식 문서 기준, 하단 출처)
- 상위 문서: [CONCEPT.md](./CONCEPT.md) §4.1(비열람의 기술적 경계) · 관련: [ARCHITECTURE.md](./ARCHITECTURE.md)(동기화 파이프라인·DB 스키마·API — 여기서 판정된 소스가 실제로 꽂히는 구조), [DATA-SOURCING.md](./DATA-SOURCING.md)

---

## 1. 판단 기준 — "구조적 프라이버시" vs "정책적 프라이버시"

| 등급 | 의미 | 강도 |
|------|------|------|
| **구조적(structural)** | OAuth 스코프 자체가 상세 접근을 **원천 차단**. free/busy 전용 권한만 요청 → 우리가 실수해도 제목을 못 읽음 | 강함 (P2 이상적) |
| **정책적(policy)** | 스코프는 상세까지 허용하지만, 우리가 **free/busy 엔드포인트만 호출**하기로 약속. 기술적으론 제목 접근 가능 | 약함 (감사·거버넌스 필요) |

이 구분이 소스별 등급을 가른다. **가능하면 구조적 등급 소스를 우선**하고, 정책적 소스는 "우리는 상세를 안 읽는다"를 감사 로그·최소권한 운영으로 보증해야 한다.

부가 판단축: **인증 모델**(OAuth = 깔끔·위임철회 가능 / 앱 암호·Basic = 취약), **실시간 갱신**(webhook 지원 여부), **쓰기**(이벤트 생성·화상 링크 자동 생성).

---

## 2. 종합 판정표

| 소스 | API / 프로토콜 | free/busy만? | 인증 | 프라이버시 등급 | 실시간 | 판정 |
|------|----------------|-------------|------|----------------|--------|------|
| **Google Calendar** | Calendar API `freeBusy.query` | ✅ 전용 스코프 | OAuth 2.0 | **구조적** (`calendar.freebusy`) | webhook ✅ | ⭐ **최적** |
| **Microsoft 365 / Outlook(업무)** | Graph `getSchedule` | ✅ (응답은 f/b) | OAuth 2.0 (Entra) | 정책적 (`Calendars.Read` = 상세도 허용) | webhook ✅ | ✅ **가능** (스코프 거침) |
| **Outlook.com(개인)** | Graph `getSchedule` | ✅ | OAuth 2.0 | 정책적 | ✅ | ✅ 가능 |
| **Microsoft Teams** | = 위 Graph 캘린더(Exchange) | ✅ | OAuth 2.0 | 정책적 | ✅ | ✅ (별도 캘린더 아님) |
| **Exchange 사내서버(on-prem)** | EWS `GetUserAvailability` / Graph | ✅ | Basic/OAuth | 정책적 | 제한 | ⚠️ EWS 지양(하단 §4) |
| **Apple / iCloud 캘린더** | CalDAV `free-busy-query` | ✅(서버 지원 시) | **앱 암호**(OAuth 없음) | 정책/구조 혼재 | webhook ❌ | ⚠️ **제한적** |
| **범용 CalDAV**(Fastmail·Nextcloud·Radicale·Yahoo) | CalDAV REPORT | ✅(편차) | 서버마다 다름 | 서버 의존 | ❌ | ✅ 표준, 편차 큼 |
| **ICS 구독 피드(.ics)** | HTTP 폴링 | ❌ 상세 포함 | 비밀 URL | **약함** | ❌(지연) | ⚠️ 폴백만 |
| **NAVER WORKS**(국내 B2B) | Works Calendar API | 이벤트 읽기(파생) | OAuth 2.0 | 정책적 | 일부 | ✅ 국내 적합 |
| **Naver 캘린더(소비자)** | 네이버 로그인 OpenAPI | ❌ 쓰기 중심 | OAuth 2.0 | — | ❌ | ✖ f/b 읽기 부적합 |
| **Kakao 톡캘린더** | Kakao 톡캘린더 REST | 이벤트 읽기(파생) | OAuth 2.0 | 정책적 | ❌ | △ 소비자용 가능 |
| **Kakao Work** | Web API(RPC, 봇) | ✖ 캘린더 소스 아님 | 토큰 | — | — | ✖ (외부 캘린더를 소비만) |

---

## 3. 소스별 상세 판단

### 3.1 Google Calendar — ⭐ 최적, 유일한 완전 구조적 소스
- `freeBusy.query` (POST `/calendar/v3/freeBusy`)는 **바쁨 구간의 start/end만** 반환한다 — 제목·참석자·장소는 애초에 응답에 없음. 한 번에 최대 50개 캘린더.
- 결정적 이점: **전용 스코프 `https://www.googleapis.com/auth/calendar.freebusy`**("View your availability in your calendars")가 실제로 존재한다. 이 스코프만 요청하면 우리 시스템은 **구조적으로 이벤트 상세를 못 읽는다** → CONCEPT P2의 "이상적" 구현.
- webhook(push notification)으로 변경 감지 가능 → 저장 없이도 재조율 시점만 갱신.
- **판정: MVP 1순위. 이 시스템의 프라이버시 모델이 그대로 성립하는 유일한 소스.**

### 3.2 Microsoft 365 / Outlook / Teams — ✅ 가능하나 스코프가 거칠다
- Graph `getSchedule` (POST `/me/calendar/getSchedule`)가 대상들의 free/busy(`availabilityView`)를 반환. 이벤트 상세 없이 가부만 받을 수 있다.
- **한계: free/busy 전용 스코프가 없다.** getSchedule의 최소 권한이 **`Calendars.Read`** 인데, 이 스코프는 **이벤트 본문까지 읽을 수 있다.** 즉 "상세를 안 본다"가 스코프로 강제되지 않고 **우리의 자제(정책)** 에 의존한다.
  - 완화: 앱 권한 최소화 + 상세 조회 코드 부재 + 감사 로그로 "getSchedule만 호출" 보증. 거버넌스 문서 필요.
- **Teams는 별도 캘린더가 아니다.** Teams 일정 = Exchange/Outlook 캘린더를 Teams UI로 보여주는 것 → free/busy는 위 Graph 그대로. "팀즈 연동"을 따로 만들 필요 없음.
- 쓰기: 이벤트 생성은 `Calendars.ReadWrite`, **Teams 화상 링크 자동 생성은 `onlineMeetings`(`OnlineMeetings.ReadWrite`)** 별도 API.
- **판정: MVP 2순위. 기업 시장 커버리지상 필수. 단 "정책적 프라이버시"임을 명시하고 감사로 보완.**

### 3.3 Exchange 사내서버 — ⚠️ EWS에 새로 짓지 말 것
- EWS `GetUserAvailability`로 free/busy 가능하지만, **Exchange Online의 EWS는 지원 종료 수순**이다: 2026-10-01부터 비-MS 앱 차단 시작, 2027-04-01 완전 종료. (사내 Exchange Server 자체는 이번 종료 대상 아님.)
- **판정: 신규는 Graph로 통일. 순수 on-prem Exchange만 있는 조직 대응 시에만 EWS/하이브리드 고려.**

### 3.4 Apple / iCloud 캘린더 — ⚠️ 가장 까다로움
- Apple은 Google·MS 같은 **서버측 OAuth 캘린더 API가 없다.** 두 경로뿐:
  1. **CalDAV** (`caldav.icloud.com`, RFC 4791) — free/busy 컴포넌트가 스펙엔 있으나 **서버별 지원 편차**. 인증은 **앱 전용 비밀번호(app-specific password)** 만 받음 → OAuth 없음, **위임 철회·스코프 제한이 약하고** 사용자 경험도 나쁨(계정에서 16자리 비번 발급). webhook 없음(폴링).
  2. **EventKit** — **온디바이스 전용**. 서버가 못 쓰고, 우리가 **네이티브 macOS/iOS 앱을 배포해야** 접근 가능.
- **판정: 서버 연동은 CalDAV로 "가능은 하나 취약". 제대로 하려면 네이티브 클라이언트가 필요. MVP 제외, P1에서 CalDAV로 제한 지원 또는 사용자에게 "iCloud→Google 미러링" 안내.**

### 3.5 범용 CalDAV — ✅ 표준이나 품질 편차
- Fastmail·Nextcloud·Radicale·Yahoo 등. `free-busy-query` REPORT 지원 시 free/busy만 받을 수 있어 등급이 좋아질 수 있음(서버 의존). 인증은 Basic/앱암호가 흔함.
- **판정: 어댑터 하나로 다수 커버. 자체호스팅 조직·프라이버시 민감 조직에 매력적. P1.**

### 3.6 ICS 구독 피드 — ⚠️ 폴백만
- `.ics` URL을 폴링. **VEVENT에 제목 등 상세가 다 들어온다** → 가져온 뒤 버려야 하므로 **구조적 보호 불가(정책적, 그것도 약함)**. 실시간 아님(캐시 지연).
- **판정: 다른 연동이 전혀 없을 때의 최후 폴백. 기본 비권장.**

### 3.7 국내 서비스 (Acryl 등 국내 기업 맥락에서 중요)
- **NAVER WORKS** (네이버웍스, B2B 협업 스위트): OAuth 2.0 기반 **Calendar API** 존재(개발자 콘솔서 앱 등록). 개인/공유/회사 캘린더 지원. free/busy 전용 여부는 미확인 → 이벤트를 읽어 free/busy를 **파생**(스코프 거칠 가능 = 정책적). **국내 기업 고객 대응 시 유력.**
- **Naver 캘린더(소비자)**: 네이버 로그인 OpenAPI는 **일정 등록(쓰기) 중심** → free/busy 읽기 용도로는 부적합.
- **Kakao 톡캘린더**: OAuth 기반 REST로 일정 CRUD 가능(소비자). free/busy 파생 가능하나 기업 free/busy 용도론 제한적.
- **Kakao Work**: Web API는 봇/메시징 중심이고, 캘린더는 오히려 **외부(Google·Outlook·iOS)를 소비**하는 구조 → 우리 free/busy 소스로는 부적합. 다만 **알림 채널**(봇으로 확정 통보)로는 활용 가치 있음.

---

## 4. 쓰기(이벤트 생성·화상 링크) 측면 — 선택 기능

CONCEPT Q2(쓰기 자동화 여부)와 연결. free/busy 읽기와 **권한이 분리**된다:

| 작업 | Google | Microsoft | CalDAV/iCloud |
|------|--------|-----------|---------------|
| 이벤트 생성 | `calendar.events` + conferenceData(Meet 링크) | `Calendars.ReadWrite` | PUT VEVENT |
| 화상 링크 | Google Meet 자동(conferenceData) | Teams `onlineMeetings` | 외부(Zoom API 등) |

- 화상 회의(온라인 형식, [FLOW.md Q3](./FLOW.md))의 "링크 자동 생성"은 이 쓰기 권한이 있어야 실제 동작. 없으면 "링크는 주최자가 첨부" 폴백.
- 쓰기는 읽기보다 훨씬 민감하므로 **선택 기능**으로, 사용자 명시 동의 하에만.

---

## 5. 권장 로드맵

| 단계 | 연동 | 근거 |
|------|------|------|
| **MVP** | **Google Calendar**(구조적) + **Microsoft Graph**(정책적, getSchedule) | 기업 캘린더 시장의 대다수. 두 개로 커버리지 확보 |
| **P1** | 범용 **CalDAV**(iCloud 포함) 어댑터, **NAVER WORKS**(국내 B2B) | Apple·자체호스팅·국내 조직 대응 |
| **P1(쓰기)** | Google Meet / Teams onlineMeetings 링크 자동 생성 | 온라인 형식 완성 |
| **P2** | Kakao 톡캘린더(소비자), ICS 폴백, Exchange on-prem(필요 조직만) | 롱테일 |

**설계 원칙(어댑터 패턴):** 조율 엔진은 소스별 API를 몰라야 한다. 각 소스를 `getFreeBusy(user, range) → BusyInterval[]` 하나로 정규화하는 **어댑터**로 감싸고, 엔진은 그 불린 구간만 받는다([DATA-SOURCING 8](./DATA-SOURCING.md)의 RoomBook 어댑터와 동일 패턴). 소스가 늘어도 엔진·프라이버시 모델은 불변.

어댑터가 꽂히는 실제 구조 — 동기화 워커(syncToken·webhook·증분), `events` 통합 저장소, `free_busy` 프로젝션, 캘린더 연동 테이블(`calendar_connections`)과 API 설계는 [ARCHITECTURE.md §2–§4](./ARCHITECTURE.md) 참고. 사이드바 "캘린더 연동" UI(index.html)는 ON AIR 로드맵의 MVP 2종(Google 연동됨 · Microsoft 연동하기)을 표시한다.

---

## 6. 결론 (한 줄)

> **Google만이 "구조적으로 상세를 못 보는" free/busy 전용 스코프를 제공해 이 시스템의 프라이버시 모델과 완벽히 맞고, Microsoft(Teams 포함)는 free/busy를 줄 수 있으나 스코프가 거칠어 정책·감사로 보완해야 하며, Apple/iCloud는 OAuth가 없어 CalDAV(앱 암호)로만 제한적으로 가능하고, 국내는 NAVER WORKS가 유력하다.**

---

## 7. 열린 질문
- **Q-I1.** Microsoft "정책적 프라이버시"를 조직 고객에게 어떻게 보증할지 — 앱 권한 심사·감사 로그·제3자 인증(SOC2 등) 범위.
- **Q-I2.** iCloud 사용자 비중이 높은 조직에서 네이티브 클라이언트(EventKit)까지 만들지, CalDAV로 감수할지.
- **Q-I3.** NAVER WORKS·Kakao의 free/busy 전용/최소 권한 실제 지원 범위(개발자 콘솔 검증 필요).
- **Q-I4.** 쓰기(이벤트 생성) 기본 정책 — 확정 통보만 vs 캘린더 자동 등록(CONCEPT Q2).

---

## 출처
- [Google Calendar API — Freebusy: query](https://developers.google.com/workspace/calendar/api/v3/reference/freebusy/query)
- [Google Calendar API — Choose scopes (calendar.freebusy 확인)](https://developers.google.com/workspace/calendar/api/auth)
- [Microsoft Graph — calendar: getSchedule](https://learn.microsoft.com/en-us/graph/api/calendar-getschedule?view=graph-rest-1.0)
- [Microsoft Graph — Get free/busy schedule (Calendars.Read 최소 권한)](https://learn.microsoft.com/en-us/graph/outlook-get-free-busy-schedule)
- [Microsoft — Retirement of Exchange Web Services in Exchange Online](https://techcommunity.microsoft.com/blog/exchange/retirement-of-exchange-web-services-in-exchange-online/3924440)
- [Microsoft — Deprecation of EWS in Exchange Online (일정)](https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/deprecation-of-ews-exchange-online)
- [iCloud CalDAV 설정(앱 전용 비밀번호·OAuth 없음)](https://cli.nylas.com/guides/icloud-caldav-settings)
- [NAVER WORKS Developers — Calendar API](https://developers.worksmobile.com/kr/docs/calendar)
- [Kakao Developers — 톡캘린더 REST API](https://developers.kakao.com/docs/latest/ko/talkcalendar/rest-api)
