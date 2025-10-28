# 기술 스택 목록

본 문서는 SignBell 프로젝트의 **핵심 기술 및 버전을 명세**하고, AI/ML 서버(Python/FastAPI)와 주 백엔드(Spring Boot) 환경 간의 **호환성 기준점**을 정의하여 팀 내 개발 표준을 확립하는 문서입니다.

**작성자:** [송민재](https://github.com/songkey06)

**문서 버전:** v1.1.0

**최초 작성일** 2025.10.10

**최종 수정일:** 2025.10.28

**대상 독자:**
- **내부 개발팀 (FE/BE/AI)**: 프로젝트의 기술 표준, 라이브러리 버전 및 환경 종속성을 확인하고 준수해야 하는 모든 개발자
- **신규 합류자**: 프로젝트가 사용하는 모든 기술 스택과 버전을 빠르게 파악하여 개발 환경 및 지식 습득에 활용하고자 하는 인원
- **외부 검토자**: GitHub 포트폴리오 검토 시, 프로젝트의 기술적 깊이, 버전 관리 능력, 그리고 AI/ML 통합 설계 역량을 평가하고자 하는 인원
- **아키텍트/PM**: 기술 결정의 근거를 파악하고, 장기적인 유지보수 및 확장 계획 수립을 위해 현재 기술 부채 및 환경을 파악해야 하는 인원

---

## 기술 스택

### Frontend

#### 언어 & 프레임워크
| 역할 | 기술 | 버전 | 설명 |
|------|------|------|------|
| 언어 | ![JavaScript](https://img.shields.io/badge/JavaScript-ES6+-F7DF1E?style=flat-square&logo=javascript&logoColor=black) | ES6+ | 프론트엔드 개발 언어 |
| UI 프레임워크 | ![React](https://img.shields.io/badge/React-19.1.1-61DAFB?style=flat-square&logo=react&logoColor=white) | 19.1.1 | 컴포넌트 기반 UI 구축 |
| 빌드 도구 | ![Vite](https://img.shields.io/badge/Vite-7.1.7-646CFF?style=flat-square&logo=vite&logoColor=white) | 7.1.7 | 빠른 개발 서버 및 프로덕션 빌드 |

#### 라우팅 & 상태 관리
| 역할 | 기술 | 버전 | 설명 |
|------|------|------|------|
| 라우팅 | ![React Router](https://img.shields.io/badge/React_Router-7.9.4-CA4245?style=flat-square&logo=react-router&logoColor=white) | 7.9.4 | SPA 페이지 라우팅 |
| 상태 관리 | ![Zustand](https://img.shields.io/badge/Zustand-5.0.8-000000?style=flat-square) | 5.0.8 | 전역 상태 관리 |

#### 스타일링 & UI
| 역할 | 기술 | 버전 | 설명 |
|------|------|------|------|
| CSS 전처리기 | ![Sass](https://img.shields.io/badge/Sass-1.93.2-CC6699?style=flat-square&logo=sass&logoColor=white) | 1.93.2 | CSS 모듈화 및 변수 관리 |
| 애니메이션 | ![Framer Motion](https://img.shields.io/badge/Framer_Motion-12.23.24-0055FF?style=flat-square&logo=framer&logoColor=white) | 12.23.24 | UI 애니메이션 및 제스처 |
| 아이콘 | ![Font Awesome](https://img.shields.io/badge/Font_Awesome-7.1.0-339AF0?style=flat-square&logo=font-awesome&logoColor=white) | 7.1.0 | 아이콘 라이브러리 |

#### 통신 & 실시간
| 역할 | 기술 | 버전 | 설명 |
|------|------|------|------|
| HTTP 클라이언트 | ![Axios](https://img.shields.io/badge/Axios-1.12.2-5A29E4?style=flat-square&logo=axios&logoColor=white) | 1.12.2 | REST API 통신, JWT 토큰 관리 |
| WebSocket | ![STOMP.js](https://img.shields.io/badge/STOMP.js-7.1.1-000000?style=flat-square) | 7.1.1 | 실시간 양방향 통신 (Spring Boot) |
| WebSocket | ![Native WebSocket](https://img.shields.io/badge/WebSocket-Native-010101?style=flat-square) | Native | 실시간 양방향 통신 (FastAPI) |
| 영상 통신 | ![WebRTC](https://img.shields.io/badge/WebRTC-Janus-333333?style=flat-square&logo=webrtc&logoColor=white) | Janus | 실시간 화상 통신 |

#### 개발 도구
| 역할 | 기술 | 버전 | 설명 |
|------|------|------|------|
| 코드 품질 | ![ESLint](https://img.shields.io/badge/ESLint-9.36.0-4B32C3?style=flat-square&logo=eslint&logoColor=white) | 9.36.0 | JavaScript 코드 린팅 |
| HTTPS 개발 | ![vite-plugin-mkcert](https://img.shields.io/badge/vite--plugin--mkcert-1.17.9-646CFF?style=flat-square) | 1.17.9 | 로컬 HTTPS 개발 환경 |

---

### Backend

#### 언어 & 프레임워크
| 역할 | 기술 | 버전 | 설명 |
|------|------|------|------|
| 언어 | ![Java](https://img.shields.io/badge/Java-17-007396?style=flat-square&logo=openjdk&logoColor=white) | 17 | 백엔드 개발 언어 |
| 프레임워크 | ![Spring Boot](https://img.shields.io/badge/Spring_Boot-3.5.6-6DB33F?style=flat-square&logo=spring-boot&logoColor=white) | 3.5.6 | 엔터프라이즈 애플리케이션 프레임워크 |

#### 보안 & 인증
| 역할 | 기술 | 버전 | 설명 |
|------|------|------|------|
| 보안 프레임워크 | ![Spring Security](https://img.shields.io/badge/Spring_Security-6.4.2-6DB33F?style=flat-square&logo=spring-security&logoColor=white) | 6.4.2 | 인증 및 권한 관리 |
| 토큰 인증 | ![JWT](https://img.shields.io/badge/JWT-0.12.3-000000?style=flat-square&logo=json-web-tokens&logoColor=white) | 0.12.3 | Access/Refresh Token 관리 |
| 소셜 로그인 | ![OAuth2](https://img.shields.io/badge/OAuth2-Kakao-FFCD00?style=flat-square&logo=kakao&logoColor=black) | Spring OAuth2 Client | 카카오 소셜 로그인 |

#### 데이터베이스 & ORM
| 역할 | 기술 | 버전 | 설명 |
|------|------|------|------|
| RDBMS | ![MariaDB](https://img.shields.io/badge/MariaDB-11.6-003545?style=flat-square&logo=mariadb&logoColor=white) | 11.6 | 프로덕션 데이터베이스 |
| ORM | ![Spring Data JPA](https://img.shields.io/badge/Spring_Data_JPA-3.4.1-6DB33F?style=flat-square&logo=spring&logoColor=white) | 3.4.1 | JPA 기반 데이터 접근 |
| 쿼리 빌더 | ![QueryDSL](https://img.shields.io/badge/QueryDSL-5.0.0-0769AD?style=flat-square) | 5.0.0 | 타입 안전 동적 쿼리 생성 |
| 테스트 DB | ![H2](https://img.shields.io/badge/H2-2.3.232-0000BB?style=flat-square) | 2.3.232 | 인메모리 테스트 데이터베이스 |

#### 실시간 통신
| 역할 | 기술 | 버전 | 설명 |
|------|------|------|------|
| WebSocket | ![WebSocket](https://img.shields.io/badge/WebSocket-STOMP-010101?style=flat-square) | Spring WebSocket | 실시간 양방향 통신 |

#### 개발 도구
| 역할 | 기술 | 버전 | 설명 |
|------|------|------|------|
| 코드 간소화 | ![Lombok](https://img.shields.io/badge/Lombok-1.18.36-BC4521?style=flat-square) | 1.18.36 | 보일러플레이트 코드 제거 |
| 개발 편의 | ![Spring DevTools](https://img.shields.io/badge/Spring_DevTools-3.5.6-6DB33F?style=flat-square&logo=spring&logoColor=white) | 3.5.6 | 자동 재시작 및 라이브 리로드 |
| 환경 변수 | ![Spring Dotenv](https://img.shields.io/badge/Spring_Dotenv-4.0.0-6DB33F?style=flat-square) | 4.0.0 | .env 파일 관리 |
| SQL 로깅 | ![P6Spy](https://img.shields.io/badge/P6Spy-1.9.1-FF6B6B?style=flat-square) | 1.9.1 | SQL 쿼리 로깅 및 디버깅 |
| 데이터 검증 | ![Bean Validation](https://img.shields.io/badge/Bean_Validation-3.0-6DB33F?style=flat-square) | 3.0 | 입력 데이터 유효성 검증 |

---

### AI/ML Server

#### 언어 & 프레임워크
| 역할 | 기술 | 버전 | 설명 |
|------|------|------|------|
| 언어 | ![Python](https://img.shields.io/badge/Python-3.10+-3776AB?style=flat-square&logo=python&logoColor=white) | 3.10+ | AI/ML 개발 언어 |
| API 프레임워크 | ![FastAPI](https://img.shields.io/badge/FastAPI-0.110.0-009688?style=flat-square&logo=fastapi&logoColor=white) | 0.110.0 | 비동기 API 서버 |
| 웹 서버 | ![Uvicorn](https://img.shields.io/badge/Uvicorn-0.29.0-499848?style=flat-square) | 0.29.0 | ASGI 웹 서버 (standard 옵션 포함) |

#### 보안 & 인증
| 역할 | 기술 | 버전 | 설명 |
|------|------|------|------|
| JWT 인증 | ![python-jose](https://img.shields.io/badge/python--jose-latest-000000?style=flat-square) | latest | JWT 토큰 검증 및 디코딩 (cryptography 포함) |

#### 딥러닝 프레임워크
| 역할 | 기술 | 버전 | 설명 |
|------|------|------|------|
| 딥러닝 | ![PyTorch](https://img.shields.io/badge/PyTorch-2.2.0-EE4C2C?style=flat-square&logo=pytorch&logoColor=white) | 2.2.0 | CNN+BiLSTM+Attention 모델 학습 및 추론 (nn.Module, LSTM, MultiheadAttention, Conv1d) |

#### 영상 처리 & 특징 추출
| 역할 | 기술 | 버전 | 설명 |
|------|------|------|------|
| 랜드마크 추출 | ![MediaPipe](https://img.shields.io/badge/MediaPipe-0.10.11-0097A7?style=flat-square) | 0.10.11 | Holistic 모델을 통한 손/얼굴/포즈 랜드마크 추출 (49개 랜드마크 × 3D 좌표) |
| 영상 처리 | ![OpenCV](https://img.shields.io/badge/OpenCV-4.9.0.80-5C3EE8?style=flat-square&logo=opencv&logoColor=white) | 4.9.0.80 | 비디오 프레임 디코딩, 색상 변환, 영상 스트리밍 처리 |
| 이미지 처리 | ![Pillow](https://img.shields.io/badge/Pillow-10.3.0-3776AB?style=flat-square) | 10.3.0 | 이미지 처리 보조 라이브러리 |

#### 데이터 처리 & 분석
| 역할 | 기술 | 버전 | 설명 |
|------|------|------|------|
| 수치 계산 | ![NumPy](https://img.shields.io/badge/NumPy-1.26.4-013243?style=flat-square&logo=numpy&logoColor=white) | 1.26.4 | 배열 및 행렬 연산, 좌표 데이터 전처리 및 정규화 |
| 데이터 분석 | ![Pandas](https://img.shields.io/badge/Pandas-2.2.0-150458?style=flat-square&logo=pandas&logoColor=white) | 2.2.0 | 메타데이터 관리, CSV 파일 읽기/쓰기, 데이터 분할 |
| 머신러닝 | ![scikit-learn](https://img.shields.io/badge/scikit--learn-1.7.2-F7931E?style=flat-square&logo=scikit-learn&logoColor=white) | 1.7.2 | LabelEncoder, train_test_split, 분류 성능 평가 (accuracy, classification_report, confusion_matrix) |
| 시각화 | ![Matplotlib](https://img.shields.io/badge/Matplotlib-3.8.0-11557C?style=flat-square) | 3.8.0 | 학습 곡선, 혼동행렬, Attention 가중치 시각화 |
| 시각화 | ![Seaborn](https://img.shields.io/badge/Seaborn-0.13.0-4C72B0?style=flat-square) | 0.13.0 | 통계 기반 데이터 시각화 (혼동행렬 히트맵) |
| 진행 표시 | ![tqdm](https://img.shields.io/badge/tqdm-4.66.0-FFC107?style=flat-square) | 4.66.0 | 학습 및 추론 진행률 표시 |



#### 실시간 통신
| 역할 | 기술 | 버전 | 설명 |
|------|------|------|------|
| WebSocket | ![WebSockets](https://img.shields.io/badge/WebSockets-11.0.3-010101?style=flat-square) | 11.0.3 | WebSocket 클라이언트 및 테스트 |
| HTTP 클라이언트 | ![httpx](https://img.shields.io/badge/httpx-0.27.0-0096D6?style=flat-square) | 0.27.0 | 비동기 HTTP 통신 및 테스트 |

---

### Build & Test

| 역할 | 기술 | 설명 |
|------|------|------|
| 빌드 도구 | ![Gradle](https://img.shields.io/badge/Gradle-8.11-02303A?style=flat-square&logo=gradle&logoColor=white) | Java 백엔드의 빌드와 라이브러리 의존성 자동 관리 |
| 패키지 관리 | ![npm](https://img.shields.io/badge/npm-10.x-CB3837?style=flat-square&logo=npm&logoColor=white) | Node.js 프로젝트의 패키지 설치 및 관리 |
| 테스트 프레임워크 | ![JUnit5](https://img.shields.io/badge/JUnit5-5.10-25A162?style=flat-square&logo=junit5&logoColor=white) | 백엔드 코드의 단위 및 통합 테스트 |
| 모킹 라이브러리 | ![Mockito](https://img.shields.io/badge/Mockito-5.x-78C257?style=flat-square) | 가짜 객체를 만들어 테스트 환경 구축 |
| 보안 테스트 | ![Spring Security Test](https://img.shields.io/badge/Spring_Security_Test-6.4.2-6DB33F?style=flat-square&logo=spring-security&logoColor=white) | Spring Security 테스트 지원 |

---

### Infrastructure & DevOps

#### 클라우드 서비스 (AWS)
| 역할 | 기술 | 설명 |
|------|------|------|
| 컴퓨팅 | ![AWS EC2](https://img.shields.io/badge/AWS_EC2-FF9900?style=flat-square&logo=amazon-ec2&logoColor=white) | Spring Boot 애플리케이션 배포 |
| 스토리지 | ![AWS S3](https://img.shields.io/badge/AWS_S3-569A31?style=flat-square&logo=amazon-s3&logoColor=white) | 정적 파일 호스팅 및 이미지 저장 |
| 데이터베이스 | ![AWS RDS](https://img.shields.io/badge/AWS_RDS-527FFF?style=flat-square&logo=amazon-rds&logoColor=white) | MariaDB 관리형 서비스 |
| 컨테이너 관리 | ![AWS EKS](https://img.shields.io/badge/AWS_EKS-FF9900?style=flat-square&logo=amazon-eks&logoColor=white) | Kubernetes 클러스터 관리 |

#### 컨테이너 & 오케스트레이션
| 역할 | 기술 | 설명 |
|------|------|------|
| 컨테이너화 | ![Docker](https://img.shields.io/badge/Docker-2496ED?style=flat-square&logo=docker&logoColor=white) | 애플리케이션 컨테이너화 |
| 오케스트레이션 | ![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat-square&logo=kubernetes&logoColor=white) | 컨테이너 배포 및 관리 |

#### CI/CD & IaC
| 역할 | 기술 | 설명 |
|------|------|------|
| CI/CD | ![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-2088FF?style=flat-square&logo=github-actions&logoColor=white) | 자동화된 빌드/테스트/배포 파이프라인 |
| CI/CD | ![Jenkins](https://img.shields.io/badge/Jenkins-D24939?style=flat-square&logo=jenkins&logoColor=white) | 지속적 통합 및 배포 자동화 |
| IaC | ![Terraform](https://img.shields.io/badge/Terraform-7B42BC?style=flat-square&logo=terraform&logoColor=white) | 인프라 코드화 및 자동 프로비저닝 |
| 이슈 관리 | ![GitHub Issues](https://img.shields.io/badge/GitHub_Issues-181717?style=flat-square&logo=github&logoColor=white) | 이슈 템플릿 기반 작업 관리 (bug, feature, chore, docs 등) |
| PR 관리 | ![GitHub PR](https://img.shields.io/badge/Pull_Request-181717?style=flat-square&logo=github&logoColor=white) | PR 템플릿 기반 코드 리뷰 프로세스 |

---

### Development Environment

#### IDE & 에디터
| 역할 | 기술 | 설명 |
|------|------|------|
| Java IDE | ![IntelliJ IDEA](https://img.shields.io/badge/IntelliJ_IDEA-000000?style=flat-square&logo=intellij-idea&logoColor=white) | 백엔드 개발 통합 환경 |
| AI 코드 에디터 | ![Kiro](https://img.shields.io/badge/Kiro-4A90E2?style=flat-square) | AI 기반 코드 에디터 |
| AI 코드 에디터 | ![Cursor](https://img.shields.io/badge/Cursor-000000?style=flat-square) | AI 페어 프로그래밍 에디터 |

#### 운영 체제
| 역할 | 기술 | 설명 |
|------|------|------|
| OS | ![Windows](https://img.shields.io/badge/Windows-0078D6?style=flat-square&logo=windows&logoColor=white) | 개발 환경 |
| OS | ![macOS](https://img.shields.io/badge/macOS-000000?style=flat-square&logo=apple&logoColor=white) | 개발 환경 |

#### 브라우저
| 역할 | 기술 | 설명 |
|------|------|------|
| 브라우저 | ![Chrome](https://img.shields.io/badge/Chrome-4285F4?style=flat-square&logo=google-chrome&logoColor=white) | 프론트엔드 개발 및 테스트 |

---

### Collaboration Tools

| 역할 | 기술 | 설명 |
|------|------|------|
| 버전 관리 | ![Git](https://img.shields.io/badge/Git-F05032?style=flat-square&logo=git&logoColor=white) | 소스 코드 버전 관리 |
| 코드 호스팅 | ![GitHub](https://img.shields.io/badge/GitHub-181717?style=flat-square&logo=github&logoColor=white) | 코드 저장소 및 협업 |
| 커뮤니케이션 | ![Discord](https://img.shields.io/badge/Discord-5865F2?style=flat-square&logo=discord&logoColor=white) | 팀 실시간 소통 |

---

* **Java 17**

  백엔드 서버의 핵심 로직을 작성하는 데 사용되는 프로그래밍 언어입니다.

* **Spring Boot 3.5.6**

  복잡한 설정 없이 독립 실행형 서버를 빠르게 구축하고 API를 개발할 수 있는 프레임워크입니다. WebSocket, Security, Data JPA 등 다양한 스타터를 제공합니다.

* **P6Spy 1.9.1**

  애플리케이션이 실행하는 모든 SQL 쿼리를 자세히 로깅하여 DB 연동 디버깅을 돕는 도구입니다.

* **Python 3.10+**

  AI/ML 모델 개발 및 실시간 추론 서버 구축에 사용되며, 3.10 이상 버전을 권장합니다.

* **FastAPI 0.110.0, uvicorn 0.29.0**

  Python 기반의 고성능 비동기 API 서버를 구축하여 수어 인식 모델 추론 결과를 제공합니다. uvicorn은 standard 옵션으로 설치하여 WebSocket 및 성능 최적화 기능을 포함합니다.

* **python-jose[cryptography]**

  JWT 토큰 검증 및 디코딩을 위한 라이브러리로, Spring Boot 백엔드와의 인증 통합을 지원합니다.

* **torch 2.2.0**

  수어 인식 모델(CNN+BiLSTM+Attention) 개발 및 추론에 사용되는 주요 딥러닝 프레임워크입니다. 1D-CNN(3층: 64→128→256 채널), 2층 Bi-LSTM(128 hidden), Multi-Head Attention(8 heads), 동작 변화점 탐지 모듈을 포함한 복합 모델 구조를 구현합니다. Dropout(0.5), Weight Decay(0.1), Label Smoothing(0.1) 등의 정규화 기법을 적용하여 과적합을 방지합니다.

* **MediaPipe 0.10.11**

  Holistic 모델을 통해 영상에서 손(양손 42개), 포즈(상체 6개), 손목 거리(1개) 총 49개 랜드마크의 3D 좌표를 추출하여 147차원 특징 벡터를 생성합니다. 어깨 중심+코 평균 기준점으로 정규화하고, Z축 감쇠(Pose 0.7, 손가락 0.6) 및 이동평균 필터링을 적용하여 안정적인 좌표 데이터를 생성합니다.

* **OpenCV 4.9.0.80, Pillow 10.3.0**

  OpenCV는 비디오 스트리밍, 프레임 디코딩, BGR↔RGB 색상 변환을 담당하며, Pillow는 이미지 처리 보조 작업을 수행합니다.

* **NumPy 1.26.4, Pandas 2.2.0, scikit-learn 1.7.2**

  NumPy는 좌표 배열 연산 및 정규화를 담당하고, Pandas는 메타데이터 CSV 관리 및 데이터 분할을 수행합니다. scikit-learn은 LabelEncoder를 통한 클래스 레이블 인코딩/복원, train_test_split을 통한 데이터 분할, accuracy_score/classification_report/confusion_matrix를 통한 모델 성능 평가를 제공합니다.

* **Matplotlib 3.8.0, Seaborn 0.13.0, tqdm 4.66.0**

  Matplotlib은 학습 곡선, Attention 가중치, 동작 변화점 시각화를 담당하고, Seaborn은 혼동행렬 히트맵을 생성합니다. tqdm은 학습 및 추론 과정의 진행률을 실시간으로 표시합니다.

* **WebSockets 11.0.3, httpx 0.27.0**

  WebSocket 클라이언트 테스트 및 비동기 HTTP 통신을 위한 라이브러리입니다.

* **Vite 7.1.7**

  빠른 개발 서버와 최적화된 프로덕션 빌드를 제공하는 차세대 프론트엔드 빌드 도구입니다.

* **React 19.1.1, React Router DOM 7.9.4**

  재사용 가능한 컴포넌트 기반 UI 구축과 SPA(단일 페이지 애플리케이션)의 페이지 라우팅 관리를 담당합니다.

* **Zustand 5.0.8, Sass/SCSS 1.93.2**

  가볍고 간단한 전역 상태 관리와 CSS 전처리기를 통한 효율적인 스타일링을 수행합니다.

* **@stomp/stompjs 7.1.1**

  WebSocket 기반 STOMP 프로토콜을 통한 실시간 양방향 통신을 지원하는 라이브러리입니다.

* **Framer Motion 12.23.24**

  React 애플리케이션에서 부드러운 애니메이션과 제스처를 구현하는 라이브러리입니다.

* **FontAwesome 7.1.0**

  다양한 아이콘을 제공하는 아이콘 라이브러리로 UI 개발을 지원합니다.

* **Spring Security, JWT (JJWT 0.12.3)**

  사용자 인증 및 권한 관리를 담당하며 토큰 기반 로그인 기능을 제공하는 보안 프레임워크입니다. OAuth2 Client를 통한 소셜 로그인을 지원합니다.

* **Axios 1.12.2**

  Promise 기반의 HTTP 클라이언트 라이브러리로, 간편한 API 요청 및 응답 처리를 지원합니다. Interceptor를 통한 JWT 토큰 자동 관리 기능을 제공합니다.

* **Janus WebRTC Gateway**

  실시간 영상 통신을 위한 WebRTC 서버로, 퀴즈 게임 중 참가자 간 화상 통신을 지원합니다.

* **MariaDB, H2 Database**

  영구적인 데이터를 저장하는 관계형 데이터베이스와 테스트용으로 사용하는 경량 인메모리 데이터베이스입니다.

* **QueryDSL 5.0.0**

  JPA 환경에서 타입 안전하고 동적인 복잡한 SQL 쿼리를 생성하여 데이터베이스 연동 효율을 높입니다.

* **Gradle**

  Java 백엔드의 빌드와 라이브러리 의존성을 자동으로 관리하는 빌드 도구입니다.

* **npm**

  Node.js 프로젝트의 패키지 설치와 관리를 담당하는 기본 패키지 매니저입니다.

* **Spring Dotenv 4.0.0**

  .env 파일을 자동으로 로딩하여 환경 변수를 관리하는 라이브러리입니다.

* **JUnit 5, Mockito**

  백엔드 코드의 단위 및 통합 테스트를 작성하고 가짜 객체를 만들어 테스트 환경을 구축하는 데 사용됩니다.

* **Lombok, Spring Boot DevTools**

  반복 코드를 줄여주고 코드 변경 시 서버를 자동으로 재시작하여 개발 편의성을 높입니다.

* **Bean Validation, Spring Data JDBC**

  Bean Validation은 @NotNull, @Size 등의 어노테이션을 통해 입력 데이터의 유효성을 검증하고, Spring Data JDBC는 JPA와 함께 JDBC 기반의 데이터베이스 접근을 지원합니다.

* **ESLint, vite-plugin-mkcert**

  ESLint는 JavaScript 코드의 품질을 유지하고 일관된 코딩 스타일을 적용하며, vite-plugin-mkcert는 로컬 개발 환경에서 HTTPS를 쉽게 설정하여 WebRTC 및 보안 기능을 테스트할 수 있게 합니다.

* **Git, GitHub, Discord**

  여러 개발자가 소스 코드를 공동 작업하고 소통하는 데 사용되는 협업 도구입니다.

* **GitHub Actions, Jenkins, Terraform**

  GitHub Actions는 코드 푸시 시 자동으로 빌드, 테스트, 배포를 수행하는 CI/CD 파이프라인을 제공합니다. Jenkins는 복잡한 빌드 및 배포 워크플로우를 관리하며, Terraform은 AWS 인프라를 코드로 정의하고 자동으로 프로비저닝하여 인프라 관리의 일관성과 재현성을 보장합니다.

* **Docker, Kubernetes (AWS EKS)**

  Docker는 애플리케이션을 컨테이너로 패키징하여 환경 독립성을 제공하고, Kubernetes는 AWS EKS를 통해 컨테이너의 배포, 스케일링, 관리를 자동화합니다.

* **AWS EC2, S3, RDS**

  EC2는 Spring Boot 애플리케이션을 호스팅하고, S3는 정적 파일 및 이미지를 저장하며, RDS는 MariaDB를 관리형 서비스로 제공하여 데이터베이스 운영을 간소화합니다.

---

## 변경 이력

| 버전         | 날짜             | 변경 내용                       | 작성자 |
|:-----------|:---------------|:----------------------------|:----|
| **v0.1**   | **2025.10.10** | **기술 스택 초안 작성**             | 송민재 |
| **v0.2**   | **2025.10.10** | **기술 스택 폴더 이동 및 오기재 사항 수정** | 송민재 |
| **v1.0.0** | **2025.10.10** | **문서 버전 업데이트**              | 송민재 |
| **v1.1.0** | **2025.10.28** | ** 최신 버전 업데이트** | 고동현 |

