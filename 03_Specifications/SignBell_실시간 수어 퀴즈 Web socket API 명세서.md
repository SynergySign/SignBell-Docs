# **실시간 수어 퀴즈 WebSocket API 명세서 - SignBell**

본 문서는 실시간 수어 학습 플랫폼 'SignBell'의 핵심 기능인 '실시간 수어 퀴즈'를 지원하는 WebSocket API 명세서입니다.
이 문서는 개발팀, 기획팀, 디자인팀 등 모든 프로젝트 구성원이 WebSocket 기반 실시간 통신 규약을 명확하게 이해하고, 일관된 사용자 경험을 구현하는 것을 목표로 합니다.
각 팀은 본 명세서를 기준으로 기능 개발, 화면 설계, 테스트 시나리오 작성을 진행하며, 이를 통해 효율적인 협업과 안정적인 서비스 구축을 지향합니다.

* **작성자:** [고동현](https://github.com/rhehdgus8831)
* **작성일**: 2025-10-21
* **최종 수정일**: 2025-10-21
* **문서 버전:** v1.0.1

**대상 독자:**

* **기획자 / PM**: 사용자 여정과 서비스 흐름을 이해하고 기획 방향성을 검증하는 담당자
* **백엔드 개발자**: WebSocket 시그널링 서버, AI 모델 서빙 API, 실시간 퀴즈 로직, 사용자 인증 및 데이터베이스 등을 구현·유지보수하는 개발자
* **프론트엔드 개발자**: 실시간 캠 영상 처리, WebSocket 기반 화상 퀴즈 UI, 거울 모드, 사용자 인터페이스 기능을 설계·구현하는 개발자
* **디자이너 (UX/UI)**: 사용자 흐름에 기반한 화면 설계, 인터랙션 디자인, 사용자 경험을 구체화하는 담당자
* **DevOps / 인프라 엔지니어**: WebSocket 서버 및 AI 모델 서빙 환경 구축, CI/CD 파이프라인 운영, 보안 설정을 관리하는 담당자
* **QA / 테스트 엔지니어**: 사용자 시나리오별 테스트 케이스 작성, AI 모델 정확도 검증, WebSocket 다중 사용자 환경 테스트를 수행하는 담당자
* **신규 합류자**: SignBell 서비스의 사용자 여정과 주요 기능 흐름을 빠르게 파악해야 하는 팀 신규 인원

---

| **WebSocket URL** | `wss://localhost:9000/ws`         |
| **프로토콜** | STOMP over WebSocket Secure (WSS)  |
| **인증 방식** | HTTP-Only Secure Cookie (JWT)     |
| **응답 형식** | JSON                              |

## 인증

모든 WebSocket 연결은 HTTPS 환경에서 Secure HTTP-Only 쿠키를 통한 JWT 토큰 인증이 필요합니다.

```
Cookie: ACCESS_TOKEN={jwt_token}; Secure; HttpOnly; SameSite=Strict
```

## WebSocket 연결 구조

### 연결 엔드포인트
- **URL**: `/ws`
- **프로토콜**: STOMP over WSS (WebSocket Secure)
- **허용 오리진**: `https://localhost:5173`

### 메시지 브로커 구조
- **구독 경로**: `/topic`, `/queue`, `/user`
- **발행 경로**: `/app`
- **사용자 메시지**: `/user`

---

## API 태그 분류

### 1. Session Management (세션 관리)
### 2. Game Room Management (게임방 관리)
### 3. Quiz Game Flow (퀴즈 게임 진행)
### 4. Real-time Notifications (실시간 알림)

---

## 1. Session Management API

### 1.1 WebSocket 세션 상태 확인
**GET** `/api/ws/session/active`

현재 사용자의 WebSocket 세션 활성 상태를 조회합니다. (중복 접속 방지용)

**Headers:**
```
Authorization: Bearer {jwt_token}
```

**Success Response:**
- **Status**: `200 OK`
- **Body**: `ApiResponse<WsSessionStatusResponse>`

```json
{
   "success": true,
   "message": "WS 세션 활성 여부",
   "timestamp": "2025-10-21T14:25:00.123456",
   "data": {
      "active": true,
      "reason": "ACTIVE_SESSION_EXISTS"
   }
}
```

**세션 없을 때 Response:**
```json
{
   "success": true,
   "message": "WS 세션 활성 여부",
   "timestamp": "2025-10-21T14:25:00.123456",
   "data": {
      "active": false,
      "reason": "NONE"
   }
}
```

**Response Fields:**
| 필드 | 타입 | 설명 |
|------|------|------|
| `active` | `Boolean` | WebSocket 세션 활성 여부 |
| `reason` | `String` | 상태 사유 ("ACTIVE_SESSION_EXISTS" 또는 "NONE") |

**Error Cases:**
- `401 Unauthorized`: `UNAUTHORIZED` - 인증되지 않은 사용자

---

### 1.2 WebSocket 연결 해제 처리
**EVENT** `SessionDisconnectEvent`

WebSocket 연결이 끊어질 때 자동으로 처리되는 이벤트입니다.

**자동 처리 내용:**
1. 사용자가 참여 중인 게임방에서 자동 퇴장
2. 같은 방 참가자들에게 퇴장 알림 브로드캐스트
3. 세션 레지스트리에서 매핑 제거

**Broadcast Response:**
- **Topic**: `/topic/room/{gameRoomId}/participant`
- **Body**: `ApiResponse<ParticipantEventResponse>`

```json
{
   "success": true,
   "message": "사용자가 연결을 끊었습니다.",
   "timestamp": "2025-10-21T14:26:00.123456",
   "data": {
      "eventType": "PARTICIPANT_LEFT",
      "participant": {
         "userId": 12345,
         "nickname": "사용자A",
         "profileImageUrl": "https://example.com/profile.jpg",
         "isHost": false,
         "isReady": true
      },
      "currentParticipants": 2,
      "gameRoomId": 1,
      "roomClosed": false
   }
}
```

**방장 퇴장 시 (방 종료):**
```json
{
   "success": true,
   "message": "사용자가 연결을 끊었습니다.",
   "timestamp": "2025-10-21T14:26:00.123456",
   "data": {
      "eventType": "ROOM_CLOSED",
      "participant": {
         "userId": 12345,
         "nickname": "방장",
         "profileImageUrl": "https://example.com/profile.jpg",
         "isHost": true,
         "isReady": false
      },
      "currentParticipants": 0,
      "gameRoomId": 1,
      "roomClosed": true
   }
}
```

**방 종료 시 개인 알림:**
- **Topic**: `/user/{userId}/queue/room-closed`
- **Body**: `ApiResponse<null>`

```json
{
   "success": false,
   "message": "방장이 나가서 방이 종료되었습니다.",
   "timestamp": "2025-10-21T14:26:00.123456",
   "data": null
}
```

---

## 2. Game Room Management API

### 2.1 게임방 입장
**SEND** `/app/room/{gameRoomId}/join`

게임방에 입장합니다.

**Path Parameters:**
- `gameRoomId` (Long, required): 입장할 게임방 ID

**Request Body:** 없음

**Success Response (개인):**
- **Topic**: `/user/{userId}/queue/room`
- **Body**: `ApiResponse<JoinRoomResponse>`

```json
{
   "success": true,
   "message": "방에 입장했습니다.",
   "timestamp": "2025-10-21T14:27:00.123456",
   "data": {
      "gameRoomId": 1,
      "gameTitle": "수어 마스터",
      "hostId": 12344,
      "participants": [
         {
            "userId": 12344,
            "nickname": "방장",
            "profileImageUrl": "https://example.com/profile1.jpg",
            "isHost": true,
            "isReady": false
         },
         {
            "userId": 12345,
            "nickname": "참가자A",
            "profileImageUrl": "https://example.com/profile2.jpg",
            "isHost": false,
            "isReady": false
         }
      ],
      "currentParticipants": 2,
      "maxParticipants": 4,
      "currentRound": 1,
      "status": "WAITING"
   }
}
```

**Success Response (브로드캐스트):**
- **Topic**: `/topic/room/{gameRoomId}/participant`
- **Body**: `ApiResponse<ParticipantEventResponse>`

```json
{
   "success": true,
   "message": "새로운 참가자가 입장했습니다.",
   "timestamp": "2025-10-21T14:27:00.123456",
   "data": {
      "eventType": "PARTICIPANT_JOINED",
      "participant": {
         "userId": 12345,
         "nickname": "참가자A",
         "profileImageUrl": "https://example.com/profile2.jpg",
         "isHost": false,
         "isReady": false
      },
      "currentParticipants": 2,
      "gameRoomId": 1,
      "roomClosed": false
   }
}
```

**Error Response:**
- **Topic**: `/user/{userId}/queue/errors`

```json
{
   "timestamp": "2025-10-21T14:27:00.123456",
   "status": 400,
   "error": "ROOM_FULL",
   "detail": "방 인원이 가득 찼습니다.",
   "path": "/app/room/join"
}
```

**Error Cases:**
- `ROOM_NOT_FOUND`: 존재하지 않는 방
- `ROOM_FULL`: 방 인원이 가득 참 (4명)
- `PARTICIPANT_ALREADY_IN_ROOM`: 이미 방에 참여 중
- `ROOM_ALREADY_STARTED`: 이미 시작된 방

---

### 2.2 참가자 준비 상태 변경
**SEND** `/app/room/{gameRoomId}/participant/ready`

참가자의 준비 상태를 변경합니다. (방장은 준비 상태 변경 불가)

**Path Parameters:**
- `gameRoomId` (Long, required): 게임방 ID

**Request Body:**
```json
{
   "isReady": true
}
```

**Request Fields:**
| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `isReady` | `Boolean` | ✅ | 준비 상태 (true: 준비완료, false: 준비취소) |

**Success Response:**
- **Topic**: `/topic/room/{gameRoomId}/participant`
- **Body**: `ApiResponse<ReadyStatusResponse>`

```json
{
   "success": true,
   "message": "참가자의 준비 상태가 변경되었습니다.",
   "timestamp": "2025-10-21T14:28:00.123456",
   "data": {
      "eventType": "PARTICIPANT_READY_UPDATED",
      "userId": 12345,
      "nickname": "참가자A",
      "isReady": true,
      "allReady": false
   }
}
```

**모든 참가자 준비 완료 시:**
```json
{
   "success": true,
   "message": "참가자의 준비 상태가 변경되었습니다.",
   "timestamp": "2025-10-21T14:28:30.123456",
   "data": {
      "eventType": "PARTICIPANT_READY_UPDATED",
      "userId": 12346,
      "nickname": "참가자B",
      "isReady": true,
      "allReady": true
   }
}
```

**Response Fields:**
| 필드 | 타입 | 설명 |
|------|------|------|
| `eventType` | `String` | 이벤트 타입 ("PARTICIPANT_READY_UPDATED") |
| `userId` | `Long` | 준비 상태를 변경한 사용자 ID |
| `nickname` | `String` | 사용자 닉네임 |
| `isReady` | `Boolean` | 변경된 준비 상태 |
| `allReady` | `Boolean` | 모든 참가자 준비 완료 여부 |

**Error Response:**
- **Topic**: `/user/{userId}/queue/errors`

```json
{
   "timestamp": "2025-10-21T14:28:00.123456",
   "status": 403,
   "error": "NOT_ALLOWED_READY_FOR_HOST",
   "detail": "방장은 준비 상태를 변경할 수 없습니다.",
   "path": "/app/room/participant/ready"
}
```

**Error Cases:**
- `NOT_ALLOWED_READY_FOR_HOST`: 방장이 준비 상태 변경 시도
- `PARTICIPANT_NOT_IN_ROOM`: 방에 참여하지 않은 사용자
- `ROOM_NOT_FOUND`: 존재하지 않는 방

---

### 2.3 게임 시작
**SEND** `/app/room/{gameRoomId}/quiz/start`

방장이 퀴즈 게임을 시작합니다.

**Path Parameters:**
- `gameRoomId` (Long, required): 게임방 ID

**Request Body:** 없음

**Success Response:**
- **Topic**: `/topic/room/{gameRoomId}/quiz/start`
- **Body**: `ApiResponse<GameStartResponse>`

```json
{
   "success": true,
   "message": "게임이 시작되었습니다.",
   "timestamp": "2025-10-21T14:30:00.123456",
   "data": {
      "totalQuestions": 10
   }
}
```

**Error Response:**
- **Topic**: `/user/{userId}/queue/errors`
- **Body**: `ApiResponse<null>`

```json
{
   "success": false,
   "message": "방장 권한이 없습니다.",
   "timestamp": "2025-10-21T14:30:00.123456",
   "data": null
}
```

**Error Cases:**
- `NOT_ROOM_HOST`: 방장이 아닌 사용자가 시작 시도
- `ROOM_MIN_PARTICIPANTS_NOT_MET`: 최소 인원 부족 (2명 미만)
- `GAME_STILL_IN_PROGRESS`: 이미 게임이 진행 중
- `ROOM_NOT_FOUND`: 존재하지 않는 방

---

### 1.2 방으로 돌아가기
**SEND** `/app/room/{gameRoomId}/quiz/return`

게임 종료 후 대기실로 돌아갑니다.

**Path Parameters:**
- `gameRoomId` (Long, required): 게임방 ID

**Request Body:** 없음

**Success Response:**
- **Topic**: `/topic/room/{gameRoomId}/quiz/return`
- **Body**: `ApiResponse<String>`

```json
{
   "success": true,
   "message": "대기실로 돌아갑니다.",
   "timestamp": "2025-10-21T14:35:00.123456",
   "data": "RETURN_TO_WAITING"
}
```

**Error Response:**
- **Topic**: `/user/{userId}/queue/errors`

```json
{
   "success": false,
   "message": "게임이 진행 중이 아닙니다.",
   "timestamp": "2025-10-21T14:35:00.123456",
   "data": null
}
```

**Error Cases:**
- `GAME_NOT_IN_PROGRESS`: 게임이 진행 중이 아님
- `PARTICIPANT_NOT_IN_ROOM`: 방에 참여하지 않은 사용자

---

## 2. Quiz Game Flow API

### 2.1 정답 도전 신청
**SEND** `/app/room/{gameRoomId}/quiz/challenge`

문제에 대한 정답 도전을 신청합니다. (선착순 4명)

**Path Parameters:**
- `gameRoomId` (Long, required): 게임방 ID

**Request Body:**
```json
{
   "questionNumber": 1
}
```

**Request Fields:**
| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `questionNumber` | `Integer` | ✅ | 도전할 문제 번호 |

**Success Response:**
- **Topic**: `/topic/room/{gameRoomId}/quiz/challenge`
- **Body**: `ApiResponse<NextChallengerResponse>`

```json
{
   "success": true,
   "message": "도전자가 등록되었습니다.",
   "timestamp": "2025-10-21T14:32:00.123456",
   "data": {
      "userId": 12345,
      "questionNumber": 1
   }
}
```

**Error Response:**
- **Topic**: `/user/{userId}/queue/errors`

```json
{
   "success": false,
   "message": "유효하지 않은 문제 번호입니다.",
   "timestamp": "2025-10-21T14:32:00.123456",
   "data": null
}
```

**Error Cases:**
- `INVALID_QUESTION_NUMBER`: 유효하지 않은 문제 번호
- `GAME_NOT_IN_PROGRESS`: 게임이 진행 중이 아님
- `PARTICIPANT_NOT_IN_ROOM`: 방에 참여하지 않은 사용자

---

### 2.2 정답 제출
**SEND** `/app/room/{gameRoomId}/quiz/answer`

AI 모델이 인식한 수어 단어를 정답으로 제출합니다.

**Path Parameters:**
- `gameRoomId` (Long, required): 게임방 ID

**Request Body:**
```json
{
   "questionNumber": 1,
   "userAnswer": "안녕하세요"
}
```

**Request Fields:**
| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `questionNumber` | `Integer` | ✅ | 문제 번호 |
| `userAnswer` | `String` | ✅ | FastAPI에서 받은 AI 추론 결과 |

**Success Response:**
- **Topic**: `/topic/room/{gameRoomId}/quiz/answer`
- **Body**: `ApiResponse<AnswerResultResponse>`

```json
{
   "success": true,
   "message": "정답입니다!",
   "timestamp": "2025-10-21T14:33:00.123456",
   "data": {
      "userId": 12345,
      "isCorrect": true,
      "score": 100,
      "totalScore": 250
   }
}
```

**오답 시 Response:**
```json
{
   "success": true,
   "message": "오답입니다.",
   "timestamp": "2025-10-21T14:33:00.123456",
   "data": {
      "userId": 12345,
      "isCorrect": false,
      "score": -50,
      "totalScore": 150
   }
}
```

**Error Response:**
- **Topic**: `/user/{userId}/queue/errors`

```json
{
   "success": false,
   "message": "유효하지 않은 문제 번호입니다.",
   "timestamp": "2025-10-21T14:33:00.123456",
   "data": null
}
```

**Error Cases:**
- `INVALID_QUESTION_NUMBER`: 유효하지 않은 문제 번호
- `GAME_NOT_IN_PROGRESS`: 게임이 진행 중이 아님
- `PARTICIPANT_NOT_IN_ROOM`: 방에 참여하지 않은 사용자

---

## 4. Real-time Notifications API

### 4.1 문제 출제 알림
**SUBSCRIBE** `/topic/room/{gameRoomId}/quiz/question`

새로운 문제가 출제될 때 받는 알림입니다.

**Response Body:**
```json
{
   "success": true,
   "message": "새 문제가 출제되었습니다.",
   "timestamp": "2025-10-21T14:31:00.123456",
   "data": {
      "questionNumber": 1,
      "totalQuestions": 10,
      "wordTitle": "안녕하세요"
   }
}
```

**Response Fields:**
| 필드 | 타입 | 설명 |
|------|------|------|
| `questionNumber` | `Integer` | 현재 문제 번호 |
| `totalQuestions` | `Integer` | 전체 문제 수 |
| `wordTitle` | `String` | 수어로 표현해야 할 단어 |

---

### 4.2 게임 종료 알림
**SUBSCRIBE** `/topic/room/{gameRoomId}/quiz/result`

게임이 종료되고 최종 순위가 발표될 때 받는 알림입니다.

**Response Body:**
```json
{
   "success": true,
   "message": "게임이 종료되었습니다.",
   "timestamp": "2025-10-21T14:40:00.123456",
   "data": {
      "eventType": "QUIZ_FINISHED",
      "completedRound": 1,
      "nextRound": 2,
      "rankings": [
         {
            "rank": 1,
            "userId": 12345,
            "nickname": "사용자A",
            "profileImageUrl": "https://example.com/profile1.jpg",
            "score": 450
         },
         {
            "rank": 2,
            "userId": 12346,
            "nickname": "사용자B",
            "profileImageUrl": "https://example.com/profile2.jpg",
            "score": 380
         },
         {
            "rank": 3,
            "userId": 12347,
            "nickname": "사용자C",
            "profileImageUrl": null,
            "score": 250
         }
      ],
      "showReturnButton": true
   }
}
```

**Response Fields:**
| 필드 | 타입 | 설명 |
|------|------|------|
| `eventType` | `String` | 이벤트 타입 ("QUIZ_FINISHED") |
| `completedRound` | `Integer` | 완료된 라운드 번호 |
| `nextRound` | `Integer` | 다음 라운드 번호 |
| `rankings` | `Array<RankingInfo>` | 순위 정보 배열 |
| `showReturnButton` | `Boolean` | "방으로 돌아가기" 버튼 표시 여부 |

**RankingInfo Fields:**
| 필드 | 타입 | 설명 |
|------|------|------|
| `rank` | `Integer` | 순위 |
| `userId` | `Long` | 사용자 ID |
| `nickname` | `String` | 닉네임 |
| `profileImageUrl` | `String` | 프로필 이미지 URL (null 가능) |
| `score` | `Integer` | 점수 |

---

### 4.3 참여자 변경 알림
**SUBSCRIBE** `/topic/room/{gameRoomId}/participants`

방에 참여자가 입장하거나 퇴장할 때 받는 알림입니다.

**Response Body:**
```json
{
   "success": true,
   "message": "참여자가 변경되었습니다.",
   "timestamp": "2025-10-21T14:25:00.123456",
   "data": {
      "eventType": "PARTICIPANT_JOINED",
      "userId": 12348,
      "nickname": "새로운사용자",
      "currentParticipants": 3,
      "maxParticipants": 4
   }
}
```

**Response Fields:**
| 필드 | 타입 | 설명 |
|------|------|------|
| `eventType` | `String` | 이벤트 타입 ("PARTICIPANT_JOINED", "PARTICIPANT_LEFT") |
| `userId` | `Long` | 변경된 사용자 ID |
| `nickname` | `String` | 사용자 닉네임 |
| `currentParticipants` | `Integer` | 현재 참여자 수 |
| `maxParticipants` | `Integer` | 최대 참여자 수 |

---

### 4.4 개인 에러 알림
**SUBSCRIBE** `/user/{userId}/queue/errors`

개인에게 발생한 에러를 받는 알림입니다.

**Response Body:**
```json
{
   "success": false,
   "message": "방장 권한이 없습니다.",
   "timestamp": "2025-10-21T14:30:00.123456",
   "data": null
}
```

---

## 5. 전체 게임 플로우 시나리오

### 5.1 방 입장부터 게임 시작까지

1. **WebSocket 연결**
   - 클라이언트: WebSocket Secure 연결 (`wss://localhost:9000/ws`)
   - 서버: JWT Secure 쿠키 검증 및 세션 등록

2. **방 입장**
   - 클라이언트: `SEND /app/room/{roomId}/join`
   - 입장자: `RECEIVE /user/queue/room` (JoinRoomResponse)
   - 전체: `RECEIVE /topic/room/{roomId}/participant` (PARTICIPANT_JOINED)

3. **준비 상태 변경**
   - 참가자: `SEND /app/room/{roomId}/participant/ready`
   - 전체: `RECEIVE /topic/room/{roomId}/participant` (PARTICIPANT_READY_UPDATED)

4. **게임 시작**
   - 방장: `SEND /app/room/{roomId}/quiz/start`
   - 전체: `RECEIVE /topic/room/{roomId}/quiz/start` (GameStartResponse)

### 5.2 정상적인 게임 진행 시나리오

### 4.1 정상적인 게임 진행 시나리오

1. **게임 시작**
   - 방장: `SEND /app/room/{roomId}/quiz/start`
   - 전체: `RECEIVE /topic/room/{roomId}/quiz/start` (GameStartResponse)

2. **문제 출제**
   - 전체: `RECEIVE /topic/room/{roomId}/quiz/question` (NextQuestionResponse)

3. **도전 신청**
   - 참여자: `SEND /app/room/{roomId}/quiz/challenge`
   - 전체: `RECEIVE /topic/room/{roomId}/quiz/challenge` (NextChallengerResponse)

4. **정답 제출**
   - 도전자: `SEND /app/room/{roomId}/quiz/answer`
   - 전체: `RECEIVE /topic/room/{roomId}/quiz/answer` (AnswerResultResponse)

5. **다음 문제** (2-4 반복)

6. **게임 종료**
   - 전체: `RECEIVE /topic/room/{roomId}/quiz/result` (GameEndResponse)

7. **방으로 돌아가기**
   - 참여자: `SEND /app/room/{roomId}/quiz/return`
   - 전체: `RECEIVE /topic/room/{roomId}/quiz/return`

### 4.2 에러 처리 시나리오

1. **권한 없는 게임 시작**
   - 일반 참여자: `SEND /app/room/{roomId}/quiz/start`
   - 해당 사용자: `RECEIVE /user/{userId}/queue/errors` (NOT_ROOM_HOST)

2. **잘못된 문제 번호로 도전**
   - 참여자: `SEND /app/room/{roomId}/quiz/challenge` (잘못된 questionNumber)
   - 해당 사용자: `RECEIVE /user/{userId}/queue/errors` (INVALID_QUESTION_NUMBER)

---

## 5. 점수 시스템

### 5.1 점수 계산 규칙

**정답 시 점수 (순위별 차등):**
- 1순위: +100점
- 2순위: +90점
- 3순위: +80점
- 4순위: +70점

**오답 시 점수:**
- 모든 순위: -50점

### 5.2 순위 결정 기준

1. **1차**: 최종 누적 점수 기준
2. **2차**: 동점 시 먼저 정답을 맞춘 횟수 우선

---

## 6. 에러 코드 정의

### 6.1 인증 관련 에러
- `UNAUTHORIZED`: 인증되지 않은 사용자
- `INVALID_TOKEN`: 유효하지 않은 토큰
- `EXPIRED_TOKEN`: 만료된 토큰

### 6.2 방 관련 에러
- `ROOM_NOT_FOUND`: 퀴즈 방을 찾을 수 없음
- `ROOM_FULL`: 방 인원이 가득 참
- `ROOM_ALREADY_STARTED`: 이미 시작된 방
- `ROOM_MIN_PARTICIPANTS_NOT_MET`: 최소 인원 부족

### 6.3 참여자 관련 에러
- `PARTICIPANT_NOT_FOUND`: 참가자를 찾을 수 없음
- `PARTICIPANT_NOT_IN_ROOM`: 방에 참여하지 않은 사용자
- `NOT_ROOM_HOST`: 방장 권한이 없음

### 6.4 퀴즈 관련 에러
- `GAME_NOT_IN_PROGRESS`: 게임이 진행 중이 아님
- `GAME_STILL_IN_PROGRESS`: 게임이 아직 진행 중
- `INVALID_QUESTION_NUMBER`: 유효하지 않은 문제 번호

### 6.5 AI/미디어 관련 에러
- `AI_MODEL_ERROR`: AI 모델 처리 중 오류
- `AI_MODEL_TIMEOUT`: AI 모델 응답 시간 초과
- `CAMERA_PERMISSION_REQUIRED`: 카메라 권한이 필요

---

## 7. 기술적 요구사항

### 7.1 연결 요구사항
- **WebSocket 프로토콜**: STOMP over WSS (WebSocket Secure)
- **인증**: HTTPS 환경에서 Secure HTTP-Only 쿠키 기반 JWT
- **재연결**: 자동 재연결 지원 필요
- **하트비트**: 30초 간격 권장

### 7.2 성능 요구사항
- **연결 시간**: 1초 이내
- **메시지 전송**: 100ms 이내
- **동시 접속**: 방당 최대 4명
- **메시지 크기**: 최대 1MB

### 7.3 보안 요구사항
- **HTTPS/WSS**: 모든 통신 암호화 필수
- **CORS**: `https://localhost:5173` 허용
- **Secure Cookie**: `Secure`, `HttpOnly`, `SameSite=Strict` 플래그 필수
- **토큰 검증**: 모든 연결 시 JWT 검증
- **세션 관리**: 중복 접속 방지
- **메시지 검증**: 모든 STOMP 메시지 유효성 검사

---

## 8. 변경 이력

| 버전     | 날짜         | 변경 내용                 | 작성자 |
|--------|------------|----------------------|-----|
| v1.0.0 | 2025.10.21 | 초기 WebSocket API 명세서 작성 | 고동현 |
| v1.0.1 | 2025.10.21 | HTTPS/WSS 보안 요구사항 반영 | 고동현 |
1. **게임 시작**
   - 방장: `SEND /app/room/{roomId}/quiz/start`
   - 전체: `RECEIVE /topic/room/{roomId}/quiz/start` (GameStartResponse)

2. **문제 출제**
   - 전체: `RECEIVE /topic/room/{roomId}/quiz/question` (NextQuestionResponse)

3. **도전 신청**
   - 참여자: `SEND /app/room/{roomId}/quiz/challenge`
   - 전체: `RECEIVE /topic/room/{roomId}/quiz/challenge` (NextChallengerResponse)

4. **정답 제출**
   - 도전자: `SEND /app/room/{roomId}/quiz/answer`
   - 전체: `RECEIVE /topic/room/{roomId}/quiz/answer` (AnswerResultResponse)

5. **다음 문제** (2-4 반복)

6. **게임 종료**
   - 전체: `RECEIVE /topic/room/{roomId}/quiz/result` (GameEndResponse)

7. **방으로 돌아가기**
   - 참여자: `SEND /app/room/{roomId}/quiz/return`
   - 전체: `RECEIVE /topic/room/{roomId}/quiz/return`

### 5.3 연결 해제 및 퇴장 시나리오

1. **일반 참가자 연결 해제**
   - 자동 처리: 방에서 퇴장
   - 전체: `RECEIVE /topic/room/{roomId}/participant` (PARTICIPANT_LEFT)

2. **방장 연결 해제**
   - 자동 처리: 방 종료
   - 전체: `RECEIVE /topic/room/{roomId}/participant` (ROOM_CLOSED)
   - 개인: `RECEIVE /user/{userId}/queue/room-closed` (방 종료 알림)

### 5.4 에러 처리 시나리오

1. **권한 없는 게임 시작**
   - 일반 참여자: `SEND /app/room/{roomId}/quiz/start`
   - 해당 사용자: `RECEIVE /user/{userId}/queue/errors` (NOT_ROOM_HOST)

2. **방장의 준비 상태 변경 시도**
   - 방장: `SEND /app/room/{roomId}/participant/ready`
   - 해당 사용자: `RECEIVE /user/{userId}/queue/errors` (NOT_ALLOWED_READY_FOR_HOST)

3. **가득 찬 방 입장 시도**
   - 참가자: `SEND /app/room/{roomId}/join`
   - 해당 사용자: `RECEIVE /user/{userId}/queue/errors` (ROOM_FULL)

---

## 6. 구독해야 할 토픽 목록

### 6.1 필수 구독 토픽

**개인 메시지 (모든 사용자)**
```typescript
// 방 입장 성공 시 방 정보
stompClient.subscribe('/user/queue/room', callback);

// 개인 에러 메시지
stompClient.subscribe('/user/queue/errors', callback);

// 방 종료 알림 (방장 퇴장 시)
stompClient.subscribe('/user/queue/room-closed', callback);
```

**방 관련 메시지 (방 입장 후)**
```typescript
// 참가자 변경 (입장/퇴장/준비상태)
stompClient.subscribe('/topic/room/' + roomId + '/participant', callback);
```

**게임 관련 메시지 (게임 시작 후)**
```typescript
// 게임 시작 알림
stompClient.subscribe('/topic/room/' + roomId + '/quiz/start', callback);

// 문제 출제
stompClient.subscribe('/topic/room/' + roomId + '/quiz/question', callback);

// 도전자 현황
stompClient.subscribe('/topic/room/' + roomId + '/quiz/challenge', callback);

// 정답 결과
stompClient.subscribe('/topic/room/' + roomId + '/quiz/answer', callback);

// 게임 종료 및 순위
stompClient.subscribe('/topic/room/' + roomId + '/quiz/result', callback);

// 방으로 돌아가기
stompClient.subscribe('/topic/room/' + roomId + '/quiz/return', callback);
```

### 6.2 구독 해제 시점

**방 퇴장 시**
```typescript
// 방 관련 모든 토픽 구독 해제
stompClient.unsubscribe('/topic/room/' + roomId + '/participant');
stompClient.unsubscribe('/topic/room/' + roomId + '/quiz/*');
```

**WebSocket 연결 해제 시**
```typescript
// 모든 구독 해제
stompClient.disconnect();
```

---

## 7. React + TypeScript 클라이언트 구현 가이드

**필수 라이브러리 설치**
```bash
npm install @stomp/stompjs sockjs-client axios
npm install -D @types/sockjs-client
```

**WebSocket 서비스 클래스 (HTTPS/WSS 환경)**
```typescript
// services/WebSocketService.ts
import { Client } from '@stomp/stompjs';
import SockJS from 'sockjs-client';

export class WebSocketService {
  private client: Client | null = null;
  private roomId: number | null = null;

  connect(): Promise<void> {
    return new Promise((resolve, reject) => {
      this.client = new Client({
        // HTTPS 환경에서 WSS 연결
        webSocketFactory: () => new SockJS('https://localhost:9000/ws'),
        connectHeaders: {},
        debug: (str) => console.log('STOMP: ' + str),
        reconnectDelay: 5000,
        heartbeatIncoming: 4000,
        heartbeatOutgoing: 4000,
      });

      this.client.onConnect = (frame) => {
        console.log('WSS Connected:', frame);
        this.subscribeToPersonalMessages();
        resolve();
      };

      this.client.onStompError = (frame) => {
        console.error('STOMP Error:', frame);
        reject(new Error('WebSocket Secure connection failed'));
      };

      this.client.activate();
    });
  }

  // ... 나머지 메서드들
}
```

**Axios 설정 (HTTPS 환경)**
```typescript
// api/axiosConfig.ts
import axios from 'axios';

const apiClient = axios.create({
  baseURL: 'https://localhost:9000', // HTTPS로 변경
  withCredentials: true, // Secure 쿠키 자동 포함
  timeout: 10000,
});

// 응답 인터셉터 (에러 처리)
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      // 인증 실패 시 로그인 페이지로 리다이렉트
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default apiClient;
```

---

## 8. 점수 시스템

### 8.1 점수 계산 규칙

**정답 시 점수 (순위별 차등):**
- 1순위: +100점
- 2순위: +90점
- 3순위: +80점
- 4순위: +70점

**오답 시 점수:**
- 모든 순위: -50점

### 8.2 순위 결정 기준

1. **1차**: 최종 누적 점수 기준
2. **2차**: 동점 시 먼저 정답을 맞춘 횟수 우선

---

## 9. 에러 코드 정의

### 9.1 인증 관련 에러
- `UNAUTHORIZED`: 인증되지 않은 사용자
- `INVALID_TOKEN`: 유효하지 않은 토큰
- `EXPIRED_TOKEN`: 만료된 토큰

### 9.2 방 관련 에러
- `ROOM_NOT_FOUND`: 퀴즈 방을 찾을 수 없음
- `ROOM_FULL`: 방 인원이 가득 참 (4명)
- `ROOM_ALREADY_STARTED`: 이미 시작된 방
- `ROOM_MIN_PARTICIPANTS_NOT_MET`: 최소 인원 부족 (2명 미만)

### 9.3 참여자 관련 에러
- `PARTICIPANT_NOT_FOUND`: 참가자를 찾을 수 없음
- `PARTICIPANT_NOT_IN_ROOM`: 방에 참여하지 않은 사용자
- `PARTICIPANT_ALREADY_IN_ROOM`: 이미 방에 참여 중
- `NOT_ROOM_HOST`: 방장 권한이 없음
- `NOT_ALLOWED_READY_FOR_HOST`: 방장은 준비 상태 변경 불가

### 9.4 퀴즈 관련 에러
- `GAME_NOT_IN_PROGRESS`: 게임이 진행 중이 아님
- `GAME_STILL_IN_PROGRESS`: 게임이 아직 진행 중
- `INVALID_QUESTION_NUMBER`: 유효하지 않은 문제 번호

### 9.5 AI/미디어 관련 에러
- `AI_MODEL_ERROR`: AI 모델 처리 중 오류
- `AI_MODEL_TIMEOUT`: AI 모델 응답 시간 초과
- `CAMERA_PERMISSION_REQUIRED`: 카메라 권한이 필요

---

## 10. 기술적 요구사항

### 10.1 연결 요구사항
- **WebSocket 프로토콜**: STOMP over WSS (WebSocket Secure)
- **인증**: HTTPS 환경에서 Secure HTTP-Only 쿠키 기반 JWT
- **재연결**: 자동 재연결 지원 필요
- **하트비트**: 30초 간격 권장

### 10.2 성능 요구사항
- **연결 시간**: 1초 이내
- **메시지 전송**: 100ms 이내
- **동시 접속**: 방당 최대 4명
- **메시지 크기**: 최대 1MB

### 10.3 보안 요구사항
- **HTTPS/WSS**: 모든 통신 암호화 필수
- **CORS**: `https://localhost:5173` 허용
- **Secure Cookie**: `Secure`, `HttpOnly`, `SameSite=Strict` 플래그 필수
- **토큰 검증**: 모든 연결 시 JWT 검증
- **세션 관리**: 중복 접속 방지
- **메시지 검증**: 모든 STOMP 메시지 유효성 검사

---

## 11. 변경 이력

| 버전     | 날짜         | 변경 내용                 | 작성자 |
|--------|------------|----------------------|-----|
| v1.0.0 | 2025.10.21 | 초기 WebSocket API 명세서 작성 | 고동현 |
| v1.0.1 | 2025.10.21 | HTTPS/WSS 보안 요구사항 반영 | 고동현 |