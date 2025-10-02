# SignBell 프로젝트 클론 및 환경설정 가이드

본 문서는 SignBell 프로젝트를 로컬 환경에 클론하여 정상적으로 개발 및 실행하기 위한 환경 설정 방법과 필수 유의사항을 안내합니다.

**작성자:** [신동준](https://github.com/sdj3959)

**문서 버전:** v1.0

**대상 독자:**
- **개발자**: 개발 환경 세팅 및 실행 방법 숙지
- **QA/테스터**: 테스트 환경 구축 및 재현 환경 설정
- **신규 합류자**: 빠른 온보딩을 위해 로컬에서 프로젝트 실행이 필요한 인원
- **운영자**: 배포 전 로컬 환경 확인 및 기본 실행 점검

---

## **Quick Start (빠른 실행 가이드)**

프로젝트를 바로 실행해보고 싶으신 경우, 아래 절차를 따라주세요.

```bash
# 1. 저장소 클론

# 2. 데이터베이스 생성 (MariaDB 실행 필요)
CREATE DATABASE ~
DEFAULT CHARACTER SET utf8mb4
DEFAULT COLLATE utf8mb4_unicode_ci;

# 3. 환경 변수 설정
# .env 파일을 생성하여 설정

# 4. 실행 (Gradle Wrapper 사용)
./gradlew bootRun   # Mac/Linux
gradlew.bat bootRun # Windows

# 5. 테스트 브라우저에서 접속하거나 프론트엔드 서버 구동 후 프론트엔드 서버 브라우저에 접속

API 테스트 url

http://localhost:~/swagger-ui/index.html

프론트엔드 서버 url

http://localhost:~

```

---

## **1. 사전 준비 사항**

프로젝트 설정을 시작하기 전에, 로컬 개발 환경에 아래 사항들이 먼저 준비되어야 합니다.

### **1-1. 필수 요구사항**

- JDK 17 이상
- MariaDB ([MariaDB Server 다운로드](https://mariadb.org/download/))
- Git

### **1-2. 데이터베이스 설정**

MariaDB에서 다음 명령을 실행하여 데이터베이스를 생성합니다

```sql
CREATE DATABASE ~
DEFAULT CHARACTER SET utf8mb4
DEFAULT COLLATE utf8mb4_unicode_ci;
```

---

## **2. 환경 변수 설정**

프로젝트 실행을 위해 `.env` 파일을 생성하여 다음 환경 변수들을 설정해야 합니다.

`.env.example` 파일을 참고하여 환경변수를 설정합니다.

.env 파일 생성위치는 [프로젝트 구조](#5-프로젝트-구조) 를 참고 바랍니다.

> API 키 등 민감 정보의 보안을 위하여 `.env` 파일은 gitignore 목록에 추가되어 있습니다.

---

### **2-1. 외부 API 키 발급**

[구글 로그인 API](https://developers.google.com/identity/protocols/oauth2)

[카카오 로그인 API](https://developers.kakao.com/docs/latest/kakaologin)

---

### **2-2. JWT 시크릿 토큰 생성**

[JWT 키 생성가이드](./generate_JWT_KEY_OpenSSL.md)


## **3. 빌드 및 실행**

### **3-1. 프로젝트 빌드**

```bash
./gradlew build
```

### **3-2. 서버 실행**

```bash
./gradlew bootRun
```

기본 실행 포트는 `~`입니다.

---

## **4. API 테스트**

서버가 정상적으로 실행되면 다음 URL에서 API를 테스트할 수 있습니다.

[http://localhost:~/swagger-ui/index.html](http://localhost:~/swagger-ui/index.html)






---

## **5. 프로젝트 구조**

주요 디렉토리 구조는 다음과 같습니다:

```
SignBell-App/
~

주요 패키지 설명:
- auth/: 사용자 인증 및 권한 관리
- ~/: ~
```

---

## **6. 문제 해결**

일반적인 문제 해결 방법:

1. 데이터베이스 연결 오류
    - MariaDB 서비스가 실행 중인지 확인
    - 데이터베이스 접속 정보가 올바른지 확인

2. 빌드 실패
    - Gradle 캐시 삭제 후 재시도: `./gradlew clean`
    - JDK 버전 확인

---

## **7. 참고 사항**

- 개발 환경에서는 JPA의 `ddl-auto`가 `create-drop`으로 설정되어 있어 서버 재시작 시 데이터가 초기화됩니다.
- 실제 운영 환경에서는 이 설정을 `update` 또는 `none`으로 변경해야 합니다.

---

이후 아래 항목(프론트엔드) 실행.

```bash
# 1. Node.js 패키지 설치
npm install

# 2. 환경 변수 설정
# .env 파일을 생성하여 설정

# 3. 개발 서버 실행
npm run dev

# 4. 브라우저에서 접속

http://localhost:~
```

---

## **1. 사전 준비 사항**

프로젝트 설정을 시작하기 전에, 로컬 개발 환경에 아래 사항들이 먼저 준비되어야 합니다.

### **1. 필수 요구사항**

- Node.js 18 이상
- npm
- Git


---

## **2. 환경 변수 설정**

프로젝트 실행을 위해 `.env` 파일을 생성하여 다음 환경 변수들을 설정해야 합니다.

`.env.example` 파일을 참고하여 환경변수를 설정합니다.

.env 파일 생성위치는 아래 프로젝트 구조를 참고 바랍니다.

> API 키 등 민감 정보의 보안을 위하여 `.env` 파일은 gitignore 목록에 추가되어 있습니다.

---

### **2-1. 필수 환경 변수**

```env
VITE_API_URL=http://localhost:~
```

## **3. 빌드 및 실행**

### **3-1. 개발 모드 실행**

```bash
npm run dev
```


기본 개발 서버 포트는 `~`입니다.

---

## **4. 컴포넌트 구조**

주요 컴포넌트는 다음과 같이 구성되어 있습니다:

- **인증 (Auth)**
  - 로컬 회원 로그인
  - 소셜 로그인 (Google, Kakao)
  - 인증 상태 관리
  - 보호된 라우트

- **~**
  - ~
  - ~
  - ~

---

## **5.1 프로젝트 구조**

주요 디렉토리 구조는 다음과 같습니다:

```
Tropical-App/
~
```

---

## **6. 스타일 가이드**

### **6-1. CSS 모듈**

- 컴포넌트별로 독립된 CSS 모듈을 사용

---

## **7. 문제 해결**

일반적인 문제 해결 방법:

1. 패키지 설치 오류
   - `node_modules` 삭제 후 재설치

2. 빌드 실패
   - 환경 변수 설정 확인

3. API 연결 오류
   - 백엔드 서버 실행 상태 확인
   - CORS 설정 확인
   - 환경 변수의 API URL 확인

---
