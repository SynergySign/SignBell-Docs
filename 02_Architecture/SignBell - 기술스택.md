# SignBell 프로젝트 기술 스택 문서

**작성자**: [신동준](https://github.com/sdj3959)

**문서 버전**: v1.0

**최종 수정일:** 2025.10.02

## 1. 개발 환경

* **운영체제:** Windows 11 Pro / MacOS Tahoe (26.0)
* **IDE:** IntelliJ IDEA Ultimate
* **데이터베이스:** MariaDB (개발/테스트/운영)
* **외부 서비스 연동:** 카카오, 구글, 네이버 로그인 API
* **버전 관리:** Git & GitHub
* **패키지 매니저:** npm (프론트엔드), Gradle (백엔드)
* **웹 브라우저:** Google Chrome, Microsoft Edge
* **배포 환경:** AWS EC2, RDS, S3

---

## 2. 프론트엔드 (Frontend)

* **프레임워크 및 라이브러리:**
    * **React:** 사용자 인터페이스 구축을 위한 JavaScript 라이브러리
    * **Zustand:** 상태 관리 라이브러리
    * **React Router DOM:** 페이지 라우팅 관리
    * **Axios:** 백엔드와 통신을 위한 HTTP 클라이언트
* **패키지 매니저:** npm
* **빌드 도구:** Vite
    * **설정:** 개발 서버는 ~ 포트를 사용하며, `/api`로 시작하는 요청은 백엔드 서버(`http://localhost:~`)로 프록시 처리하여 CORS 문제를 해결
* **스타일링:** Sass / SCSS
* **테스트 및 린트:** ESlint

---

## 3. 백엔드 (Backend)

* **프레임워크:** Spring Boot 3.5.6
* **언어:** Java 17
* **주요 라이브러리:**
    * **Spring Data JPA & QueryDSL:** JPA(Hibernate) 기반의 ORM 및 타입-세이프 쿼리 작성
    * **Spring Security & OAuth2 Client:** 사용자 인증, 인가 및 소셜 로그인 기능
    * **Spring Boot Starter Web & Webflux:** RESTful API 및 비동기 HTTP 클라이언트(WebClient) 구축
    * **Lombok:** 코드 간소화 유틸리티
    * **JJWT:** JWT 기반 인증 토큰 생성/검증
    * **Spring Dotenv:** 환경 변수 관리
    * **Spring WebSocket:** 실시간 통신 기능 제공
    * **Springdoc OpenAPI:** API 문서 자동화
    * **~:** ~
* **데이터베이스:**
    * **개발/테스트/배포 주요 DB:** MariaDB
* **빌드 도구:** Gradle
* **API:** RESTful API
* **API 테스트 도구:** Swagger

---

## 4. 기타

* **외부 서비스 연동:** 카카오, 구글 로그인 API
* **협업 도구:** Git, GitHub
* **API 문서화:** Swagger