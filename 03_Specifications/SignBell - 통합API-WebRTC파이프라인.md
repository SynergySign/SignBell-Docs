# 클라이언트-REST API-FastAPI 통합 기능 파이프라인

* **작성자:** [백승현](https://github.com/sirosho)
* **작성일**: 2025-10-10
* **문서 버전:** v1.0.0

## 1. 전역 상태 및 통신 정의

### A. 통신 채널 정의
1.  **WebRTC DataChannel (DC):** **클라이언트 ↔ FastAPI** 간의 **실시간 영상 프레임 전송** 및 **실시간 추론 결과 수신** 전용 채널. (최소 지연)
2.  **Spring REST API (HTTP POST/GET):** **클라이언트 ↔ Spring 서버** 간의 모든 명령, 사용자 상태(점수) 변경, **DB 기록**을 위한 주 통신 채널.
3.  **내부 REST API (HTTP POST):** **Spring 서버 → FastAPI 서버**로의 **저장 작업 명령**을 위한 단방향 채널.

### B. 역할 분담 명확화 (FastAPI 자율 저장)
| 서버 | 주요 책임 영역 | 핵심 역할 |
| :--- | :--- | :--- |
| **클라이언트** | UX 및 WebRTC 영상 전송/수신 | **Spring의 권한 확인 통과 후**에만 FastAPI에 연결 시도. |
| **Spring REST API** | **DB/상태 관리 및 보안 통제** | **WebRTC 연결 전 게이트키퍼 역할** 수행. **FastAPI 저장 여부 통제 대신 DB 기록 최종 책임**. |
| **FastAPI** | 고성능 엔진 역할 | **Spring의 검증을 신뢰**. 실시간 좌표 처리, 모델 추론 후 DC로 즉시 전달. **S3 저장을 자율적으로 백그라운드 처리 (MVP 단순화)**. |

---

## 2. 수어 단어 퀴즈 파이프라인

**FastAPI가 추론 후 S3 저장을 자율적으로 시작**하고, Spring은 DB 기록만 담당합니다.

| # | 단계 | 주체 | 채널 | 상세 설명 및 서버 역할 |
| :--- | :--- | :--- | :--- | :--- |
| **1** | **퀴즈 기회 요청** | 클라이언트 ↔ Spring | Spring REST API (예: `/quiz/lock`) | 사용자가 발언권 획득 요청. **Spring 서버는 사용자 인증/인가 및 퀴즈 잠금 상태를 확인 (게이트키퍼)**. |
| **2** | **WebRTC 연결 정보 반환** | Spring ↔ 클라이언트 | Spring REST API | **권한 확인 통과 시**, Spring은 FastAPI의 **WebRTC 연결 정보 및 임시 세션 ID**를 클라이언트에게 반환. |
| **3** | **DC 연결 및 전송 시작** | **클라이언트 ↔ FastAPI** | **DC** | 클라이언트는 반환 정보로 FastAPI에 DC 연결을 수립하고, 영상 프레임 전송 시작. |
| **4** | **프레임 수집 및 추론** | **FastAPI** | **DC 콜백/내부 로직** | FastAPI는 프레임 수집 후 **모델 실시간 추론** 실행. |
| **5** | **실시간 예측 결과 전달** | **FastAPI ↔ 클라이언트** | **DC** | FastAPI는 예측 결과 (`label`, `score`)를 **DC를 통해 클라이언트에 직접 전달**하여 지연 최소화. |
| **6** | **내부 데이터 저장 (자율적)** | **FastAPI** | 내부 로직 (비동기) | FastAPI는 추론 완료 후, **Spring의 추가 명령 없이** 좌표를 추출하여 **S3에 NPY/CSV 파일을 백그라운드 태스크로 자율 저장**합니다. |
| **7** | **정답 처리/DB 반영** | 클라이언트 ↔ Spring | Spring REST API (예: `/quiz/submit-answer`) | 클라이언트가 DC로 수신한 최종 `score`를 바탕으로 Spring 서버에 **DB 점수 반영 및 기록** 명령 요청. |

---

## 3. 수어 개인 학습 파이프라인

**FastAPI가 저장 명령을 받아 S3 저장을 수행**하고 Spring은 학습 기록만 업데이트합니다.

| # | 단계 | 주체 | 채널 | 상세 설명 및 서버 역할 |
| :--- | :--- | :--- | :--- | :--- |
| **1** | **학습 세션 요청** | 클라이언트 ↔ Spring | Spring REST API (예: `/learning/session`) | 클라이언트가 학습 세션 시작 요청. Spring은 **사용자 인증 및 학습 권한을 확인 (게이트키퍼)**. |
| **2** | **WebRTC 연결 정보 반환** | Spring ↔ 클라이언트 | Spring REST API | **권한 통과 시**, Spring은 FastAPI 연결 정보와 학습 단어 정보를 클라이언트에 반환. |
| **3** | **DC 연결 및 녹화 시작** | **클라이언트 ↔ FastAPI** | **DC** | 클라이언트는 FastAPI에 DC 연결을 수립하고 녹화 시작 후 영상 프레임을 전송. |
| **4** | **프레임 수집** | **FastAPI** | **DC 콜백** | `SequenceCollector`가 프레임을 버퍼링. 녹화 종료 시 수집 중단. |
| **5** | **데이터 저장 명령 요청** | 클라이언트 ↔ Spring | Spring REST API (예: `/data/collect-command`) | 녹화 종료 후, 클라이언트는 Spring 서버에 **데이터 저장 명령**을 요청합니다. |
| **6** | **작업 요청 및 저장 위임** | **Spring → FastAPI** | **내부 REST API** (예: `/fast-api/save-learning`) | Spring 서버는 FastAPI에 **데이터 저장 명령을 전달**합니다. FastAPI는 이 명령을 받아 **좌표 추출 및 S3 저장을 실행**합니다. |
| **7** | **학습 기록 업데이트** | 클라이언트 ↔ Spring | Spring REST API (예: `/learning/update-status`) | Spring 서버는 DB에 학습 시간, 진행률 등 **학습 기록을 최종적으로 반영**하여 응답. |

---

## 4. 통합 엔드포인트 역할 분담 목록

| 엔드포인트 | 방식 | 통신 주체 | 역할 | 관련 기능 |
| :--- | :--- | :--- | :--- | :--- |
| **`DC 콜백`** | `WebRTC` | 클라이언트 ↔ FastAPI | **FastAPI: 프레임 수집, 추론 후 DC로 결과 직접 전달** | 퀴즈, 학습 |
| **(예시)** `/quiz/lock` | `REST` | 클라이언트 ↔ Spring | **Spring: 퀴즈 잠금 처리 및 WebRTC 연결 정보 반환** (게이트키퍼) | 퀴즈 |
| **(예시)** `/quiz/submit-answer` | `REST` | 클라이언트 ↔ Spring | **Spring: 최종 score 기반 DB 점수 반영 및 기록** | 퀴즈 |
| **(예시)** `/learning/session` | `REST` | 클라이언트 ↔ Spring | **Spring: 학습 권한 확인 및 WebRTC 연결 정보 반환** (게이트키퍼) | 학습 |
| **(예시)** `/data/collect-command` | `REST` | 클라이언트 ↔ Spring | **Spring: 데이터 수집 명령 접수** (FastAPI 저장 요청 전달용) | 학습 |
| **(예시)** `/learning/update-status` | `REST` | 클라이언트 ↔ Spring | **Spring: 개인 학습 기록 (시간, 진행률) DB 반영** | 학습 |
| **(예시)** `/fast-api/save-quiz` | `REST` | Spring → FastAPI (내부) | **FastAPI: 추론 완료된 퀴즈 데이터 자율 저장** (FastAPI 내부 호출) | 퀴즈 |
| **(예시)** `/fast-api/save-learning` | `REST` | Spring → FastAPI (내부) | **FastAPI: 좌표 추출 후 S3/DB에 학습 데이터 저장** | 학습 |

**(주의: `(예시)`로 표시된 엔드포인트는 합의 전 임시 명칭이며, 실제 구현 시 팀원 간 합의가 필요합니다.)**



