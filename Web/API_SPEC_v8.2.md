# Rookies Bank API 명세서 v8.0

> **Base URL:** `http://localhost:8080`
> **인증 방식:** Session-based (Redis) + OTP 2단계 인증
> **Content-Type:** `application/json` (multipart 엔드포인트는 별도 표기)
> **버전:** v9.0 | **작성일:** 2026-03-31

---

## 공통 응답 형식

### 에러 응답
```json
{
  "status": 400,
  "error": "BAD_REQUEST",
  "message": "에러 메시지"
}
```

### HTTP 상태 코드
| 코드 | 설명 |
|------|------|
| 200 | 성공 |
| 201 | 생성 성공 |
| 202 | 수락 (처리 진행 중) |
| 204 | 성공 (응답 바디 없음) |
| 400 | 잘못된 요청 |
| 401 | 미인증 |
| 403 | 권한 없음 |
| 404 | 리소스 없음 |
| 409 | 충돌 (중복 등) |

---

## 1. 인증 (Auth)

### POST /auth/login
로그인 1단계: 자격증명 검증 후 OTP를 등록 전화번호로 발송

**인증 불필요**

**Request Body**
```json
{
  "loginId": "user@example.com",
  "password": "password123"
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| loginId | String | Y | 로그인 ID |
| password | String | Y | 비밀번호 |

**Response** `202 Accepted`
```json
{
  "message": "OTP가 발송되었습니다.",
  "maskedPhone": "010-****-5678"
}
```

---

### POST /auth/login/otp
로그인 2단계: OTP 검증 후 세션 발급. 기존 세션 자동 무효화(중복 로그인 방지)

**인증 불필요**

**Request Body**
```json
{
  "loginId": "user@example.com",
  "otp": "123456"
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| loginId | String | Y | 로그인 ID |
| otp | String | Y | SMS로 수신한 OTP |

**Response** `200 OK`
```json
{
  "userId": 1,
  "loginId": "user@example.com",
  "role": "USER"
}
```

---

### POST /auth/logout
세션 및 SESSION 쿠키 무효화

**인증 필요**

**Request Body** 없음

**Response** `200 OK` (빈 바디)

---

### POST /auth/verify-password
비밀번호 일치 여부만 검증 (OTP 발송 없음)

> **모바일 전용** — 간편 로그인(생체 인증) 등록 전 비밀번호 확인 용도

**인증 필요** (SESSION 쿠키)

**Request Body**
```json
{
  "password": "사용자입력비밀번호"
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| password | String | Y | 검증할 비밀번호 |

**Response**

| 상태 코드 | 설명 |
|-----------|------|
| `200 OK` | 비밀번호 일치 |
| `401 Unauthorized` | 비밀번호 불일치 또는 미인증 |

---

### POST /auth/sms/send
SMS OTP 발송 (회원가입 인증 / 비밀번호 재설정)

**인증 불필요**

**Request Body**
```json
{
  "phone": "010-1234-5678",
  "purpose": "SIGNUP",
  "name": "홍길동"
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| phone | String | Y | 전화번호. 형식: `010-XXXX-XXXX` |
| purpose | String | Y | 목적: `SIGNUP` \| `PASSWORD_RESET` |
| name | String | N | 비밀번호 재설정 시 본인 확인용 이름 |

**Response** `200 OK`
```json
{
  "message": "인증번호가 발송되었습니다."
}
```

---

### POST /auth/sms/verify
SMS OTP 검증

**인증 불필요**

**Request Body**
```json
{
  "phone": "010-1234-5678",
  "otp": "123456",
  "purpose": "SIGNUP"
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| phone | String | Y | 전화번호 |
| otp | String | Y | 수신한 OTP |
| purpose | String | Y | `SIGNUP` \| `PASSWORD_RESET` |

**Response** `200 OK`
```json
{
  "message": "전화번호 인증이 완료되었습니다."
}
```

---

### POST /auth/otp/send
이체·민감 작업용 OTP 발송

**인증 필요**

**Request Body**
```json
{
  "purpose": "TRANSFER"
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| purpose | String | N | 목적 (기본값: `TRANSFER`) |

**Response** `200 OK` (빈 바디)

---

### POST /auth/otp/verify
이체·민감 작업용 OTP 검증

**인증 필요**

**Request Body**
```json
{
  "purpose": "TRANSFER",
  "otp": "123456"
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| purpose | String | N | 목적 (기본값: `TRANSFER`) |
| otp | String | Y | OTP 코드 |

**Response** `200 OK` (빈 바디)

---

## 2. 회원 (User)

### POST /users
회원가입

**인증 불필요**

**Request Body**
```json
{
  "loginId": "user01@test.com",
  "password": "password123",
  "name": "홍길동",
  "residentId": "900101-1234567",
  "phone": "010-1234-5678",
  "email": "test@test.com",
  "address": "서울시 강남구"
}
```

| 필드 | 타입 | 필수 | 검증 |
|------|------|------|------|
| loginId | String | Y | 4~20자 |
| password | String | Y | 최소 8자 |
| name | String | Y | NotBlank |
| residentId | String | Y | 주민등록번호 |
| phone | String | Y | `010-XXXX-XXXX` 형식 |
| email | String | Y | 이메일 형식 |
| address | String | Y | NotBlank |

**Response** `201 Created` (빈 바디)

---

### GET /users/profile
내 프로필 조회

**인증 필요**

**Response** `200 OK`
```json
{
  "userId": 1,
  "loginId": "user01@test.com",
  "name": "홍길동",
  "phone": "010-1234-5678",
  "email": "test@test.com",
  "address": "서울시 강남구",
  "createdAt": "2025-01-01T09:00:00",
  "lastLogin": "2026-03-26T10:00:00",
  "status": "ACTIVE",
  "role": "USER",
  "accCount": 2
}
```

| 필드 | 타입 | 설명 |
|------|------|------|
| accCount | Integer | 활성 계좌 수 |

---

### PATCH /users/profile
프로필 수정

**인증 필요**

**Request Body**
```json
{
  "name": "홍길동",
  "phone": "010-9999-0000",
  "email": "new@test.com",
  "address": "서울시 서초구"
}
```

**Response** `200 OK` (빈 바디)

---

### PATCH /users/password
비밀번호 변경 (로그인 상태)

**인증 필요**

**Request Body**
```json
{
  "currentPassword": "oldPassword1",
  "newPassword": "newPassword1"
}
```

**Response** `200 OK` (빈 바디)

---

### PATCH /users/password/reset
비밀번호 재설정 (SMS 인증 완료 후)

**인증 불필요**

**Request Body**
```json
{
  "phone": "010-1234-5678",
  "newPassword": "newPassword1"
}
```

**Response** `200 OK` (빈 바디)

---

### GET /users/login-history
로그인 이력 조회

**인증 필요**

**Response** `200 OK`
```json
[
  {
    "logId": 1,
    "ipAddress": "192.168.0.1",
    "userAgent": "Mozilla/5.0 ...",
    "loginTime": "2026-03-26T10:00:00",
    "result": "SUCCESS"
  }
]
```

---

## 3. 계좌 (Account)

### AccountResponseDto 구조
```json
{
  "accountId": 1,
  "accountNumber": "110-012-345678",
  "accountType": "CHECKING",
  "balance": 100000.00,
  "amount": 50000.00,
  "currency": "KRW",
  "status": "ACTIVE",
  "createdAt": "2026-01-01T09:00:00",
  "accountNickname": "생활비 계좌",
  "applicationPhotoUrl": "https://...",
  "userName": "홍길동",
  "email": "user@test.com",
  "phone": "010-1234-5678"
}
```

> `amount`: 관리자 심사용 초기입금 신청액 (`pendingInitDeposit`)
> `applicationPhotoUrl`, `userName`, `email`, `phone`: 관리자 심사 상세 조회 시에만 포함
> `accountNickname`: 미설정 시 `null`

---

### GET /accounts
내 계좌 목록 조회 (REJECTED 제외)

**인증 필요**

**Response** `200 OK` — `AccountResponseDto[]`

---

### POST /accounts/apply
계좌 개설 신청 (JSON)

**인증 필요**

**Request Body**
```json
{
  "accountType": "CHECKING",
  "currency": "KRW",
  "accountPassword": "1234",
  "fromAccountId": 2,
  "initDeposit": 50000.00,
  "applyMemo": "신청 메모",
  "accountNickname": "생활비 계좌"
}
```

| 필드 | 타입 | 필수 | 검증 | 설명 |
|------|------|------|------|------|
| accountType | String | Y | NotBlank | 계좌 종류 |
| currency | String | N | — | 통화 (기본: KRW) |
| accountPassword | String | Y | 숫자 4자리 | 계좌 비밀번호 (BCrypt 저장) |
| fromAccountId | Long | 조건부 | — | initDeposit > 0 이면 필수 |
| initDeposit | BigDecimal | N | ≥ 0 | 초기입금액 (승인 시 실 이체) |
| applyMemo | String | N | — | 신청 메모 |
| accountNickname | String | N | 최대 255자 | 계좌 별명 |

> **초기입금 로직:** 신청 시 출금 계좌의 소유권·활성 상태·잔액만 검증. 실제 차감은 관리자 승인 시 수행.

**Response** `201 Created` — `AccountResponseDto`

---

### POST /accounts/apply (Multipart)
계좌 개설 신청 (서류 첨부)

**인증 필요**
**Content-Type:** `multipart/form-data`

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| accountType | String | Y | 계좌 종류 |
| currency | String | N | 통화 (기본: KRW) |
| accountPassword | String | Y | 숫자 4자리 계좌 비밀번호 |
| initDeposit | BigDecimal | N | 초기입금액 (기본: 0) |
| fromAccountId | Long | 조건부 | initDeposit > 0 이면 필수 |
| applyMemo | String | N | 신청 메모 |
| accountNickname | String | N | 계좌 별명 (최대 255자) |
| doc | File | N | 신청 서류 |

**Response** `201 Created` — `AccountResponseDto` (신청인 정보 + 사진 URL 포함)

---

### GET /accounts/applications
신청 현황 조회

**인증 필요**

- **일반 사용자:** 본인의 PENDING / REJECTED 신청 목록
- **관리자(ADMIN):** 전체 PENDING 신청 목록 (신청인 정보 포함)

**Response** `200 OK` — `AccountResponseDto[]`

---

### GET /accounts/applications/{accountId}
계좌 신청 상세 조회 (관리자)

**인증 필요 (ADMIN)**

**Path Parameter:** `accountId` (Long)

**Response** `200 OK` — `AccountResponseDto` (신청인 이름·이메일·전화번호·서류 사진 URL 포함)

---

### POST /accounts/{accountId}/activate
계좌 활성화 / 승인

**인증 필요**

**Path Parameter:** `accountId` (Long)

- **관리자:** PENDING → ACTIVE 승인. `pendingInitDeposit > 0` 이면 출금 계좌에서 실 이체 후 Transaction 2건(WITHDRAWAL, DEPOSIT) 저장
- **사용자:** 본인 계좌 자체 활성화

**Response** `200 OK` — `AccountResponseDto`

---

### POST /accounts/{accountId}/reject
계좌 신청 반려 (관리자)

**인증 필요 (ADMIN)**

**Path Parameter:** `accountId` (Long)

**Response** `200 OK` — `AccountResponseDto`

---

### GET /accounts/{accountId}
계좌 단건 조회

**인증 필요 (본인 소유)**

**Path Parameter:** `accountId` (Long)

**Response** `200 OK` — `AccountResponseDto`

---

### PATCH /accounts/{accountId}/nickname
계좌 별명 수정

**인증 필요 (본인 소유)**

**소유권 검증:** 세션의 `userId` + Path Parameter `accountId` 조합으로 본인 계좌 여부 확인

**Path Parameter:** `accountId` (Long)

**Request Body**
```json
{
  "accountNickname": "새 별명"
}
```

| 필드 | 타입 | 필수 | 검증 | 설명 |
|------|------|------|------|------|
| accountNickname | String | Y | NotBlank, 최대 255자 | 변경할 계좌 별명 |

**Response** `200 OK` — `AccountResponseDto`

---

### PATCH /accounts/{accountId}/status
계좌 상태 변경 (사용자)

**인증 필요 (본인 소유, ACTIVE 계좌만 가능)**

**Path Parameter:** `accountId` (Long)

**Request Body**
```json
{
  "status": "SUSPENDED"
}
```

| status 값 | 설명 |
|-----------|------|
| SUSPENDED | 계좌 정지 |
| LOST | 분실 신고 |

**Response** `200 OK` (빈 바디)

---

### GET /accounts/{accountId}/transactions
거래 내역 조회 (페이지네이션)

**인증 필요 (본인 소유)**

**Path Parameter:** `accountId` (Long)

**Query Parameters**

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| start | ISO DateTime | Y | 조회 시작 시각 (예: `2026-01-01T00:00:00`) |
| end | ISO DateTime | Y | 조회 종료 시각 |
| page | Integer | N | 페이지 번호 (기본: 0) |
| size | Integer | N | 페이지 크기 (기본: 20) |

**Response** `200 OK` — Page\<Transaction\>
```json
{
  "content": [
    {
      "transactionId": 1,
      "accountId": 1,
      "targetAccount": "110-012-345678",
      "amount": 50000.00,
      "transactionType": "WITHDRAWAL",
      "description": "신규 계좌 초기입금",
      "balanceAfter": 950000.00,
      "createdAt": "2026-03-26T10:00:00"
    }
  ],
  "totalElements": 100,
  "totalPages": 5,
  "size": 20,
  "number": 0
}
```

---

## 4. 카드 (Card)

### CardResponseDto 구조
```json
{
  "cardId": 1,
  "cardNumber": "1234-5678-9012-3456",
  "cardType": "CREDIT",
  "accountId": 1,
  "paymentDay": 15,
  "status": "ACTIVE",
  "createdAt": "2026-01-01T09:00:00",
  "applicationPhotoUrl": "https://...",
  "userName": "홍길동",
  "email": "user@test.com",
  "phone": "010-1234-5678"
}
```

---

### GET /cards/{cardId}
카드 상세 조회

**인증 필요**

**Path Variable**

| 파라미터 | 타입 | 설명 |
|----------|------|------|
| cardId | Long | 조회할 카드 ID |

**Response** `200 OK`
```json
{
  "accountNumber": "110-001-000001",
  "paymentDay": 15,
  "cardNumber": "4000123456780001"
}
```

> 카드 번호는 원본 16자리 문자열을 그대로 반환합니다. 마스킹은 프론트엔드에서 처리합니다.

**Error**
- `403 Forbidden` — 본인 카드가 아닌 경우
- `404 Not Found` — 존재하지 않는 카드 ID

---

### PATCH /cards/{cardId}/password
카드 비밀번호 변경

**인증 필요**

**Path Variable**

| 파라미터 | 타입 | 설명 |
|----------|------|------|
| cardId | Long | 비밀번호를 변경할 카드 ID |

**Request Body**
```json
{
  "oldPassword": "1234",
  "newPassword": "5678"
}
```

| 필드 | 타입 | 필수 | 검증 |
|------|------|------|------|
| oldPassword | String | Y | NotBlank |
| newPassword | String | Y | NotBlank, 4자리 숫자 |

**Response** `204 No Content`

**Error**
- `400 Bad Request` — 현재 비밀번호 불일치
- `403 Forbidden` — 본인 카드가 아닌 경우
- `404 Not Found` — 존재하지 않는 카드 ID

---

### GET /cards
내 카드 목록 조회

**인증 필요**

**Response** `200 OK` — `CardResponseDto[]`

---

### POST /cards/apply
카드 발급 신청 (JSON)

**인증 필요**

**Request Body**
```json
{
  "accountId": 1,
  "accountNumber": "110-001-000001",
  "cardType": "CREDIT",
  "paymentDay": 15,
  "cardPw": "1234"
}
```

| 필드 | 타입 | 필수 | 검증 |
|------|------|------|------|
| accountId | Long | Y | NotNull |
| accountNumber | String | Y | NotBlank |
| cardType | String | Y | NotBlank |
| paymentDay | Integer | Y | 1~31 |
| cardPw | String | Y | NotBlank, 4자리 숫자 |

**Response** `201 Created` — `CardResponseDto`

---

### POST /cards/apply (Multipart)
카드 발급 신청 (서류 첨부)

**인증 필요**
**Content-Type:** `multipart/form-data`

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| accountId | Long | Y | 연결 계좌 ID |
| accountNumber | String | Y | 연결 계좌번호 |
| cardType | String | Y | 카드 종류 |
| paymentDay | Integer | Y | 결제일 (1~31) |
| cardPw | String | Y | 카드 비밀번호 (4자리 숫자) |
| doc | File | N | 신청 서류 |

**Response** `201 Created` — `CardResponseDto`

---

### GET /cards/applications
카드 신청 현황

**인증 필요**

- **사용자:** 본인 신청 목록
- **관리자:** 전체 PENDING 목록 (신청인 정보 포함)

**Response** `200 OK` — `CardResponseDto[]`

---

### POST /cards/{cardId}/activate
카드 신청 승인 (관리자)

**인증 필요 (ADMIN)**

**Path Parameter:** `cardId` (Long)

**Response** `200 OK` — `CardResponseDto`

---

### POST /cards/{cardId}/reject
카드 신청 반려 (관리자)

**인증 필요 (ADMIN)**

**Path Parameter:** `cardId` (Long)

**Response** `200 OK` — `CardResponseDto`

---

### PATCH /cards/{cardId}/status
카드 상태 변경 (사용자)

**인증 필요 (본인 소유)**

**Path Parameter:** `cardId` (Long)

**Request Body**
```json
{
  "status": "SUSPENDED"
}
```

| status 값 | 설명 |
|-----------|------|
| SUSPENDED | 카드 정지 |
| LOST | 분실 신고 |
| CANCELED | 카드 해지 |

**Response** `200 OK` — `CardResponseDto`

---

### POST /cards/{cardId}/restore-request
카드 재발급/복원 신청 (사용자)

**인증 필요 (본인 소유)**

**Path Parameter:** `cardId` (Long) — SUSPENDED 또는 LOST 상태

**Response** `200 OK` — `CardResponseDto` (상태: RESTORE_PENDING)

---

### GET /cards/restore-requests
카드 복원 신청 목록 (관리자)

**인증 필요 (ADMIN)**

**Response** `200 OK` — `CardResponseDto[]`

---

### POST /cards/{cardId}/restore-approve
카드 복원 승인 (관리자)

**인증 필요 (ADMIN)**

**Path Parameter:** `cardId` (Long)

**Response** `200 OK` — `CardResponseDto`

---

### POST /cards/{cardId}/restore-reject
카드 복원 반려 (관리자)

**인증 필요 (ADMIN)**

**Path Parameter:** `cardId` (Long)

**Response** `200 OK` — `CardResponseDto`

---

## 5. 대출 (Loan)

### LoanResponseDto 구조
```json
{
  "loanId": 1,
  "loanType": "PERSONAL",
  "amount": 10000000.00,
  "remainingAmount": 9500000.00,
  "interestRate": 6.50,
  "termMonths": 24,
  "startDate": "2026-01-01",
  "endDate": "2028-01-01",
  "fromAccountId": 1,
  "toAccountId": 2,
  "status": "ACTIVE",
  "createdAt": "2026-01-01T09:00:00",
  "applicationPhotoUrl": "https://...",
  "userName": "홍길동",
  "email": "user@test.com",
  "phone": "010-1234-5678"
}
```

| 필드 | 타입 | 설명 |
|------|------|------|
| loanId | Long | 대출 ID |
| loanType | String | 대출 유형 (PERSONAL/MORTGAGE/AUTO/BUSINESS) |
| amount | BigDecimal | 대출 원금 |
| remainingAmount | BigDecimal | 남은 상환 잔액 |
| interestRate | BigDecimal | 연이율 (%) |
| termMonths | Integer | 대출 기간 (개월) |
| startDate | LocalDate | 대출 시작일 |
| endDate | LocalDate | 대출 만기일 |
| fromAccountId | Long | 상환 출금 계좌 ID |
| toAccountId | Long | 대출금 수령 계좌 ID |
| status | String | 상태 (PENDING/ACTIVE/CLOSED/REJECTED) |
| createdAt | LocalDateTime | 신청 일시 |
| applicationPhotoUrl | String | 서류 사진 URL (관리자용) |
| userName | String | 신청인 이름 (관리자용) |
| email | String | 신청인 이메일 (관리자용) |
| phone | String | 신청인 전화번호 (관리자용) |

### RepaymentResponseDto 구조
```json
{
  "repaymentId": 1,
  "repaymentDate": "2026-03-01T09:00:00",
  "amount": 500000,
  "principal": 480000,
  "interest": 20000,
  "remainingAmount": 9500000
}
```

| 필드 | 타입 | 설명 |
|------|------|------|
| repaymentId | Long | 상환 이력 ID |
| repaymentDate | LocalDateTime | 상환 일시 |
| amount | BigDecimal | 상환 총액 (원금 + 이자) |
| principal | BigDecimal | 상환 원금 |
| interest | BigDecimal | 상환 이자 (잔액 × 연이율 / 1200) |
| remainingAmount | BigDecimal | 상환 후 남은 대출 잔액 |

---

### POST /loans/apply
대출 신청 (JSON)

**인증 필요**

**Request Body**
```json
{
  "loanType": "PERSONAL",
  "amount": 10000000.00,
  "termMonths": 24,
  "fromAccountId": 1,
  "toAccountId": 2
}
```

| 필드 | 타입 | 필수 | 검증 |
|------|------|------|------|
| loanType | String | Y | NotBlank |
| amount | BigDecimal | Y | ≥ 100,000 |
| termMonths | Integer | Y | 1~360 |
| fromAccountId | Long | Y | 상환 출금 계좌 ID |
| toAccountId | Long | Y | 대출금 수령 계좌 ID |

**Response** `201 Created` — `LoanResponseDto`

---

### POST /loans/apply (Multipart)
대출 신청 (서류 첨부)

**인증 필요**
**Content-Type:** `multipart/form-data`

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| loanType | String | Y | 대출 종류 |
| amount | BigDecimal | Y | 대출 금액 |
| termMonths | Integer | Y | 대출 기간(개월) |
| fromAccountId | Long | Y | 상환 출금 계좌 ID |
| toAccountId | Long | Y | 대출금 수령 계좌 ID |
| doc | File | N | 신청 서류 |

**Response** `201 Created` — `LoanResponseDto`

---

### GET /loans/applications
대출 신청 현황

**인증 필요**

- **사용자:** 본인 신청 목록
- **관리자:** 전체 PENDING 목록 (신청인 정보 포함)

**Response** `200 OK` — `LoanResponseDto[]`

---

### GET /loans/active
활성 대출 목록 조회

**인증 필요**

**Response** `200 OK` — `LoanResponseDto[]` (ACTIVE 상태만)

---

### POST /loans/{loanId}/activate
대출 신청 승인 (관리자)

**인증 필요 (ADMIN)**

**Path Parameter:** `loanId` (Long)

**Response** `200 OK` — `LoanResponseDto`

---

### POST /loans/{loanId}/reject
대출 신청 반려 (관리자)

**인증 필요 (ADMIN)**

**Path Parameter:** `loanId` (Long)

**Response** `200 OK` — `LoanResponseDto`

---

### GET /loans/{loanId}
대출 단건 조회

**인증 필요 (본인)**

**Path Parameter:** `loanId` (Long)

**Response** `200 OK` — `LoanResponseDto`

---

### PATCH /loans/{loanId}/repayment
대출 상환

**인증 필요 (본인)**

**Path Parameter:** `loanId` (Long)

**Request Body**
```json
{
  "amount": 500000.00,
  "fromAccountId": 1
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| amount | BigDecimal | Y | 상환 금액 |
| fromAccountId | Long | Y | 출금 계좌 ID |

**Response** `200 OK` — `LoanResponseDto`

> 상환 시 원금/이자 분리 계산 후 상환 이력(`repayments` 테이블)이 자동 저장됩니다.

**Error**
- `400 Bad Request` — 상환 금액이 잔여 잔액 초과 또는 ACTIVE 상태가 아닌 경우
- `403 Forbidden` — 본인 대출이 아닌 경우
- `404 Not Found` — 대출 또는 계좌 없음

---

### GET /loans/{loanId}/repayments
상환 이력 조회

**인증 필요 (본인)**

**Path Parameter:** `loanId` (Long)

**Response** `200 OK` — `RepaymentResponseDto[]`
```json
[
  {
    "repaymentId": 1,
    "repaymentDate": "2026-03-01T09:00:00",
    "amount": 500000,
    "principal": 480000,
    "interest": 20000,
    "remainingAmount": 9500000
  }
]
```

**Error**
- `403 Forbidden` — 본인 대출이 아닌 경우
- `404 Not Found` — 존재하지 않는 대출 ID

---

## 6. 이체 (Transfer)

### POST /transfers/validate
이체 사전 검증 (실행 전 확인)

**인증 필요**

**Request Body**
```json
{
  "fromAccountNumber": "110-000-000000",
  "toAccountNumber": "110-012-345678",
  "amount": 100000.00,
  "description": "용돈",
  "otp": "123456",
  "accountPassword": "1234"
}
```

| 필드 | 타입 | 필수 | 검증 |
|------|------|------|------|
| fromAccountNumber | String | Y | NotBlank |
| toAccountNumber | String | Y | NotBlank |
| amount | BigDecimal | Y | ≥ 1 |
| description | String | N | — |
| otp | String | Y | NotBlank |
| accountPassword | String | Y | NotBlank — 출금 계좌 비밀번호 |

**Response** `200 OK`
```json
{
  "toAccountNumber": "110-012-345678",
  "toAccountHolder": "김철수",
  "amount": 100000.00,
  "fee": 0.00,
  "totalAmount": 100000.00,
  "balanceAfterTransfer": 900000.00,
  "valid": true
}
```

**Error**
- `400 Bad Request` — 계좌 비밀번호 불일치 / 출금·수취 계좌 비활성 / 잔액 부족
- `404 Not Found` — 수취 계좌 없음

---

### POST /transfers/execute
이체 실행

**인증 필요**

**Request Body** — `/transfers/validate`와 동일 (`accountPassword` 포함)

**Response** `200 OK` — `Transaction`
```json
{
  "transactionId": 1,
  "accountId": 1,
  "targetAccount": "110-012-345678",
  "amount": 100000.00,
  "transactionType": "WITHDRAWAL",
  "description": "용돈",
  "balanceAfter": 900000.00,
  "createdAt": "2026-03-26T10:00:00"
}
```

**Error**
- `400 Bad Request` — OTP 불일치 / 계좌 비밀번호 불일치 / 잔액 부족
- `404 Not Found` — 수취 계좌 없음

---

### POST /auto-transfers
자동이체 등록

**인증 필요**

**Request Body**
```json
{
  "fromAccountId": 1,
  "toAccountNumber": "110-012-345678",
  "amount": 100000.00,
  "description": "월세",
  "transferDay": 25
}
```

| 필드 | 타입 | 필수 | 검증 | 설명 |
|------|------|------|------|------|
| fromAccountId | Long | Y | NotNull | 출금 계좌 |
| toAccountNumber | String | Y | NotBlank | 입금 계좌번호 |
| amount | BigDecimal | Y | ≥ 1 | 이체 금액 |
| description | String | N | — | 메모 |
| transferDay | Integer | Y | 0~31 | 이체일 (0=매주 일요일, 1~31=매월 N일) |

**Response** `201 Created` — `AutoTransfer`

---

### GET /auto-transfers
자동이체 목록 조회

**인증 필요**

**Query Parameters**

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| accountId | Long | N | 특정 계좌 필터 |

**Response** `200 OK` — `AutoTransfer[]`

---

### DELETE /auto-transfers/{autoTransferId}
자동이체 해제

**인증 필요 (본인)**

**Path Parameter:** `autoTransferId` (Long)

**Response** `204 No Content`

---

## 7. 알림 (Notification)

### NotificationResponseDto 구조
```json
{
  "notificationId": 1,
  "notificationType": "ACCOUNT",
  "title": "계좌 개설 승인",
  "message": "CHECKING 계좌(110-012-345678) 개설이 승인되었습니다.",
  "isRead": false,
  "createdAt": "2026-03-26T10:00:00"
}
```

---

### GET /notifications
내 알림 목록 조회

**인증 필요**

**Response** `200 OK` — `NotificationResponseDto[]`

---

### PATCH /notifications/{id}/read
알림 단건 읽음 처리

**인증 필요**

**Path Parameter:** `id` (Long)

**Response** `200 OK` (빈 바디)

---

### PATCH /notifications/read-all
전체 알림 읽음 처리

**인증 필요**

**Response** `200 OK` (빈 바디)

---

### DELETE /notifications/{id}
알림 삭제

**인증 필요**

**Path Parameter:** `id` (Long)

**Response** `204 No Content`

---

## 8. 공지사항 (Notice)

### NoticeResponseDto 구조
```json
{
  "noticeId": 1,
  "title": "서비스 점검 안내",
  "content": "...",
  "category": "서비스",
  "pinned": true,
  "createdAt": "2026-03-26T09:00:00",
  "updatedAt": "2026-03-26T09:00:00",
  "attachmentUrl": "https://...",
  "prevId": 2,
  "prevTitle": "이전 공지",
  "nextId": null,
  "nextTitle": null
}
```

---

### GET /notices
공지사항 목록 (페이지네이션)

**인증 불필요**

**Query Parameters**

| 파라미터 | 타입 | 기본값 | 설명 |
|----------|------|--------|------|
| page | Integer | 0 | 페이지 번호 |
| size | Integer | 10 | 페이지 크기 |

**Response** `200 OK` — Page\<NoticeResponseDto\>

---

### GET /notices/{noticeId}
공지사항 단건 조회 (이전/다음 글 포함)

**인증 불필요**

**Path Parameter:** `noticeId` (Long)

**Response** `200 OK` — `NoticeResponseDto`

---

### POST /notices
공지사항 등록 (JSON)

**인증 필요 (ADMIN)**

**Request Body**
```json
{
  "title": "서비스 점검 안내",
  "content": "점검 일시: ...",
  "category": "서비스",
  "pinned": false
}
```

| 필드 | 타입 | 필수 | 기본값 |
|------|------|------|--------|
| title | String | Y | — |
| content | String | Y | — |
| category | String | N | 서비스 |
| pinned | Boolean | N | false |

**Response** `201 Created` — `NoticeResponseDto`

---

### POST /notices (Multipart)
공지사항 등록 (파일 첨부)

**인증 필요 (ADMIN)**
**Content-Type:** `multipart/form-data`

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| title | String | Y | 제목 |
| content | String | Y | 내용 |
| category | String | N | 카테고리 (기본: 서비스) |
| pinned | Boolean | N | 상단 고정 (기본: false) |
| file | File | N | 첨부 파일 |

**Response** `201 Created` — `NoticeResponseDto`

---

### PATCH /notices/{noticeId}
공지사항 수정 (JSON)

**인증 필요 (ADMIN)**

**Path Parameter:** `noticeId` (Long)

**Request Body**
```json
{
  "title": "수정된 제목",
  "content": "수정된 내용",
  "category": "이벤트",
  "pinned": true
}
```

**Response** `200 OK` — `NoticeResponseDto`

---

### PATCH /notices/{noticeId} (Multipart)
공지사항 수정 (파일 교체)

**인증 필요 (ADMIN)**
**Content-Type:** `multipart/form-data`

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| title | String | Y | 수정 제목 |
| content | String | Y | 수정 내용 |
| category | String | N | 카테고리 |
| pinned | Boolean | N | 상단 고정 |
| file | File | N | 교체할 파일 |

**Response** `200 OK` — `NoticeResponseDto`

---

### DELETE /notices/{noticeId}
공지사항 삭제

**인증 필요 (ADMIN)**

**Path Parameter:** `noticeId` (Long)

**Response** `204 No Content`

---

## 9. 보안 뉴스 (Security News)

### SecurityNewsResponseDto 구조
```json
{
  "securityNewsId": 1,
  "title": "피싱 사기 주의",
  "content": "...",
  "createdAt": "2026-03-26T09:00:00",
  "updatedAt": "2026-03-26T09:00:00",
  "attachmentUrl": "https://...",
  "prevId": 2,
  "prevTitle": "이전 뉴스",
  "nextId": null,
  "nextTitle": null
}
```

---

### GET /security-news
보안 뉴스 목록 (페이지네이션, 최신순)

**인증 불필요**

**Query Parameters**

| 파라미터 | 타입 | 기본값 |
|----------|------|--------|
| page | Integer | 0 |
| size | Integer | 10 |

**Response** `200 OK` — Page\<SecurityNewsResponseDto\>

---

### GET /security-news/{securityNewsId}
보안 뉴스 단건 조회

**인증 불필요**

**Path Parameter:** `securityNewsId` (Long)

**Response** `200 OK` — `SecurityNewsResponseDto`

---

### POST /security-news
보안 뉴스 등록 (JSON)

**인증 필요 (ADMIN)**

**Request Body**
```json
{
  "title": "피싱 사기 주의",
  "content": "최근 피싱 사기가 증가하고 있습니다..."
}
```

**Response** `201 Created` — `SecurityNewsResponseDto`

---

### POST /security-news (Multipart)
보안 뉴스 등록 (파일 첨부)

**인증 필요 (ADMIN)**
**Content-Type:** `multipart/form-data`

| 파라미터 | 타입 | 필수 |
|----------|------|------|
| title | String | Y |
| content | String | Y |
| file | File | N |

**Response** `201 Created` — `SecurityNewsResponseDto`

---

### PATCH /security-news/{securityNewsId}
보안 뉴스 수정 (JSON)

**인증 필요 (ADMIN)**

**Path Parameter:** `securityNewsId` (Long)

**Request Body**
```json
{
  "title": "수정 제목",
  "content": "수정 내용"
}
```

**Response** `200 OK` — `SecurityNewsResponseDto`

---

### PATCH /security-news/{securityNewsId} (Multipart)
보안 뉴스 수정 (파일 교체)

**인증 필요 (ADMIN)**
**Content-Type:** `multipart/form-data`

| 파라미터 | 타입 | 필수 |
|----------|------|------|
| title | String | Y |
| content | String | Y |
| file | File | N |

**Response** `200 OK` — `SecurityNewsResponseDto`

---

### DELETE /security-news/{securityNewsId}
보안 뉴스 삭제

**인증 필요 (ADMIN)**

**Path Parameter:** `securityNewsId` (Long)

**Response** `204 No Content`

---

## 10. 배너 (Banner)

### BannerResponseDto 구조
```json
{
  "bannerId": 1,
  "title": "신규 계좌 개설 이벤트",
  "subtitle": "지금 개설하고 혜택 받으세요",
  "colors": "#1e3a8a,#3b82f6",
  "isActive": true,
  "startDate": "2026-03-01",
  "endDate": "2026-03-31",
  "sortOrder": 1,
  "imageUrl": "https://presigned-url..."
}
```

| 필드 | 타입 | 설명 |
|------|------|------|
| bannerId | Long | 배너 ID |
| title | String | 배너 제목 |
| subtitle | String | 부제목 |
| colors | String | 배경 그라데이션 색상 (쉼표 구분) |
| isActive | Boolean | 활성 여부 |
| startDate | LocalDate | 게시 시작일 |
| endDate | LocalDate | 게시 종료일 |
| sortOrder | Integer | 표시 순서 |
| imageUrl | String | S3 presigned URL (60분 유효) — 이미지 없으면 null, 이때 colors로 배경 표시 |

---

### GET /banners
활성 배너 목록 조회 (홈 화면 / 대시보드)

**인증 불필요**

**Response** `200 OK` — `BannerResponseDto[]` (활성 배너만, imageUrl 포함)

---

### GET /banners/all
전체 배너 목록 (관리자)

**인증 필요 (ADMIN)**

**Response** `200 OK` — `BannerResponseDto[]` (비활성 포함, imageUrl 포함)

---

### POST /banners
배너 등록

**인증 필요 (ADMIN)**
**Content-Type:** `multipart/form-data`

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| title | String | Y | 배너 제목 |
| subtitle | String | N | 부제목 |
| colors | String | N | 배경 색상 (쉼표 구분, 기본값: `#1e3a8a,#3b82f6`) |
| isActive | Boolean | N | 활성 여부 |
| startDate | LocalDate | N | 게시 시작일 (yyyy-MM-dd) |
| endDate | LocalDate | N | 게시 종료일 (yyyy-MM-dd) |
| sortOrder | Integer | N | 표시 순서 (기본값: 0) |
| image | File | N | 배너 이미지 (JPG/PNG/PDF, 10MB 이하) — 미첨부 시 colors로 배경 표시 |

**Response** `201 Created` — `BannerResponseDto`

---

### PATCH /banners/{bannerId}
배너 수정

**인증 필요 (ADMIN)**
**Content-Type:** `multipart/form-data`

**Path Parameter:** `bannerId` (Long)

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| title | String | N | 배너 제목 |
| subtitle | String | N | 부제목 |
| colors | String | N | 배경 색상 |
| isActive | Boolean | N | 활성 여부 |
| startDate | LocalDate | N | 게시 시작일 (yyyy-MM-dd) |
| endDate | LocalDate | N | 게시 종료일 (yyyy-MM-dd) |
| sortOrder | Integer | N | 표시 순서 |
| image | File | N | 새 배너 이미지 — 미첨부 시 기존 이미지 유지 |

> 새 이미지 첨부 시 기존 S3 이미지는 자동 삭제됩니다.

**Response** `200 OK` — `BannerResponseDto`

---

### PATCH /banners/{bannerId}/toggle
배너 활성/비활성 토글

**인증 필요 (ADMIN)**

**Path Parameter:** `bannerId` (Long)

**Response** `200 OK` — `BannerResponseDto`

---

### DELETE /banners/{bannerId}
배너 삭제

**인증 필요 (ADMIN)**

**Path Parameter:** `bannerId` (Long)

> 삭제 시 연결된 S3 이미지도 함께 삭제됩니다.

**Response** `204 No Content`

---

## 11. 통계 (Statistics)

### GET /admin/statistics
전체 신청 건수 통계 조회

**인증 필요 (ADMIN)**

**Response** `200 OK`
```json
{
  "totalAccounts": 42,
  "totalCards": 17,
  "totalLoans": 8
}
```

| 필드 | 타입 | 설명 |
|------|------|------|
| totalAccounts | long | 계좌 전체 신청 건수 (상태 무관) |
| totalCards | long | 카드 전체 신청 건수 (상태 무관) |
| totalLoans | long | 대출 전체 신청 건수 (상태 무관) |

**Error**
- `403 Forbidden` — ADMIN이 아닌 경우

---

## 12. 1:1 문의 (Inquiry)

### InquiryResponseDto 구조
```json
{
  "inquiryId": 1,
  "title": "카드 발급 문의",
  "content": "카드 발급이 너무 오래 걸립니다.",
  "answer": "확인 후 처리해드리겠습니다.",
  "status": "RESOLVED",
  "createdAt": "2026-03-31T10:00:00",
  "answeredAt": "2026-03-31T11:00:00"
}
```

| 필드 | 타입 | 설명 |
|------|------|------|
| inquiryId | Long | 문의 ID |
| title | String | 문의 제목 |
| content | String | 문의 내용 |
| answer | String | 관리자 답변 (미답변 시 null) |
| status | String | 처리 상태 (PENDING / IN_PROGRESS / RESOLVED) |
| createdAt | LocalDateTime | 문의 등록 일시 |
| answeredAt | LocalDateTime | 답변 일시 (미답변 시 null) |

---

### POST /inquiries
문의 등록

**인증 필요**

**Request Body**
```json
{
  "title": "카드 발급 문의",
  "content": "카드 발급이 너무 오래 걸립니다."
}
```

| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| title | String | Y | 문의 제목 |
| content | String | Y | 문의 내용 |

**Response** `200 OK` — `InquiryResponseDto`

---

### GET /inquiries
내 문의 목록 조회

**인증 필요**

**Response** `200 OK` — `InquiryResponseDto[]` (최신순)

---

### GET /inquiries/{inquiryId}
내 문의 단건 조회

**인증 필요**

**Path Parameter:** `inquiryId` (Long)

**Response** `200 OK` — `InquiryResponseDto`

**Error**
- `403 Forbidden` — 본인 문의가 아닌 경우
- `404 Not Found` — 존재하지 않는 문의 ID

---

### 문의 상태값 (InquiryStatus)
| 값 | 설명 |
|----|------|
| PENDING | 접수 대기 |
| IN_PROGRESS | 처리 중 |
| RESOLVED | 답변 완료 |

---

## 부록: 도메인 상태값

### 계좌 상태 (AccountStatus)
| 값 | 설명 |
|----|------|
| PENDING | 심사 대기 |
| ACTIVE | 활성 |
| SUSPENDED | 정지 |
| LOST | 분실 신고 |
| REJECTED | 심사 반려 |

### 카드 상태 (CardStatus)
| 값 | 설명 |
|----|------|
| PENDING | 심사 대기 |
| ACTIVE | 활성 |
| SUSPENDED | 정지 |
| LOST | 분실 신고 |
| CANCELED | 해지 |
| RESTORE_PENDING | 복원 심사 중 |
| REJECTED | 반려 |

### 거래 유형 (transactionType)
| 값 | 설명 |
|----|------|
| WITHDRAWAL | 출금 |
| DEPOSIT | 입금 |

### 알림 유형 (notificationType)
| 값 | 설명 |
|----|------|
| ACCOUNT | 계좌 관련 |
| CARD | 카드 관련 |
| LOAN | 대출 관련 |
| TRANSFER | 이체 관련 |
| INQUIRY | 문의 관련 |

---

## 부록: 엔드포인트 요약

| # | 도메인 | 엔드포인트 수 |
|---|--------|-------------|
| 1 | Auth | 7 |
| 2 | User | 6 |
| 3 | Account | 10 |
| 4 | Card | 11 |
| 5 | Loan | 9 |
| 6 | Transfer | 5 |
| 7 | Notification | 4 |
| 8 | Notice | 7 |
| 9 | Security News | 7 |
| 10 | Banner | 6 |
| 11 | Statistics | 1 |
| 12 | Inquiry | 3 |
| 13 | Exchange | 1 |
| | **합계** | **77** |

---

## 13. 환율 조회 (Exchange)

### GET /exchange/rate
환율 데이터 조회

**인증 불필요** (비로그인 사용자 접근 가능)

**Query Parameter**

| 파라미터 | 타입 | 필수 | 설명 |
|----------|------|------|------|
| url | String | Y | 환율 API URL. `{key}` 플레이스홀더 포함 시 서버 API 키로 자동 치환 |

**요청 예시**
```
GET /exchange/rate?url=https://oapi.koreaexim.go.kr/site/program/financial/exchangeJSON?authkey={key}&searchdate=20180102&data=AP01
```

**Response** `200 OK` — 외부 API 응답 본문 (JSON 문자열)

**Error**
- `500 Internal Server Error` — URL 요청 실패
