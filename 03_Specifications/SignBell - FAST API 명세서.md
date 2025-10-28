# **SignSense - API 명세서 (FastAPI)**

본 문서는 SignBell 서비스의 실시간 수어 추론을 담당하는 \*\*'SignSense 추론 서버 (FastAPI)'\*\*의 API를 사용하는 개발자를 위한 공식 기술 명세서입니다.

이 서버는 메인 애플리케이션 서버(SignBell)와 독립적으로 동작하며, WebSocket을 통한 실시간 프레임 수신 및 AI 모델 추론을 전담합니다.

* **작성자:** [백승현](https://github.com/sirosho)
* **작성일**: 2025-10-28
* **문서 버전:** v1.0.0

**대상 독자:**

* **프론트엔드 개발자**: AI 추론 서버의 WebSocket API와 통신하고, 프레임 스트림을 전송하며, 추론 결과를 수신하는 클라이언트(웹, 앱)를 구현하는 개발자.
* **백엔드 개발자**: WebSocket 핸들러(`ws_handler`), 인증 로직(`jwt_validator`), 비동기 저장 파이프라인(`inference_pipeline`)을 구현·유지보수하는 개발자.
* **AI / 머신러닝 엔지니어**: `predictor.py`의 모델 로직, `landmark_extractor.py`의 전처리 과정을 이해하고 모델을 개선·교체하는 담당자.
* **QA / 테스트 엔지니어**: WebSocket 통신, 랜드마크 추출, 모델 추론 정확도 및 부하 테스트 시나리오를 작성하고 검증하는 담당자.

-----

## 1\. 공통 가이드

### 1.1. 기본 정보

* **Base URL (WebSocket)**: `wss://{host}/fastapi/ws/{session_id}`
* **Base URL (REST)**: `https://{host}/fastapi`
* **서버**: FastAPI (Python)
* **주요 역할**: 실시간 랜드마크 추출 및 AI 모델 추론

### 1.2. 인증 (WebSocket Handshake)

본 서버는 REST API 방식의 인증이 아닌, **WebSocket 연결 핸드셰이크** 시점에 JWT 토큰을 검증합니다.

* `ws_handler.py`는 WebSocket 연결 요청 시 다음 순서로 JWT 토큰을 탐색하여 검증합니다:
    1.  **HTTP-Only Cookie**: `settings.COOKIE_ACCESS_TOKEN_NAME` ('ACCESS\_TOKEN')에 포함된 토큰. (권장 방식)
    2.  **Authorization Header**: `Authorization: Bearer <token>` 헤더.
    3.  **Query Parameter**: `?token=<token>` 쿼리 파라미터.
* 토큰 검증(`jwt_validator.py`)에 실패하면, 서버는 HTTP `401 Unauthorized`에 해당하는 WebSocket 종료 코드(`4001` 또는 `1008` 등)로 연결을 즉시 종료합니다.

-----

## 2\. WebSocket API (핵심 기능)

실시간 퀴즈 및 학습 데이터 전송을 위한 메인 API입니다.

* **Endpoint**: `WS /fastapi/ws/{session_id}`
* **설명**: 클라이언트와 서버 간의 실시간 양방향 통신 채널. 클라이언트는 비디오 프레임(Binary)과 제어 신호(Text)를 전송하고, 서버는 처리 결과(Text)를 응답합니다.
* `{session_id}`: 클라이언트를 식별하기 위한 고유 ID (예: `quiz-session-12345`).

### 2.1. 클라이언트 ➔ 서버 (C2S) 메시지

| 타입 | 메시지 (JSON) | `quizFastApiWebSocket.js` 함수 | 설명 |
| :--- | :--- | :--- | :--- |
| **Text** | `{"type": "meta", "word_pk": ..., "word_name": ...}` | `sendMeta(wordPk, wordName)` | **(퀴즈/학습)** 추론할 단어의 메타데이터를 서버에 전송합니다.. 프레임 전송 전에 호출되어야 하며, 서버의 프레임 버퍼를 초기화합니다. |
| **Binary** | `[JPEG Blob Bytes]` | `sendFrame(blob)` | **(퀴즈/학습)** 웹캠에서 캡처된 `image/jpeg` 형식의 비디오 프레임(Blob)을 바이너리 메시지로 전송합니다.. 서버는 이 프레임들을 버퍼에 누적합니다. |
| **Text** | `{"type": "flush"}` | `sendFlush()` | **(퀴즈)** 누적된 프레임에 대한 \*\*AI 추론(퀴즈 정답)\*\*을 요청합니다. 서버는 즉시 추론을 실행하고 `inference_result`로 응답합니다. |
| **Text** | `{"type": "save_learning"}` | `sendSaveLearning()` | **(학습)** 누적된 프레임의 랜드마크 데이터를 **저장**하도록 요청합니다.. (추론은 수행하지 않음). 서버는 `learning_ack`로 응답합니다. |
| **Text** | `{"type": "ping"}` | `startHeartbeat()` (내부) | **(퀴즈)** 30초마다 전송하여 WebSocket 연결을 유지하고 비정상적인 종료를 감지합니다.. |

### 2.2. 서버 ➔ 클라이언트 (S2C) 메시지

| 타입 | 메시지 (JSON) | `ws_handler.py` 전송 시점 | 설명 |
| :--- | :--- | :--- | :--- |
| **Text** | `{"type": "meta_ack"}` | `meta` 메시지 수신 직후 | `meta` 메시지를 성공적으로 수신했음을 확인합니다.. |
| **Text** | `{"type": "inference_result", "result": {...}}` | `flush` 요청 처리 및 추론 완료 직후 | **(퀴즈)** AI 모델의 추론 결과를 반환합니다. `result` 객체에는 `predicted` (예측 단어), `score` (신뢰도), `inference_ms` (추론 시간) 등이 포함됩니다. |
| **Text** | `{"type": "learning_ack", "status": "..."}` | `save_learning` 요청 처리 직후 | **(학습)** 데이터 저장 요청 수락 여부를 반환합니다. `status`는 `"accepted"` (성공) 또는 `"failed"` (실패)가 될 수 있습니다. |
| **Text** | `{"type": "pong"}` | `ping` 메시지 수신 시 (서버 구현에 따라 다름) | **(퀴즈)** `ping`에 대한 응답으로, 연결이 활성 상태임을 확인합니다.. |
| **Text** | `{"type": "error", "message": "..."}` | 서버 내부 예외 발생 시 | 핸들러 등에서 예측하지 못한 오류가 발생했을 때 전송됩니다. |

-----

## 3\. REST API (진단 및 상태)

WebSocket 외에 서버의 상태를 확인하기 위한 보조 REST API입니다. (인증 불필요)

### 3.1. `GET /health`

* **설명**: AI 추론기(Predictor)가 성공적으로 로드되었는지 확인합니다.
* **응답 (200 OK)**:
  ```json
  {
    "ok": true
  }
  ```
    * `ok`: `true` (모델 로드 성공), `false` (모델 로드 실패)

### 3.2. `GET /model/status`

* **설명**: 로드된 AI 모델의 상세 정보를 반환합니다.
* **응답 (200 OK)**:
  ```json
  {
    "predictor_loaded": true,
    "model_path": "/app/models/cnn_bilstm_attention_model.pth",
    "device": "cuda"
  }
  ```
    * `predictor_loaded`: 모델 로드 여부
    * `model_path`: 로드된 `.pth` 파일 경로
    * `device`: 추론에 사용 중인 장치 (`cuda` 또는 `cpu`)

### 3.3. `GET /config`

* **설명**: 서버의 주요 설정값(수집 프레임 수 등)을 반환합니다.
* **응답 (200 OK)**:
  ```json
  {
    "target_frame_count": 24,
    "collection_duration_seconds": 1.0,
    "max_frames_to_collect": 1024
  }
  ```
    * `max_frames_to_collect`: `SequenceCollector`가 버퍼에 저장할 수 있는 최대 프레임 수

### 3.4. `GET /api/diagnostics/status`

* **설명**: 진단용 라우터의 상태를 확인합니다.
* **응답 (200 OK)**:
  ```json
  {
    "ok": true,
    "message": "diagnostics router alive"
  }
  ```

### 3.5. `POST /api/diagnostics/echo`

* **설명**: 스모크 테스트용 엔드포인트로, 요청 본문을 그대로 반환합니다.
* **요청 (application/json)**:
  ```json
  {
    "test_key": "test_value"
  }
  ```
* **응답 (200 OK)**:
  ```json
  {
    "echo": {
      "test_key": "test_value"
    }
  }
  ```

-----

## 4\. 변경 이력

| 버전 | 날짜 | 변경 내용 | 작성자 |
| :--- | :--- | :--- |:----|
| v1.0.0 | 2025-10-28 | 신규 작성 | [백승현](https://github.com/sirosho) |