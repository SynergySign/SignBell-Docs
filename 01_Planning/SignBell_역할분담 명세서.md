# SignBell 역할분담 명세서

본 문서는 SignBell 프로젝트의 팀원별 역할과 기여 내역을 상세히 정의하여 프로젝트 관리 및 협업 효율성을 높이기 위한 가이드를 제공합니다.

* **작성자**: [고동현](https://github.com/Donghyeon0915)
* **작성일**: 2025-10-28
* **최종 수정일**: 2025-10-28
* **문서 버전**: v1.0.0

**대상 독자:**

* **기획자 / PM**: 프로젝트 리소스 배분과 일정 관리를 위해 팀원별 역할과 책임을 파악하는 담당자
* **개발팀 전체**: 팀원 간 역할 분담을 명확히 이해하고 협업 효율성을 높이고자 하는 모든 개발자
* **신규 합류자**: SignBell 프로젝트의 팀 구성과 각 팀원의 기여 영역을 빠르게 파악해야 하는 신규 인원
* **이해관계자**: 프로젝트 진행 상황과 팀원별 기여도를 확인하고자 하는 관리자 및 외부 협력자

**관련 문서:**
* [프로젝트 개요](./SignBell_프로젝트%20개요.md): SignBell 서비스의 전체 개요 및 목표
* [기술 스택](../02_Architecture/SignBell%20-%20기술스택.md): 프로젝트에서 사용된 기술 스택 및 라이브러리
* [시스템 아키텍처](../02_Architecture/SignBell%20시스템%20아키텍처.md): 시스템 구조 및 인프라 설계
* [개발 일정 및 마일스톤](./SignBell_개발일정%20및%20마일스톤.md): 프로젝트 일정 및 주요 마일스톤

---

## 1. 개요 (Overview)

SignBell 프로젝트는 **AI 기반 실시간 수어 학습 플랫폼**으로, 5명의 팀원이 Frontend, Backend, ML/AI, Infrastructure, Documentation 영역에 걸쳐 협업하여 개발되었습니다. 본 문서는 각 팀원의 주요 역할과 기술적 기여 내역을 명확히 정리하여 프로젝트의 투명성과 협업 효율성을 높이는 것을 목표로 합니다.

---

## 2. 팀 구성 및 역할 요약 (Team Structure & R&R)

### 2.1. 팀원별 핵심/서브 역할

| 이름 | 핵심 역할 (Primary Role) | 서브 역할 (Secondary Role) | GitHub |
|------|------------------------|--------------------------|--------|
| **신동준** | Team Lead / Frontend Architect | Project Manager, UI/UX Designer, Git/Infra-Ops | [@sdj3959](https://github.com/sdj3959) |
| **고동현** | Backend Lead (Game Logic) / System Architect | Frontend Developer (WebRTC), ML Researcher (Initial), Lead Documenter | [@rhehdgus8831](https://github.com/rhehdgus8831) |
| **백승현** | AI/ML Lead / FastAPI Engineer | Full-Stack Developer (Personal Study), ML Documenter | [@sirosho](https://github.com/sirosho) |
| **송민재** | DevOps Lead / Auth Engineer (Backend) | Frontend Developer (Auth & Mypage), Infra Documenter | [@songkey06](https://github.com/songkey06) |
| **강관주** | Full-Stack Developer (Lobby & Room) | Backend Developer (Query Optimization), API Documenter, ML Researcher (Initial) | [@Kanggwanju](https://github.com/Kanggwanju) |

### 2.2. 주요 담당 영역

| 이름 | Frontend | Backend | ML/AI | Infrastructure | Documentation |
|------|----------|---------|-------|----------------|---------------|
| **신동준** | 핵심 | - | - | 핵심 | 핵심 |
| **고동현** | 주요 | 핵심 | 주요 | 보조 | 핵심 |
| **백승현** | 주요 | 주요 | 핵심 | 보조 | 주요 |
| **송민재** | 주요 | 핵심 | - | 주요 | 주요 |
| **강관주** | 핵심 | 핵심 | 보조 | 보조 | 주요 |

---

## 3. 팀원별 상세 역할 (Detailed Roles)

### 3.1. 고동현 (Backend / ML)

**주요 역할**: 실시간 퀴즈 게임의 핵심 백엔드 로직(WebSocket, 타이머)과 프론트엔드의 WebRTC (Janus) 연동을 설계하고 개발했습니다.

#### Backend (signbell-app/backend)

| 구분 | 담당 기능 | 주요 구현 내용 |
|------|----------|---------------|
| **실시간 퀴즈 엔진** | 퀴즈 게임 로직 | Spring WebSocket (`@MessageMapping`)을 이용한 실시간 퀴즈 진행 상태 관리 (게임 시작, 다음 문제, 정답 제출) |
| **게임 상태 관리** | 인메모리 캐시 | `ConcurrentHashMap`을 활용한 다중 퀴즈 방의 상태 (참가자, 도전자, 점수, 현재 문제) 동시성 제어 |
| **타이머 시스템** | 동시성 타이머 관리 | `ScheduledExecutorService` 기반 타이머 매니저 개발, 준비/사인 시간 스케줄링 및 엣지 케이스 처리 |
| **게임 진행 로직** | 턴 기반 시스템 | 도전자 순서 관리, 정답 제출 처리, 점수 계산 및 라운드 진행 로직 구현 |
| **WebSocket 메시지 처리** | STOMP 엔드포인트 | `/app/quiz/{roomId}/start`, `/app/quiz/{roomId}/answer`, `/app/quiz/{roomId}/next` 등 퀴즈 관련 WebSocket 엔드포인트 구현 |
| **브로드캐스트 시스템** | 실시간 알림 | SimpMessagingTemplate을 활용한 게임 상태 변경 실시간 브로드캐스트 (`/topic/room/{roomId}`) |
| **트랜잭션 관리** | 데이터 일관성 | 게임 종료 시 점수 업데이트, 게임 기록 저장 등 트랜잭션 처리 (QuizTransactionService) |
| **엔티티 설계** | 게임 데이터 모델 | GameRoom, GameParticipant, GameHistory, QuizWord 엔티티 설계 및 JPA 연관관계 매핑 |
| **Repository 구현** | 데이터 접근 계층 | GameRoomRepository, GameParticipantRepository, GameHistoryRepository, QuizWordRepository 구현 |
| **단위 테스트** | 테스트 코드 | QuizService 단위 테스트 작성 (게임 시작, 진행, 종료, 예외 상황 시나리오 검증) |

#### Frontend (signbell-app/frontend)

| 구분 | 담당 기능 | 주요 구현 내용 |
|------|----------|---------------|
| **WebRTC 통합** | Janus Gateway 연동 | Janus JavaScript 라이브러리를 활용한 WebRTC 연결 설정 및 VideoRoom 플러그인 통합 |
| **JanusContext** | 전역 상태 관리 | React Context API를 활용한 Janus 연결 전역 관리 및 대기방-게임방 간 연결 유지 |
| **비디오 스트림 관리** | 미디어 제어 | pluginHandle 생성, 로컬/원격 비디오 스트림 관리, offer/answer SDP 처리 |
| **퀴즈 게임 페이지** | 게임 진행 UI | 문제 표시, 도전자 턴 표시, 타이머 UI, 정답 피드백 애니메이션 구현 |
| **게임 상태 훅** | useQuizGame | 퀴즈 게임 상태 관리 커스텀 훅 (현재 문제, 도전자, 점수, 라운드 등) |
| **WebSocket 통신** | 게임 이벤트 처리 | STOMP를 통한 게임 시작, 정답 제출, 다음 라운드 메시지 송수신 |
| **참가자 비디오 그리드** | 멀티 비디오 레이아웃 | 4명의 참가자 비디오를 그리드 형태로 배치 및 도전자 강조 표시 |
| **FastAPI WebSocket** | 수어 인식 연동 | FastAPI 서버와 WebSocket 연결하여 실시간 수어 인식 결과 수신 및 정답 처리 |
| **게임 결과 모달** | 최종 결과 표시 | 게임 종료 시 순위, 점수, 우승자 표시 모달 컴포넌트 구현 |

#### ML (signbell-ml)

| 구분 | 담당 기능 | 주요 구현 내용 |
|------|----------|---------------|
| **모델 아키텍처** | 수어 인식 모델 설계 | PyTorch 기반 CNN + BiLSTM + Attention 조합의 수어 인식 모델 초기 아키텍처 설계 |
| **데이터 전처리** | 랜드마크 추출 | MediaPipe를 활용한 손/얼굴/포즈 랜드마크 추출 및 npy 파일 변환 파이프라인 구축 |

#### Documentation (signbell-docs)

| 구분 | 담당 문서 | 주요 내용 |
|------|----------|----------|
| **아키텍처** | 시스템 설계 | MSA 구조 시스템 아키텍처 다이어그램 설계 및 문서 작성 |
| **데이터베이스** | ERD 및 스키마 | 개념/논리 ERD 및 데이터베이스 스키마 가이드 명세서 (DDL) 작성 |
| **API 명세** | WebSocket API | 실시간 수어 퀴즈 WebSocket API 명세서 작성 |
| **프로젝트 관리** | 일정 및 로드맵 | 개발 일정 및 마일스톤, 프로젝트 로드맵 명세서 작성 |
| **회의록** | 팀 커뮤니케이션 | 프로젝트 전반의 팀 회의록 작성 및 관리 |

---

### 3.2. 강관주 (Fullstack)

**주요 역할**: 퀴즈 게임방의 생성, 조회(무한 스크롤), 입장/퇴장/준비 로직을 FE/BE 전반에 걸쳐 개발했습니다.

#### Frontend (signbell-app/frontend)

| 구분 | 담당 기능 | 주요 구현 내용 |
|------|----------|---------------|
| **대기방 UI** | 퀴즈 대기방 | 대기방 메인 UI 및 참가자 상태 (방장, 준비 완료) 표시 로직 개발 |
| **WebSocket 연동** | 실시간 상태 동기화 | Stomp.js 기반 WebSocket 연동, 참가자 입장/퇴장/준비 상태 실시간 동기화 |
| **게임방 목록** | 사이드바 및 무한 스크롤 | axios 연동 게임 방 목록 조회 UI, IntersectionObserver API를 활용한 무한 스크롤 구현 |
| **모달 UI** | 방 생성/검색 | 방 생성 및 방 번호 입력 검색 모달 UI 개발 |
| **방 카드 컴포넌트** | 방 정보 표시 | 게임방 정보 카드 컴포넌트 (방 제목, 인원, 상태) |

#### Backend (signbell-app/backend)

| 구분 | 담당 기능 | 주요 구현 내용 |
|------|----------|---------------|
| **방 목록 조회** | 무한 스크롤 API | Querydsl 및 Slice 페이징을 활용한 게임 방 목록 무한 스크롤 조회 API |
| **방 생성/입장** | REST API | 게임 방 생성 및 고유 방 코드 생성, 방 입장 및 번호 입장 API 개발 |
| **전역 예외 처리** | GlobalExceptionHandler | @RestControllerAdvice를 활용한 전역 예외 처리 및 일관된 에러 응답 구조 구현 |
| **에러 코드 관리** | ErrorCode Enum | 비즈니스 예외 코드 정의 및 HTTP 상태 코드 매핑 (방 없음, 방 꽉 참, 게임 중 등) |
| **API 응답 표준화** | ApiResponse | 성공/실패 응답을 위한 공통 API 응답 DTO 설계 및 적용 |
| **비즈니스 예외** | BusinessException | 커스텀 비즈니스 예외 클래스 구현 및 ErrorCode 기반 예외 처리 |
| **참가자 관리** | 퇴장/준비 상태 | WebSocket DISCONNECT 이벤트 기반 퇴장 처리, 방장 퇴장 시 방 삭제, 준비 상태 변경 API |
| **Querydsl 구현** | 동적 쿼리 | GameRoomRepositoryCustom 인터페이스 및 구현체 작성                       |
| **WebSocket 세션 관리** | 세션 추적 | WebSocket 연결/해제 이벤트 리스너 및 세션 서비스 구현                           |
| **테스트 코드** | 통합 테스트 | 방 생성, 입장, 준비 상태 변경 통합 테스트 작성                                  |

#### ML (signbell-ml)

| 구분 | 담당 기능 | 주요 구현 내용 |
|------|----------|---------------|
| **모델 학습 지원** | 데이터셋 구축 | AI 모델 학습용 수어 동작 비디오 데이터 촬영 및 제공 |
| **모델 검증** | 테스트 및 피드백 | 초기 모델 아키텍처 검증 및 학습 결과 피드백 제공 |

#### Documentation (signbell-docs)

| 구분 | 담당 문서 | 주요 내용 |
|------|----------|----------|
| **요구사항** | 기능 명세 | 기능 요구사항 명세서 및 사용자 흐름도 명세서 작성 |
| **API 명세** | REST/WebSocket API | API 명세서 (REST API) 및 실시간 수어 퀴즈 WebSocket API 명세서 공동 작성 |

---

### 3.3. 신동준 (팀장 / Frontend Lead)

**주요 역할**: 프론트엔드 아키텍처(상태 관리, 스타일), UI/UX 디자인 시스템, 미디어 제어를 총괄하고 프로젝트를 리딩했습니다.

#### Frontend (signbell-app/frontend)

| 구분 | 담당 기능 | 주요 구현 내용 |
|------|----------|---------------|
| **UI 컴포넌트** | 공통 컴포넌트 시스템 | 재사용 가능한 공통 UI 컴포넌트 (Button, Modal, AlertModal, Toast, SkeletonLoader 등) 설계 및 개발 |
| **스타일 시스템** | SCSS 모듈화 | 전역 스타일 시스템 구축, CSS Modules 패턴 도입으로 스타일 충돌 방지, 변수 및 애니메이션 정의 |
| **상태 관리** | Zustand 아키텍처 | Zustand를 활용한 전역 상태 (로그인 유저 정보, 인증 토큰) 관리 아키텍처 설계 |
| **미디어 제어** | 웹캠 제어 훅 | MediaDevices.getUserMedia API를 활용한 웹캠 스트림 제어 커스텀 훅 및 전역 상태 관리 |
| **레이아웃** | 반응형 디자인 | Header, Footer, Sidebar를 포함한 반응형 웹 메인 레이아웃 개발 |
| **라우팅** | React Router 설정 | 페이지 라우팅 구조 설계 및 PrivateRoute 인증 가드 구현 |
| **페이지 컴포넌트** | 주요 페이지 개발 | 랜딩, 메인, 약관 동의, 마이페이지, 퀴즈 대기방/게임, 개인 학습 페이지 개발 |
| **비디오 플레이어** | 영상 재생 컴포넌트 | 수어 학습 영상 재생 컴포넌트 (로딩 상태, 에러 처리) |

#### Infrastructure (AWS & CI/CD)

| 구분 | 담당 영역 | 주요 구현 내용 |
|------|----------|---------------|
| **Backend 배포** | EC2 | Spring Boot .jar 파일 EC2 배포 및 systemd 서비스 등록 |
| **Frontend 배포** | S3 | React 정적 빌드 결과물 S3 정적 웹사이트 호스팅 배포 |
| **Database** | RDS | AWS RDS (MySQL) 인스턴스 구축 및 application-prod.yml 연동 |
| **컨테이너 오케스트레이션** | Kubernetes | Kubernetes (EKS) 클러스터 구축 및 Pod/Service/Deployment 관리 |
| **CI/CD 파이프라인** | 자동화 배포 | GitHub Actions 및 Jenkins를 활용한 빌드/테스트/배포 자동화 파이프라인 구축 |
| **인프라 자동화** | IaC | Terraform/CloudFormation을 활용한 인프라 코드화 및 자동 프로비저닝 |
| **모니터링** | 로깅 및 알림 | CloudWatch를 활용한 서버 모니터링 및 알림 설정 |

#### Documentation (signbell-docs)

| 구분 | 담당 문서 | 주요 내용 |
|------|----------|----------|
| **프로젝트 기획** | 기획 문서 | 기능 우선 순위, 사용자 페르소나, 프로젝트 개요 작성 |
| **UI/UX 설계** | 와이어프레임 | 서비스 전체 와이어프레임 제작 |
| **기능 명세** | 사용자 흐름 | 사용자흐름 기반 기능명세서, 통합API-WebRTC파이프라인 작성 |
| **협업 가이드** | Git 전략 | git-workflow (Git Flow 전략) 등 팀 협업 룰 가이드 작성 |
| **프로젝트 문서** | README 및 발표 | 프로젝트 README 작성 및 최종 발표 자료 (PPT) 제작 |

#### Project Management

| 구분 | 담당 영역 | 주요 내용 |
|------|----------|----------|
| **레포지토리 관리** | GitHub 설정 | 4개 레포지토리 (app, docs, fastapi, ml) 생성 및 브랜치 전략 (Git-Flow) 수립 |
| **협업 프로세스** | PR/Issue 템플릿 | PULL_REQUEST_TEMPLATE, ISSUE_TEMPLATE 설정 및 코드 리뷰 총괄 |

---

### 3.4. 송민재 (Backend)

**주요 역할**: Spring Security 기반의 인증/인가(OAuth2, JWT) 시스템과 마이페이지, 약관 동의 기능을 개발했습니다.

#### Backend (signbell-app/backend)

| 구분 | 담당 기능 | 주요 구현 내용 |
|------|----------|---------------|
| **소셜 로그인** | 카카오 SSO | Spring Security 및 OAuth2 Client를 활용한 카카오 소셜 로그인 인증 파이프라인 개발 |
| **JWT 인증** | 토큰 관리 | jjwt 라이브러리 기반 JWT (Access/Refresh Token) 생성, 서명, 검증, 파싱 모듈 개발 |
| **OAuth2 연동** | 토큰 발급 핸들러 | 카카오 로그인 성공 시 자체 JWT 발급 및 HTTP-Only 쿠키 전송 핸들러 개발 |
| **마이페이지 API** | 프로필 관리 | multipart/form-data 처리, S3 연동 프로필 이미지 업로드 및 닉네임 변경 REST API |
| **약관 동의** | 회원가입 로직 | 사용자 필수/선택 약관 동의 상태 DB 저장 및 관리 로직 개발 |
| **쿠키 유틸** | 쿠키 관리 | HTTP-Only 쿠키 생성, 읽기, 삭제 유틸리티 클래스 개발 |
| **사용자 Repository** | 데이터 접근 | User 엔티티 조회 메서드 (providerId 기반 조회) 구현 |

#### Frontend (signbell-app/frontend)

| 구분 | 담당 기능 | 주요 구현 내용 |
|------|----------|---------------|
| **소셜 로그인 UI** | 카카오 로그인 | 카카오 SDK 연동 및 OAuth 2.0 팝업창 방식의 소셜 로그인 UI 개발 |
| **마이페이지 UI** | 프로필 수정 | authService.js를 통한 사용자 정보 조회 및 프로필 (닉네임, 이미지) 수정 UI |
| **Axios 인터셉터** | 토큰 자동 관리 | 요청 시 JWT 자동 삽입, 401 에러 시 Refresh Token으로 재발급 또는 자동 로그아웃 처리 |
| **약관 동의 UI** | 약관 페이지 | 약관 동의 UI 및 authService.js를 통한 약관 동의 API 연동 |
| **인증 서비스** | API 통신 | 사용자 정보 조회, 프로필 수정, 약관 동의 API 호출 모듈 |

#### Infrastructure (AWS 지원)

| 구분 | 담당 영역 | 주요 구현 내용 |
|------|----------|---------------|
| **AWS S3** | 파일 스토리지 | 프로필 이미지 업로드 및 관리를 위한 S3 연동 |
| **RDS 연동** | 데이터베이스 설정 | application-prod.yml 환경 설정 및 RDS 연결 지원 |
| **배포 지원** | 인프라 협업 | EC2 배포 환경 설정 및 systemd 서비스 등록 지원 |

#### Documentation (signbell-docs)

| 구분 | 담당 문서 | 주요 내용 |
|------|----------|----------|
| **기술 스택** | 기술 명세 | 기술 스택 문서 작성 |
| **API 명세** | 카카오 SSO | 카카오 SSO API 명세서 작성 |
| **정책 문서** | 약관 | 서비스 필수/선택 동의 약관 작성 |

---

### 3.5. 백승현 (ML/AI / Fullstack)

**주요 역할**: FastAPI 기반 AI 추론 서버(WebRTC, WebSocket)를 구축하고, ML 모델 개발을 주도했으며, 개인 수어 학습 페이지(FE/BE)를 개발했습니다.

#### ML & FastAPI (signbell-fastapi / signbell-ml)

| 구분 | 담당 기능 | 주요 구현 내용 |
|------|----------|---------------|
| **AI 추론 서버** | FastAPI 아키텍처 | FastAPI 프레임워크 기반 AI 추론 서버 아키텍처 설계 및 구축 |
| **WebSocket 시그널링** | 실시간 통신 | WebSocket 엔드포인트 개발, JWT 토큰 기반 인증 및 세션 관리 |
| **프레임 수집 시스템** | 실시간 프레임 처리 | SequenceCollector를 통한 프레임 수집 및 관리, 최대 프레임 수 기반 수집 완료 판단 |
| **랜드마크 추출 파이프라인** | MediaPipe 연동 | RealtimeLandmarkExtractor 클래스 구현, MediaPipe Holistic을 활용한 손/얼굴/포즈 랜드마크 추출 (147차원 특징 벡터 생성) |
| **특징 정규화** | 좌표 정규화 | 어깨 중심+코 평균 기준 상대 좌표 변환, Z축 감쇠(Pose 0.7, 손가락 0.6) 및 이동평균 필터링, 양손 손목 3D 상대거리 벡터 계산 |
| **ML 모델 아키텍처** | CNN+BiLSTM+Attention | 1D-CNN(3층: 64→128→256 채널, 특징 추출) + 2층 Bi-LSTM(128 hidden, 시퀀스 모델링) + Multi-Head Attention(8 heads) + 동작 변화점 탐지 가중치 학습 모델 설계 |
| **모델 학습 최적화** | 과적합 방지 전략 | Dropout 0.5, Weight Decay 0.1, Label Smoothing 0.1, Batch Size 4, Learning Rate 0.0005, ReduceLROnPlateau 스케줄러, Early Stopping(patience 8) 적용 |
| **좌표 전처리** | 데이터 파이프라인 | coordinate_extractor.py를 통한 DB/로컬 영상 좌표 추출, 49개 랜드마크(Pose 6개 + 양손 42개 + 손목 거리 1개) × 3D 좌표 = 147차원 특징 벡터 생성 |
| **모델 로드 및 추론** | Predictor 클래스 | 체크포인트에서 전체 모델 객체 로드(weights_only=False), 클래스 레이블 복원, predict() 메서드를 통한 추론 및 신뢰도 반환 |
| **추론 파이프라인** | 통합 추론 흐름 | run_inference() 함수로 랜드마크 추출 → 모델 추론 → 결과 반환 통합 처리 |
| **데이터 저장 스케줄링** | 비동기 저장 | schedule_quiz_save/schedule_learning_save를 통한 추론 결과 및 학습 데이터 비동기 저장 |
| **서버 수명주기 관리** | Lifespan 이벤트 | FastAPI lifespan을 통한 Predictor 초기화 및 전역 상태 관리 |
| **모델 진단 엔드포인트** | REST API | /health, /model/status, /config 엔드포인트를 통한 모델 상태 및 설정 확인 |
| **모델 평가 및 테스트** | 테스트 스크립트 | test_model.py를 통한 로컬 비디오 기반 모델 성능 평가, 클래스별 정확도 분석, Attention 가중치 시각화 |
| **HTTPS 설정** | SSL/TLS | mkcert 인증서를 활용한 로컬 HTTPS 서버 구동 설정 |

#### Frontend (signbell-app/frontend)

| 구분 | 담당 기능 | 주요 구현 내용 |
|------|----------|---------------|
| **학습 목록 페이지** | 수어 학습 UI | 수어 학습 목록 조회, 키워드 검색, 무한 스크롤 UI 개발 |
| **학습 상세 페이지** | WebRTC 연동 | WebRTC MediaStream을 FastAPI 추론 서버로 전송하는 UI 및 로직 개발 |
| **WebSocket 모듈** | 추론 결과 수신 | WebSocket을 통해 FastAPI 서버로부터 실시간 수어 추론 결과 및 피드백 수신 |
| **학습 데이터 제공** | 영상 데이터 수집 | 사용자 동의 기반 학습용 수어 영상 데이터 제공 페이지 |
| **수어 학습 컴포넌트** | UI 컴포넌트 | 수어 단어 카드, 검색 입력, 상세 모달, 거울 모드 등 학습 관련 컴포넌트 |

#### Backend (signbell-app/backend)

| 구분 | 담당 기능 | 주요 구현 내용 |
|------|----------|---------------|
| **학습 콘텐츠 API** | REST API | 수어 학습 콘텐츠 (단어 목록, 카테고리, 상세 정보) 제공 REST API 개발 |
| **키워드 검색** | 동적 쿼리 | Querydsl을 활용한 수어 학습 콘텐츠 키워드 기반 동적 검색 기능 구현 |
| **수어 데이터 관리** | CRUD 서비스 | Sign 엔티티 CRUD 및 학습 상태 관리 서비스 |
| **외부 API 연동** | 데이터 동기화 | 한국정보화진흥원 수어 API 연동 및 데이터 동기화 서비스 |
| **퀴즈 단어 관리** | QuizWord 서비스 | 학습 완료된 수어를 퀴즈 단어로 등록하는 서비스 |
| **Sign Repository** | 데이터 접근 | Sign 엔티티 조회 및 검색 메서드 구현 |

#### Documentation (signbell-docs)

| 구분 | 담당 문서 | 주요 내용 |
|------|----------|----------|
| **AI 서버 명세** | 기술 문서 | 수어인식 서버 기술 명세서 작성 |
| **파이프라인** | 통신 구조 | 클라이언트 - 수어인식 파이프라인 문서 작성 |
| **API 명세** | WebSocket API | 개인수어학습 WebSocket API 명세서 작성 |
| **디렉토리 구조** | FastAPI 구조 | FastAPI 디렉토리 구조 명세서 작성 |

---

## 4. 공통 기여 (Shared Contributions)

모든 팀원이 함께 참여한 공통 작업 영역입니다.

| 구분 | 담당 영역 | 주요 내용 | 참여 인원 |
|------|----------|----------|----------|
| **AI 데이터셋 구축** | 학습 데이터 수집 | AI 모델 학습에 필요한 수어 동작 비디오 데이터 직접 촬영 및 제공 | 전체 팀원 |
| **통합 테스트** | 배포 환경 검증 | AWS (EC2, S3, RDS) 통합 배포 환경에서 기능 테스트 및 버그 리포팅 | 전체 팀원 |
| **코드 리뷰** | 품질 관리 | Pull Request 리뷰 및 코드 품질 개선 제안 | 전체 팀원 |
| **문서화** | 기술 문서 작성 | 각자 담당 영역의 기술 문서 및 API 명세서 작성 | 전체 팀원 |

---

## 5. 버전 히스토리 (Version History)

| 버전 | 날짜         | 작성자 | 변경 내용 |
|------|------------|--------|----------|
| v1.0.0 | 2025-10-28 | 고동현 | 초기 역할분담 명세서 작성 |

---

## 6. 참고 사항 (Notes)

- 본 문서는 프로젝트 종료 시점 기준으로 작성되었으며, 실제 개발 과정에서 역할이 유동적으로 조정되었을 수 있습니다.
- 각 팀원의 기여도는 코드 커밋 수가 아닌 기능의 복잡도와 중요도를 기준으로 평가되었습니다.
- 추가 문의사항은 각 팀원의 GitHub 프로필을 통해 연락 가능합니다.