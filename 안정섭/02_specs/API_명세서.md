# Hana Safe Lease Passport API 명세서 초안

문서 버전: v0.3  
작성 기준: 기능명세서 기반 설명형 MVP API 초안  
작성일: 2026-04-01

---

## 0. 이 문서는 무엇을 위한 문서인가

이 문서는 Hana Safe Lease Passport 서비스를 만들 때,  
프론트엔드와 백엔드가 어떤 순서로 데이터를 주고받아야 하는지를 정리한 `API 초안`이다.

쉽게 말하면 아래 질문에 답하는 문서다.

- 사용자가 주소를 입력하면 서버는 무엇을 해야 하는가?
- 등기부 PDF를 올리면 서버는 어떤 작업을 시작하는가?
- 분석 결과는 어떤 형태로 돌려주는가?
- 리포트는 어떤 데이터 구조로 저장하는가?

이 문서는 최종 확정본이 아니다.  
현재 논의 중인 구조를 `처음 보는 사람도 이해할 수 있도록` 풀어서 정리한 설명형 초안이다.

---

## 1. 이 문서를 읽기 전에 알아두면 좋은 개념

### 1.1 API란 무엇인가

API는 화면과 서버가 대화하는 약속이다.

예를 들어 사용자가 앱에서 주소를 입력하면:

1. 화면은 서버에 "이 주소가 맞는지 확인해줘"라고 요청한다.
2. 서버는 주소를 검증하고 결과를 돌려준다.
3. 화면은 그 결과를 바탕으로 다음 화면을 보여준다.

이때 사용하는 "요청 주소", "보내는 값", "돌려받는 값"을 정리한 것이 API 명세다.

### 1.2 Request와 Response

- `Request`: 화면이 서버로 보내는 요청
- `Response`: 서버가 화면에 돌려주는 결과

예:

- Request: `주소 = 성동구 왕십리로 123`
- Response: `이 주소는 유효하고, 수도권 지원 대상입니다`

### 1.3 Resource란 무엇인가

이 문서에서 `resource`는 서버가 관리하는 핵심 데이터 단위를 뜻한다.

예:

- 사용자가 지금 분석 중인 매물
- 업로드한 등기부 문서
- 발급된 Passport 리포트

### 1.4 동기 처리와 비동기 처리

- `동기 처리`: 요청을 보내면 바로 결과가 오는 방식
- `비동기 처리`: 오래 걸리는 작업은 먼저 접수만 하고, 나중에 상태를 다시 조회하는 방식

이 프로젝트에서는:

- 주소 검증, 기본 분석은 비교적 빠르므로 동기 처리 후보
- 등기부 파싱, 정밀 분석은 오래 걸릴 수 있으므로 비동기 처리 후보

### 1.5 Bearer Token이란 무엇인가

로그인한 사용자인지 확인하기 위한 인증 토큰이다.

이 초안은 `하나원큐 인앱 + 하나원큐 로그인 연동`을 전제로 한다.  
따라서 이 문서에 등장하는 `Authorization: Bearer ...`는 하나원큐 로그인 완료 후 전달받은 액세스 토큰이라고 이해하면 된다.

---

## 2. 이 프로젝트에서 API가 필요한 이유

이 서비스의 핵심 흐름은 아래와 같다.

1. 사용자가 서비스에 들어온다
2. 주소와 보증금을 입력한다
3. 서버가 기본 위험 분석을 수행한다
4. 필요하면 등기부 PDF를 업로드한다
5. 서버가 등기부를 파싱하고 정밀 분석을 수행한다
6. 서버가 Lease Passport 리포트를 발급한다
7. 사용자는 보험, 대출, 상담 같은 후속 행동으로 이동한다

즉 이 API는 단순 조회용이 아니라,

- 입력값 검증
- 외부 데이터 조회
- 분석 실행
- 리포트 생성
- 후속 액션 연결

을 하나의 흐름으로 이어주는 역할을 한다.

---

## 3. 범위

이 초안은 기능명세서 기준으로 아래 범위를 담는다.

- F-02 매물 정보 입력
- F-03 등기부등본 업로드 + AI 파싱
- F-05 AI 위험 분석 엔진
- F-06 Lease Passport 리포트
- F-07 후속 금융 액션 연결

선택 기능:

- F-04 마이데이터 연동
- F-08 발급 이력 기록

Phase 2 이후 기능:

- 리포트 해설형 AI Q&A
- PDF 다운로드
- 카카오 공유 카드
- 대안 매물 탐색

---

## 4. 이 초안에서 고정한 전제와 아직 남은 미결정 사항

이 문서는 아래 두 가지를 먼저 고정하고 읽는 것이 좋다.

### 4.1 서비스 채널 전제

이 초안은 `하나원큐 인앱 기능`을 전제로 작성한다.

즉:

- 주소 입력, 분석, 리포트 조회는 하나원큐 앱 내부 화면에서 진행한다
- 후속 행동 연결은 하나원큐 내부 라우팅 또는 인앱 딥링크를 우선 사용한다
- 외부 웹 URL로 바로 노출하기보다, 하나원큐 내부 플로우를 통해 연결하는 구조를 기본값으로 둔다

### 4.2 인증/유저 전략 전제

이 초안은 `하나원큐 로그인 연동`을 전제로 작성한다.

즉:

- 사용자는 하나원큐 로그인 완료 후 서비스를 이용한다
- 서버는 `Authorization: Bearer {accessToken}` 헤더를 기준으로 사용자 문맥을 식별한다
- `me/*`, 상담 예약, 마이데이터 동의뿐 아니라 분석 케이스 생성과 조회도 로그인 사용자 기준으로 관리한다

### 4.3 주소/시세/보증 데이터 소스

아직 아래가 최종 확정되지 않았다.

- 주소 검색 공급자
- 시세 산정 데이터 소스
- 보증 가능 여부 판정 방식

이 문서에서는 대표 후보를 예시로 적었다.

### 4.4 MVP 범위

아직 아래도 미결정이다.

- 기본 분석만으로 Passport를 발급할지
- 등기부 업로드까지 해야 Passport를 발급할지
- 보험/대출을 추천만 할지, 실제 연결까지 할지

---

## 5. 이 서비스의 전체 흐름을 API 관점에서 보면

### 5.1 가장 단순한 흐름

1. 사용자가 서비스에 들어온다
2. `/addresses/suggestions`로 주소 후보를 찾는다
3. `/properties/validate`로 주소와 보증금을 검증한다
4. `/assessment-cases`로 분석 케이스를 만든다
5. `/assessment-cases/{caseId}/basic-analysis`로 기본 분석을 실행한다
6. 필요하면 `/assessment-cases/{caseId}/passports`로 리포트를 발급한다

### 5.2 정밀 분석까지 가는 흐름

1. 기본 분석 완료
2. `/assessment-cases/{caseId}/registry-cache`로 기존 등기 캐시가 있는지 확인한다
3. 없으면 `/registry-documents`로 PDF 업로드
4. `/parse`로 OCR/NLP 파싱 시작
5. `/analysis-jobs/{jobId}`로 파싱 완료 여부 확인
6. 필요하면 `/registry-manual-input`으로 수기 입력 보완
7. `/precision-analysis`로 정밀 분석 시작
8. `/analysis-jobs/{jobId}`로 정밀 분석 완료 여부 확인
9. `/passports`로 최종 리포트 발급

### 5.3 리포트 이후 흐름

1. `/passports/{passportId}`로 리포트 상세 조회
2. `/passports/{passportId}/actions`로 후속 행동 목록 조회
3. 필요하면 `/consultations/reservations`로 상담 예약

---

## 6. 핵심 용어 설명

이 문서에서 자주 나오는 용어를 먼저 정리한다.

### 6.1 Assessment Case

사용자가 `특정 매물`에 대해 `한 번 분석을 진행하는 작업 단위`다.

쉽게 말하면:

- 어떤 주소인지
- 보증금이 얼마인지
- 지금 분석이 어디까지 진행됐는지

를 묶어놓은 단위다.

### 6.2 Registry Document

사용자가 업로드한 등기부 문서 또는 재사용한 등기 데이터다.

여기에는 아래 같은 정보가 들어간다.

- 언제 발급된 등기부인지
- 파싱이 성공했는지
- 재사용 동의가 있는지
- 언제까지 유효한지

### 6.3 Analysis Job

오래 걸리는 작업을 뜻한다.

예:

- OCR/NLP 파싱
- 정밀 분석

이런 작업은 바로 끝나지 않을 수 있으므로 `jobId`를 발급하고 상태를 나중에 조회한다.

### 6.4 Passport

분석 결과를 스냅샷처럼 저장한 최종 리포트다.

여기에는 보통 아래가 포함된다.

- 종합 판정
- 위험 노출 구간
- 각 위험 항목 설명
- 추천 액션

### 6.5 Consultation Reservation

상담 예약 정보다.

예:

- 어떤 Passport를 바탕으로 상담을 요청했는지
- 희망 날짜/시간이 언제인지
- 상태가 접수인지 확정인지

---

## 7. 공통 규약

### 7.1 Base URL

```text
/api/v1
```

모든 API는 이 주소를 기준으로 시작한다고 보면 된다.

### 7.2 Headers

```http
Content-Type: application/json
X-Request-Id: {uuid}
Authorization: Bearer {accessToken}
```

설명:

- `Content-Type`: JSON 형식으로 통신한다는 의미
- `X-Request-Id`: 요청 추적용 고유 ID
- `Authorization`: 하나원큐 로그인 연동으로 전달된 액세스 토큰

파일 업로드 API는 `multipart/form-data`를 사용한다.

### 7.3 공통 응답 형식

성공 시:

```json
{
  "requestId": "4d18d7f8-85df-4f9a-95a0-1e6f2ecbe71b",
  "data": {}
}
```

실패 시:

```json
{
  "requestId": "4d18d7f8-85df-4f9a-95a0-1e6f2ecbe71b",
  "error": {
    "code": "UNSUPPORTED_REGION",
    "message": "현재는 서울·수도권 매물만 지원합니다.",
    "details": {}
  }
}
```

이 구조를 통일하면 화면 쪽에서 에러 처리를 쉽게 할 수 있다.

### 7.4 공통 Enum

```text
RiskGrade = GREEN | YELLOW | RED
Decision = SAFE | REVIEW_AFTER_FIX | CAUTION
CaseStatus = DRAFT | BASIC_ANALYZED | REGISTRY_UPLOADED | REGISTRY_PARSED | PRECISION_ANALYZED | PASSPORT_ISSUED
JobStatus = QUEUED | PROCESSING | SUCCEEDED | FAILED
RegistrySource = USER_UPLOAD | CACHE_REUSE | MANUAL_INPUT
```

쉽게 말하면:

- `RiskGrade`: 각 항목 위험도
- `Decision`: 최종 종합 판정
- `CaseStatus`: 분석이 현재 어디까지 진행됐는지
- `JobStatus`: 오래 걸리는 작업이 현재 어떤 상태인지
- `RegistrySource`: 등기 데이터가 어디서 왔는지

---

## 8. 핵심 리소스 예시

### 8.1 Assessment Case 예시

```json
{
  "caseId": "case_01JQW6M2D7JH9M3E8Q3J2M7D4T",
  "status": "BASIC_ANALYZED",
  "property": {
    "roadAddress": "서울특별시 성동구 왕십리로 123",
    "jibunAddress": "서울특별시 성동구 성수동1가 123-45",
    "lat": 37.5442,
    "lng": 127.0557,
    "regionCode": "11200",
    "supportedRegion": true
  },
  "depositAmount": 350000000
}
```

이 객체는 "지금 어떤 매물에 대해, 어떤 금액 기준으로, 분석이 어디까지 진행됐는가"를 보여준다.

### 8.2 Registry Document 예시

```json
{
  "registryDocumentId": "reg_01JQW6PT9F7G2A7XW2TPP7YQ9B",
  "source": "USER_UPLOAD",
  "issuedDate": "2026-03-28",
  "parseStatus": "SUCCEEDED",
  "reuseConsent": true,
  "validUntil": "2026-04-27",
  "rawFileDeleteAt": "2026-04-02T09:00:00+09:00"
}
```

이 객체는 "업로드한 등기부가 어떤 상태인지"를 보여준다.

### 8.3 Passport 예시

```json
{
  "passportId": "pass_01JQW6VYRF2Y3CT3NZ7FD2FJ5W",
  "issuedAt": "2026-04-01T15:20:00+09:00",
  "decision": "REVIEW_AFTER_FIX",
  "riskExposure": {
    "minAmount": 85000000,
    "maxAmount": 110000000,
    "currency": "KRW"
  },
  "resultHash": "sha256:5dcf96d1..."
}
```

이 객체는 "최종 리포트 1건"이라고 보면 된다.

---

## 9. API 한눈에 보기

아래 표는 `하나원큐 인앱 + 하나원큐 로그인 연동`을 전제로 정리한 MVP 초안이다.  
MVP 포함 여부는 바뀔 수 있지만, 채널과 인증 방식은 이 문서 기준으로 고정한다.

| Method | Path | 쉬운 설명 | 언제 호출하는가 | 인증 방식 | 근거 기능명세 | MVP |
|------|------|------|------|------|------|:---:|
| GET | `/addresses/suggestions` | 주소 검색 후보 조회 | 사용자가 주소 입력 중일 때 | 하나원큐 로그인 연동 | F-02 매물 정보 입력 | 필수 |
| POST | `/properties/validate` | 주소/보증금이 유효한지 확인 | 주소 선택 후 다음 단계 직전 | 하나원큐 로그인 연동 | F-02 매물 정보 입력 | 필수 |
| POST | `/assessment-cases` | 분석 케이스 생성 | 본격 분석 시작 시 | 하나원큐 로그인 연동 | 사용자 여정 §3 + F-05 진입 | 필수 |
| GET | `/assessment-cases/{caseId}` | 현재 케이스 상태 조회 | 화면 재진입 또는 상태 새로고침 | 하나원큐 로그인 연동 | 사용자 여정 §3 단계형 흐름 | 필수 |
| POST | `/assessment-cases/{caseId}/basic-analysis` | 기본 분석 실행 | 주소+보증금 입력 후 | 하나원큐 로그인 연동 | F-05 1단계 즉시 분석 | 필수 |
| GET | `/assessment-cases/{caseId}/registry-cache` | 기존 등기 캐시 존재 여부 확인 | 등기 업로드 전에 | 하나원큐 로그인 연동 | F-03 등기 저장 및 재사용 정책 | 필수 |
| POST | `/assessment-cases/{caseId}/registry-documents` | 등기부 PDF 업로드 | 사용자가 파일을 올릴 때 | 하나원큐 로그인 연동 | F-03 등기부 업로드 | 필수 |
| POST | `/assessment-cases/{caseId}/registry-documents/{registryDocumentId}/parse` | OCR/NLP 파싱 시작 | 업로드 직후 | 하나원큐 로그인 연동 | F-03 OCR + NLP 처리 | 필수 |
| POST | `/assessment-cases/{caseId}/registry-manual-input` | 파싱 실패 시 수기 입력 | OCR 실패 후 | 하나원큐 로그인 연동 | F-03 파싱 실패 시 수기 입력 | 필수 |
| POST | `/assessment-cases/{caseId}/precision-analysis` | 정밀 분석 시작 | 등기 정보가 준비된 후 | 하나원큐 로그인 연동 | F-05 2단계 정밀 분석 | 필수 |
| GET | `/analysis-jobs/{jobId}` | 오래 걸리는 작업 상태 조회 | 파싱/정밀분석 진행 중 | 하나원큐 로그인 연동 | F-03/F-05 비동기 처리 보조 | 필수 |
| POST | `/assessment-cases/{caseId}/passports` | 리포트 발급 | 분석 결과를 고정하고 싶을 때 | 하나원큐 로그인 연동 | F-06 Lease Passport 리포트 | 필수 |
| GET | `/passports/{passportId}` | 리포트 상세 조회 | 리포트 화면 진입 시 | 하나원큐 로그인 연동 | F-06 Layer 1~3 리포트 | 필수 |
| GET | `/passports/{passportId}/actions` | 후속 행동 목록 조회 | 리포트 하단 액션 노출 시 | 하나원큐 로그인 연동 | F-07 후속 금융 액션 연결 | 필수 |
| POST | `/consultations/reservations` | 상담 예약 생성 | 상담 버튼 클릭 시 | 하나원큐 로그인 연동 | F-07 판정별 액션: 상담 예약 | 필수 |
| POST | `/mydata/consents` | 마이데이터 동의 저장 | 동의 버튼 클릭 시 | 하나원큐 로그인 연동 | F-04 마이데이터 연동 | 선택 |
| GET | `/me/passports` | 내 리포트 목록 조회 | 마이페이지/이력 화면 | 하나원큐 로그인 연동 | F-08 발급 이력 기록 | 선택 |
| GET | `/me/passports/{passportId}` | 내 리포트 단건 조회 | 이력 상세 보기 | 하나원큐 로그인 연동 | F-08 발급 이력 기록 | 선택 |

위 `근거 기능명세`는 기능명세서의 F-01~F-08 및 사용자 여정에서 직접 가져온 것이다.  
기능명세서에 API가 직접 적혀 있지 않은 경우에도, 사용자 여정과 단계별 처리 요구를 서버 입장에서 분해해 도출했다.

---

## 10. 하나원큐 인앱 전제에서 정리되는 API 해석

이 문서는 서비스 채널을 `하나원큐 인앱`, 인증 방식을 `하나원큐 로그인 연동`으로 고정한다.

따라서 아래 API들은 더 이상 `채널 미정` 상태로 보지 않고, 인앱 서비스 기준으로 해석한다.

### 10.1 인앱 전제에서 달라지는 점

하나원큐 인앱 전제를 두면 아래가 함께 정리된다.

- 로그인 방식은 하나원큐 로그인 연동으로 고정된다
- 화면 이동은 하나원큐 내부 라우트 또는 인앱 딥링크를 우선 사용한다
- 사용자 식별은 하나원큐 로그인 사용자 문맥 기준으로 처리한다
- 마이페이지/이력은 하나원큐 내 `내 Passport` 영역으로 수렴한다

### 10.2 인앱 전제에서 구체화되는 API

| API | 하나원큐 인앱 기준 해석 |
|------|------|
| `GET /passports/{passportId}/actions` | 액션 타깃은 `INTERNAL_ROUTE` 또는 인앱 딥링크를 기본값으로 둔다. 외부 웹 URL 직접 노출은 우선순위를 낮춘다. |
| `POST /consultations/reservations` | 하나원큐 로그인 사용자 정보를 바탕으로 예약을 생성하고, 실제 상담 운영 시스템은 내부 구축 또는 기존 시스템 연동 중 하나로 구현한다. |
| `POST /mydata/consents` | 하나원큐 로그인 사용자 기준으로 동의 이력을 저장한다. 별도 회원 체계는 두지 않는다. |
| `GET /me/passports` | 하나원큐 로그인 사용자 기준의 `내 Passport` 목록 조회로 본다. |
| `GET /me/passports/{passportId}` | 하나원큐 로그인 사용자 본인 소유 Passport만 조회 가능하다고 가정한다. |

### 10.3 인앱 전제와 무관하게 그대로 가는 API

아래 API들은 하나원큐 인앱으로 고정하더라도 본질이 크게 바뀌지 않는다.

- `GET /addresses/suggestions`
- `POST /properties/validate`
- `POST /assessment-cases`
- `GET /assessment-cases/{caseId}`
- `POST /assessment-cases/{caseId}/basic-analysis`
- `GET /assessment-cases/{caseId}/registry-cache`
- `POST /assessment-cases/{caseId}/registry-documents`
- `POST /assessment-cases/{caseId}/registry-documents/{registryDocumentId}/parse`
- `POST /assessment-cases/{caseId}/registry-manual-input`
- `POST /assessment-cases/{caseId}/precision-analysis`
- `GET /analysis-jobs/{jobId}`
- `POST /assessment-cases/{caseId}/passports`
- `GET /passports/{passportId}`

이 API들은 본질적으로 `주소 입력 -> 분석 -> 리포트 생성`이라는 핵심 분석 흐름에 해당하므로, 인앱 여부와 관계없이 구조가 비교적 안정적이다.

---

## 11. 상세 명세
아래부터는 각 API를 하나씩 설명한다.

### 11.1 GET `/addresses/suggestions`

#### 쉽게 설명하면

주소 입력창에서 자동완성 후보를 보여주는 API다.

#### 왜 필요한가

사용자가 주소를 직접 정확히 다 입력하기 어렵기 때문이다.  
자동완성을 제공해야 오입력을 줄일 수 있다.

#### 언제 호출하는가

사용자가 주소 검색창에 글자를 입력할 때마다 호출할 수 있다.

#### Request Query

```text
q=성동구 왕십리로 123
limit=10
```

#### Response

```json
{
  "requestId": "29aa0f9f-8bb0-455e-b0d7-b1d35ad4c4d5",
  "data": {
    "items": [
      {
        "roadAddress": "서울특별시 성동구 왕십리로 123",
        "jibunAddress": "서울특별시 성동구 성수동1가 123-45",
        "lat": 37.5442,
        "lng": 127.0557,
        "regionCode": "11200",
        "supportedRegion": true
      }
    ]
  }
}
```

#### 메모

실제 주소 검색 공급자는 아직 미정이다.  
이 API는 외부 공급자가 무엇이든 화면에는 공통 형식으로 돌려주기 위한 추상화 계층으로 이해하면 된다.

#### 기능명세서 연동

- 근거: F-02 매물 정보 입력
- 연결 이유: 기능명세서_v3에서 `주소 검색·자동완성`을 입력 단계의 핵심으로 명시하고 있으므로, 이를 화면에서 호출 가능한 API로 분리했다.

---

### 11.2 POST `/properties/validate`

#### 쉽게 설명하면

사용자가 고른 주소와 보증금이 분석 가능한 입력값인지 확인하는 API다.

#### 왜 필요한가

주소가 틀렸거나, 지원하지 않는 지역이거나, 보증금 범위가 이상하면 분석을 시작하면 안 되기 때문이다.

#### Request

```json
{
  "roadAddress": "서울특별시 성동구 왕십리로 123",
  "regionCode": "11200",
  "depositAmount": 350000000
}
```

#### Response

```json
{
  "requestId": "cd526ba3-8515-4af1-9be6-b8d24b520bf1",
  "data": {
    "supportedRegion": true,
    "propertyExists": true,
    "depositAmountValid": true,
    "normalizedProperty": {
      "roadAddress": "서울특별시 성동구 왕십리로 123",
      "regionCode": "11200",
      "lat": 37.5442,
      "lng": 127.0557
    }
  }
}
```

#### Possible Errors

```text
ADDRESS_NOT_FOUND
UNSUPPORTED_REGION
INVALID_DEPOSIT_AMOUNT
```

#### 기능명세서 연동

- 근거: F-02 매물 정보 입력
- 연결 이유: 기능명세서_v3의 `실존 여부 확인`, `보증금 범위 검증`, `지원 지역 판정`, `주소 매칭 실패 예외 처리`를 서버 검증 API로 풀어낸 것이다.

---

### 11.3 POST `/assessment-cases`

#### 쉽게 설명하면

이제부터 이 매물에 대해 분석을 시작하겠다는 `작업 단위`를 만드는 API다.

#### 왜 필요한가

이 프로젝트는 주소 입력, 기본 분석, 등기 업로드, 정밀 분석, 리포트 발급이 단계적으로 이어진다.  
이 모든 흐름을 하나의 `caseId` 아래 묶어 관리해야 하기 때문이다.

#### Request

```json
{
  "entrySessionId": "entry_01JQW7BCS8N6R1CZ6M0Y4J2S3N",
  "property": {
    "roadAddress": "서울특별시 성동구 왕십리로 123",
    "jibunAddress": "서울특별시 성동구 성수동1가 123-45",
    "lat": 37.5442,
    "lng": 127.0557,
    "regionCode": "11200"
  },
  "depositAmount": 350000000
}
```

#### Response

```json
{
  "requestId": "52bb3e1f-4706-49d8-a4ce-c5368ff980dd",
  "data": {
    "caseId": "case_01JQW6M2D7JH9M3E8Q3J2M7D4T",
    "status": "DRAFT"
  }
}
```

#### 기능명세서 연동

- 근거: 사용자 여정 §3, F-05 AI 위험 분석 엔진
- 연결 이유: 기능명세서_v3의 흐름이 `입력 -> 기본 분석 -> 등기 업로드 -> 정밀 분석 -> 리포트 발급`처럼 단계형이므로, 이 흐름을 묶는 작업 단위가 필요해 `assessment case`를 도입했다.

---

### 11.4 GET `/assessment-cases/{caseId}`

#### 쉽게 설명하면

현재 분석 케이스가 어디까지 진행됐는지 조회하는 API다.

#### 왜 필요한가

사용자가 화면을 나갔다가 다시 들어올 수도 있고,  
분석 도중 상태를 새로고침해야 할 수도 있기 때문이다.

#### Response

```json
{
  "requestId": "d3383303-1f43-49c1-878e-854aa21c4986",
  "data": {
    "caseId": "case_01JQW6M2D7JH9M3E8Q3J2M7D4T",
    "status": "PRECISION_ANALYZED",
    "property": {
      "roadAddress": "서울특별시 성동구 왕십리로 123",
      "regionCode": "11200"
    },
    "depositAmount": 350000000,
    "latestAnalysisSummary": {
      "decision": "REVIEW_AFTER_FIX",
      "riskExposure": {
        "minAmount": 85000000,
        "maxAmount": 110000000
      }
    },
    "latestPassportId": "pass_01JQW6VYRF2Y3CT3NZ7FD2FJ5W"
  }
}
```

#### 기능명세서 연동

- 근거: 사용자 여정 §3
- 연결 이유: 사용자가 중간 단계에서 이탈하거나 재진입해도 현재 진행 상태를 이어가야 하므로, 단계형 사용자 여정을 케이스 상태 조회 API로 구체화했다.

---

### 11.5 POST `/assessment-cases/{caseId}/basic-analysis`

#### 쉽게 설명하면

주소와 보증금만으로 하는 1차 분석 API다.

#### 이 단계에서 하는 일

- 시세 추정
- 전세가율 계산
- 보증 가입 가능성 1차 판정
- 대략적인 위험 수준 계산

#### Request

```json
{
  "forceRefreshMarketData": false
}
```

#### Response

```json
{
  "requestId": "82c11c0b-56a3-4879-8952-15d4c748a0d7",
  "data": {
    "caseId": "case_01JQW6M2D7JH9M3E8Q3J2M7D4T",
    "status": "BASIC_ANALYZED",
    "basicAnalysis": {
      "marketPrice": 470000000,
      "jeonseRate": 74.5,
      "riskGrades": {
        "jeonseRateAdequacy": "YELLOW",
        "guaranteeEligibility": "GREEN"
      },
      "guaranteeEligibility": {
        "hugEligible": true,
        "hfEligible": true,
        "blockingReasons": []
      },
      "roughRiskLevel": "YELLOW",
      "disclaimer": "본 추정치는 법적 확정 금액이 아니며, 계약 의사결정 참고용 정보입니다."
    }
  }
}
```

#### 기능명세서 연동

- 근거: F-05 AI 위험 분석 엔진, 분석 단계별 산출물 1단계
- 연결 이유: 기능명세서_v3의 `주소 + 보증금만으로 전세가율, 보증 가입 가능성, 개략적 위험 수준`을 계산하는 요구를 1차 분석 API로 정의했다.

---

### 11.6 GET `/assessment-cases/{caseId}/registry-cache`

#### 쉽게 설명하면

예전에 이 매물에 대해 저장된 등기 데이터가 있으면 다시 쓰는 API다.

#### 왜 필요한가

- 사용자가 매번 등기부를 다시 올리지 않게 하기 위해
- 비용과 시간을 줄이기 위해

#### 주의

무엇을 `동일 매물`로 볼지는 아직 미정이다.

#### Response

```json
{
  "requestId": "a8c5a9f5-05db-4bfe-80ad-c0ad1c4c439c",
  "data": {
    "available": true,
    "registryDocumentId": "reg_01JQW6PT9F7G2A7XW2TPP7YQ9B",
    "issuedDate": "2026-03-28",
    "validUntil": "2026-04-27",
    "parseStatus": "SUCCEEDED"
  }
}
```

만료된 경우:

```json
{
  "requestId": "637ea6f0-5d7d-4038-90c1-02f85e3774ec",
  "error": {
    "code": "REGISTRY_CACHE_EXPIRED",
    "message": "등기부가 30일 이상 경과했습니다. 최신 등기부를 업로드해주세요.",
    "details": {
      "expiredAt": "2026-03-30"
    }
  }
}
```

#### 기능명세서 연동

- 근거: F-03 등기부 저장 및 재사용 정책
- 연결 이유: 기능명세서_v3가 `동일 매물 재조회`, `30일 유효기간`, `만료 시 재업로드 안내`를 명시하므로, 업로드 전에 캐시 존재 여부를 확인하는 API가 필요하다.

---

### 11.7 POST `/assessment-cases/{caseId}/registry-documents`

#### 쉽게 설명하면

등기부 PDF 파일을 서버에 올리는 API다.

#### 왜 필요한가

기본 분석만으로는 선순위 채권 등 세부 정보가 부족하기 때문이다.  
정밀 분석을 하려면 등기부 데이터가 필요하다.

#### Request

```text
multipart/form-data
- file: registry.pdf
- issuedDate: 2026-03-28
- reuseConsent: true
```

#### Response

```json
{
  "requestId": "526e31e8-dc7f-4c63-9150-cf340a4440cf",
  "data": {
    "registryDocumentId": "reg_01JQW6PT9F7G2A7XW2TPP7YQ9B",
    "source": "USER_UPLOAD",
    "parseStatus": "QUEUED",
    "rawFileDeleteAt": "2026-04-02T09:00:00+09:00"
  }
}
```

#### 기능명세서 연동

- 근거: F-03 등기부등본 업로드 + AI 파싱
- 연결 이유: 기능명세서_v3에서 사용자가 인터넷등기소 PDF를 직접 올리는 흐름이 MVP 필수이므로, 업로드 자체를 담당하는 엔드포인트가 필요하다.

---

### 11.8 POST `/assessment-cases/{caseId}/registry-documents/{registryDocumentId}/parse`

#### 쉽게 설명하면

업로드한 등기부를 OCR/NLP로 읽기 시작하는 API다.

#### 왜 필요한가

PDF 파일 자체는 사람이 보기 위한 문서라서,  
서버가 분석하려면 구조화된 데이터로 바꿔야 하기 때문이다.

#### Response

```json
{
  "requestId": "f8be6f85-39ad-4426-8d7d-b0dfe7f9f3d0",
  "data": {
    "jobId": "job_01JQW7VY4EHCAGVXGR36BQA9EV",
    "jobType": "REGISTRY_PARSE",
    "status": "QUEUED"
  }
}
```

#### 기능명세서 연동

- 근거: F-03 등기부등본 업로드 + AI 파싱
- 연결 이유: 기능명세서_v3의 `PDF -> OCR 텍스트 추출 -> NLP 구조화 -> 위험 요소 자동 태깅` 중 OCR/NLP 시작 단계를 별도 비동기 작업으로 분리한 것이다.

---

### 11.9 POST `/assessment-cases/{caseId}/registry-manual-input`

#### 쉽게 설명하면

OCR이 실패했을 때, 사람이 최소 핵심 항목을 직접 입력하는 API다.

#### 왜 필요한가

초기 서비스에서는 OCR 실패가 완전히 없어지기 어렵다.  
그래서 최소한의 수기 입력 경로를 열어두는 것이 안전하다.

#### Request

```json
{
  "mortgageAmount": 120000000,
  "hasSeizureOrProvisionalSeizure": false,
  "ownershipTransferredAt": "2025-11-10",
  "hasTrustRegistration": false
}
```

#### Response

```json
{
  "requestId": "3995cc7e-8e30-4364-b230-373b4534ff00",
  "data": {
    "registryDocumentId": "reg_01JQW6PT9F7G2A7XW2TPP7YQ9B",
    "source": "MANUAL_INPUT",
    "status": "REGISTRY_PARSED"
  }
}
```

#### 기능명세서 연동

- 근거: F-03 등기부등본 업로드 + AI 파싱
- 연결 이유: 기능명세서_v3가 `자동 인식 실패 시 핵심 3항목 수기 입력`을 명시하고 있으므로, OCR 실패 복구 경로를 독립 API로 두었다.

---

### 11.10 POST `/assessment-cases/{caseId}/precision-analysis`

#### 쉽게 설명하면

등기 데이터까지 포함해서 2차 정밀 분석을 시작하는 API다.

#### 이 단계에서 하는 일

- 등기 권리관계 반영
- 선순위 채권 반영
- 위험 노출 구간 계산
- 종합 판정 계산

#### Request

```json
{
  "registrySource": "CACHE_REUSE"
}
```

#### Response

```json
{
  "requestId": "2f60f3f4-d3c6-4c4d-91d8-86bdbba2cf70",
  "data": {
    "jobId": "job_01JQW83FXY4B2QVTW0M4H2KX5N",
    "jobType": "PRECISION_ANALYSIS",
    "status": "QUEUED"
  }
}
```

#### 기능명세서 연동

- 근거: F-05 AI 위험 분석 엔진, 분석 단계별 산출물 2단계
- 연결 이유: 기능명세서_v3의 `등기 권리관계 반영 + 위험 노출 구간 계산 + 종합 판정`을 수행하는 정밀 분석 단계를 API로 분리한 것이다.

---

### 11.11 GET `/analysis-jobs/{jobId}`

#### 쉽게 설명하면

비동기 작업이 끝났는지 확인하는 API다.

#### 언제 쓰는가

- OCR 파싱 상태 확인
- 정밀 분석 상태 확인

#### 성공 응답 예시

```json
{
  "requestId": "979d4ad3-0eb1-4907-a70b-c0d1f78435a7",
  "data": {
    "jobId": "job_01JQW83FXY4B2QVTW0M4H2KX5N",
    "jobType": "PRECISION_ANALYSIS",
    "status": "SUCCEEDED",
    "result": {
      "riskGrades": {
        "registryRights": "YELLOW",
        "jeonseRateAdequacy": "YELLOW",
        "guaranteeEligibility": "GREEN"
      },
      "decision": "REVIEW_AFTER_FIX",
      "riskExposure": {
        "minAmount": 85000000,
        "maxAmount": 110000000,
        "currency": "KRW"
      },
      "analysisBasis": {
        "marketPrice": 470000000,
        "auctionRateRange": [0.68, 0.73],
        "seniorClaimsAmount": 140000000
      }
    }
  }
}
```

#### 실패 응답 예시

```json
{
  "requestId": "6694c8f0-a31d-4347-a3cf-b2ed8c1b5a72",
  "data": {
    "jobId": "job_01JQW83FXY4B2QVTW0M4H2KX5N",
    "jobType": "PRECISION_ANALYSIS",
    "status": "FAILED",
    "failure": {
      "code": "REGISTRY_PARSE_FAILED",
      "message": "자동 인식에 실패했습니다. 핵심 3항목을 수기 입력해주세요."
    }
  }
}
```

#### 기능명세서 연동

- 근거: F-03 OCR 파싱, F-05 정밀 분석
- 연결 이유: 기능명세서_v3의 핵심 작업 중 OCR 파싱과 정밀 분석은 시간이 걸릴 수 있으므로, 상태 polling용 공통 잡 조회 API가 필요하다.

---

### 11.12 POST `/assessment-cases/{caseId}/passports`

#### 쉽게 설명하면

현재 분석 결과를 바탕으로 `정식 리포트 1건`을 발급하는 API다.

#### 왜 필요한가

분석 결과는 계속 바뀔 수 있지만,  
Passport는 특정 시점의 결과를 `스냅샷`처럼 고정해두는 역할을 한다.

#### 주의

기본 분석만으로도 Passport를 발급할지 여부는 아직 미정이다.

#### Request

```json
{
  "issueReason": "USER_REQUEST"
}
```

#### Response

```json
{
  "requestId": "3aeed583-8126-45bc-bc3e-e0a3ab89ce28",
  "data": {
    "passportId": "pass_01JQW6VYRF2Y3CT3NZ7FD2FJ5W",
    "issuedAt": "2026-04-01T15:20:00+09:00",
    "decision": "REVIEW_AFTER_FIX",
    "resultHash": "sha256:5dcf96d1..."
  }
}
```

#### 기능명세서 연동

- 근거: F-06 Lease Passport 리포트
- 연결 이유: 기능명세서_v3에서 Passport를 `특정 시점의 분석 결과 스냅샷`처럼 다루므로, 발급 시점을 고정하는 API가 필요하다.

---

### 11.13 GET `/passports/{passportId}`

#### 쉽게 설명하면

Lease Passport 리포트 화면에 필요한 상세 데이터를 가져오는 API다.

#### 왜 필요한가

리포트는 단순 점수 하나가 아니라 아래 3단 구조를 가진다.

- Layer 1: 종합 판정 + 핵심 수치
- Layer 2: 항목별 위험 설명
- Layer 3: 추천 액션

#### Response

```json
{
  "requestId": "ac395900-1663-4f88-bb7f-2c51580ffdbd",
  "data": {
    "passportId": "pass_01JQW6VYRF2Y3CT3NZ7FD2FJ5W",
    "decision": "REVIEW_AFTER_FIX",
    "layer1": {
      "decisionLabel": "보완 후 가능",
      "riskExposure": {
        "minAmount": 85000000,
        "maxAmount": 110000000,
        "currency": "KRW"
      },
      "guaranteeImprovement": {
        "expectedReducedExposureMax": 20000000
      },
      "disclaimer": "본 추정치는 법적 확정 금액이 아니며, 계약 의사결정 참고용 정보입니다."
    },
    "layer2": {
      "riskCards": [
        {
          "factor": "registryRights",
          "grade": "YELLOW",
          "title": "등기 권리관계",
          "evidence": "근저당 2건, 최근 1년 이내 소유권 이전",
          "comment": "선순위 채권 규모를 추가 확인하세요."
        },
        {
          "factor": "jeonseRateAdequacy",
          "grade": "YELLOW",
          "title": "전세가율",
          "evidence": "전세가율 74.5%",
          "comment": "지역 평균 대비 다소 높습니다."
        }
      ]
    },
    "layer3": {
      "recommendedActionSet": "YELLOW_DEFAULT"
    }
  }
}
```

#### 기능명세서 연동

- 근거: F-06 Lease Passport 리포트, Layer 1~3
- 연결 이유: 기능명세서_v3가 리포트를 `종합 판정`, `항목별 위험 카드`, `맞춤형 액션 플랜`의 3-Layer 구조로 정의하고 있으므로, 이를 그대로 내려주는 상세 조회 API가 필요하다.

---

### 11.14 GET `/passports/{passportId}/actions`

#### 쉽게 설명하면

리포트 결과에 따라 사용자에게 보여줄 후속 행동 목록을 가져오는 API다.

#### 예시 행동

- 위험 항목 확인 가이드
- 보증기관 연결
- 대출 비교
- 상담 예약

#### Response

```json
{
  "requestId": "c43f803a-9c1b-4d43-a3f2-7cd8a8de6437",
  "data": {
    "items": [
      {
        "actionType": "GUIDE_RISK_FACTOR",
        "label": "위험 항목 확인 가이드",
        "target": {
          "type": "INTERNAL_ROUTE",
          "routeKey": "PASSPORT_RISK_GUIDE"
        }
      },
      {
        "actionType": "BOOK_CONSULTATION",
        "label": "전문 상담 예약",
        "target": {
          "type": "INTERNAL_ROUTE",
          "routeKey": "CONSULTATION_RESERVATION"
        }
      }
    ]
  }
}
```

#### 메모

이 초안에서는 후속 행동 타깃을 `INTERNAL_ROUTE` 또는 인앱 딥링크로 본다.  
즉, 하나원큐 앱 안에서 다음 화면으로 이동시키는 구조를 기본값으로 둔다.

#### 기능명세서 연동

- 근거: F-07 후속 금융 액션 연결
- 연결 이유: 기능명세서_v3에서 리포트 이후 `보증`, `대출`, `체크리스트`, `상담` 같은 액션 세트를 판정별로 다르게 보여주므로, 액션 목록 조회 API가 필요하다.

---

### 11.15 POST `/consultations/reservations`

#### 쉽게 설명하면

전문 상담을 예약하는 API다.

#### 왜 필요한가

특히 노란색/빨간색 판정 사용자에게는  
자동 추천만으로 부족할 수 있으므로 사람 상담 연결이 필요하다.

#### 주의

이 초안은 상담 예약도 하나원큐 로그인 사용자 기준으로 처리한다고 가정한다.  
다만 실제 예약 처리 엔진을 내부 구축할지, 기존 하나은행 시스템과 연동할지는 구현 단계에서 정하면 된다.

#### Request

```json
{
  "passportId": "pass_01JQW6VYRF2Y3CT3NZ7FD2FJ5W",
  "preferredDate": "2026-04-03",
  "preferredTimeSlot": "14:00-15:00",
  "contactPhone": "010-1234-5678",
  "memo": "보증 가입 불가 사유를 먼저 확인하고 싶어요."
}
```

#### Response

```json
{
  "requestId": "8f704a38-e880-4640-aa2e-887d8828ae32",
  "data": {
    "reservationId": "consult_01JQW8JHQG14Q22T5PNS4D7N7X",
    "status": "REQUESTED",
    "attachedPassportId": "pass_01JQW6VYRF2Y3CT3NZ7FD2FJ5W"
  }
}
```

#### 기능명세서 연동

- 근거: F-07 후속 금융 액션 연결, 판정별 액션 `전문 상담 예약`
- 연결 이유: 기능명세서_v3가 노란색/빨간색 사용자를 상담으로 연결하는 흐름을 MVP에 포함하므로, 예약 생성 API를 정의했다.

---

### 11.16 POST `/mydata/consents`

#### 쉽게 설명하면

사용자가 마이데이터 연동에 동의했는지 저장하는 API다.

#### 왜 필요한가

Phase 2에서 개인 자산/대출 상황을 분석에 반영하려면  
우선 동의 정보부터 관리해야 하기 때문이다.

#### 주의

현재 초안에서는 `동의 여부 저장`만 다룬다.  
실제 개인화 분석 반영은 후속 단계다.

#### Request

```json
{
  "agreed": true,
  "agreedAt": "2026-04-01T15:30:00+09:00"
}
```

#### Response

```json
{
  "requestId": "ae59dfdc-5cfe-4a93-908d-644ed20c9651",
  "data": {
    "agreed": true,
    "phaseApplied": "PHASE_2_PENDING"
  }
}
```

#### 기능명세서 연동

- 근거: F-04 마이데이터 연동
- 연결 이유: 기능명세서_v3에서 MVP는 `동의 UI만 구현, 실제 분석 반영은 Phase 2`로 정의되어 있으므로, 현재 초안도 `동의 저장`까지만 다룬다.

---

### 11.17 GET `/me/passports`

#### 쉽게 설명하면

내가 발급했던 Passport 목록을 보는 API다.

#### 언제 필요한가

- 마이페이지
- 이전 리포트 다시 보기
- 재상담 요청

#### Response

```json
{
  "requestId": "81bbebde-9fa6-4b39-a8cc-b8867c83f7d5",
  "data": {
    "items": [
      {
        "passportId": "pass_01JQW6VYRF2Y3CT3NZ7FD2FJ5W",
        "issuedAt": "2026-04-01T15:20:00+09:00",
        "propertySummary": "서울특별시 성동구 왕십리로 123",
        "decision": "REVIEW_AFTER_FIX",
        "riskExposure": {
          "minAmount": 85000000,
          "maxAmount": 110000000
        }
      }
    ],
    "nextCursor": null
  }
}
```

#### 기능명세서 연동

- 근거: F-08 발급 이력 기록, F-06 부가 기능 `앱 내 내 Passport 저장`
- 연결 이유: 기능명세서_v3가 선택 기능으로 `내 Passport 이력`을 제공하므로, 목록 조회 API를 선택 항목으로 두었다.

---

### 11.18 GET `/me/passports/{passportId}`

#### 쉽게 설명하면

내가 발급했던 Passport 중 하나를 다시 자세히 보는 API다.

#### 설명

응답 본문은 `GET /passports/{passportId}`와 거의 같고,  
추가로 `본인 것이 맞는지`만 더 검사한다고 이해하면 된다.

#### 기능명세서 연동

- 근거: F-08 발급 이력 기록, F-06 부가 기능 `앱 내 내 Passport 저장`
- 연결 이유: 기능명세서_v3의 이력 조회 요구를 상세 보기까지 확장한 API이며, 본문 구조는 기존 Passport 상세 응답을 재사용한다.

---

## 12. 상태 전이

### 11.1 케이스 상태 흐름

```text
DRAFT
  -> BASIC_ANALYZED
  -> REGISTRY_UPLOADED
  -> REGISTRY_PARSED
  -> PRECISION_ANALYZED
  -> PASSPORT_ISSUED
```

### 11.2 쉽게 설명하면

- `DRAFT`: 케이스만 만들어진 상태
- `BASIC_ANALYZED`: 주소+보증금 기준 기본 분석 완료
- `REGISTRY_UPLOADED`: 등기부 파일 업로드 완료
- `REGISTRY_PARSED`: 등기 데이터 해석 완료
- `PRECISION_ANALYZED`: 정밀 분석 완료
- `PASSPORT_ISSUED`: 리포트 발급 완료

### 11.3 예외 흐름

- 등기 파싱 실패 시 `REGISTRY_UPLOADED` 상태에서 수기 입력으로 대체 가능
- 등기 미업로드 상태에서 Passport 발급을 허용할지는 아직 미결정이다
- 허용한다면 위험 노출 구간은 `null` 또는 빈 값 처리하는 안이 후보다

---

## 13. 주요 에러 코드

| HTTP | Code | 설명 |
|------|------|------|
| 400 | INVALID_DEPOSIT_AMOUNT | 보증금 범위 오류 |
| 400 | ADDRESS_NOT_FOUND | 주소 매칭 실패 |
| 400 | UNSUPPORTED_REGION | 서울·수도권 외 지역 |
| 409 | REGISTRY_CACHE_EXPIRED | 저장된 등기 유효기간 만료 |
| 409 | PASSPORT_ALREADY_ISSUED | 동일 스냅샷 기준 중복 발급 |
| 422 | REGISTRY_PARSE_FAILED | OCR/NLP 파싱 실패 |
| 422 | ANALYSIS_INPUT_INCOMPLETE | 정밀 분석 필수 입력 누락 |
| 404 | CASE_NOT_FOUND | 케이스 없음 |
| 404 | PASSPORT_NOT_FOUND | Passport 없음 |
| 503 | MARKET_DATA_UNAVAILABLE | 시세 API 일시 장애 |
| 503 | GUARANTEE_RULESET_UNAVAILABLE | 보증기관 기준 데이터 사용 불가 |

---

## 14. 외부 연동 경계와 후보

| 연동 대상 | 용도 | 현재 메모 |
|------|------|------|
| 주소 검색 API (예: 카카오 로컬 API) | 주소 검색/좌표 표준화 | 실제 공급자와 호출 구조 미결정 |
| 시세 데이터 소스 (예: 국토부 실거래가 API) | 시세/전세가율 계산 | 단독 사용 여부와 보조 시세 결합 여부 미결정 |
| 건축물대장 API | 용도/기본 정보 보조 확인 | 등기 해석 보조 |
| 보증기관 기준 데이터 | HUG/HF 가입 가능성 판정 | 룰 엔진 기준 테이블화 권장 |
| OCR/NLP 엔진 | 등기부 구조화 파싱 | 비동기 처리 권장 |
| 하나원큐 딥링크 또는 내부 라우팅 | 후속 금융 액션 연결 | 이 초안의 기본 전제. 액션 타깃은 인앱 이동을 우선 사용 |

### 13.1 쉽게 설명하면

이 서비스는 자체 데이터만으로 완성되지 않는다.  
아래 외부 정보에 의존한다.

- 주소
- 시세
- 건물 기본 정보
- 보증 가능 여부 기준
- 등기부 해석
- 금융 액션 이동 경로

그래서 외부 API 선택은 단순 구현 문제가 아니라  
`서비스 신뢰도`, `비용`, `속도`, `정확도`를 함께 결정하는 요소다.

---

## 15. 보안 및 저장 정책 반영 포인트

- 등기 원본 PDF 메타데이터에는 `rawFileDeleteAt`를 포함한다
- 파싱 데이터 저장 시 `reuseConsent`, `validUntil`, `encrypted=true` 속성을 관리한다
- Passport 발급 시 `resultHash`를 저장해 이력 무결성을 보강한다
- 마이데이터 동의 철회 API는 추후 별도 정의가 필요하다

### 14.1 쉽게 설명하면

이 프로젝트는 민감한 정보가 들어갈 수 있으므로  
무조건 많이 저장하는 방향보다 `필요한 것만 저장하고, 빨리 지우는 구조`가 중요하다.

대표적으로:

- 원본 PDF는 오래 보관하지 않는다
- 파싱된 데이터는 동의가 있을 때만 재사용한다
- 결과값은 나중에 증빙할 수 있게 해시를 남긴다

---

## 16. 구현 우선순위 제안

아래 순서는 현재 `기본 분석 -> 등기 파싱 -> 정밀 분석 -> 리포트/액션`을 MVP 범위로 보는 임시안이다.

1. `addresses`, `properties/validate`, `assessment-cases`, `basic-analysis`
2. `registry-documents`, `registry parse job`, `registry manual input`
3. `precision-analysis`, `analysis-jobs`
4. `passports`, `actions`, `consultations`
5. `mydata/consents`, `me/passports`

### 16.1 쉽게 설명하면

처음부터 모든 기능을 한 번에 만들지 말고:

1. 입력과 기본 분석 먼저
2. 그다음 문서 업로드와 파싱
3. 그다음 정밀 분석
4. 마지막으로 리포트와 후속 행동 연결

순으로 가는 것이 안전하다는 뜻이다.

---

## 17. 최종 검토 필요 사항

아래는 아직 의사결정이 필요한 핵심 항목이다.

- 주소 검색 공급자를 무엇으로 할지
- 주소 검색을 백엔드 중계로 할지, 프론트 직접 호출 후 백엔드 검증만 할지
- 시세/전세가율 계산 시 `국토부 실거래가 단독`으로 갈지, `보조 시세`를 결합할지
- 기본 분석 결과만으로 Passport 발급을 허용할지
- 후속 금융 액션을 추천만 제공할지, 실제 신청/예약 연결까지 포함할지
- 등기 캐시 재사용 시 동일 매물 판정 키를 주소만으로 둘지, 건물 식별자까지 둘지
- 상담 예약 시스템을 내부 구축할지, 기존 하나은행 시스템과 연동할지

---

## 18. 이 문서를 처음 보는 사람이 읽을 때 추천 순서

1. `0~4장`: 문서 목적과 아직 미결정인 것 확인
2. `5장`: 서비스 전체 흐름 이해
3. `6~8장`: 용어와 핵심 객체 이해
4. `9~10장`: 실제 API 의미 파악
5. `11~18장`: 상태, 외부 연동, 보안, 우선순위, 미결정 사항 확인

이 순서로 읽으면 API를 하나도 모르는 사람도 전체 구조를 비교적 쉽게 따라갈 수 있다.
