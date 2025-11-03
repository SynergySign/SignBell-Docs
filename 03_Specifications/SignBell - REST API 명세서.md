# Signbell REST API 명세서

본 문서는 SignBell 서비스의 RESTful API를 사용하는 개발자를 위한 공식 기술 명세서입니다.

* **작성자**: [신동준](https://github.com/sdj3959)
* **작성일**: 2025-10-08
* **최종 수정일**: 2025-10-29
* **문서 버전**: v2.1.0

**대상 독자:**

* **백엔드 개발자**: RESTful API 엔드포인트를 구현·유지보수하고, 인증/인가 로직, 데이터 검증, 에러 처리를 담당하는 개발자
* **프론트엔드 개발자**: API를 호출하여 사용자 인터페이스와 데이터를 연동하고, 요청/응답 처리 로직을 구현하는 개발자
* **QA / 테스트 엔지니어**: API 엔드포인트별 테스트 케이스를 작성하고, 기능 검증 및 통합 테스트를 수행하는 담당자
* **DevOps / 인프라 엔지니어**: API 서버 배포 환경 구축, 모니터링, 로깅, API Gateway 설정 등을 관리하는 담당자
* **기획자 / PM**: API 기능 범위와 데이터 구조를 이해하고, 프론트엔드-백엔드 간 협업을 조율하는 담당자
* **신규 합류자**: SignBell 프로젝트의 API 구조와 데이터 흐름을 빠르게 파악해야 하는 팀 신규 인원

---

## 1. 공통 가이드

### 1.1. 기본 정보

- **Base URL**: `https://localhost:9000/api` (개발 환경 기준)

### 1.2. 인증

- 인증이 필요한 모든 API는 쿠키에 저장된 JWT 기반 토큰을 사용합니다.

- **쿠키 설정**:
    - `HttpOnly`: `true` (JavaScript 접근 차단으로 XSS 공격 방지)
    - `Secure`: 개발 환경에서는 `false`, 프로덕션에서는 `true` (HTTPS 연결에서만 전송)
    - `SameSite`: `None` (크로스 도메인 허용)
    - `Domain`: `localhost` (개발 환경 기준)
- **액세스 토큰**:
    - 쿠키명: `ACCESS_TOKEN`
    - 유효 시간: 15분 (900초)
    - 사용자 인증 상태를 확인하기 위한 단기 유효 토큰
- **리프레시 토큰**:
    - 쿠키명: `REFRESH_TOKEN`
    - 유효 시간: 7일 (604,800초)
    - 액세스 토큰을 재발급하기 위한 장기 유효 토큰

- 액세스 토큰 만료 시 리프레시 토큰을 통해 자동으로 갱신됩니다.
- 클라이언트는 쿠키를 직접 관리할 필요 없이, 브라우저가 자동으로 쿠키를 포함하여 요청을 전송합니다.

### 1.3. 공통 응답 포맷

- 모든 API 응답은 아래의 `ApiResponse` DTO 형식으로 통일됩니다.
```json
  {
    "success": true,
    "message": "요청 처리 성공 메시지",
    "timestamp": "2025-10-21T12:00:00.000Z",
    "data": { ... } // API 별 실제 데이터
  }
```

### 1.4. 공통 에러 응답

- API 요청 실패 시, `success`는 `false`가 되며 에러 정보는 `ErrorResponse` 형식으로 반환됩니다.
```json
  {
    "timestamp": "2025-10-21T12:00:00.000Z",
    "status": 404,
    "error": "USER_NOT_FOUND",
    "detail": "사용자를 찾을 수 없습니다.",
    "path": "/api/v1/users/123"
  }
```

#### 주요 에러 코드

| HTTP 상태 | 에러 코드 (ErrorCode) | 설명 |
|---|---|---|
| 400 Bad Request | `BAD_REQUEST` | 잘못된 요청 |
| 400 Bad Request | `INVALID_INPUT` | 유효하지 않은 입력값 |
| 400 Bad Request | `VALIDATION_ERROR` | 유효성 검사 실패 |
| 401 Unauthorized | `UNAUTHORIZED` | 인증되지 않은 사용자 |
| 401 Unauthorized | `INVALID_TOKEN` | 유효하지 않은 토큰 |
| 401 Unauthorized | `EXPIRED_TOKEN` | 만료된 토큰 |
| 403 Forbidden | `FORBIDDEN` | 접근 권한 없음 |
| 403 Forbidden | `TERMS_NOT_AGREED` | 약관 동의 필요 |
| 403 Forbidden | `NOT_ROOM_HOST` | 방장 권한 없음 |
| 404 Not Found | `USER_NOT_FOUND` | 사용자를 찾을 수 없음 |
| 404 Not Found | `ROOM_NOT_FOUND` | 퀴즈 방을 찾을 수 없음 |
| 404 Not Found | `WORD_NOT_FOUND` | 단어를 찾을 수 없음 |
| 409 Conflict | `ALREADY_REGISTERED_USER` | 이미 가입된 사용자 |
| 409 Conflict | `PARTICIPANT_ALREADY_IN_ROOM` | 이미 방에 참여 중 |
| 500 Internal Server Error | `INTERNAL_SERVER_ERROR` | 서버 내부 오류 |
| 500 Internal Server Error | `AI_MODEL_ERROR` | AI 모델 처리 오류 |
| 500 Internal Server Error | `WEBRTC_CONNECTION_FAILED` | WebRTC 연결 실패 |

#### 유효성 검증 에러 응답

- `@Valid`, `@Validated` 어노테이션으로 검증 실패 시 다음과 같은 형식으로 반환됩니다.
```json
  {
    "timestamp": "2025-10-21T12:00:00.000Z",
    "status": 400,
    "error": "VALIDATION_ERROR",
    "validationErrors": [
      {
        "field": "nickname",
        "message": "닉네임은 2~10자 사이여야 합니다.",
        "rejectedValue": "a"
      }
    ]
  }
```

---

## 2. 사용자 정보 조회 (User Information)

### 2.1. 내 정보 조회

- **Endpoint**: `GET /api/users/me`
- **설명**: 현재 로그인한 사용자의 전체 정보를 조회합니다. 프로필 정보뿐만 아니라 인증 관련 정보와 누적 점수를 포함합니다.
- **인증**: **필수**
- **요청**:
  - 별도의 파라미터 없음 (인증 토큰에서 사용자 식별)
- **응답 (200 OK)**:
  - **Body**: `ApiResponse<UserInfoResponse>`
  - **JSON 응답 예시**:
    ```json
    {
      "success": true,
      "message": "사용자의 정보가 성공적으로 조회되었습니다.",
      "timestamp": "2025-10-29T12:00:00.123Z",
      "data": {
        "userId": 1,
        "nickname": "수어마스터",
        "profileImageUrl": "http://k.kakaocdn.net/dn/jnklU/btsPFZKQSrJ/aHNecgxEqPLdnpIVw9ZR0K/img_640x640.jpg",
        "providerId": "123456789",
        "provider": "KAKAO",
        "email": "user@example.com",
        "requiredAgree": true,
        "optionalAgree": true,
        "totalScore": 1500
      }
    }
    ```
  - **상세 스펙 (`data`)**:

    | 필드 | 타입 | 설명 |
     |---|---|---|         
    | `userId` | `Long` | 사용자 고유 ID |
    | `nickname` | `String` | 닉네임 |
    | `profileImageUrl` | `String` | 프로필 이미지 URL |
    | `providerId` | `String` | OAuth 제공자의 사용자 ID |
    | `provider` | `String` | 로그인 제공자 (KAKAO, GOOGLE 등) |
    | `email` | `String` | 이메일 주소 |
    | `requiredAgree` | `Boolean` | 필수 약관 동의 여부 |
    | `optionalAgree` | `Boolean` | 선택 약관 동의 여부 (학습 데이터 수집 동의) |
    | `totalScore` | `Long` | 누적 점수 |
- **오류**:
  - `400 Bad Request`: 잘못된 사용자 ID 형식
  - `401 Unauthorized`: `UNAUTHORIZED` (인증 정보 없음 또는 유효하지 않음)
  - `404 Not Found`: `USER_NOT_FOUND` (해당 사용자 없음)
  - `500 Internal Server Error`: 서버 내부 오류

---

## 3. 사용자 랭킹 (User Ranking)

### 3.1. 랭킹 조회

- **Endpoint**: `GET /api/users/rank`
- **설명**: 누적 점수(totalScore) 기준 상위 8명의 사용자 랭킹을 조회합니다.
- **인증**: **필수**
- **요청**:
  - 별도의 파라미터 없음
- **응답 (200 OK)**:
  - **Body**: `ApiResponse<List<UserRankResponse>>`
  - **JSON 응답 예시**:
    ```json
    {
      "success": true,
      "message": "랭킹 조회에 성공했습니다.",
      "timestamp": "2025-10-29T12:01:00.456Z",
      "data": [
        {
          "rank": 1,
          "nickname": "랭킹1위",
          "score": 5000,
          "profileImage": "http://example.com/profile1.jpg"
        },
        {
          "rank": 2,
          "nickname": "랭킹2위",
          "score": 4500,
          "profileImage": "http://example.com/profile2.jpg"
        }
      ]
    }
    ```
  - **상세 스펙 (`data` 배열의 각 요소)**:

    | 필드 | 타입 | 설명 |
      |---|---|---|        
    | `rank` | `Integer` | 사용자 순위 (1~8) |
    | `nickname` | `String` | 닉네임 |
    | `score` | `Long` | 누적 점수 |
    | `profileImage` | `String` | 프로필 이미지 URL (nullable) |
- **오류**:
  - `500 Internal Server Error`: `DATABASE_ERROR` (데이터베이스 조회 실패)

---

## 4. 마이페이지 - 프로필 관리 (My Page - Profile Management)

### 4.1. 내 프로필 조회

- **Endpoint**: `GET /api/my-page/users/{userId}/profile`
- **설명**: 현재 로그인한 사용자의 상세 프로필 정보를 조회합니다. `userId`는 현재 인증된 사용자의 ID여야 합니다.
- **인증**: **필수**
- **요청**:
    - **Path Parameter**:
      | 이름 | 타입 | 필수 | 설명 |
      |---|---|---|---|
      | `userId` | `Long` | 필수 | 조회할 사용자 고유 ID (현재 로그인된 사용자) |
- **응답 (200 OK)**:
    - **Body**: `ApiResponse<UserProfileResponse>`
    - **JSON 응답 예시**:
    ```json
      {
        "success": true,
        "message": "프로필 조회 성공",
        "timestamp": "2025-10-21T12:01:00.123Z",
        "data": {
          "nickname": "수어마스터",
          "profileImageUrl": "http://k.kakaocdn.net/dn/jnklU/btsPFZKQSrJ/aHNecgxEqPLdnpIVw9ZR0K/img_640x640.jpg",
          "optionalAgree": true
        }
      }
    ```
    - **상세 스펙 (`data`)**:
  
      | 필드 | 타입 | 설명 |
      |---|---|---|
      | `nickname` | `String` | 닉네임 |
      | `profileImageUrl` | `String` | 프로필 이미지 URL (카카오 프로필 이미지) |
      | `optionalAgree` | `Boolean` | 선택 약관 동의 여부 (학습 데이터 수집 동의) |
- **오류**:
    - `401 Unauthorized`: `UNAUTHORIZED` (인증 정보 없음 또는 유효하지 않음)
    - `403 Forbidden`: `FORBIDDEN` (요청한 userId와 로그인 사용자 불일치)
    - `404 Not Found`: `USER_NOT_FOUND` (해당 ID의 사용자 없음)

### 4.2. 내 프로필 수정

- **Endpoint**: `PATCH /api/my-page/users/{userId}/profile`
- **설명**: 현재 로그인한 사용자의 프로필 정보를 수정합니다. `userId`는 현재 인증된 사용자의 ID여야 합니다.
- **인증**: **필수**
- **요청**:
    - **Path Parameter**:
      | 이름 | 타입 | 필수 | 설명 |
      |---|---|---|---|
      | `userId` | `Long` | 필수 | 수정할 사용자 고유 ID (현재 로그인된 사용자) |
    - **Body (`application/json`)**:
    ```json
      {
        "nickname": "변경된닉네임",
        "optionalAgree": true
      }
    ```
    - **상세 스펙 (Request Body)**:
  
      | 필드 | 타입 | 필수 | 제약사항 | 설명 |
      |---|---|---|---|---|
      | `nickname` | `String` | 필수 | 1~20자 | 변경할 닉네임 |
      | `optionalAgree` | `Boolean` | 선택 | - | 선택 약관 동의 여부 (학습 데이터 수집 동의) |
- **응답 (200 OK)**:
    - **Body**: `ApiResponse<UserProfileResponse>`
    - **JSON 응답 예시**:
    ```json
      {
        "success": true,
        "message": "프로필 수정 성공",
        "timestamp": "2025-10-21T12:02:00.456Z",
        "data": {
          "nickname": "변경된닉네임",
          "profileImageUrl": "http://k.kakaocdn.net/dn/jnklU/btsPFZKQSrJ/aHNecgxEqPLdnpIVw9ZR0K/img_640x640.jpg",
          "optionalAgree": true
        }
      }
    ```
    - **상세 스펙 (`data`)**:
  
      | 필드 | 타입 | 설명 |
      |---|---|---|
      | `nickname` | `String` | 수정된 닉네임 |
      | `profileImageUrl` | `String` | 프로필 이미지 URL |
      | `optionalAgree` | `Boolean` | 수정된 선택 약관 동의 여부 |
- **오류**:
    - `400 Bad Request`: `VALIDATION_ERROR` (닉네임 길이 제약 위반 등 유효성 검사 실패)
    - `401 Unauthorized`: `UNAUTHORIZED` (인증 정보 없음 또는 유효하지 않음)
    - `403 Forbidden`: `FORBIDDEN` (요청한 userId와 로그인 사용자 불일치)
    - `404 Not Found`: `USER_NOT_FOUND` (해당 ID의 사용자 없음)
    - `409 Conflict`: `NICKNAME_ALREADY_EXISTS` (이미 사용 중인 닉네임)

### 4.3. 닉네임 변경

- **Endpoint**: `PATCH /api/my-page/users/{userId}/nickname`
- **설명**: 현재 로그인한 사용자의 닉네임만 변경합니다. 프로필 전체 수정(3.2)과 달리 닉네임만 부분 업데이트합니다.
- **인증**: **필수**
- **요청**:
  - **Path Parameter**:
    | 이름 | 타입 | 필수 | 설명 |
    |---|---|---|---|
    | `userId` | `Long` | 필수 | 수정할 사용자 고유 ID (현재 로그인된 사용자) |
  - **Body (`application/json`)**:
    ```json
    {
      "nickname": "새로운닉네임"
    }
    ```
  - **상세 스펙 (Request Body)**:

    | 필드 | 타입 | 필수 | 제약사항 | 설명 |
    |---|---|---|---|---|          
    | `nickname` | `String` | 필수 | 1~10자, 공백 불가 | 변경할 닉네임 |
- **응답 (200 OK)**:
  - **Body**: `ApiResponse<UserProfileResponse>`
  - **JSON 응답 예시**:
    ```json
    {
      "success": true,
      "message": "닉네임 변경 성공",
      "timestamp": "2025-10-29T12:04:00.456Z",
      "data": {
        "nickname": "새로운닉네임",
        "profileImageUrl": "http://k.kakaocdn.net/dn/jnklU/btsPFZKQSrJ/aHNecgxEqPLdnpIVw9ZR0K/img_640x640.jpg",
        "optionalAgree": true
      }
    }
    ```
  - **상세 스펙 (`data`)**:

    | 필드 | 타입 | 설명 |
   |---|---|---|    
    | `nickname` | `String` | 변경된 닉네임 |
    | `profileImageUrl` | `String` | 프로필 이미지 URL |
    | `optionalAgree` | `Boolean` | 선택 약관 동의 여부 |
- **오류**:
  - `400 Bad Request`: `VALIDATION_ERROR` (닉네임 길이 제약 위반 등 유효성 검사 실패)
  - `401 Unauthorized`: `UNAUTHORIZED` (인증 정보 없음 또는 유효하지 않음)
  - `403 Forbidden`: `FORBIDDEN` (요청한 userId와 로그인 사용자 불일치)
  - `404 Not Found`: `USER_NOT_FOUND` (해당 ID의 사용자 없음)
  - `409 Conflict`: `NICKNAME_ALREADY_EXISTS` (이미 사용 중인 닉네임)

### 4.4. 약관 동의 처리

- **Endpoint**: `PATCH /api/my-page/users/{userId}/terms-agreement`
- **설명**: 사용자의 약관 동의를 처리합니다. 필수 약관(requiredAgree)은 자동으로 true로 설정되며, 선택 약관(optionalAgree)은 요청값에 따라 설정됩니다.
- **인증**: **필수**
- **요청**:
  - **Path Parameter**:
    | 이름 | 타입 | 필수 | 설명 |
    |---|---|---|---|
    | `userId` | `Long` | 필수 | 약관 동의를 처리할 사용자 고유 ID (현재 로그인된 사용자) |
  - **Body (`application/json`)**:
    ```json
    {
      "optionalAgree": true
    }
    ```
  - **상세 스펙 (Request Body)**:

    | 필드 | 타입 | 필수 | 설명 |
    |---|---|---|---|
    | `optionalAgree` | `Boolean` | 선택 | 선택 약관 동의 여부 (학습 데이터 수집 동의). null인 경우 기존 값 유지 |
- **응답 (200 OK)**:
  - **Body**: `ApiResponse<UserProfileResponse>`
  - **JSON 응답 예시**:
    ```json
    {
      "success": true,
      "message": "약관 동의 처리 성공",
      "timestamp": "2025-10-29T12:05:00.789Z",
      "data": {
        "nickname": "수어마스터",
        "profileImageUrl": "http://k.kakaocdn.net/dn/jnklU/btsPFZKQSrJ/aHNecgxEqPLdnpIVw9ZR0K/img_640x640.jpg",
        "optionalAgree": true
      }
    }
    ```
  - **상세 스펙 (`data`)**:

    | 필드 | 타입 | 설명 |
    |---|---|---|
    | `nickname` | `String` | 닉네임 |
    | `profileImageUrl` | `String` | 프로필 이미지 URL |
    | `optionalAgree` | `Boolean` | 업데이트된 선택 약관 동의 여부 |
- **오류**:
  - `401 Unauthorized`: `UNAUTHORIZED` (인증 정보 없음 또는 유효하지 않음)
  - `403 Forbidden`: `FORBIDDEN` (요청한 userId와 로그인 사용자 불일치)
  - `404 Not Found`: `USER_NOT_FOUND` (해당 ID의 사용자 없음)


---

## 3. 수어 데이터 관리 (Sign Data Management)

### 3.1. 외부 API 데이터 로드

- **Endpoint**: `GET /api/sign-data/getApiData`
- **설명**: 외부 수어 API로부터 데이터를 가져와 데이터베이스에 저장합니다. 최초 1회 실행되는 초기 데이터 세팅용 API입니다.
- **인증**: **필수** (관리자 전용)
- **요청**: 없음
- **응답 (200 OK)**:
    - **Body**: `SignDataLoadResponseDto`
    - **JSON 응답 예시**:
    ```json
      {
        "loadedCount": 1542,
        "message": "1542개의 수어 데이터가 성공적으로 로드되었습니다."
      }
    ```
    - **상세 스펙**:
  
      | 필드 | 타입 | 설명 |
      |---|---|---|
      | `loadedCount` | `Long` | 로드된 수어 데이터 개수 |
      | `message` | `String` | 결과 메시지 |
- **참고**:
    - 이미 데이터가 존재하는 경우, API 호출을 건너뛰고 기존 데이터를 유지합니다.
    - 외부 API로부터 데이터를 페이징 처리하여 모두 가져온 후, 임시 테이블(`sign_api`)을 거쳐 서비스 테이블(`sign`)에 저장됩니다.
- **오류**:
    - `401 Unauthorized`: `UNAUTHORIZED` (인증 정보 없음)
    - `403 Forbidden`: `FORBIDDEN` (관리자 권한 없음)
    - `500 Internal Server Error`: `INTERNAL_SERVER_ERROR` (외부 API 호출 실패 또는 데이터 저장 오류)

### 3.2. 수어 학습 상태 완료 처리

- **Endpoint**: `POST /api/sign-data/{signId}/complete`
- **설명**: 특정 수어 데이터의 학습 상태를 '완료(COMPLETED)'로 변경합니다. ML 모델이 학습 완료 후 이 API를 호출하여 상태를 업데이트합니다.
- **인증**: **필수**
- **요청**:
    - **Path Parameter**:
      | 이름 | 타입 | 필수 | 설명 |
      |---|---|---|---|
      | `signId` | `Long` | 필수 | 수어 데이터 고유 ID |
- **응답 (200 OK)**:
    - **Body**: `QuizWordUpdateResponse`
    - **JSON 응답 예시**:
    ```json
      {
        "signId": 123,
        "title": "안녕하세요",
        "newStatus": "COMPLETED",
        "message": "ID 123번 단어 '안녕하세요'의 상태가 '완료'(으)로 성공적으로 업데이트되었습니다."
      }
    ```
    - **상세 스펙**:
  
      | 필드 | 타입 | 설명 |
      |---|---|---|
      | `signId` | `Long` | 수어 데이터 ID |
      | `title` | `String` | 수어 단어 제목 |
      | `newStatus` | `String` | 변경된 학습 상태 (`COMPLETED`) |
      | `message` | `String` | 상태 변경 결과 메시지 |
- **참고**:
    - 상태가 `COMPLETED`로 변경되면 해당 단어가 퀴즈 단어 테이블(`quiz_word`)에 자동으로 추가됩니다.
- **오류**:
    - `401 Unauthorized`: `UNAUTHORIZED` (인증 정보 없음)
    - `404 Not Found`: `WORD_NOT_FOUND` (해당 ID의 수어 데이터가 없음)
    - `400 Bad Request`: `INVALID_SIGN_STATUS` (유효하지 않은 상태 전환)

### 3.3. 수어 학습 상태 진행중 처리

- **Endpoint**: `POST /api/sign-data/{signId}/start-progress`
- **설명**: 특정 수어 데이터의 학습 상태를 '진행중(IN_PROGRESS)'으로 변경합니다.
- **인증**: **필수**
- **요청**:
    - **Path Parameter**:
      | 이름 | 타입 | 필수 | 설명 |
      |---|---|---|---|
      | `signId` | `Long` | 필수 | 수어 데이터 고유 ID |
- **응답 (200 OK)**:
    - **Body**: `QuizWordUpdateResponse`
    - **JSON 응답 예시**:
    ```json
      {
        "signId": 123,
        "title": "감사합니다",
        "newStatus": "IN_PROGRESS",
        "message": "ID 123번 단어 '감사합니다'의 상태가 '진행중'(으)로 성공적으로 업데이트되었습니다."
      }
    ```
    - **상세 스펙**:
  
      | 필드 | 타입 | 설명 |
      |---|---|---|
      | `signId` | `Long` | 수어 데이터 ID |
      | `title` | `String` | 수어 단어 제목 |
      | `newStatus` | `String` | 변경된 학습 상태 (`IN_PROGRESS`) |
      | `message` | `String` | 상태 변경 결과 메시지 |
- **참고**:
    - 상태가 `COMPLETED`가 아닌 경우, 퀴즈 단어 테이블(`quiz_word`)에서 해당 단어가 제거됩니다.
- **오류**:
    - `401 Unauthorized`: `UNAUTHORIZED` (인증 정보 없음)
    - `404 Not Found`: `WORD_NOT_FOUND` (해당 ID의 수어 데이터가 없음)
    - `400 Bad Request`: `INVALID_SIGN_STATUS` (유효하지 않은 상태 전환)

### 3.4. 수어 학습 상태 초기화

- **Endpoint**: `POST /api/sign-data/{signId}/reset`
- **설명**: 특정 수어 데이터의 학습 상태를 '미진행(PENDING)'으로 초기화합니다.
- **인증**: **필수**
- **요청**:
    - **Path Parameter**:
      | 이름 | 타입 | 필수 | 설명 |
      |---|---|---|---|
      | `signId` | `Long` | 필수 | 수어 데이터 고유 ID |
- **응답 (200 OK)**:
    - **Body**: `QuizWordUpdateResponse`
    - **JSON 응답 예시**:
    ```json
      {
        "signId": 123,
        "title": "미안합니다",
        "newStatus": "PENDING",
        "message": "ID 123번 단어 '미안합니다'의 상태가 '미진행'(으)로 성공적으로 업데이트되었습니다."
      }
    ```
    - **상세 스펙**:
  
      | 필드 | 타입 | 설명 |
      |---|---|---|
      | `signId` | `Long` | 수어 데이터 ID |
      | `title` | `String` | 수어 단어 제목 |
      | `newStatus` | `String` | 변경된 학습 상태 (`PENDING`) |
      | `message` | `String` | 상태 변경 결과 메시지 |
- **참고**:
    - 상태가 `COMPLETED`가 아닌 경우, 퀴즈 단어 테이블(`quiz_word`)에서 해당 단어가 제거됩니다.
- **오류**:
    - `401 Unauthorized`: `UNAUTHORIZED` (인증 정보 없음)
    - `404 Not Found`: `WORD_NOT_FOUND` (해당 ID의 수어 데이터가 없음)
    - `400 Bad Request`: `INVALID_SIGN_STATUS` (유효하지 않은 상태 전환)

---

---

## 4. 수어 학습 콘텐츠 (Sign Education)

### 4.1. 카테고리 목록 조회

- **Endpoint**: `GET /api/sign-edu/categories`
- **설명**: 수어 단어의 모든 카테고리(태그) 목록을 중복 없이 조회합니다.
- **인증**: **필수**
- **요청**: 없음
- **응답 (200 OK)**:
    - **Body**: `ApiResponse<List<String>>`
    - **JSON 응답 예시**:
    ```json
      {
        "success": true,
        "message": "카테고리 목록 조회 성공",
        "timestamp": "2025-10-21T12:00:00.000Z",
        "data": ["삶", "감정", "인사", "가족", "시간", "장소"]
      }
    ```
    - **상세 스펙 (`data`)**:
  
      | 필드 | 타입 | 설명 |
      |---|---|---|
      | `data` | `List<String>` | 카테고리 문자열 배열 |
- **참고**:
    - 카테고리가 없는 경우 빈 배열(`[]`)을 반환합니다.
- **오류**:
    - `401 Unauthorized`: `UNAUTHORIZED` (인증 정보 없음)

### 4.2. 수어 단어 목록 조회

- **Endpoint**: `GET /api/sign-edu`
- **설명**: 모든 수어 단어 또는 특정 카테고리의 수어 단어 목록을 페이징하여 조회합니다. 키워드 검색 기능을 지원합니다.
- **인증**: **필수**
- **요청**:
    - **Query Parameters**:
      | 이름 | 타입 | 필수 | 기본값 | 설명 |
      |---|---|---|---|---|
      | `category` | `String` | 선택 | - | 조회할 카테고리명 (미입력 또는 "전체" 입력 시 전체 조회) |
      | `keyword` | `String` | 선택 | - | 검색할 단어명 (부분 일치 검색) |
      | `page` | `Integer` | 선택 | `0` | 페이지 번호 (0부터 시작) |
      | `size` | `Integer` | 선택 | `20` | 페이지당 항목 수 |
      | `sort` | `String` | 선택 | `title` | 정렬 기준 필드 |

- **검색 우선순위**:
    - `keyword` 파라미터가 있는 경우, 다음 우선순위로 정렬됩니다:
      1. **완전 일치**: 단어명이 키워드와 정확히 일치
      2. **앞부분 일치**: 단어명이 키워드로 시작
      3. **부분 일치**: 단어명에 키워드가 포함
    - 같은 우선순위 내에서는 단어명 가나다순(오름차순)으로 정렬됩니다.
    - `keyword`가 없는 경우, 기본 정렬은 `title`(단어명) 오름차순입니다.

- **파라미터 우선순위**:
    - `keyword` > `category` > 전체 조회
    - `keyword`가 있으면 `category`는 무시됩니다.

- **응답 (200 OK)**:
    - **Body**: `ApiResponse<Page<SignSimpleResponseDto>>`
    - **JSON 응답 예시**:
    ```json
      {
        "success": true,
        "message": "수어 단어 목록 조회 성공",
        "timestamp": "2025-10-21T12:01:00.000Z",
        "data": {
          "content": [
            {
              "signId": 1,
              "wordName": "감사합니다"
            },
            {
              "signId": 2,
              "wordName": "미안합니다"
            },
            {
              "signId": 3,
              "wordName": "사랑해요"
            }
          ],
          "pageable": {
            "sort": {
              "sorted": true,
              "unsorted": false,
              "empty": false
            },
            "pageNumber": 0,
            "pageSize": 20,
            "offset": 0,
            "paged": true,
            "unpaged": false
          },
          "totalPages": 5,
          "totalElements": 92,
          "last": false,
          "size": 20,
          "number": 0,
          "sort": {
            "sorted": true,
            "unsorted": false,
            "empty": false
          },
          "numberOfElements": 20,
          "first": true,
          "empty": false
        }
      }
    ```
    - **상세 스펙 (`data.content[]`)**:
  
      | 필드 | 타입 | 설명 |
      |---|---|---|
      | `signId` | `Long` | 수어 단어 고유 ID |
      | `wordName` | `String` | 수어 단어명 |

    - **페이징 정보 (`data`)**:
  
      | 필드 | 타입 | 설명 |
      |---|---|---|
      | `content` | `Array` | 현재 페이지의 수어 단어 목록 |
      | `pageable` | `Object` | 페이징 요청 정보 |
      | `totalPages` | `Integer` | 전체 페이지 수 |
      | `totalElements` | `Long` | 전체 항목 수 |
      | `size` | `Integer` | 페이지당 항목 수 |
      | `number` | `Integer` | 현재 페이지 번호 (0부터 시작) |
      | `first` | `Boolean` | 첫 페이지 여부 |
      | `last` | `Boolean` | 마지막 페이지 여부 |
      | `empty` | `Boolean` | 빈 페이지 여부 |
- **요청 예시**:
    - 전체 조회: `GET /api/sign-edu?page=0&size=20`
    - 카테고리별 조회: `GET /api/sign-edu?category=삶&page=0&size=20`
    - 키워드 검색: `GET /api/sign-edu?keyword=감사&page=0&size=20`
    - 키워드 검색 (카테고리는 무시됨): `GET /api/sign-edu?keyword=감사&category=삶&page=0`

- **참고**:
    - 존재하지 않는 카테고리로 조회 시 빈 페이지(`content: []`)를 반환합니다.
    - `keyword` 검색 시 부분 일치(LIKE '%keyword%')로 검색되며, 결과는 관련성 순으로 정렬됩니다.
    - `category`에 "전체"를 입력하거나 빈 값을 보내면 전체 조회가 됩니다.
    - 키워드 검색 결과는 QueryDSL의 CaseBuilder를 사용하여 최적화된 정렬을 제공합니다.

- **오류**:
  - `401 Unauthorized`: `UNAUTHORIZED` (인증 정보 없음 또는 유효하지 않음)

### 4.3. 수어 단어 상세 조회

- **Endpoint**: `GET /api/sign-edu/{signId}`
- **설명**: 특정 수어 단어의 상세 정보를 조회합니다. 단어명, 설명, 영상 URL, 카테고리 정보를 포함합니다.
- **인증**: **필수**
- **요청**:
    - **Path Parameter**:
      | 이름 | 타입 | 필수 | 설명 |
      |---|---|---|---|
      | `signId` | `Long` | 필수 | 수어 단어 고유 ID |
- **응답 (200 OK)**:
    - **Body**: `ApiResponse<SignDetailResponseDto>`
    - **JSON 응답 예시**:
    ```json
      {
        "success": true,
        "message": "수어 단어 상세 조회 성공",
        "timestamp": "2025-10-21T12:02:00.000Z",
        "data": {
          "signId": 1,
          "wordName": "안녕하세요",
          "description": "오른손을 들어 인사하는 동작입니다. 손바닥을 펴서 상대방을 향하게 합니다.",
          "videoUrl": "https://sldict.korean.go.kr/multimedia/multimedia_files/convert/20191230/625954/MOV000249969_700X466.webm",
          "tag": "인사"
        }
      }
    ```
    - **상세 스펙 (`data`)**:
  
      | 필드 | 타입 | 설명 |
      |---|---|---|
      | `signId` | `Long` | 수어 단어 고유 ID |
      | `wordName` | `String` | 수어 단어명 |
      | `description` | `String` | 수어 동작 설명 |
      | `videoUrl` | `String` | 수어 동작 영상 URL |
      | `tag` | `String` | 카테고리(태그) |
- **오류**:
    - `401 Unauthorized`: `UNAUTHORIZED` (인증 정보 없음)
    - `404 Not Found`: `WORD_NOT_FOUND` (해당 ID의 수어 단어가 없음)

---

## 5. 퀴즈 방 관리 (Quiz Room)

### 5.1. 퀴즈 방 목록 조회

- **Endpoint**: `GET /api/quiz/rooms`
- **설명**: 현재 활성 상태(대기 중 또는 진행 중)인 퀴즈 방 목록을 Slice 페이징 방식으로 조회합니다. 무한 스크롤을 지원합니다.
- **인증**: **필수**
- **요청**:
    - **Query Parameters**:
      | 이름 | 타입 | 필수 | 기본값 | 제약사항 | 설명 |
      |---|---|---|---|---|---|
      | `page` | `Integer` | 선택 | `0` | 0 이상 | 페이지 번호 (0부터 시작) |
      | `size` | `Integer` | 선택 | `10` | 1~100 | 페이지당 항목 수 |
- **응답 (200 OK)**:
    - **Body**: `ApiResponse<RoomListSliceResponse>`
    - **JSON 응답 예시**:
    ```json
      {
        "success": true,
        "message": "퀴즈 방 목록을 조회했습니다.",
        "timestamp": "2025-10-21T12:00:00.000Z",
        "data": {
          "roomList": [
            {
              "gameRoomId": 1,
              "gameTitle": "초보자 환영 퀴즈방",
              "hostNickname": "수어마스터",
              "currentParticipants": 2,
              "maxParticipants": 4,
              "currentRound": 0,
              "status": "WAITING"
            },
            {
              "gameRoomId": 2,
              "gameTitle": "실력자 모여라",
              "hostNickname": "수어고수",
              "currentParticipants": 3,
              "maxParticipants": 4,
              "currentRound": 5,
              "status": "IN_PROGRESS"
            }
          ],
          "hasNext": true
        }
      }
    ```
    - **상세 스펙 (`data.roomList[]`)**:
  
      | 필드 | 타입 | 설명 |
      |---|---|---|
      | `gameRoomId` | `Long` | 퀴즈 방 고유 ID |
      | `gameTitle` | `String` | 방 제목 |
      | `hostNickname` | `String` | 방장 닉네임 |
      | `currentParticipants` | `Integer` | 현재 참여자 수 |
      | `maxParticipants` | `Integer` | 최대 참여자 수 (4명) |
      | `currentRound` | `Integer` | 현재 진행 중인 라운드 |
      | `status` | `String` | 방 상태 (`WAITING`, `IN_PROGRESS`) |

    - **페이징 정보 (`data`)**:
  
      | 필드 | 타입 | 설명 |
      |---|---|---|
      | `roomList` | `Array` | 현재 페이지의 퀴즈 방 목록 |
      | `hasNext` | `Boolean` | 다음 페이지 존재 여부 (무한 스크롤용) |
- **참고**:
    - `WAITING` (대기 중) 또는 `IN_PROGRESS` (진행 중) 상태의 방만 조회됩니다.
    - `FINISHED` (종료) 상태의 방은 목록에 표시되지 않습니다.
    - `size` 파라미터가 1~100 범위를 벗어나면 기본값 10으로 자동 설정됩니다.
- **오류**:
    - `401 Unauthorized`: `UNAUTHORIZED` (인증 정보 없음 또는 유효하지 않음)

### 5.2. 퀴즈 방 상세 조회

- **Endpoint**: `GET /api/quiz/rooms/{roomId}`
- **설명**: 특정 퀴즈 방의 상세 정보를 조회합니다.
- **인증**: **필수**
- **요청**:
    - **Path Parameter**:
      | 이름 | 타입 | 필수 | 설명 |
      |---|---|---|---|
      | `roomId` | `Long` | 필수 | 퀴즈 방 고유 ID |
- **응답 (200 OK)**:
    - **Body**: `ApiResponse<RoomDetailResponse>`
    - **JSON 응답 예시**:
    ```json
      {
        "success": true,
        "message": "퀴즈 방 상세 정보를 조회했습니다.",
        "timestamp": "2025-10-21T12:01:00.000Z",
        "data": {
          "gameRoomId": 1,
          "gameTitle": "초보자 환영 퀴즈방",
          "currentParticipants": 2,
          "maxParticipants": 4,
          "status": "WAITING"
        }
      }
    ```
    - **상세 스펙 (`data`)**:
  
      | 필드 | 타입 | 설명 |
      |---|---|---|
      | `gameRoomId` | `Long` | 퀴즈 방 고유 ID |
      | `gameTitle` | `String` | 방 제목 |
      | `currentParticipants` | `Integer` | 현재 참여자 수 |
      | `maxParticipants` | `Integer` | 최대 참여자 수 (4명) |
      | `status` | `String` | 방 상태 (`WAITING`, `IN_PROGRESS`) |
- **참고**:
    - `FINISHED` (종료) 상태의 방은 조회할 수 없습니다.
- **오류**:
    - `401 Unauthorized`: `UNAUTHORIZED` (인증 정보 없음 또는 유효하지 않음)
    - `404 Not Found`: `ROOM_NOT_FOUND` (해당 ID의 방이 없거나 종료된 방)

### 5.3. 퀴즈 방 생성

- **Endpoint**: `POST /api/quiz/rooms`
- **설명**: 새로운 퀴즈 방을 생성합니다. 방을 생성한 사용자가 자동으로 방장이 되며, 첫 번째 참가자로 등록됩니다.
- **인증**: **필수**
- **요청**:
    - **Body (`application/json`)**:
    ```json
      {
        "gameTitle": "초보자 환영 퀴즈방"
      }
    ```
    - **상세 스펙 (Request Body)**:
  
      | 필드 | 타입 | 필수 | 제약사항 | 설명 |
      |---|---|---|---|---|
      | `gameTitle` | `String` | 필수 | 1~50자, 공백 불가 | 방 제목 |
- **응답 (200 OK)**:
    - **Body**: `ApiResponse<CreateRoomResponse>`
    - **JSON 응답 예시**:
    ```json
      {
        "success": true,
        "message": "퀴즈 방이 성공적으로 생성되었습니다.",
        "timestamp": "2025-10-21T12:02:00.000Z",
        "data": {
          "gameRoomId": 123
        }
      }
    ```
    - **상세 스펙 (`data`)**:
  
      | 필드 | 타입 | 설명 |
      |---|---|---|
      | `gameRoomId` | `Long` | 생성된 퀴즈 방 ID |
- **참고**:
    - 사용자가 이미 다른 `WAITING` 또는 `IN_PROGRESS` 상태의 방에 참여 중인 경우 방 생성이 불가능합니다.
    - 방 생성 시 초기 상태는 `WAITING`이며, 최대 참여자 수는 4명입니다.
    - 방장은 자동으로 첫 번째 참가자로 등록됩니다.
- **오류**:
    - `400 Bad Request`: `VALIDATION_ERROR` (방 제목 유효성 검사 실패)
    - `401 Unauthorized`: `UNAUTHORIZED` (인증 정보 없음 또는 유효하지 않음)
    - `404 Not Found`: `USER_NOT_FOUND` (사용자 정보를 찾을 수 없음)
    - `409 Conflict`: `PARTICIPANT_ALREADY_IN_ROOM` (이미 다른 방에 참여 중)

### 5.4. WebSocket 세션 상태 확인

- **Endpoint**: `GET /api/ws/session/active`
- **설명**: 현재 로그인한 사용자의 WebSocket 세션 활성 상태를 확인합니다. 중복 접속 방지를 위한 세션 체크 기능입니다.
- **인증**: **필수**
- **요청**: 없음
- **응답 (200 OK)**:
    - **Body**: `ApiResponse<WsSessionStatusResponse>`
    - **JSON 응답 예시 (활성 세션 존재)**:
    ```json
      {
        "success": true,
        "message": "WS 세션 활성 여부",
        "timestamp": "2025-10-21T12:03:00.000Z",
        "data": {
          "active": true,
          "reason": "ACTIVE_SESSION_EXISTS"
        }
      }
    ```
    - **JSON 응답 예시 (활성 세션 없음)**:
    ```json
      {
        "success": true,
        "message": "WS 세션 활성 여부",
        "timestamp": "2025-10-21T12:03:00.000Z",
        "data": {
          "active": false,
          "reason": "NONE"
        }
      }
    ```
    - **상세 스펙 (`data`)**:
  
      | 필드 | 타입 | 설명 |
      |---|---|---|
      | `active` | `Boolean` | WebSocket 세션 활성 여부 |
      | `reason` | `String` | 상태 사유 (`ACTIVE_SESSION_EXISTS`: 활성 세션 존재, `NONE`: 세션 없음) |
- **참고**:
    - 이 API는 사용자당 하나의 WebSocket 연결만 허용하는 "단일 탭 강제" 정책을 지원합니다.
    - 동일 사용자가 다른 탭/브라우저에서 중복 접속을 시도하면 기존 세션이 활성 상태로 감지됩니다.
    - WebSocket 연결 전에 이 API를 호출하여 기존 세션 존재 여부를 확인할 수 있습니다.
- **오류**:
    - `401 Unauthorized`: `UNAUTHORIZED` (인증 정보 없음 또는 유효하지 않음)

---

## 6. 데이터 타입 및 제약사항

### 공통 제약사항
- **방 제목**: 1~50자, 공백 불가
- **닉네임**: 1~20자, 공백 불가
- **페이지 번호**: 0 이상의 정수
- **페이지 크기**:
    - 수어 학습: 기본값 20, 1~100
    - 퀴즈 방 목록: 기본값 10, 1~100
- **수어 단어 ID**: 양의 정수
- **퀴즈 방 ID**: 양의 정수
- **사용자 ID**: 양의 정수

### 날짜/시간 형식
- 타임스탬프: ISO 8601 형식 (예: `2025-10-21T12:00:00.000Z`)
- 모든 시간은 UTC 기준

### 열거형(Enum) 값

#### 퀴즈 방 상태 (GameRoomStatus)
| 값 | 한글명 | 설명 |
|---|---|---|
| `WAITING` | 대기 중 | 참가자를 모집 중인 상태 |
| `IN_PROGRESS` | 진행 중 | 퀴즈가 진행 중인 상태 |
| `FINISHED` | 종료 | 퀴즈가 완료된 상태 |

#### 수어 학습 상태 (SignStatus)
| 값 | 한글명 | 설명 |
|---|---|---|
| `PENDING` | 미진행 | AI 모델 학습이 시작되지 않은 초기 상태 |
| `IN_PROGRESS` | 진행중 | AI 모델 학습이 진행 중인 상태 |
| `COMPLETED` | 완료 | AI 모델 학습이 완료되어 퀴즈에 사용 가능한 상태 |

**참고**: `COMPLETED` 상태의 단어만 실시간 퀴즈에서 출제됩니다.

#### 로그인 방식 (LoginMethod)
| 값 | 설명 |
|---|---|
| `KAKAO` | 카카오톡 소셜 로그인 |

**참고**: 현재는 카카오 로그인만 지원합니다.

---

## 변경 이력

| 버전     | 날짜         | 변경 내용                                 | 작성자 |
|--------|------------|---------------------------------------|-----|
| v1.0.0 | 2025.10.08 | 초기 문서 작성                              | 신동준 |
| v2.0.0 | 2025.10.21 | 백엔드 코드 기반으로 API 명세서 현행화               | 강관주 |
| v2.1.0 | 2025.10.29 | 사용자 정보 조회 API, 사용자 랭킹 API 추가 및 문서 재구성 | 강관주 |
