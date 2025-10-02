# **API 명세서 - TropiCal**

| 항목 | 내용                                |
| :--- |:----------------------------------|
| **팀명** | SynergeSign                       |
| **프로젝트명** | SignBell                          |
| **플랫폼명** | SignBell                          |
| **문서 버전** | v1.0                              |
| **작성자** | [신동준](https://github.com/sdj3959) |
| **최종 수정일**| 2025-10-02                        |
| **Base URL** | `https://localhost:~`             |
| **인증 방식** | JWT Bearer Token (쿠키 전달)          |
| **응답 형식** | JSON                              |



## 인증
모든 API 엔드포인트는 Bearer 토큰 인증이 필요합니다.

```
Authorization: Bearer {token}
```

## API 태그 분류

### 1. Authentication (인증/회원가입 API)
### 2. Todo (할 일 API)
### 3. Schedule (일정 API)
### 4. Diary (일기 API)
### 5. BucketList (버킷리스트 API)
### 6. User Preferences (사용자 선호 설정 통합 관리 API)
### 7. Terms (약관 및 동의서 조회 API)
### 8. Holiday (공휴일 조회 API)
### 9. Admin (관리자 API)

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

## 2. ~ API

### 2.1 ~
**GET** `/api/v1/~`

~을 반환합니다.

**Response:**
```json
[
  ~
]
```
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

