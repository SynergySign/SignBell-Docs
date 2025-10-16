# **API 명세서 - SignBell**

| 항목 | 내용                                |
| :--- |:----------------------------------|
| **팀명** | SynergeSign                       |
| **프로젝트명** | SignBell                          |
| **플랫폼명** | SignBell                          |
| **문서 버전** | v1.0.1                            |
| **작성자** | [신동준](https://github.com/sdj3959) |
| **최종 수정일**| 2025-10-16                        |
| **Base URL** | `https://localhost:9000`          |
| **인증 방식** | JWT Bearer Token (쿠키 전달)          |
| **응답 형식** | JSON                              |



## 인증
모든 API 엔드포인트는 Bearer 토큰 인증이 필요합니다.

```
Authorization: Bearer {token}
```

## API 태그 분류

### 1. Authentication (인증/회원가입 API)
### 2. MyPage (마이페이지 API)
### 3. Agreements (필수/선택 약관동의 API)

---

## 1. Authentication API

### 1.1 회원가입
**POST** `/api/v1/auth/signup`

로컬 계정 회원가입. 이메일과 비밀번호를 사용하는 로컬 계정을 생성합니다.

**Request Body:**
```json
{
  ~
}
```

**Responses:**
- `200`: 회원가입 성공
- `400`: 회원가입 실패

### 1.2 로그인
**POST** `/api/v1/~`

**Request Body:**
```json
{
  ~
}
```

### 1.3 로그아웃
**POST** `/api/v1/~`

**Query Parameters:**
- `token` (required): 인증 토큰

### 1.4 토큰 갱신
**POST** `/api/v1/~`

**Cookies:**
- `REFRESH_TOKEN`: 리프레시 토큰

---

요청하신 형식과 내용을 기반으로 유저 프로필 조회 및 수정 API 문서를 전체 마크다운 형식으로 작성해 드립니다.

새로운 Base URL Prefix (`/api/my-page`)와 Path Variable (`{userId}`) 구조를 반영했습니다.

-----

## 2\. 사용자 (User)

### 2.1. 내 프로필 조회

- **Endpoint**: `GET /my-page/users/{userId}/profile`
- **설명**: 현재 로그인한 사용자의 상세 프로필 정보를 조회합니다. `userId`는 현재 인증된 사용자의 ID여야 합니다.
- **인증**: **필수** (JWT Bearer Token)
- **요청**: 없음
- **응답 (200 OK)**:
    - **Body**: `ApiResponse<UserProfileResponse>`
    - **JSON 응답 예시**:
      ```json
      {
          "success": true,
          "message": "프로필 조회 성공",
          "timestamp": "2025-10-16T17:16:40.4370385",
          "data": {
              "nickname": "송민재",
              "profileImageUrl": "http://k.kakaocdn.net/dn/jnklU/btsPFZKQSrJ/aHNecgxEqPLdnpIVw9ZR0K/img_640x640.jpg",
              "optionalAgree": false
          }
      }
      ```
    - **상세 스펙 (`data`)**:

      | 필드 | 타입 | 설명 |
      |---|---|---|
      | `nickname` | `String` | 닉네임 |
      | `profileImageUrl` | `String` | 프로필 이미지 URL (null 가능) |
      | `optionalAgree` | `Boolean` | 선택 약관 동의 여부 |
- **오류**:
    - `401 Unauthorized`: `UNAUTHORIZED` (인증 정보 없음/유효하지 않음)
    - `403 Forbidden`: `FORBIDDEN` (요청 `userId`와 로그인 사용자 불일치 등 권한 없음)
    - `404 Not Found`: `USER_NOT_FOUND` (해당 ID의 사용자 정보가 DB에 없음)

### 2.2. 내 프로필 수정

- **Endpoint**: `PATCH /my-page/users/{userId}/profile`
- **설명**: 현재 로그인한 사용자의 프로필 정보를 수정합니다. `userId`는 현재 인증된 사용자의 ID여야 합니다.
- **인증**: **필수** (JWT Bearer Token)
- **요청**:
    - **Path Parameter**: `userId` (Long, 수정할 사용자 고유 ID (현재 로그인된 사용자))
    - **Body**: `ApiResponse<UserProfileUpdateRequest>`
    - **JSON 요청 예시**:
      ```json
      {
          "nickname":"변경된이름",
          "optionalAgree":true
      }
      ```
- **응답 (200 OK)**:
    - **Body**: `ApiResponse<UserProfileResponse>` 
    - **JSON 응답 예시**:
      ```json
      {
          "success": true,
          "message": "프로필 수정 성공",
          "timestamp": "2025-10-16T17:19:21.7728209",
          "data": {
              "nickname": "변경된이름",
              "profileImageUrl": "http://k.kakaocdn.net/dn/jnklU/btsPFZKQSrJ/aHNecgxEqPLdnpIVw9ZR0K/img_640x640.jpg",
              "optionalAgree": true
          }
      }
      ```
    - **상세 스펙 (`data`)**:

      | 필드                | 타입        | 설명              |
      |-------------------|-----------|-----------------|
      | `nickname`        | `String`  | 수정된 닉네임         |
      | `optionalAgree`   | `Boolean` | 수정된 선택 약관 동의 여부 |
- **오류**:
    - `400 Bad Request`: `INVALID_INPUT_VALUE` (요청 Body 유효성 검사 실패)
    - `401 Unauthorized`: `UNAUTHORIZED` (인증 정보 없음/유효하지 않음)
    - `403 Forbidden`: `FORBIDDEN` (요청 `userId`와 로그인 사용자 불일치 등 권한 없음)
    - `404 Not Found`: `USER_NOT_FOUND` (해당 ID의 사용자 정보가 DB에 없음)
---

## 10. Test API

### 10.1 API 테스트
**GET** `/api/test`

### 10.2 헬스 체크
**GET** `/api/health`

### 10.3 홈
**GET** `/`

---

## 공통 응답 형식

### 성공 응답
- `200 OK`: 요청 성공
- `201 Created`: 리소스 생성 성공

### 오류 응답
- `400 Bad Request`: 잘못된 요청
- `401 Unauthorized`: 인증 실패
- `403 Forbidden`: 권한 없음
- `404 Not Found`: 리소스 없음
- `500 Internal Server Error`: 서버 내부 오류

## 데이터 타입 및 제약사항

### 공통 제약사항
- ~
- ~

### 날짜/시간 형식
- 날짜: `yyyy-MM-dd` (예: `2025-09-22`)
- 시간: `HH:mm:ss` (예: `14:30:00`)
- 일시: ISO 8601 형식 (예: `2025-09-22T14:30:00`)

### 열거형(Enum) 값

#### ~ 타입 (~Type)
- `~`: ~
- `~`: ~

