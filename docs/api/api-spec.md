# Fridgely API 명세서

출처: [구글 시트 "스마트 냉장고 API 시트"](https://docs.google.com/spreadsheets/d/1GCKBe8YhoNOzQ3taUI94wlg1xR-9Ky_g_RRe93bUe6M/edit?gid=1310724013)
스냅샷 일자: 2026-09-09 (시트 탭: 공통 / 회원 / 인증 / 냉장고 / 재고 / 추천 / 알림)

이 문서는 링크가 아니라 실제 명세 내용을 프로젝트 컨텍스트 안에 둔 사본이다. 일반 작업은 이 문서를 읽고 진행하고, 시트를 매번 열지 않는다. 시트가 갱신되면 사용자가 요청할 때 이 문서를 다시 동기화한다.
문서 하단 [12. 명세서 검토 메모](#12-명세서-검토-메모)에 기준 문서·확정 이슈와 어긋나는 지점을 정리해 두었다.

---

## 1. 공통 규약

### 성공 응답 포맷

```json
{
  "code": "USER-200-001",
  "message": "회원 정보를 수정했습니다.",
  "data": {
    "userId": "7",
    "nickname": "주방장"
  }
}
```

### 에러 응답 포맷 (RFC 7807 계열)

```json
{
  "title": "닉네임 중복",
  "status": 409,
  "detail": "이미 사용 중인 닉네임입니다. 다른 닉네임을 입력해 주세요.",
  "instance": "/api/v1/users/me",
  "code": "USER-409-001"
}
```

- `instance`는 요청 URI(`{REQUEST_URI}`)를 담는다.
- 에러 코드는 `{도메인}-{HTTP 상태}-{기능 순번}` 형태다.
- 아래 각 API의 에러 표는 이 포맷을 공유하므로 `title` / `detail` / `code`만 표기한다.

### 인증 헤더

- 보호된 API는 `Authorization: Bearer {accessToken}`을 사용한다.
- 로그인·회원가입 성공 시 `Set-Cookie: refreshToken={REFRESH_TOKEN}; Max-Age=1209600; Path=/api/v1/auth; HttpOnly; Secure; SameSite=Lax`, `Cache-Control: no-store`를 내려준다.

---

## 2. 이미지 업로드 (공통)

### POST /api/image/presigned-url — 사진 업로드 URL 요청

화면: REG-002(재고 등록), MY-003(마이페이지) · 헤더: `Authorization`

Request body

```json
{
  "purpose": "ANALYSIS",
  "contentType": "image/jpeg",
  "byteSize": 512000,
  "sha256": "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"
}
```

필수 필드

| 필드 | 타입 | 설명 |
|---|---|---|
| `purpose` | String | 이미지 업로드 용도 |
| `contentType` | String | 업로드할 파일의 MIME 타입 |
| `byteSize` | Long | 파일 크기(바이트), 0보다 큰 정수 |
| `sha256` | String | 파일 바이너리의 SHA-256 해시, 소문자 16진수 64자 |

- `purpose` 유형: `ANALYSIS`(식재료 분석용), `PROFILE`(프로필 사진용)
- 검증: `purpose`는 서버 지원 값, `contentType`은 허용 이미지 MIME, `byteSize`는 용도별 최대 크기 이하, `sha256`은 `^[0-9a-f]{64}$`

200 응답

```json
{
  "code": "IMAGE-200-001",
  "message": "presigned url 발급 성공",
  "data": {
    "uploadUrl": "https://storage.example.com/signed-upload",
    "method": "PUT",
    "headers": {
      "Content-Type": "image/jpeg",
      "x-amz-checksum-sha256": "<base64-sha256>"
    },
    "expiresAt": "2026-09-07T09:10:00+09:00"
  }
}
```

에러

| 상태 | title | code |
|---|---|---|
| 400 | 입력 형식 위반 (입력 규칙 위반 혹은 필수 입력 누락) | IMAGE-400-001 |
| 400 | 이미지 크기 제한 초과 (`byteSize` 최대 크기 초과) | IMAGE-400-001 |
| 401 | 로그인이 필요합니다. | IMAGE-401-001 |
| 500 | 서버 오류 | IMAGE-500-001 |

> 별도 `presigned-url` 탭에 `POST /api/image`(사진 업로드) 행이 남아 있으나 상세 명세는 비어 있다.

---

## 3. 회원 (users)

### POST /api/users — 로컬 회원가입

화면: AUTH-001

```json
{
  "loginId": "fridge01",
  "password": "example-only-123",
  "nickname": "fridge01"
}
```

- 필수: `loginId`(String), `password`(String) / 비필수: `nickname`(String)
- `nickname` 미입력 시 아이디와 동일한 값으로 저장한다.
- `loginId`는 2~10자 한글·영문·숫자만 허용하고 공백은 제거한다.
- `password`는 8자 이상, 영문과 숫자를 각각 1자 이상 포함한다.
- 회원가입 성공은 로그인 성공과 동일하게 처리한다.

200 응답 (헤더: `Location: {FRONTEND_URL}/`, refreshToken 쿠키, `Cache-Control: no-store`)

```json
{
  "code": "USER-201-001",
  "message": "회원가입 성공",
  "data": {
    "userId": "1",
    "nickname": "fridge01",
    "profileImageUrl": null,
    "activeRefrigeratorId": "1",
    "cookingCount": 0,
    "accessToken": "{JWT_ACCESS_TOKEN}"
  }
}
```

| 상태 | title | code |
|---|---|---|
| 400 | 입력 형식이 잘못됐습니다. | USER-400-001 |
| 409 | 닉네임 중복 | USER-409-001 |
| 409 | 이미 회원가입한 계정입니다. | USER-409-001 |
| 422 | 부적절한 닉네임입니다. (금칙어) | USER-422-001 |
| 500 | 서버 오류 | USER-500-001 |

### POST /api/users/oauth — 소셜 회원가입 완료

화면: AUTH-003

```json
{
  "registrationToken": "가입전용토큰",
  "nickname": "dave",
  "notificationSetting": true
}
```

- 모든 필드 필수. `registrationToken`으로 서버가 사용자의 OAuth 정보를 식별한다.
- 닉네임은 2~10자, 한글·영문·숫자만 허용하고 공백은 제거한다.

201 응답 (헤더: `Location`, accessToken 쿠키 `Max-Age=900; Path=/api/v1`, refreshToken 쿠키, `Cache-Control: no-store`)

```json
{
  "code": "USER-201-002",
  "message": "회원가입 성공",
  "data": {
    "userId": "1",
    "nickname": "냉장고주인",
    "profileImageUrl": null,
    "activeRefrigeratorId": "1",
    "cookingCount": 0
  }
}
```

| 상태 | 상황 | title | code |
|---|---|---|---|
| 400 | 필수값 누락·형식 오류 | 입력 형식이 잘못됐습니다. | USER-400-002 |
| 400 | 가입 토큰 만료·무효 | 회원가입 인증이 유효하지 않습니다. | USER-400-002 |
| 409 | 닉네임 중복 | 닉네임 중복 | USER-409-001 |
| 409 | 이메일 중복 금지 정책 적용 시 | 중복된 이메일입니다. | USER-409-002 |
| 409 | 이미 가입한 소셜 계정 | 이미 회원가입한 계정입니다. | USER-409-002 |
| 422 | 닉네임 금칙어 | 부적절한 닉네임입니다. | USER-422-002 |
| 500 | 서버 오류 | 서버 오류 | USER-500-002 |

### POST /api/users/nickname-validations — 닉네임 검사

화면: AUTH-003, MY-002 · body `{ "nickname": "david" }`

- 중복·금칙어·형식을 검사한다. 2~10자, 한글·영문·숫자, 공백 제거.
- **요청 실패가 아니므로 400이 아닌 200으로 응답하고 본문에 검사 결과를 담는다.**

```json
{ "code": "USER-200-006", "message": "사용 가능한 닉네임", "data": { "valid": true, "reason": null } }
```

| 결과 | code | message | reason |
|---|---|---|---|
| 금칙어 | USER-200-005 | 부적절한 닉네임입니다. | 금칙어 |
| 형식 위반 | USER-200-005 | 닉네임은 2~10자, 한글·영문·숫자만 가능합니다. | 닉네임 형식 위반 |
| 중복 | USER-200-005 | 중복된 닉네임입니다. | 닉네임 중복 |

500: 서버 오류 (`USER-500-006`)

### GET /api/users/me — 회원정보 조회

화면: MY-001 · 헤더: `Authorization`

```json
{
  "code": "USER-200-003",
  "message": "회원정보 조회 성공",
  "data": {
    "userId": "1",
    "nickname": "dave",
    "profileImageUrl": null,
    "activeRefrigeratorId": "1",
    "cookingCount": 3
  }
}
```

401 로그인이 필요합니다. (`USER-401-003`) / 500 서버 오류 (`USER-500-003`)

### PATCH /api/users/me — 회원정보 수정

화면: MY-002 · body `{ "nickname": "david", "profileImage": "imageObjectKey" }`

- 필수: `nickname`(2~10자, 한글·영문·숫자, 공백 제거)

```json
{
  "code": "USER-200-004",
  "message": "회원정보 수정 성공",
  "data": { "userId": "1", "nickname": "david", "profileImageUrl": "imageObjectKey" }
}
```

| 상태 | title | code |
|---|---|---|
| 400 | 입력 형식이 잘못됐습니다. | USER-400-004 |
| 401 | 로그인이 필요합니다. | USER-401-004 |
| 409 | 닉네임 중복 | USER-409-004 |
| 422 | 부적절한 닉네임입니다. | USER-422-004 |
| 500 | 서버 오류 | USER-500-004 |

### DELETE /api/users/me — 회원탈퇴

화면: MY-002 · 성공 시 **204 No Content**, 응답 본문 없음

탈퇴 처리
- 회원 탈퇴 상태로 전환하고 개인정보를 익명화한다.
- 재가입 시 기존 데이터를 자동 복구하지 않는다.

냉장고
- 방장 탈퇴 시 남은 유효 구성원에게 자동 위임한다. 기준은 `created_at ASC`, 동률이면 `refrigerator_member_id ASC`.
- 구성원이 남으면 냉장고·재고를 유지한다.
- 마지막 구성원이 탈퇴하면 `deleted_at`을 기록한다(soft delete).
- 삭제된 냉장고는 조회·가입·재고 변경·추천·푸시를 모두 차단한다.
- 위임·마지막 구성원 판단·탈퇴는 같은 트랜잭션과 잠금으로 처리한다.

인증·푸시
- 모든 기기의 refresh token을 폐기하고 푸시 구독을 비활성화한다.
- 만료 전 access token으로도 탈퇴 계정 접근을 차단한다.

데이터 정리
- 일괄 연쇄 삭제는 하지 않는다.
- 기록 보관이 끝나고 FK를 정리한 뒤 냉장고를 hard delete 한다.
- 보관 기간·정리 주기는 별도 운영 정책이며 확정 전에는 실행하지 않는다.

401 로그인이 필요합니다. (`USER-401-005`) / 500 서버 오류 (`USER-500-005`)

### DELETE /api/auth/sessions — 로그아웃

화면: MY-001 · 헤더: `Authorization`

```json
{ "code": "AUTH-200-005", "message": "로그아웃 성공", "data": null }
```

401 로그인이 필요합니다. (`AUTH-200-005`) / 500 서버 오류 (`AUTH-500-005`)

---

## 4. 인증 (auth)

### POST /api/auth/sessions — 로그인

화면: AUTH-001 · body `{ "loginId": "testID", "password": "password" }`

- `loginId` 2~10자 한글·영문·숫자, 공백 제거. `password` 8자 이상, 영문·숫자 각 1자 이상.

200 (헤더: `Location`, refreshToken 쿠키, `Cache-Control: no-store`)

```json
{
  "code": "AUTH-200-001",
  "mesage": "로그인 성공",
  "data": { "userId": "1", "accessToken": "{JWT_ACCESS_TOKEN}" }
}
```

| 상태 | title | code |
|---|---|---|
| 400 | 입력 형식이 잘못됐습니다. | AUTH-400-001 |
| 401 | 아이디·비밀번호가 틀렸습니다. | AUTH-401-001 |
| 404 | 존재하지 않는 아이디입니다. | AUTH-404-001 |
| 500 | 서버 오류 | AUTH-500-001 |

### GET /api/auth/oauth/{provider} — OAuth 인증 시작

- Path `provider`: `kakao` / `google`, body 없음
- 302 → `Location: {OAuth 인증 경로}`, body 없음
- 500 서버 오류 (`AUTH-500-002`)

### GET /login/oauth2/code/{provider} — OAuth 로그인 콜백

프론트를 거치지 않는다. Path `provider`, Query `code`, `state`

| 상황 | 응답 |
|---|---|
| 기존 회원 | 302 `Location: {FRONTEND_URL}/oauth/callback?code={LOGIN_CODE}`, `Cache-Control: no-store` |
| 신규 회원 (소셜 정보 임시 저장) | 302 `Location: {FRONT_URL}/signup?signup_code={SIGNUP_CODE}` |
| 사용자 동의 거부 등 접근 거부 | 302 `Location: {FRONT_URL}/login?error=OAUTH_CANCELLED` |
| OAuth 검증·인증 실패 | 302 `Location: {FRONT_URL}/login?error=OAUTH_LOGIN_FAILED` |
| 서버 오류 | 500 (`AUTH-500-003`) |

### POST /api/auth/token — 소셜 로그인 토큰 발급

화면: AUTH-003 · body `{ "loginCode": "일회용코드" }` · 성공 시 일회용 코드 소비

```json
{
  "code": "AUTH-200-004",
  "mesage": "로그인 성공",
  "data": { "userId": "1", "accessToken": "{JWT_ACCESS_TOKEN}" }
}
```

400 회원가입 인증이 유효하지 않습니다. (코드 누락·만료·무효·사용됨, `AUTH-400-004`) / 500 (`AUTH-500-004`)

### POST /api/auth/oauth/registration-tokens — 소셜 회원가입 정보 교환

화면: AUTH-003 · body `{ "signupCode": "일회용코드" }` · 성공 시 일회용 코드 소비

```json
{
  "code": "AUTH-200-005",
  "message": "회원가입 정보 조회 성공",
  "data": { "registrationToken": "가입전용토큰", "expiresIn": 600 }
}
```

400 회원가입 인증이 유효하지 않습니다. (`AUTH-400-005`) / 500 (`AUTH-500-005`)

### POST /api/auth/token-renewals — 인증정보 갱신

필수 쿠키: `refresh: {refresh token}`

```json
{
  "code": "AUTH-200-006",
  "message": "인증정보 갱신 성공",
  "data": { "userId": "1", "accessToken": "{JWT_ACCESS_TOKEN}" }
}
```

400 리프레시 토큰이 유효하지 않습니다. (`AUTH-400-006`) / 500 (`AUTH-500-006`)

---

## 5. 냉장고 · 공유 (refrigerators)

모든 API 헤더: `Authorization: Bearer {accessToken}`

### POST /api/refrigerators/{refrigerator-id}/invitations — 초대코드 발급

화면: SHARE-002 · Path `refrigerator-id`(Long)

```json
{ "code": "REFRIGERATOR-200-001", "message": "초대코드 발급 성공", "data": { "code": "P2F32Y" } }
```

401 (`-401-001`) / 403 본인이 소유한 냉장고가 아닙니다. (`-403-001`) / 404 존재하지 않는 냉장고입니다. (`-404-001`) / 500 (`-500-001`)

### POST /api/refrigerators/members — 공유 참여 [신규 API]

화면: SHARE-003 · body `{ "inviteCode": "P2F32Y" }`

- `inviteCode` 필수. 총 6자리 영문·숫자 조합만 허용한다.
- 사용자와 냉장고 사이의 멤버 리소스가 새로 생성되므로 201을 사용한다. (시트 메모: UUID 도입 시 변경 필요)

```json
{ "code": "REFRIGERATOR-200-002", "message": "냉장고 참여 성공.", "data": { "refrigeratorId": "2" } }
```

| 상태 | title | code |
|---|---|---|
| 400 | 본인의 냉장고에는 참여할 수 없습니다. | REFRIGERATOR-400-002 |
| 400 | 초대코드 형식이 잘못됐습니다. (6자 영문 대문자+숫자) | REFRIGERATOR-400-002 |
| 400 | 냉장고 공유 정원이 가득 차 참여할 수 없습니다. (최대 4명) | REFRIGERATOR-400-002 |
| 400 | 초대코드가 만료됐습니다. | REFRIGERATOR-400-002 |
| 401 | 로그인이 필요합니다. | REFRIGERATOR-401-002 |
| 404 | 존재하지 않는 초대코드입니다. | REFRIGERATOR-404-002 |
| 409 | 이미 다른 냉장고에 참여 중입니다. | REFRIGERATOR-409-002 |
| 500 | 서버 오류 | REFRIGERATOR-500-002 |

### GET /api/refrigerators/{refrigerator-id}/members — 참여자 목록 조회

화면: SHARE-001 · Path `refrigerator-id`(Long)

```json
{
  "code": "REFRIGERATOR-200-003",
  "message": "냉장고 공유 참여자 목록 조회 성공",
  "data": {
    "refrigeratorName": "dave 냉장고",
    "memberNum": 2,
    "memberMax": 4,
    "members": [
      { "userId": "1", "nickname": "dave", "profileImage": "default_image", "role": "owner" },
      { "userId": "2", "nickname": "jade", "profileImage": "default_image", "role": "member" }
    ]
  }
}
```

401 (`-401-003`) / 403 본인이 참여 중인 냉장고만 조회 가능 (`-403-003`) / 404 (`-404-003`) / 500 (`-500-003`)

### GET /api/refrigerators/{refrigerator-id}/members/stats — 참여자 인원수 조회

화면: SHARE-001

```json
{ "code": "REFRIGERATOR-200-004", "message": "냉장고 참여 인원 조회 성공", "data": { "memberNum": 2 } }
```

401 (`-401-004`) / 403 (`-403-004`) / 404 (`-404-004`) / 500 (`-500-004`)

### DELETE /api/refrigerators/{refrigerator-id}/members/me — 공유 나가기

화면: SHARE-001 · 성공 시 **204**

| 상태 | title | code |
|---|---|---|
| 400 | 본인이 소유한 냉장고에서는 나갈 수 없습니다. | REFRIGERATOR-400-005 |
| 401 | 로그인이 필요합니다. | REFRIGERATOR-401-005 |
| 403 | 접근 권한이 부족합니다. | REFRIGERATOR-403-005 |
| 404 | 존재하지 않는 냉장고입니다. | REFRIGERATOR-404-005 |
| 500 | 서버 오류 | REFRIGERATOR-500-005 |

---

## 6. 재고 (ingredients)

### GET /api/refrigerators/{refrigerator-id}/expired — 이번 달 만료된 재고 수 조회

화면: 마이페이지

```json
{
  "code": "REFRIGERATOR-200-010",
  "message": "이번 달 만료된 재고 목록 조회 성공",
  "data": { "expiredIngredientsNum": 30 }
}
```

401 (`-401-010`) / 403 (`-403-010`) / 404 (`-404-010`) / 500 (`-500-010`)

### GET /api/refrigerators/{refrigerator-id}/ingredients — 재고 목록 조회

화면: STOCK-001 · 헤더 `Authorization`

Query: `cursor`(String, 첫 요청 생략), `size`(int), `sort`(String), `category`(String), `filter`(String), `keyword`(String)

| 파라미터 | 값 |
|---|---|
| `sort` | `EXPIRATION_ASC`(유통기한 오름순), `CREATED_DESC`(등록순), `NAME_ASC`(이름순, 가나다순·한영 중 한글 우선) |
| `category` | `VEGETABLES`, `FRUITS`, `MEAT`, `SEAFOOD`, `DAIRY`, `TOFU_BEANS`, `GRAINS_NOODLES`, `PROCESSED_FOODS`, `SEASONINGS`, `BEVERAGES`, `OTHER` |
| `filter` | `TOTAL`, `EXPIRING`, `EXPIRED`, `REFRIGERATED`, `FROZEN` |

```json
{
  "code": "REFRIGERATOR-200-008",
  "message": "냉장고 재고 목록 조회 성공",
  "data": {
    "ingredientsNum": 30,
    "refrigeratorCapacity": 100,
    "ingredients": {
      "ingredientId": "1",
      "name": "달걀",
      "category": "TOFU_BEAN",
      "quantity": 10,
      "weightValue": null,
      "weightUnit": "NONE",
      "storageType": "REFRIGERATED",
      "status": "EXPIRED",
      "daysUntilExpiration": 1
    },
    "nextCursor": "eyJpZCI6MTA0fQ"
  }
}
```

401 (`-401-008`) / 403 (`-403-008`) / 404 (`-403-008`, 시트 표기 그대로) / 500 (`-500-008`)

### DELETE /api/refrigerators/{refrigerator_id}/ingredients/expired — 만료 재고 모두 만료 처리

화면: STOCK-001 · 성공 시 **204**

401 (`-401-015`) / 403 (`-403-015`) / 404 존재하지 않는 냉장고 (`-404-015`) / 404 존재하지 않는 재료 (`-404-015`) / 500 (`-500-015`)

### GET /api/ingredients/{ingredient-id} — 재고 상세 조회

화면: STOCK-002-1 · 응답 헤더 `ETag: "7"`

```json
{
  "code": "REFRIGERATOR-200-011",
  "message": "재고 상세 조회 성공",
  "data": {
    "ingredientId": "1",
    "name": "두부",
    "category": "TOFU_BEAN",
    "storageType": "REFRIGERATED",
    "measureType": "WEIGHT",
    "quantity": null,
    "weightValue": "300.000",
    "weightUnit": "g",
    "expirationDate": "2026-09-09",
    "imageUrl": null,
    "status": "EXPIRED",
    "daysUntilExpiration": 1,
    "version": 7
  }
}
```

401 (`-401-011`) / 403 (`-403-011`) / 404 (`-404-011`) / 500 (`-500-011`)

### PATCH /api/refrigerators/{refrigerator-id}/ingredients/{ingredient-id} — 재고 수정

화면: STOCK-003 · 헤더 `Authorization`, **`If-Match: {ETag}`**

```json
{
  "name": "두부",
  "category": "TOFU_BEAN",
  "storageType": "REFRIGERATED",
  "measureType": "WEIGHT",
  "quantity": null,
  "weightValue": "300.000",
  "weightUnit": "G",
  "expirationDate": "2026-09-10",
  "imageUploadId": null
}
```

- 필수: `name`, `category`, `storageType`, `measureType`, `expirationDate`, `weightUnit`
- 조건부 필수: `measureType`이 `WEIGHT`면 `weightValue`(double), `COUNT`면 `quantity`(int)
- `measureType` 유형: `WEIGHT`, `COUNT`

200 응답은 상세 조회와 동일한 형태이며 `version`이 증가한다 (`REFRIGERATOR-200-013`, "재고 수정 성공").

| 상태 | title | code |
|---|---|---|
| 400 | 입력 형식이 잘못됐습니다. | REFRIGERATOR-400-013 |
| 400 | 재료 허용 수량이 아닙니다. | REFRIGERATOR-400-013 |
| 400 | 재료는 개수만 사용하거나 무게만 사용할 수 있습니다. | REFRIGERATOR-400-013 |
| 401 | 로그인이 필요합니다. | REFRIGERATOR-401-013 |
| 403 | 접근 권한이 부족합니다. | REFRIGERATOR-403-013 |
| 404 | 존재하지 않는 냉장고입니다. | REFRIGERATOR-404-013 |
| 412 | 버전 정보가 맞지 않습니다. (조회 이후 변경됨) | REFRIGERATOR-412-013 |
| 422 | 부적절한 재고이름입니다. | REFRIGERATOR-422-013 |
| 428 | 수정하려면 현재 버전 정보가 필요합니다. (`If-Match` 누락) | REFRIGERATOR-428-013 |
| 500 | 서버 오류 | REFRIGERATOR-500-013 |

### POST /api/ingredients/name-validation — 재고 이름 검사

화면: STOCK-003 · 닉네임 검사와 같은 방식으로 200 + 본문에 결과를 담는다.

| 결과 | code | message | reason |
|---|---|---|---|
| 사용 가능 | REFRIGERATOR-200-013 | 사용 가능한 재고 이름 | null |
| 금칙어 | REFRIGERATOR-200-013 | 부적절한 재고 이름입니다. | 금칙어 |
| 형식 위반 | REFRIGERATOR-200-013 | 재고 이름은 2~10자, 한글·영문·숫자만 가능합니다. | 닉네임 형식 위반 |

500 서버 오류 (`REFRIGERATOR-500-013`)

### PATCH /api/refrigerators/{refrigerator-id}/ingredients/{ingredient-id} — 재고 만료 처리

화면: STOCK-002-4 · 헤더 `Authorization`, `If-Match: {ETag}` · body `{ "dispose-quantity": 3 }`

- `dispose-quantity`(int)로 폐기 수량을 정할 수 있다.
- 성공 시 **204**

401 (`-401-014`) / 403 (`-403-014`) / 404 냉장고·재료 (`-404-014`) / 412 버전 불일치 (`-412-014`) / 428 버전 정보 필요 (`-428-014`) / 500 (`-500-014`)

### POST /api/v1/refrigerators/{refrigerator-id}/image-analyses — 영수증/실물 촬영 인식 요청

화면: REG-002

```json
{
  "imageObjectKeys": ["foodImage1", "reciptImage1", "foodImage2"],
  "inputHint": "AUTO"
}
```

- 필수: `imageObjectKeys`(String[], 인식할 이미지 키 목록)

202 응답

```json
{
  "code": "REFRIGERATOR-202-016",
  "message": "이미지 인식 시작",
  "data": {
    "analysisId": "ana_1",
    "status": "QUEUED",
    "submittedAt": "2026-09-07T09:00:00+09:00",
    "expiresAt": "2026-09-08T09:00:00+09:00",
    "pollAfterMs": 1000
  }
}
```

| 상태 | title | code |
|---|---|---|
| 400 | 입력 형식이 잘못됐습니다. (키 누락·최대 개수 초과) | REFRIGERATOR-400-016 |
| 401 | 로그인이 필요합니다. | REFRIGERATOR-401-016 |
| 403 | 접근 권한이 부족합니다. | REFRIGERATOR-403-016 |
| 404 | 존재하지 않는 냉장고입니다. | REFRIGERATOR-404-016 |
| 422 | 이미지 제한을 위반했습니다. | REFRIGERATOR-422-016 |
| 429 | 요청 횟수 제한 | REFRIGERATOR-429-016 |
| 500 | 서버 오류 | REFRIGERATOR-500-016 |
| 503 | 분석 접수 일시 불가 | REFRIGERATOR-503-016 |

### GET /api/v1/refrigerators/{refrigerator-id}/image-analyses/{analysisId} — 인식 결과 조회

화면: REG-003

- 작업 status: `QUEUED`, `PROCESSING`, `COMPLETED`, `PARTIALLY_COMPLETED`, `FAILED`
- 항목 status(#14 확정과 연결)
  - `RECOGNIZED` 인식 — 결과를 그대로 등록하거나 수정
  - `UNRECOGNIZED` 미인식 — 다시 촬영하거나 직접 입력
  - `NEEDS_REVIEW` 일부 값 누락 — 표시된 필드를 확인·수정
  - `AI_ESTIMATED` AI 예상 — 추정 근거를 보고 날짜를 입력·확정

200 응답 (COMPLETED · 인식 성공 예시)

```json
{
  "code": "REFRIGERATOR-200-017",
  "message": "이미지 인식 결과 조회 성공",
  "data": {
    "analysisId": "ana_1",
    "status": "COMPLETED",
    "submittedAt": "2026-09-07T09:00:00+09:00",
    "expiresAt": "2026-09-08T09:00:00+09:00",
    "pollAfterMs": null,
    "results": [
      {
        "imageObjectKey": "foodImage1",
        "status": "COMPLETED",
        "recognitionStatus": "RECOGNIZED",
        "items": [
          {
            "itemId": "item_001",
            "displayStatus": "RECOGNIZED",
            "name": { "value": "서울우유 나100%", "confidence": 0.94 },
            "category": { "value": "DAIRY", "confidence": 0.99 },
            "storageType": { "value": "REFRIGERATED", "confidence": 1.0 },
            "measureType": "WEIGHT",
            "quantity": null,
            "weight": { "value": 1000, "unit": "ML", "confidence": 0.96 },
            "expiration": { "date": "2026-09-10", "confidence": 0.93 },
            "reviewReasons": []
          }
        ],
        "error": null
      }
    ],
    "error": null
  }
}
```

- `QUEUED` / `PROCESSING` 단계에서는 `results[].items`가 비고 `recognitionStatus`가 null이며 `pollAfterMs`가 채워진다.
- `UNRECOGNIZED`는 `items: []`.
- `NEEDS_REVIEW` 항목은 누락 필드가 null이고 `reviewReasons`에 안내 문구가 들어간다. 예: `"수량 또는 중량을 입력해 주세요."`, `"소비기한을 직접 확인해 주세요."`
- 이미지별 실패는 `results[].error`에 담는다. 예: `{"code":"MODEL_UNAVAILABLE","message":"...","retryable":true,"retryAfterMs":30000}`, `DEPENDENCY_UNAVAILABLE`
- 전체 실패면 작업 status가 `FAILED`, 일부만 실패면 `PARTIALLY_COMPLETED`.

401 (`-401-017`) / 403 (`-403-017`) / 404 (`-404-017`) / 429 조회 횟수 제한 (`-429-017`) / 500 (`-500-017`) / 503 분석 조회 일시 불가 (`-503-017`)

### POST /api/refrigerators/{refrigerator-id}/ingredients — 재고 일괄 등록

화면: REG-003 · body는 `items` 배열에 단건 등록과 같은 객체를 담는다.

```json
{
  "code": "REFRIGERATOR-201-018",
  "message": "재고 일괄등록 성공",
  "data": { "createdCount": 3, "ingredientsNum": 30, "refrigeratorCapacity": 100 }
}
```

401 (`-401-018`) / 403 (`-403-018`) / 409 냉장고 용량이 가득 참 (`-409-018`) / 500 (`-500-018`)

### POST /api/refrigerators/{refrigerato-id}/ingredient — 재고 단건 등록

화면: REG-003, REG-004

```json
{
  "name": "두부",
  "category": "TOFU_BEAN",
  "storageType": "REFRIGERATED",
  "measureType": "WEIGHT",
  "quantity": null,
  "weightValue": "300.000",
  "weightUnit": "G",
  "expirationDate": "2026-09-09",
  "imageUploadId": null
}
```

- 필수/조건부 필수 규칙은 재고 수정과 같다.
- **200** — 기존 품목에 합산 처리 (`REFRIGERATOR-200-012`, weightValue가 합산된 값으로 응답)
- **201** — 신규 등록 (`REFRIGERATOR-201-012`)

| 상태 | title | code |
|---|---|---|
| 400 | 재료 허용 수량이 아닙니다. | REFRIGERATOR-400-012 |
| 400 | 입력 형식이 잘못됐습니다. | REFRIGERATOR-400-012 |
| 400 | 재료는 개수만 사용하거나 무게만 사용할 수 있습니다. | REFRIGERATOR-400-012 |
| 401 | 로그인이 필요합니다. | REFRIGERATOR-401-012 |
| 403 | 접근 권한이 부족합니다. | REFRIGERATOR-403-012 |
| 404 | 존재하지 않는 냉장고입니다. | REFRIGERATOR-404-012 |
| 409 | 합산 시 재료 허용 수량을 초과합니다. | REFRIGERATOR-409-012 |
| 409 | 냉장고 용량이 가득 차서 등록할 수 없습니다. | REFRIGERATOR-409-012 |
| 422 | 부적절한 재고이름입니다. | REFRIGERATOR-422-012 |
| 500 | 서버 오류 | REFRIGERATOR-500-012 |

---

## 7. 추천 (recommendations)

### GET /api/v1/refrigerators/{refrigeratorId}/recommendations — 추천 목록 조회

동작
- 동일 재고는 추천 재사용, 변경 시 중복 없이 생성
- 생성 중에는 기존 결과 유지
- 냉장고당 하루 생성 10회 제한

응답 규칙
- AI에서 받아온 10개 전체 반환, 결과 없음은 `recipes: []`
- 생성 중이면 `isGenerating: true`, 최초 생성 중이면 `recommendationId: null`
- 재료별 보유·부족·미보유는 백엔드가 계산한다.
- `ingredientCounts`는 필요 재료 **종류 수** 기준: `total`(전체 필요 종류), `owned`(보유 종류, 수량 부족 포함), `missing`(미보유 종류), `expiring`(활용 가능한 임박 종류)

정렬
1. 임박 재료 종류 수 내림차순
2. 부족·미보유 재료 종류 수 오름차순
3. 보유 재료 종류 수 내림차순

백엔드가 정렬한 `recipes` 배열 순서대로 표시한다.

```json
{
  "code": "REFRIGERATOR-200-002",
  "message": "추천 레시피 조회 성공",
  "data": {
    "recommendationId": "rec_001",
    "isGenerating": false,
    "recipes": [
      {
        "recipeId": "51",
        "name": "두부김치",
        "imageUrl": null,
        "cookingTimeMinutes": 15,
        "ingredientCounts": { "total": 4, "owned": 3, "expiring": 2 }
      }
    ]
  }
}
```

400 `refrigeratorId`는 양의 정수 (`COMMON-400-002`) / 401 (`COMMON-401-001`) / 403 (`FRIDGE-403-001`) / 404 (`FRIDGE-404-001`) / 500 (`COMMON-500-001`)

### GET /api/v1/refrigerators/{refrigeratorId}/recipes/{recipeId} — 요리 상세 조회

Path: `refrigeratorId`(Long), `recipeId`(Long, **백엔드 레시피 ID**)

조회
- 레시피 사본과 해당 냉장고의 현재 재고를 반환한다. AI 호출·추천 생성·재고 차감은 없다.
- 적용 시 재고 검증·차감은 백엔드에서 다시 수행한다.

재료 필드
- `requiredAmount`: 기준 인분의 필요량 / `amountText`: 원천 필요량 표현 / `unit`: `EA`·`G`·`ML`, 수치화 불가 시 null
- `stocks`: 매칭된 재고 목록, 미보유는 `[]`
  - `ingredientId`(냉장고 재고 ID), `ownedAmount`, `unit`, `expirationDate`(YYYY-MM-DD)
- 같은 재료라도 만료일별 재고를 분리해 반환한다.
- 매칭·단위 환산은 백엔드, 비교·문구·D-day 표시는 프론트가 한다.
- 환산 가능한 값만 같은 단위로 통일하며, 단위가 다르거나 `requiredAmount`가 null이면 수량 비교가 불가하다.
- 수치화 불가 필요량은 null이고 `amountText`를 유지한다(예: "약간").

표시
- 재료·조리 단계는 응답 배열 순서대로 표시한다.
- 조리 시간·기준 인분·이미지 정보가 없으면 null.
- `expiringIngredientCount`: 활용 가능한 임박 재료 종류 수

> 확인 필요: `cookingTimeMinutes` 저장 컬럼·AI 동기화 보완

```json
{
  "code": "REFRIGERATOR-200-002",
  "message": "레시피 상세 조회 성공",
  "data": {
    "recipeId": "51",
    "name": "두부김치",
    "imageUrl": null,
    "cookingTimeMinutes": 15,
    "baseServings": 2,
    "expiringIngredientCount": 1,
    "ingredients": [
      {
        "recipeIngredientId": "11",
        "name": "두부",
        "requiredAmount": 2,
        "amountText": "2개",
        "unit": "EA",
        "stocks": [
          { "ingredientId": "101", "ownedAmount": 1, "unit": "EA", "expirationDate": "2026-09-09" },
          { "ingredientId": "102", "ownedAmount": 2, "unit": "EA", "expirationDate": "2026-09-15" }
        ]
      },
      {
        "recipeIngredientId": "13",
        "name": "참기름",
        "requiredAmount": null,
        "amountText": "1큰술",
        "unit": null,
        "stocks": []
      }
    ],
    "steps": [
      { "stepNo": 1, "instruction": "두부를 도톰하게 썰어 물기를 뺀다." },
      { "stepNo": 2, "instruction": "팬에 기름을 두르고 두부를 노릇하게 굽는다." }
    ]
  }
}
```

400 (`COMMON-400-002`) / 401 (`COMMON-401-001`) / 403 (`FRIDGE-403-001`) / 404 냉장고 (`FRIDGE-404-001`) / 404 레시피 (`RECOMMEND-404-001`) / 500 (`COMMON-500-001`)

### POST /api/v1/refrigerators/{refrigeratorId}/cooking-records — 요리 적용

헤더: `Authorization`, `Content-Type: application/json`, **`Idempotency-Key: {requestKey}`**
body: `{ "recommendationId": "rec_001", "recipeId": "51" }`

차감
- 백엔드가 현재 재고·재료 매칭·필요량을 재검증한다.
- 기준 인분만큼 적용하며 별도 인분 변경은 없다.
- 충분하면 필요량만, 부족하면 보유량만 차감하고 미보유 재료는 건너뛴다.
- 필요량 수치화·단위 환산이 불가한 재료는 차감에서 제외한다.
- 차감할 재료가 없어도 요리 적용은 성공한다. 재료 부족·미보유 자체는 오류로 반환하지 않는다.
- 동시 요청에도 음수 재고·중복 차감을 방지한다.

기록
- 차감 · 조리 기록 생성 · 본인 누적 요리 수 +1은 하나의 트랜잭션이다.
- 추천은 요청 냉장고 소속이어야 하고, 레시피는 해당 추천에 포함되어야 한다.
- 적용 직후 AI 호출은 없다.

멱등성
- `Idempotency-Key` 필수, 1~64자
- 같은 조리의 재시도는 같은 키, 새로운 조리는 새 키를 쓴다.
- 성공한 동일 요청의 재시도도 추가 처리 없이 204를 반환한다.
- 같은 키로 사용자·냉장고·요청 내용이 다르면 409.
- 동시 동일 요청도 조리 기록·차감·횟수 증가는 1회.
- 성공 재시도는 현재 추천 캐시 만료와 무관하게 204를 반환한다.

응답: **204 No Content**, 본문 없음. 조리 기록·누적 요리 횟수는 서버 내부에 반영한다.

> 확인 필요: 동일 재료 여러 재고의 차감 순서 / 전량 소진 재고 처리(현재 DB는 수량 0 저장 불가) / 멱등 처리 결과·요청 비교 정보 보존 방식과 재시도 보장 기간

| 상태 | detail | code |
|---|---|---|
| 400 | 요청 값의 형식 또는 필수 항목을 확인해 주세요. | RECOMMEND-400-002 |
| 401 | 로그인이 필요합니다. | COMMON-401-001 |
| 403 | 해당 냉장고에 접근할 권한이 없습니다. | RECOMMEND-403-001 |
| 404 | 존재하지 않는 냉장고입니다. | RECOMMEND-404-001 |
| 404 | 존재하지 않거나 해당 추천에 포함되지 않은 레시피입니다. | RECOMMEND-404-001 |
| 409 | 추천 결과를 확인할 수 없습니다. 추천 목록을 다시 조회해 주세요. | RECOMMEND-409-001 |
| 409 | 같은 멱등키로 다른 요청을 전송할 수 없습니다. | RECOMMEND-409-001 |
| 500 | 요청을 처리하는 중 오류가 발생했습니다. | COMMON-500-001 |

### POST /ai/v1/recipe-recommendations — 추천 생성·재고 매칭 (BE → AI)

신규 명세 · 제품 v3 · 헤더: `Content-Type: application/json`, 내부 서비스 인증(방식 미정)

```json
{
  "requestId": "req_recipe_001",
  "refrigeratorId": 17,
  "inventoryFingerprint": "7eb9556c...c67",
  "recommendationMode": "INCLUDE_EXPIRED",
  "inventory": [
    {
      "ingredientId": 101,
      "name": "두부",
      "category": "TOFU_BEAN",
      "storageType": "REFRIGERATED",
      "measureType": "COUNT",
      "quantity": 2,
      "weightValue": null,
      "weightUnit": "NONE",
      "expirationDate": "2026-09-09"
    }
  ],
  "excludedIngredientNames": [],
  "limit": 10
}
```

계약 (AI팀 합의 필요)
- 기존 경로·요청 필드는 유지하고 `limit=10` 고정. 레시피 ID·재고 매칭 응답은 변경 제안 상태다.

매핑
- `recipeIngredientId`: AI의 레시피 재료 항목 ID
- `ingredientIds`: 요청 `inventory`의 재고 ID 목록. 매칭 없음은 `[]`, 여러 재고는 배열.
- AI ID와 백엔드 PK는 별도로 매핑한다.

검증
- 백엔드가 재고 소속·이름·단위를 재검증하고, 보유·부족·미보유를 계산한다.
- 매칭만으로 재고를 차감하지 않는다.

응답
- 추천순으로 최대 10개, 결과 없음은 `[]`. 상세는 동기화된 레시피 DB에서 조회한다.

```json
{
  "code": "AI-200-XXX",
  "message": "레시피 추천 생성 성공",
  "data": {
    "recommendationId": "rec_001",
    "refrigeratorId": 17,
    "inventoryFingerprint": "7eb9556c...c67",
    "recommendationMode": "INCLUDE_EXPIRED",
    "recommendations": [
      {
        "recipeId": 1024,
        "ingredients": [
          { "recipeIngredientId": 11, "ingredientIds": [101] },
          { "recipeIngredientId": 12, "ingredientIds": [] }
        ]
      }
    ],
    "generatedAt": "2026-09-08T14:00:00+09:00"
  }
}
```

400 `INVALID_REQUEST` / 401 `UNAUTHORIZED_SERVICE` / 429 `RATE_LIMITED` / 503 `AI_GATEWAY_UNAVAILABLE` / 500 `AI-500-001`

---

## 8. 알림 (notifications)

### GET /api/v1/notifications?type=ALL&cursor={cursor} — 알림 목록 조회

화면: NOTI-001

- `type` 기본값 `ALL`. 첫 요청은 `cursor` 생략. 필터·냉장고 변경 시 커서를 초기화한다.
- `expired_ingredients_num`은 첫 페이지에만 값이 있고 이후는 null.
- 마지막 페이지: `next_cursor: null`, `has_next: false`
- 미읽음 수는 전역 폴링 조회 값을 사용한다.

```json
{
  "code": "NOTI-200-001",
  "message": "알림 목록 조회 성공",
  "data": {
    "expired_ingredients_num": 3,
    "notifications": [
      {
        "notification_id": "1051",
        "type": "MEMBER",
        "title": "새 참여자가 들어왔어요",
        "body": "민주님이 참여했어요",
        "read_at": null,
        "created_at": "2026-08-27T14:22:10+09:00"
      },
      {
        "notification_id": "1042",
        "type": "EXPIRING_SOON",
        "title": "오늘 만료되는 재료가 있어요",
        "body": "달걀 10개 외 1개 · 오늘까지예요",
        "read_at": null,
        "created_at": "2026-08-27T08:00:00+09:00"
      }
    ],
    "next_cursor": "eyJjcmVhdGVkX2F0Ijoi...",
    "has_next": true
  }
}
```

400 `type`은 `ALL`·`EXPIRING_SOON`·`EXPIRED`·`MEMBER` 중 하나 (`NOTI-400-001`) / 400 유효하지 않은 커서 (`COMMON-400-001`) / 401 (`COMMON-401-001`) / 500 (`COMMON-500-001`)

### PATCH /api/v1/notifications/{notification-id} — 알림 읽기

- 이미 읽은 알림도 변경 없이 **204**
- 읽음 처리 성공 후 프론트가 미읽음 수 조회 API를 재호출한다.

401 (`COMMON-401-001`) / 404 존재하지 않는 알림 (`NOTI-404-001`) / 500 (`COMMON-500-001`)

### PATCH /api/v1/notifications — 알림 모두 읽기

- 활성 냉장고의 안 읽은 알림 전체를 필터와 무관하게 읽음 처리한다.
- 안 읽은 알림이 0건이어도 **204**
- 읽음 처리 성공 후 프론트가 미읽음 수 조회 API를 재호출한다.

401 (`COMMON-401-001`) / 500 (`COMMON-500-001`)

### GET /api/v1/notifications/unread-count — 미읽음 수 조회

헤더: `Authorization`, `Accept: application/json`

응답 헤더
- 200: `Content-Type: application/json`, `Cache-Control: no-store`
- 오류: `Content-Type: application/problem+json`
- 401: `WWW-Authenticate: Bearer`

집계
- 본인·활성 냉장고 기준 미읽음 수, 필터 무관
- 요청마다 현재 집계값 반환, 0건이면 `unreadCount: 0`
- Web Push 구독·발송은 별도로 유지

폴링
- 앱 전역에서 폴링 하나를 공유하고 화면 이동마다 호출하지 않는다.
- 로그인 후 최초 · 앱 전경 복귀 · 냉장고 변경 시 즉시 조회
- 앱 전경에서 주기적으로 조회하고 백그라운드·로그아웃 시 중단
- 개별·전체 읽음 성공 후 즉시 재조회
- 진행 중 요청과 중복 호출하지 않는다.

프론트
- `unreadCount`로 덮어쓰며 직접 +1/-1 하지 않는다.
- 최초 로딩·조회 실패는 0건과 구분한다.
- 냉장고·계정 변경 전의 늦은 응답은 무시한다.
- 로그아웃 시 진행 중 요청을 취소하고 상태를 초기화한다.

인증·오류
- 매 요청 인증·활성 냉장고 접근 권한을 검증한다.
- 401이면 토큰 갱신 후 재시도하고 실패하면 폴링을 중단한다.
- 409면 폴링을 중단하고 활성 냉장고 선택 후 재개한다.
- 네트워크·500 오류 시 연속 즉시 재시도를 금지한다.

> 확인 필요: 폴링 주기·실패 재시도 간격

```json
{
  "code": "NOTI-200-001",
  "message": "미읽음 알림 개수 조회 성공",
  "data": { "refrigeratorId": "17", "unreadCount": 3 }
}
```

401 (`COMMON-401-001`) / 409 활성 냉장고를 선택한 뒤 다시 요청 (`FRIDGE-409-001`) / 500 (`COMMON-500-001`)

### POST /api/v1/push-subscriptions — 푸시 구독 등록·갱신

```json
{
  "endpoint": "https://push.example.com/subscriptions/example",
  "keys": { "p256dh": "{BASE64URL_PUBLIC_KEY}", "auth": "{BASE64URL_AUTH_SECRET}" }
}
```

- `endpoint`, `keys.p256dh`, `keys.auth` 필수

계약
- 신규는 201, 동일 endpoint 갱신은 200. 동일 구독은 같은 `subscriptionId`를 반환한다.
- 동일 요청은 중복 생성·버전 증가가 없다. 키·활성 상태가 바뀌면 구독 버전이 증가한다.
- 다른 계정의 구독은 활성 여부와 무관하게 재사용·소유자 변경 금지 (409).

계정 변경
- 기존 계정 로그아웃 시 서버 구독을 비활성화하고 브라우저 구독을 해제한다.
- 새 계정 로그인 후 새 브라우저 구독으로 등록한다.
- 409 수신 시 프론트가 기존 브라우저 구독을 해제하고 새 구독으로 재등록한다.

ERD
- `user_devices` 등록·갱신, 상태 `ACTIVE`
- `subscriptionId` = `user_device_id`(문자열)
- `auth`는 암호화 저장하며 응답·로그에서 제외한다.

201 응답 (`Location: /api/v1/push-subscriptions/501`)

```json
{ "code": "PUSH-201-001", "message": "푸시 구독 생성 성공", "data": { "subscriptionId": "501" } }
```

200 응답: `PUSH-200-001`, "푸시 구독 등록 성공"

400 (`PUSH-400-001`) / 401 (`COMMON-401-001`) / 409 다른 계정에 연결된 구독 재사용 불가 (`PUSH-409-001`) / 500 (`COMMON-500-001`)

### DELETE /api/v1/push-subscriptions/{subscriptionId} — 푸시 구독 해제

- 본인 구독만 해제하며 `DISABLED`로 변경하고 구독 버전을 증가시킨다.
- 이미 `DISABLED`면 변경 없이 **204**. 응답 본문 없음.
- 알림함 데이터는 유지한다.
- 발송 전 상태·버전을 재검증해 차단하며, 이미 전송된 푸시는 회수할 수 없다.
- 브라우저 구독 해제는 프론트에서 별도로 수행한다.
- 로그아웃 시 인증 종료 전에 현재 구독을 비활성화한다.

400 `subscriptionId` 형식 (`PUSH-400-002`) / 401 (`COMMON-401-001`) / 403 본인 구독만 해제 가능 (`PUSH-403-001`) / 404 (`PUSH-404-001`) / 500 (`COMMON-500-001`)

### GET /api/v1/notifications/settings — 알림 설정 조회

- 유형: `EXPIRATION`(임박·만료 알림), `RECIPE`(추천 알림)
- 본인의 유형별 수신 설정을 반환하며 가입 시 기본값은 true.

```json
{
  "code": "NOTI-200-001",
  "message": "알림 설정 조회 성공",
  "data": {
    "notificationPreferences": [
      { "type": "EXPIRATION", "isEnabled": true },
      { "type": "RECIPE", "isEnabled": true }
    ]
  }
}
```

401 (`COMMON-401-001`) / 500 (`COMMON-500-001`)

### PATCH /api/v1/notifications/setting — 알림 설정 변경

```json
{ "notificationPreferences": [ { "type": "RECIPE", "isEnabled": false } ] }
```

- `type`: `EXPIRATION` / `RECIPE`, `isEnabled`: Boolean(null 불가)
- 전달한 유형만 수정하고 생략한 유형은 유지한다. 같은 값으로 재요청해도 동일 상태를 유지한다.
- 200으로 저장된 전체 설정을 반환한다 (`NOTI-200-002`).

400 (`COMMON-400-002`) / 401 (`COMMON-401-001`) / 500 (`COMMON-500-001`)

---

## 9. 엔드포인트 한눈에 보기

| 도메인 | Method | URL | 기능 |
|---|---|---|---|
| 공통 | POST | `/api/image/presigned-url` | 사진 업로드 URL 요청 |
| 회원 | POST | `/api/users` | 로컬 회원가입 |
| 회원 | POST | `/api/users/oauth` | 소셜 회원가입 완료 |
| 회원 | POST | `/api/users/nickname-validations` | 닉네임 검사 |
| 회원 | GET | `/api/users/me` | 회원정보 조회 |
| 회원 | PATCH | `/api/users/me` | 회원정보 수정 |
| 회원 | DELETE | `/api/users/me` | 회원탈퇴 |
| 인증 | POST | `/api/auth/sessions` | 로그인 |
| 인증 | DELETE | `/api/auth/sessions` | 로그아웃 |
| 인증 | GET | `/api/auth/oauth/{provider}` | OAuth 인증 시작 |
| 인증 | GET | `/login/oauth2/code/{provider}` | OAuth 로그인 콜백 |
| 인증 | POST | `/api/auth/token` | 소셜 로그인 토큰 발급 |
| 인증 | POST | `/api/auth/oauth/registration-tokens` | 소셜 회원가입 정보 교환 |
| 인증 | POST | `/api/auth/token-renewals` | 인증정보 갱신 |
| 냉장고 | POST | `/api/refrigerators/{id}/invitations` | 초대코드 발급 |
| 냉장고 | POST | `/api/refrigerators/members` | 공유 참여 |
| 냉장고 | GET | `/api/refrigerators/{id}/members` | 참여자 목록 조회 |
| 냉장고 | GET | `/api/refrigerators/{id}/members/stats` | 참여자 인원수 조회 |
| 냉장고 | DELETE | `/api/refrigerators/{id}/members/me` | 공유 나가기 |
| 재고 | GET | `/api/refrigerators/{id}/expired` | 이번 달 만료 재고 수 |
| 재고 | GET | `/api/refrigerators/{id}/ingredients` | 재고 목록 조회 |
| 재고 | POST | `/api/refrigerators/{id}/ingredients` | 재고 일괄 등록 |
| 재고 | POST | `/api/refrigerators/{id}/ingredient` | 재고 단건 등록 |
| 재고 | DELETE | `/api/refrigerators/{id}/ingredients/expired` | 만료 재고 일괄 처리 |
| 재고 | GET | `/api/ingredients/{id}` | 재고 상세 조회 |
| 재고 | PATCH | `/api/refrigerators/{id}/ingredients/{id}` | 재고 수정 / 만료 처리 |
| 재고 | POST | `/api/ingredients/name-validation` | 재고 이름 검사 |
| 재고 | POST | `/api/v1/refrigerators/{id}/image-analyses` | 이미지 인식 요청 |
| 재고 | GET | `/api/v1/refrigerators/{id}/image-analyses/{analysisId}` | 인식 결과 조회 |
| 추천 | GET | `/api/v1/refrigerators/{id}/recommendations` | 추천 목록 조회 |
| 추천 | GET | `/api/v1/refrigerators/{id}/recipes/{recipeId}` | 요리 상세 조회 |
| 추천 | POST | `/api/v1/refrigerators/{id}/cooking-records` | 요리 적용 |
| 추천 | POST | `/ai/v1/recipe-recommendations` | 추천 생성·재고 매칭 (BE→AI) |
| 알림 | GET | `/api/v1/notifications` | 알림 목록 조회 |
| 알림 | PATCH | `/api/v1/notifications/{id}` | 알림 읽기 |
| 알림 | PATCH | `/api/v1/notifications` | 알림 모두 읽기 |
| 알림 | GET | `/api/v1/notifications/unread-count` | 미읽음 수 조회 |
| 알림 | GET | `/api/v1/notifications/settings` | 알림 설정 조회 |
| 알림 | PATCH | `/api/v1/notifications/setting` | 알림 설정 변경 |
| 알림 | POST | `/api/v1/push-subscriptions` | 푸시 구독 등록·갱신 |
| 알림 | DELETE | `/api/v1/push-subscriptions/{id}` | 푸시 구독 해제 |

---

## 10. 열거값 모음

| 구분 | 값 |
|---|---|
| 카테고리 | `VEGETABLES`, `FRUITS`, `MEAT`, `SEAFOOD`, `DAIRY`, `TOFU_BEANS`, `GRAINS_NOODLES`, `PROCESSED_FOODS`, `SEASONINGS`, `BEVERAGES`, `OTHER` |
| 보관 방식 | `REFRIGERATED`, `FROZEN` |
| 측정 방식 | `WEIGHT`, `COUNT` |
| 무게 단위 | `G`, `ML`, `NONE` |
| 재고 상태 | `EXPIRING`, `EXPIRED` (+정상) |
| 목록 정렬 | `EXPIRATION_ASC`, `CREATED_DESC`, `NAME_ASC` |
| 목록 필터 | `TOTAL`, `EXPIRING`, `EXPIRED`, `REFRIGERATED`, `FROZEN` |
| 이미지 용도 | `ANALYSIS`, `PROFILE` |
| 분석 작업 상태 | `QUEUED`, `PROCESSING`, `COMPLETED`, `PARTIALLY_COMPLETED`, `FAILED` |
| 분석 항목 상태 | `RECOGNIZED`, `UNRECOGNIZED`, `NEEDS_REVIEW`, `AI_ESTIMATED` |
| 알림 유형 | `ALL`, `EXPIRING_SOON`, `EXPIRED`, `MEMBER` |
| 알림 설정 유형 | `EXPIRATION`, `RECIPE` |
| 추천 모드 | `INCLUDE_EXPIRED` |
| 레시피 단위 | `EA`, `G`, `ML`, null |
| 멤버 역할 | `owner`, `member` |

---

## 11. 도메인별 에러 코드 접두어

`IMAGE` · `USER` · `AUTH` · `REFRIGERATOR` · `RECOMMEND` · `NOTI` · `PUSH` · `FRIDGE` · `COMMON`, AI 연동은 `AI-*` 및 문자열 코드(`INVALID_REQUEST`, `UNAUTHORIZED_SERVICE`, `RATE_LIMITED`, `AI_GATEWAY_UNAVAILABLE`).

---

## 12. 명세서 검토 메모

시트를 그대로 옮기면서 발견한, 기준 문서·확정 이슈와 어긋나거나 시트 내부에서 충돌하는 지점이다. 아직 확정 사항이 아니라 확인이 필요한 목록이다. (2026-09-09 작성 당시 기준. 1·2번은 이후 PROJECT_CONTEXT 갱신으로 해소됐을 수 있으므로 [consistency-checker](../agents/consistency-checker.md)로 다시 대조한다.)

### 기준 문서와의 충돌

1. **로컬 회원가입·로그인 API가 살아 있다.** `POST /api/users`, `POST /api/auth/sessions`(loginId·password)는 `PROJECT_CONTEXT.md` 3장의 "v2·v3는 카카오·구글 OAuth만 제공"과 어긋난다. v1 잔존 명세인지, v3에서도 유지할지 확인이 필요하다.
2. **백엔드 레시피 DB를 전제한다.** 추천 상세가 "레시피 사본", "동기화된 레시피 DB에서 조회", "백엔드 레시피 ID"를 쓰는데, `PROJECT_CONTEXT.md` 7장·13장은 레시피 원본을 관리하지 않고 추천 결과 JSON만 인메모리 캐시에 둔다고 되어 있다. PL 피드백 재설계안(`erd/pl-redesign-2026-09-05.md`)의 백엔드 레시피 사본 방향을 API가 먼저 반영한 상태로 보인다.
3. **알림 보관 정책이 다르다.** `PROJECT_CONTEXT.md` 9장은 "삭제하지 않고 조회 시 LIMIT 99"인데, 이슈 #33 최종 결론은 "DB에 사용자별 99개만 보관하고 100번째 생성 시 읽은 알림 → 오래된 순으로 즉시 삭제"다. 기준 문서 갱신이 필요하다.
4. **비활성 냉장고 알림 생성 여부.** 이슈 #33 최종 결론은 "비활성 개인 냉장고는 알림 생성·푸시 중단, 복귀 시 요약 영역에 만료·임박 개수 표시"인데 기준 문서에는 이 결론이 반영돼 있지 않다.
5. **알림 문구 규칙 미반영.** 이슈 #33에서 확정한 `n일 후/오늘 … 만료됩니다 + 실제 유통기한` 형식이 시트의 `title`·`body` 예시("오늘 만료되는 재료가 있어요")와 다르다.

### 시트 내부 불일치

6. **경로 prefix가 섞여 있다.** 회원·인증·냉장고·재고 일부는 `/api/...`, 추천·알림·이미지 분석은 `/api/v1/...`을 쓴다. 반면 refreshToken 쿠키의 `Path`는 `/api/v1/auth`다.
7. **재고 수정과 만료 처리가 같은 메서드·경로다.** 둘 다 `PATCH /api/refrigerators/{refrigerator-id}/ingredients/{ingredient-id}`이므로 구분자가 필요하다. 만료 처리 설명은 `dispose-quantity`를 Query라고 적었는데 예시는 body로 되어 있다.
8. **토큰 전달 방식이 다르다.** 로컬 회원가입·로그인은 `accessToken`을 body로, 소셜 회원가입 완료는 `Set-Cookie: accessToken`으로 내려준다.
9. **상태 코드와 code 접두어가 어긋난 행이 있다.** 로그아웃 401 응답의 코드가 `AUTH-200-005`, 재고 목록 404 응답의 코드가 `REFRIGERATOR-403-008`이다.
10. **응답 네이밍이 섞여 있다.** 알림 목록만 snake_case(`notification_id`, `read_at`, `next_cursor`)이고 나머지는 camelCase다.
11. **알림 설정 경로 불일치.** 조회는 `/settings`, 변경은 `/setting`이다.
12. **카테고리 상수 불일치.** 목록 조회 필터는 `TOFU_BEANS`인데 재고 응답 예시는 모두 `TOFU_BEAN`이다.
13. **`ingredientCounts` 필드 누락.** 설명에는 `total`·`owned`·`missing`·`expiring` 4개인데 응답 예시에는 `missing`이 없다.
14. **재고 목록 응답의 `ingredients`가 배열이 아니라 객체로 적혀 있다.** 커서 `nextCursor`도 `ingredients` 밖이 아니라 `data` 안에 나란히 있어 구조 확정이 필요하다.
15. **재고 이름 검사 결과의 `reason`이 "닉네임 형식 위반"으로 되어 있다.** 재고 문구로 교체가 필요하다.
16. **오탈자.** 컬럼명 `응답 Response ststus code`, 로그인 응답의 `mesage`, 설명의 `logind`·`Stirng`·`refrigerato-id`·`reciptImage1`.
17. **`inputHint`가 요청 예시에만 있고 설명·허용값이 없다.**
