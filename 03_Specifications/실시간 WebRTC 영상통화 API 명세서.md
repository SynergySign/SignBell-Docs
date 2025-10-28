# **실시간 WebRTC 영상통화 명세서 - SignBell**

본 문서는 실시간 수어 학습 플랫폼 'SignBell'의 WebRTC 기반 영상통화 기능을 지원하는 Janus Gateway 연동 명세서입니다.
이 문서는 개발팀, 기획팀, 디자인팀 등 모든 프로젝트 구성원이 WebRTC 기반 실시간 영상통화 규약을 명확하게 이해하고, 일관된 사용자 경험을 구현하는 것을 목표로 합니다.
각 팀은 본 명세서를 기준으로 기능 개발, 화면 설계, 테스트 시나리오 작성을 진행하며, 이를 통해 효율적인 협업과 안정적인 서비스 구축을 지향합니다.

* **작성자:** [고동현](https://github.com/rhehdgus8831)
* **작성일**: 2025-10-28
* **최종 수정일**: 2025-10-28
* **문서 버전:** v1.0.0

**대상 독자:**

* **기획자 / PM**: 사용자 여정과 서비스 흐름을 이해하고 기획 방향성을 검증하는 담당자
* **백엔드 개발자**: Janus Gateway 서버 구축, 방 관리, 사용자 인증 등을 구현·유지보수하는 개발자
* **프론트엔드 개발자**: WebRTC 연결, 영상 스트림 처리, Janus API 연동을 설계·구현하는 개발자
* **디자이너 (UX/UI)**: 사용자 흐름에 기반한 화면 설계, 인터랙션 디자인, 사용자 경험을 구체화하는 담당자
* **DevOps / 인프라 엔지니어**: Janus Gateway 서버 구축, TURN/STUN 서버 설정, 네트워크 최적화를 관리하는 담당자
* **QA / 테스트 엔지니어**: 사용자 시나리오별 테스트 케이스 작성, 영상 품질 검증, 다중 사용자 환경 테스트를 수행하는 담당자
* **신규 합류자**: SignBell 서비스의 WebRTC 영상통화 기능을 빠르게 파악해야 하는 팀 신규 인원

---

| **Janus Gateway URL** | `https://janus.jsflux.co.kr/janus` |
| **프로토콜** | WebRTC (SDP/ICE) |
| **플러그인** | janus.plugin.videoroom |
| **응답 형식** | JSON |

## 인증

Janus Gateway는 별도의 인증 없이 연결 가능하지만, 방 입장 시 사용자 ID를 `display` 필드로 전달하여 식별합니다.

---

## WebRTC 연결 구조

### Janus Gateway 아키텍처
- **서버**: Janus Gateway (SFU - Selective Forwarding Unit)
- **플러그인**: VideoRoom Plugin
- **역할**:
    - **Publisher**: 자신의 영상/음성을 방에 송출
    - **Subscriber**: 다른 참가자의 영상/음성을 수신

### 연결 흐름
```
1. Janus 세션 생성
2. VideoRoom 플러그인 연결 (Attach)
3. 방 생성 또는 입장 (Join as Publisher)
4. 로컬 스트림 Publish (Offer/Answer)
5. 다른 참가자 구독 (Subscribe as Subscriber)
6. 원격 스트림 수신
```

---

## API 태그 분류

### 1. Session Management (세션 관리)
### 2. Room Management (방 관리)
### 3. Publisher Operations (송출자 작업)
### 4. Subscriber Operations (구독자 작업)
### 5. Stream Management (스트림 관리)

---

## 1. Session Management API

### 1.1 Janus 세션 생성
**POST** `https://janus.jsflux.co.kr/janus`

Janus Gateway와의 세션을 생성합니다.

**Request Body:**
```json
{
  "janus": "create",
  "transaction": "random-string-123"
}
```

**Success Response:**
```json
{
  "janus": "success",
  "transaction": "random-string-123",
  "data": {
    "id": 1234567890
  }
}
```

**Response Fields:**
| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | `Long` | Janus 세션 ID |

---

### 1.2 플러그인 연결 (Attach)
**POST** `https://janus.jsflux.co.kr/janus/{sessionId}`

VideoRoom 플러그인에 연결합니다.

**Path Parameters:**
- `sessionId` (Long, required): Janus 세션 ID

**Request Body:**
```json
{
  "janus": "attach",
  "plugin": "janus.plugin.videoroom",
  "opaque_id": "user-12345",
  "transaction": "random-string-456"
}
```

**Request Fields:**
| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `plugin` | `String` | ✅ | 플러그인 이름 ("janus.plugin.videoroom") |
| `opaque_id` | `String` | ❌ | 사용자 식별자 (디버깅용) |

**Success Response:**
```json
{
  "janus": "success",
  "session_id": 1234567890,
  "transaction": "random-string-456",
  "data": {
    "id": 9876543210
  }
}
```

**Response Fields:**
| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | `Long` | 플러그인 핸들 ID |

---

## 2. Room Management API

### 2.1 방 생성
**POST** `https://janus.jsflux.co.kr/janus/{sessionId}/{handleId}`

VideoRoom을 생성합니다.

**Request Body:**
```json
{
  "janus": "message",
  "transaction": "random-string-789",
  "body": {
    "request": "create",
    "room": 1234,
    "description": "Game Room 1234",
    "publishers": 10,
    "bitrate": 128000,
    "fir_freq": 10,
    "audiocodec": "opus",
    "videocodec": "vp8",
    "record": false,
    "permanent": false
  }
}
```

**Request Fields:**
| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `room` | `Integer` | ✅ | 방 번호 (게임방 ID와 동일) |
| `description` | `String` | ❌ | 방 설명 |
| `publishers` | `Integer` | ❌ | 최대 송출자 수 (기본: 3) |
| `bitrate` | `Integer` | ❌ | 비트레이트 제한 (bps) |
| `fir_freq` | `Integer` | ❌ | FIR 요청 빈도 (초) |
| `audiocodec` | `String` | ❌ | 오디오 코덱 ("opus", "pcma", "pcmu") |
| `videocodec` | `String` | ❌ | 비디오 코덱 ("vp8", "vp9", "h264") |
| `record` | `Boolean` | ❌ | 녹화 여부 |
| `permanent` | `Boolean` | ❌ | 영구 방 여부 |

**Success Response:**
```json
{
  "janus": "success",
  "session_id": 1234567890,
  "transaction": "random-string-789",
  "plugindata": {
    "plugin": "janus.plugin.videoroom",
    "data": {
      "videoroom": "created",
      "room": 1234,
      "permanent": false
    }
  }
}
```

**Error Response (방이 이미 존재):**
```json
{
  "janus": "success",
  "plugindata": {
    "plugin": "janus.plugin.videoroom",
    "data": {
      "videoroom": "event",
      "error_code": 427,
      "error": "Room already exists"
    }
  }
}
```

---

### 2.2 방 입장 (Publisher)
**POST** `https://janus.jsflux.co.kr/janus/{sessionId}/{handleId}`

Publisher로 방에 입장합니다.

**Request Body:**
```json
{
  "janus": "message",
  "transaction": "random-string-101",
  "body": {
    "request": "join",
    "room": 1234,
    "ptype": "publisher",
    "display": "12345"
  }
}
```

**Request Fields:**
| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `room` | `Integer` | ✅ | 방 번호 |
| `ptype` | `String` | ✅ | 참가자 타입 ("publisher") |
| `display` | `String` | ✅ | 사용자 ID (문자열) |

**Success Response:**
```json
{
  "janus": "event",
  "session_id": 1234567890,
  "plugindata": {
    "plugin": "janus.plugin.videoroom",
    "data": {
      "videoroom": "joined",
      "room": 1234,
      "description": "Game Room 1234",
      "id": 5555555555,
      "private_id": 6666666666,
      "publishers": [
        {
          "id": 7777777777,
          "display": "12346",
          "audio_codec": "opus",
          "video_codec": "vp8"
        }
      ]
    }
  }
}
```

**Response Fields:**
| 필드 | 타입 | 설명 |
|------|------|------|
| `id` | `Long` | 내 Feed ID (Publisher ID) |
| `private_id` | `Long` | 비공개 ID (방 관리용) |
| `publishers` | `Array` | 방에 이미 있는 다른 Publisher 목록 |

---

### 2.3 방 퇴장
**POST** `https://janus.jsflux.co.kr/janus/{sessionId}/{handleId}`

방에서 퇴장합니다.

**Request Body:**
```json
{
  "janus": "message",
  "transaction": "random-string-102",
  "body": {
    "request": "leave"
  }
}
```

**Success Response:**
```json
{
  "janus": "success",
  "session_id": 1234567890,
  "transaction": "random-string-102"
}
```

---

## 3. Publisher Operations API

### 3.1 스트림 송출 설정 (Configure)
**POST** `https://janus.jsflux.co.kr/janus/{sessionId}/{handleId}`

Publisher의 오디오/비디오 송출을 설정합니다.

**Request Body:**
```json
{
  "janus": "message",
  "transaction": "random-string-103",
  "body": {
    "request": "configure",
    "audio": true,
    "video": true
  },
  "jsep": {
    "type": "offer",
    "sdp": "v=0\r\no=- ... (SDP Offer)"
  }
}
```

**Request Fields:**
| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `audio` | `Boolean` | ✅ | 오디오 송출 여부 |
| `video` | `Boolean` | ✅ | 비디오 송출 여부 |
| `jsep` | `Object` | ✅ | WebRTC SDP Offer |

**Success Response:**
```json
{
  "janus": "event",
  "session_id": 1234567890,
  "plugindata": {
    "plugin": "janus.plugin.videoroom",
    "data": {
      "videoroom": "event",
      "configured": "ok"
    }
  },
  "jsep": {
    "type": "answer",
    "sdp": "v=0\r\no=- ... (SDP Answer)"
  }
}
```

---

### 3.2 새 Publisher 입장 알림
**EVENT** `janus.plugin.videoroom`

새로운 Publisher가 방에 입장하면 자동으로 알림을 받습니다.

**Event Message:**
```json
{
  "janus": "event",
  "plugindata": {
    "plugin": "janus.plugin.videoroom",
    "data": {
      "videoroom": "event",
      "room": 1234,
      "publishers": [
        {
          "id": 8888888888,
          "display": "12347",
          "audio_codec": "opus",
          "video_codec": "vp8"
        }
      ]
    }
  }
}
```

---

### 3.3 Publisher 퇴장 알림
**EVENT** `janus.plugin.videoroom`

Publisher가 방에서 퇴장하면 자동으로 알림을 받습니다.

**Event Message:**
```json
{
  "janus": "event",
  "plugindata": {
    "plugin": "janus.plugin.videoroom",
    "data": {
      "videoroom": "event",
      "room": 1234,
      "leaving": 8888888888
    }
  }
}
```

**Response Fields:**
| 필드 | 타입 | 설명 |
|------|------|------|
| `leaving` | `Long` | 퇴장한 Publisher의 Feed ID |

---

## 4. Subscriber Operations API

### 4.1 Publisher 구독 (Subscribe)
**POST** `https://janus.jsflux.co.kr/janus/{sessionId}/{handleId}`

다른 Publisher의 스트림을 구독합니다.

**Request Body:**
```json
{
  "janus": "message",
  "transaction": "random-string-104",
  "body": {
    "request": "join",
    "room": 1234,
    "ptype": "subscriber",
    "feed": 7777777777
  }
}
```

**Request Fields:**
| 필드 | 타입 | 필수 | 설명 |
|------|------|------|------|
| `room` | `Integer` | ✅ | 방 번호 |
| `ptype` | `String` | ✅ | 참가자 타입 ("subscriber") |
| `feed` | `Long` | ✅ | 구독할 Publisher의 Feed ID |

**Success Response:**
```json
{
  "janus": "event",
  "plugindata": {
    "plugin": "janus.plugin.videoroom",
    "data": {
      "videoroom": "attached",
      "room": 1234,
      "feed": 7777777777,
      "display": "12346"
    }
  },
  "jsep": {
    "type": "offer",
    "sdp": "v=0\r\no=- ... (SDP Offer)"
  }
}
```

---

### 4.2 구독 시작 (Start)
**POST** `https://janus.jsflux.co.kr/janus/{sessionId}/{handleId}`

구독을 시작하고 스트림을 수신합니다.

**Request Body:**
```json
{
  "janus": "message",
  "transaction": "random-string-105",
  "body": {
    "request": "start",
    "room": 1234
  },
  "jsep": {
    "type": "answer",
    "sdp": "v=0\r\no=- ... (SDP Answer)"
  }
}
```

**Success Response:**
```json
{
  "janus": "event",
  "plugindata": {
    "plugin": "janus.plugin.videoroom",
    "data": {
      "videoroom": "event",
      "started": "ok"
    }
  }
}
```

---

## 5. Stream Management API

### 5.1 로컬 스트림 획득
**JavaScript API**

사용자의 웹캠과 마이크에 접근하여 로컬 스트림을 획득합니다.

```javascript
navigator.mediaDevices.getUserMedia({
  video: true,
  audio: true
})
.then(function(stream) {
  // 로컬 스트림 획득 성공
  localStream = stream;
})
.catch(function(error) {
  console.error('미디어 접근 실패:', error);
});
```

---

### 5.2 원격 스트림 수신
**EVENT** `onremotestream`

Subscriber가 원격 스트림을 수신하면 자동으로 콜백이 호출됩니다.

```javascript
pluginHandle.onremotestream = function(stream) {
  // 원격 스트림 수신
  remoteStream = stream;
  
  // 비디오 엘리먼트에 연결
  const videoElement = document.getElementById('remote-video');
  videoElement.srcObject = stream;
};
```

---

## 6. 전체 연결 플로우 시나리오

### 6.1 대기실 입장 시나리오

1. **Janus 초기화**
   ```javascript
   Janus.init({ debug: 'all', callback: initCallback });
   ```

2. **세션 생성**
   ```javascript
   janus = new Janus({
     server: 'https://janus.jsflux.co.kr/janus',
     success: sessionSuccess
   });
   ```

3. **플러그인 연결 (Publisher)**
   ```javascript
   janus.attach({
     plugin: 'janus.plugin.videoroom',
     success: pluginSuccess
   });
   ```

4. **방 생성 또는 입장**
   ```javascript
   // 방이 없으면 생성
   pluginHandle.send({
     message: { request: 'create', room: 1234 }
   });
   
   // 방 입장
   pluginHandle.send({
     message: { request: 'join', room: 1234, ptype: 'publisher', display: '12345' }
   });
   ```

5. **로컬 스트림 송출**
   ```javascript
   pluginHandle.createOffer({
     stream: localStream,
     success: function(jsep) {
       pluginHandle.send({
         message: { request: 'configure', audio: true, video: true },
         jsep: jsep
       });
     }
   });
   ```

6. **다른 참가자 구독**
   ```javascript
   // 새 플러그인 핸들 생성 (Subscriber)
   janus.attach({
     plugin: 'janus.plugin.videoroom',
     success: function(subscriberHandle) {
       subscriberHandle.send({
         message: { request: 'join', room: 1234, ptype: 'subscriber', feed: 7777777777 }
       });
     }
   });
   ```

---

### 6.2 게임 페이지 이동 시나리오

1. **Janus 연결 유지**
    - 대기실에서 게임 페이지로 이동 시 Janus 연결을 끊지 않음
    - `JanusContext`를 통해 연결 상태 공유

2. **스트림 계속 송수신**
    - Publisher와 Subscriber 연결 유지
    - 원격 스트림 계속 수신

---

### 6.3 게임 종료 후 대기실 복귀 시나리오

1. **Janus 방 떠나기**
   ```javascript
   pluginHandle.send({ message: { request: 'leave' } });
   ```

2. **플러그인 Detach**
   ```javascript
   pluginHandle.detach();
   ```

3. **Janus 세션 종료**
   ```javascript
   janus.destroy();
   ```

4. **재연결**
    - 대기실로 돌아오면 Janus 재연결
    - 새로운 세션 및 플러그인 핸들 생성

---

### 6.4 참가자 퇴장 시나리오

1. **퇴장 알림 수신**
   ```javascript
   onmessage: function(msg) {
     if (msg.leaving) {
       const leavingFeedId = msg.leaving;
       // Subscriber 정리
       if (remoteFeedsRef.current[leavingFeedId]) {
         remoteFeedsRef.current[leavingFeedId].detach();
         delete remoteFeedsRef.current[leavingFeedId];
       }
     }
   }
   ```

2. **원격 스트림 제거**
   ```javascript
   setRemoteStreams(prev => {
     const newStreams = { ...prev };
     delete newStreams[userId];
     return newStreams;
   });
   ```

---

## 7. React + Janus.js 클라이언트 구현 가이드

### 7.1 필수 라이브러리 설치

```bash
# Janus.js는 CDN으로 로드
# index.html에 추가:
<script src="https://cdn.jsdelivr.net/npm/janus-gateway@1.2.0/html/janus.js"></script>
```

### 7.2 JanusContext 구현

```javascript
// contexts/JanusContext.jsx
import { createContext, useContext, useRef, useState } from 'react';

const JanusContext = createContext(null);

export const JanusProvider = ({ children }) => {
  const janusRef = useRef(null);
  const pluginHandleRef = useRef(null);
  const remoteFeedsRef = useRef({});
  const userIdToFeedIdRef = useRef({});
  const [remoteStreams, setRemoteStreams] = useState({});
  const [isJanusConnected, setIsJanusConnected] = useState(false);

  const value = {
    janusRef,
    pluginHandleRef,
    remoteFeedsRef,
    userIdToFeedIdRef,
    remoteStreams,
    setRemoteStreams,
    isJanusConnected,
    setIsJanusConnected,
  };

  return <JanusContext.Provider value={value}>{children}</JanusContext.Provider>;
};

export const useJanus = () => {
  const context = useContext(JanusContext);
  if (!context) {
    throw new Error('useJanus must be used within a JanusProvider');
  }
  return context;
};
```

### 7.3 Janus 연결 훅 구현

```javascript
// hooks/useWaitingRoomJanus.js
import { useEffect } from 'react';

export const useWaitingRoomJanus = ({
  roomId,
  myUserId,
  stream,
  janusRef,
  pluginHandleRef,
  setIsJanusConnected,
}) => {
  const JANUS_SERVER = 'https://janus.jsflux.co.kr/janus';

  useEffect(() => {
    if (!window.Janus || !stream) return;

    const Janus = window.Janus;

    Janus.init({
      debug: 'all',
      callback: function () {
        janusRef.current = new Janus({
          server: JANUS_SERVER,
          success: function () {
            janusRef.current.attach({
              plugin: 'janus.plugin.videoroom',
              success: function (pluginHandle) {
                pluginHandleRef.current = pluginHandle;
                
                // 방 입장
                const register = {
                  request: 'join',
                  room: parseInt(roomId),
                  ptype: 'publisher',
                  display: String(myUserId),
                };
                
                pluginHandle.send({ message: register });
              },
              onmessage: handleJanusMessage,
            });
          },
        });
      },
    });

    return () => {
      if (janusRef.current) {
        janusRef.current.destroy();
      }
    };
  }, [roomId, myUserId, stream]);
};
```

---

## 8. 에러 코드 정의

### 8.1 방 관련 에러
- `426`: 방이 존재하지 않음 (Room does not exist)
- `427`: 방이 이미 존재함 (Room already exists)
- `428`: 방이 가득 참 (Room is full)

### 8.2 참가자 관련 에러
- `429`: 참가자를 찾을 수 없음 (No such feed)
- `430`: 이미 참가 중 (Already in room)

### 8.3 스트림 관련 에러
- `431`: 미디어 협상 실패 (Media negotiation failed)
- `432`: ICE 연결 실패 (ICE connection failed)

---

## 9. 기술적 요구사항

### 9.1 연결 요구사항
- **프로토콜**: WebRTC (HTTPS 필수)
- **Janus Gateway**: v1.2.0 이상
- **브라우저**: Chrome 90+, Firefox 88+, Safari 14+
- **STUN/TURN**: Google STUN 서버 사용

### 9.2 성능 요구사항
- **연결 시간**: 3초 이내
- **비디오 해상도**: 640x480 (VGA)
- **프레임레이트**: 25fps
- **비트레이트**: 128kbps (오디오), 512kbps (비디오)
- **동시 접속**: 방당 최대 4명

### 9.3 네트워크 요구사항
- **대역폭**: 최소 1Mbps (업로드/다운로드)
- **지연시간**: 200ms 이하
- **패킷 손실률**: 5% 이하

---

## 10. 변경 이력

| 버전 | 날짜 | 변경 내용 | 작성자 |
|------|------|----------|--------|
| v1.0.0 | 2025.10.28 | 초기 WebRTC 명세서 작성<br>- Janus Gateway 연동 방식 정의<br>- Publisher/Subscriber 플로우 명시<br>- React 구현 가이드 추가 | 고동현 |
