# SignBell 백엔드 디렉토리 구조


## 1. 개요

본 문서는 SignBell 프로젝트의 **백엔드(Spring)** 부분 디렉토리 구조를 설명합니다. 각 디렉토리의 역할 중심으로 기술하여 프로젝트 구조 이해를 돕고 코드 관리의 일관성을 유지하는 것을 목표로 합니다.


* **작성자:** [고동현](https://github.com/rhehdgus8831)
* **작성일**: 2025-10-27
* **최종 수정일**: 2025-10-27
* **문서 버전**: v1.0.0

**대상 독자:**

* **기획자 / PM**: 백엔드 시스템의 구성 요소를 이해하고 개발 일정 관리에 참고
* **백엔드 개발자**: 서버 애플리케이션의 코드 구조를 파악하고 개발 및 유지보수 수행
* **프론트엔드 개발자**: 백엔드 API와의 연동 지점을 이해하고 협업 시 참고
* **DevOps / 인프라 엔지니어**: 서버 배포 및 운영 환경 구성 시 구조 참고
* **QA / 테스트 엔지니어**: 기능 테스트 및 통합 테스트 설계 시 구조 참고
* **신규 합류자**: 백엔드 코드 베이스의 전체적인 구조를 빠르게 파악



## 2. 최상위 디렉토리 구조

```

backend/
├── .github/      \# GitHub 관련 설정 (워크플로우, 템플릿)
├── gradle/       \# Gradle Wrapper 관련 파일
└── src/          \# 소스 코드 루트 디렉토리
├── main/     \# 애플리케이션 메인 코드
└── test/     \# 테스트 코드

```

* `.github/`: GitHub Actions 워크플로우, 이슈/PR 템플릿 등 GitHub 관련 설정 파일 위치.
* `gradle/`: Gradle Wrapper 관련 파일 위치. 특정 버전의 Gradle을 사용하도록 강제하여 빌드 환경 일관성 유지.
* `src/`: 모든 소스 코드(메인, 테스트)가 위치하는 최상위 디렉토리.

---

## 3. `src/main/java/app/signbell/backend` 디렉토리 구조

애플리케이션의 핵심 로직이 위치합니다.

```

backend/src/main/java/app/signbell/backend/
├── config/       \# 애플리케이션 설정 (보안, 웹소켓, DB 등)
├── controller/   \# API 엔드포인트 정의 (HTTP 요청 처리)
├── dto/          \# 데이터 전송 객체 (Request/Response 형식 정의)
├── entity/       \# 데이터베이스 테이블과 매핑되는 객체 (JPA Entity)
├── exception/    \# 예외 처리 관련 클래스
├── repository/   \# 데이터베이스 접근 로직 (JPA Repository, Querydsl)
├── service/      \# 핵심 비즈니스 로직 구현
└── util/         \# 공통 유틸리티 클래스 (JWT, 쿠키, 보안 관련 등)

```

* `config/`: Security, WebSocket, Querydsl, CORS 등 애플리케이션의 주요 설정 클래스들을 포함.
* `controller/`: 기능별(게임방, 마이페이지, 수어 데이터, 인증 등) HTTP 요청을 받아 처리하는 컨트롤러 클래스들을 포함.
* `dto/`: API 요청(Request) 및 응답(Response) 시 데이터 형식을 정의하는 클래스들을 포함. 계층 간 데이터 전달 역할.
* `entity/`: 데이터베이스 테이블 구조와 매핑되는 JPA Entity 클래스들을 포함.
* `exception/`: 비즈니스 로직 중 발생하는 예외 상황을 정의하고, 전역적으로 처리하는 핸들러 포함.
* `repository/`: 데이터베이스 CRUD(Create, Read, Update, Delete) 작업을 담당하는 인터페이스 및 구현체. Querydsl을 이용한 복잡한 쿼리 포함.
* `service/`: 실제 비즈니스 로직을 수행하는 서비스 클래스들을 포함. Controller와 Repository 사이의 중간 계층.
* `util/`: JWT 토큰 생성/검증, 쿠키 관리, 보안 관련 유틸리티 등 프로젝트 전반에서 사용되는 보조 기능 클래스들을 포함.

---

## 4. `src/main/resources` 디렉토리 구조

애플리케이션 설정 파일 및 정적 리소스가 위치합니다.

```

backend/src/main/resources/
└── application\*.yml \# Spring Boot 설정 파일 (환경별 분리)

```

* `application.yml`: 공통 설정 및 기본 프로필 설정.
* `application-dev.yml`: 개발 환경 설정.
* `application-prod.yml`: 운영 환경 설정.

---

## 5. `src/test/java/app/signbell/backend` 디렉토리 구조

단위 테스트 및 통합 테스트 코드가 위치합니다. 구조는 `src/main`과 유사하게 가져갑니다.

```

backend/src/test/java/app/signbell/backend/
├── controller/   \# 컨트롤러 테스트
├── repository/   \# 리포지토리 테스트 (DB 연동 테스트 포함)
└── service/      \# 서비스 로직 테스트

```

---

## 6. 변경 이력

| 버전 | 날짜 | 변경 내용    | 작성자 |
| :--- | :--- |:---------| :--- |
| v1.0.0 | 2025-10-27 | 문서 초안 작성 | [고동현](https://github.com/rhehdgus8831) |
