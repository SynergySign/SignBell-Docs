# SignBell - 데이터베이스 스키마 가이드

본 문서는 SignBell 프로젝트의 MariaDB 데이터베이스 스키마(SQL DDL)에 대한 공식 가이드입니다. 각 테이블의 설계 의도와 컬럼의 역할을 설명하고, 애플리케이션 개발 시 반드시 고려해야 할 핵심 원칙과 구현 가이드를 제공합니다.

* **문서 버전**: v1.1
* **작성자**: [고동현](https://github.com/rhehdgus8831)
* **작성일:** 2025-10-08
* **최종 수정일:** 2025-10-21

**대상 독자:**

* **기획자 / PM**: signbell의 철학과 기획 의도를 이해하고 프로젝트 방향성을 설정하는 담당자
* **백엔드 개발자**: WebRTC 화상 통신, AI 모델 서빙 및 유사도 판정 로직을 구현·유지보수하는 개발자
* **프론트엔드 개발자**: 캠 화면 렌더링, 실시간 퀴즈 UI/UX를 설계·구현하는 개발자
* **디자이너**: 포용성과 직관성에 기반하여 서비스의 사용자 경험(UX)과 상호작용을 구체화하는 담당자
* * **DevOps / 인프라 엔지니어**: 배포 환경, 보안(CORS, HTTPS, 쿠키 설정) 등을 운영하는 담당자
* **AI 모델러**: 수어 좌표 데이터 파이프라인 및 학습 모델을 설계·개선하는 담당자
* **신규 합류자**: signbell의 문제의식과 기획 철학을 빠르게 파악해야 하는 팀 신규 인원



### 1. 데이터베이스 스키마(SQL DDL)

```
-- SignBell Project MariaDB Schema
-- -----------------------------------------------------
-- Table `user`
-- -----------------------------------------------------
CREATE TABLE IF NOT EXISTS `user` (
  `user_id` BIGINT NOT NULL AUTO_INCREMENT COMMENT '사용자 고유 ID',
  `nickname` VARCHAR(255) NOT NULL COMMENT '닉네임',
  `email` VARCHAR(255) NULL COMMENT '이메일',
  `profile_image_url` VARCHAR(1024) NULL COMMENT '프로필 이미지 URL',
  `provider` ENUM('KAKAO') NOT NULL COMMENT 'OAuth 공급자 (로그인 방식)',
  `provider_id` VARCHAR(255) NOT NULL COMMENT 'OAuth 공급자에서 부여한 ID',
  `required_agree` BOOLEAN NOT NULL DEFAULT FALSE COMMENT '필수 약관 동의 여부',
  `optional_agree` BOOLEAN NOT NULL DEFAULT FALSE COMMENT '선택 약관 동의 여부',
  `total_score` BIGINT NOT NULL DEFAULT 0 COMMENT '사용자 누적 점수',
  `created_at` TIMESTAMP NULL DEFAULT CURRENT_TIMESTAMP COMMENT '가입 시간',
  `updated_at` TIMESTAMP NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '마지막 수정 시간',
  PRIMARY KEY (`user_id`),
  UNIQUE INDEX `email_UNIQUE` (`email` ASC) VISIBLE,
  UNIQUE INDEX `uk_provider_id` (`provider`, `provider_id`) VISIBLE)
ENGINE = InnoDB
COMMENT = '사용자 기본 정보 테이블';


-- -----------------------------------------------------
-- Table `terms`
-- -----------------------------------------------------
CREATE TABLE IF NOT EXISTS `terms` (
  `terms_id` BIGINT NOT NULL AUTO_INCREMENT COMMENT '약관 ID',
  `title` VARCHAR(255) NOT NULL COMMENT '약관 제목',
  `content` TEXT NOT NULL COMMENT '약관 내용',
  `version` VARCHAR(50) NOT NULL COMMENT '약관 버전',
  `is_required` TINYINT NOT NULL COMMENT '필수 약관 여부',
  `created_at` TIMESTAMP NULL DEFAULT CURRENT_TIMESTAMP COMMENT '약관 등록 일시',
  `updated_at` TIMESTAMP NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '마지막 수정 시간',
  PRIMARY KEY (`terms_id`))
ENGINE = InnoDB
COMMENT = '약관 정보 및 버전 관리 테이블';


-- -----------------------------------------------------
-- Table `terms_agreement`
-- -----------------------------------------------------
CREATE TABLE IF NOT EXISTS `terms_agreement` (
  `terms_agreement_id` BIGINT NOT NULL AUTO_INCREMENT COMMENT '약관별 동의여부 ID',
  `user_id` BIGINT NOT NULL COMMENT '사용자 ID (FK)',
  `terms_id` BIGINT NOT NULL COMMENT '약관 ID (FK)',
  `agreed` TINYINT NOT NULL COMMENT '동의여부 (T/F)',
  `agreed_at` TIMESTAMP NULL COMMENT '동의일시',
  `updated_at` TIMESTAMP NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '마지막 수정시간',
  PRIMARY KEY (`terms_agreement_id`),
  INDEX `fk_terms_agreement_user_idx` (`user_id` ASC) VISIBLE,
  INDEX `fk_terms_agreement_terms_idx` (`terms_id` ASC) VISIBLE,
  CONSTRAINT `fk_terms_agreement_user`
    FOREIGN KEY (`user_id`)
    REFERENCES `user` (`user_id`)
    ON DELETE CASCADE
    ON UPDATE NO ACTION,
  CONSTRAINT `fk_terms_agreement_terms`
    FOREIGN KEY (`terms_id`)
    REFERENCES `terms` (`terms_id`)
    ON DELETE CASCADE
    ON UPDATE NO ACTION)
ENGINE = InnoDB
COMMENT = '사용자별 약관 동의 이력 테이블';


-- -----------------------------------------------------
-- Table `sign_api`
-- -----------------------------------------------------
CREATE TABLE IF NOT EXISTS `sign_api` (
  `id` BIGINT NOT NULL AUTO_INCREMENT COMMENT '국어원 API 수어단어 ID',
  `title` VARCHAR(255) NOT NULL COMMENT '이름',
  `url` VARCHAR(1024) NOT NULL COMMENT '영상 URL',
  `sign_description` TEXT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NULL COMMENT '수어 설명',
  `category_type` VARCHAR(100) NULL COMMENT '카테고리',
  PRIMARY KEY (`id`))
ENGINE = InnoDB
COMMENT = '국어원 API 기반 수어 단어 리스트 (Staging Table)';


-- -----------------------------------------------------
-- Table `sign`
-- -----------------------------------------------------
CREATE TABLE IF NOT EXISTS `sign` (
  `id` BIGINT NOT NULL AUTO_INCREMENT COMMENT '수어 데이터 ID',
  `title` VARCHAR(255) NOT NULL COMMENT '단어 제목',
  `url` VARCHAR(1024) NOT NULL COMMENT '영상 URL',
  `sign_description` TEXT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci NULL COMMENT '수어 설명',
  `category_type` VARCHAR(100) NULL COMMENT '카테고리',
  `learning_status` ENUM('PENDING', 'IN_PROGRESS', 'COMPLETED') NOT NULL DEFAULT 'PENDING' COMMENT '모델 학습 상태',
  PRIMARY KEY (`id`),
  INDEX `idx_learning_status` (`learning_status` ASC) VISIBLE)
ENGINE = InnoDB
COMMENT = '수어 데이터 관리 테이블 (AI 모델 학습 상태 포함)';


-- -----------------------------------------------------
-- Table `quiz_word`
-- -----------------------------------------------------
CREATE TABLE IF NOT EXISTS `quiz_word` (
  `quiz_word_id` BIGINT NOT NULL AUTO_INCREMENT COMMENT '퀴즈용 단어 ID',
  `sign_id` BIGINT NOT NULL COMMENT '학습 완료된 수어 단어 ID (FK)',
  PRIMARY KEY (`quiz_word_id`),
  UNIQUE INDEX `uk_quiz_word_sign_id` (`sign_id` ASC) VISIBLE,
  CONSTRAINT `fk_quiz_word_sign`
    FOREIGN KEY (`sign_id`)
    REFERENCES `sign` (`id`)
    ON DELETE CASCADE
    ON UPDATE NO ACTION)
ENGINE = InnoDB
COMMENT = 'AI 학습 완료된 퀴즈용 단어 테이블';


-- -----------------------------------------------------
-- Table `game_room`
-- -----------------------------------------------------
CREATE TABLE IF NOT EXISTS `game_room` (
  `game_room_id` BIGINT NOT NULL AUTO_INCREMENT COMMENT '게임방 ID',
  `game_title` VARCHAR(255) NOT NULL COMMENT '게임방 제목',
  `host_id` BIGINT NOT NULL COMMENT '방장 User ID (FK)',
  `max_participants` INT NULL DEFAULT 4 COMMENT '게임방 최대 인원 수 (4 고정)',
  `current_participants` INT NULL DEFAULT 1 COMMENT '게임방 현재 인원 수',
  `current_round` INT NOT NULL DEFAULT 1 COMMENT '현재 라운드',
  `status` ENUM('WAITING', 'PLAYING', 'FINISHED') NOT NULL COMMENT '상태값 (대기중/진행중/종료)',
  `created_at` TIMESTAMP NULL DEFAULT CURRENT_TIMESTAMP COMMENT '생성 시간',
  `updated_at` TIMESTAMP NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '수정 시간',
  PRIMARY KEY (`game_room_id`),
  INDEX `fk_game_room_user_idx` (`host_id` ASC) VISIBLE,
  CONSTRAINT `fk_game_room_user`
    FOREIGN KEY (`host_id`)
    REFERENCES `user` (`user_id`)
    ON DELETE NO ACTION
    ON UPDATE NO ACTION)
ENGINE = InnoDB
COMMENT = '실시간 퀴즈 게임방 테이블';


-- -----------------------------------------------------
-- Table `game_participant`
-- -----------------------------------------------------
CREATE TABLE IF NOT EXISTS `game_participant` (
  `game_participant_id` BIGINT NOT NULL AUTO_INCREMENT COMMENT '게임 참가자 ID',
  `game_room_id` BIGINT NOT NULL COMMENT '게임방 ID (FK)',
  `participant_id` BIGINT NOT NULL COMMENT '참가자 User ID (FK)',
  `is_ready` BOOLEAN NOT NULL DEFAULT FALSE COMMENT '준비 상태 (방장은 항상 FALSE)',
  `is_host` BOOLEAN NOT NULL COMMENT '방장 여부',
  `joined_at` TIMESTAMP NULL DEFAULT CURRENT_TIMESTAMP COMMENT '입장 시각',
  `updated_at` TIMESTAMP NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '수정 시간',
  PRIMARY KEY (`game_participant_id`),
  INDEX `fk_game_participant_game_room_idx` (`game_room_id` ASC) VISIBLE,
  INDEX `fk_game_participant_user_idx` (`participant_id` ASC) VISIBLE,
  CONSTRAINT `fk_game_participant_game_room`
    FOREIGN KEY (`game_room_id`)
    REFERENCES `game_room` (`game_room_id`)
    ON DELETE CASCADE
    ON UPDATE NO ACTION,
  CONSTRAINT `fk_game_participant_user`
    FOREIGN KEY (`participant_id`)
    REFERENCES `user` (`user_id`)
    ON DELETE CASCADE
    ON UPDATE NO ACTION)
ENGINE = InnoDB
COMMENT = '게임 참가자 목록 테이블';


-- -----------------------------------------------------
-- Table `game_history`
-- -----------------------------------------------------
CREATE TABLE IF NOT EXISTS `game_history` (
  `game_history_id` BIGINT NOT NULL AUTO_INCREMENT COMMENT '게임 기록 고유 ID',
  `game_room_id` BIGINT NOT NULL COMMENT '게임방 ID (FK)',
  `participant_id` BIGINT NOT NULL COMMENT '유저 ID (FK)',
  `score` INT NOT NULL COMMENT '점수',
  `round` INT NOT NULL COMMENT '회차',
  `created_at` TIMESTAMP NULL DEFAULT CURRENT_TIMESTAMP COMMENT '생성 시간',
  `updated_at` TIMESTAMP NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '수정 시간',
  PRIMARY KEY (`game_history_id`),
  INDEX `fk_game_history_game_room_idx` (`game_room_id` ASC) VISIBLE,
  INDEX `fk_game_history_user_idx` (`participant_id` ASC) VISIBLE,
  CONSTRAINT `fk_game_history_game_room`
    FOREIGN KEY (`game_room_id`)
    REFERENCES `game_room` (`game_room_id`)
    ON DELETE CASCADE
    ON UPDATE NO ACTION,
  CONSTRAINT `fk_game_history_user`
    FOREIGN KEY (`participant_id`)
    REFERENCES `user` (`user_id`)
    ON DELETE CASCADE
    ON UPDATE NO ACTION)
ENGINE = InnoDB
COMMENT = '게임 기록 테이블';

```


-----

### 2\. 설계 철학 및 주요 결정사항

본 스키마는 **데이터의 일관성**, **조회 성능**, 그리고 **유지보수성** 사이의 균형을 맞추는 것을 목표로 설계되었습니다.

#### 2.1. 조회 성능을 위한 비정규화: `current_participants` & `is_host`

* **설계 결정**: `game_room` 테이블에 `current_participants`(현재 인원 수)를, `game_participant` 테이블에 `is_host`(방장 여부) 컬럼을 유지하기로 결정했습니다.
* **의도**:
  * 메인 페이지에서 여러 게임방 목록을 조회할 때, 각 방의 인원수를 실시간으로 `COUNT`하는 대신 저장된 값을 바로 읽어 **조회 성능을 최적화**하기 위함입니다.
  * 클라이언트(화면)에서 방장 여부를 쉽게 판단할 수 있도록 `is_host` 값을 직접 제공하여 **API 응답 구조를 단순화**하기 위함입니다.
* **주의사항**: 이 방식은 데이터 중복을 허용하므로, 데이터의 일관성을 유지하기 위해 **반드시 트랜잭션(Transaction)을 사용**해야 합니다. (아래 3.1. 트랜잭션 관리 참조)

#### 2.2. 데이터 파이프라인 설계: `sign_api` → `sign` → `quiz_word`

* **설계 결정**: 수어 데이터를 3단계 파이프라인으로 관리하기로 결정했습니다.
* **데이터 흐름**:
  1. `sign_api`: 외부 API에서 받은 원본 데이터를 가공 없이 저장 (Staging Table)
  2. `sign`: 가공된 수어 데이터와 AI 모델 학습 상태(`learning_status`) 관리
  3. `quiz_word`: 학습 완료(`COMPLETED`) 상태인 데이터만 퀴즈용으로 활용
* **의도**:
  * **데이터 품질 관리**: 원본 데이터 보존과 가공 데이터 분리로 데이터 무결성 보장
  * **성능 최적화**: 퀴즈 출제 시 학습 완료된 소수의 데이터만 조회하여 성능 향상
  * **AI 모델 연동**: `learning_status`를 통해 AI 모델 학습 진행 상황 추적 가능

#### 2.3. OAuth2 인증 시스템 설계

* **설계 결정**: `user` 테이블에 OAuth2 기반 인증 정보를 저장하도록 설계했습니다.
* **주요 컬럼**:
  * `provider`: OAuth 공급자 (`ENUM('KAKAO')`)
  * `provider_id`: OAuth 공급자에서 부여한 고유 ID
  * `required_agree`, `optional_agree`: 약관 동의 상태를 사용자 테이블에서 직접 관리
* **의도**:
  * **보안 강화**: `provider`와 `provider_id` 조합으로 유니크 제약조건 설정
  * **성능 최적화**: 약관 동의 상태를 별도 조인 없이 바로 조회 가능
  * **확장성**: 향후 다른 OAuth 공급자 추가 시 `ENUM` 값만 확장하면 됨

#### 2.4. 게임 상태 관리 최적화

* **설계 결정**: `game_room` 테이블의 `status` 값을 실제 구현에 맞게 조정했습니다.
* **상태 값**:
  * `WAITING`: 참여자 모집 중
  * `PLAYING`: 게임 진행 중 (기존 `IN_PROGRESS`에서 변경)
  * `FINISHED`: 게임 종료
* **추가 필드**: `current_round`를 통해 게임 진행 상황 추적

-----

### 3\. 애플리케이션 구현 가이드

아래 가이드는 데이터베이스를 안정적으로 운영하기 위해 백엔드 애플리케이션 개발 시 반드시 지켜야 할 규칙입니다.

#### 3.1. 트랜잭션(Transaction) 관리: 데이터 일관성 보장의 핵심

`current_participants`와 `is_host`처럼 여러 테이블에 걸쳐 데이터의 상태를 동시에 변경해야 하는 모든 로직은 **반드시 하나의 트랜잭션**으로 묶어야 합니다.

* **주요 대상 로직**:

  * **유저가 방에 참여/퇴장하는 로직**: `game_participant`에 데이터 추가/삭제 + `game_room`의 `current_participants` 값 증감.
  * **방장이 변경되는 로직**: `game_room`의 `host_id` 변경 + 이전 방장과 새 방장의 `is_host` 값 변경.

* **Spring Boot `@Transactional` 어노테이션 활용**:

  * 관련 로직을 처리하는 서비스 메소드에 `@Transactional` 어노테이션을 추가하면, 해당 메소드 내의 모든 DB 작업이 하나의 트랜잭션으로 처리됩니다.
  * 만약 중간에 하나라도 실패하면 모든 작업이 자동으로 원상 복구(Rollback)되어 데이터가 꼬이는 문제를 방지할 수 있습니다.

  <!-- end list -->

  ```java
  // 예시: 유저가 게임방에 참여하는 서비스 로직 (실제 구현 기반)
  import org.springframework.transaction.annotation.Transactional;

  @Service
  public class GameRoomJoinService {
      // ... repository 의존성 주입

      @Transactional
      public JoinRoomResponse joinRoom(Long userId, Long gameRoomId) {
          // 1. 게임방과 사용자 조회
          GameRoom room = gameRoomRepository.findById(gameRoomId)
              .orElseThrow(() -> new BusinessException(ErrorCode.ROOM_NOT_FOUND));
          User user = userRepository.findById(userId)
              .orElseThrow(() -> new BusinessException(ErrorCode.USER_NOT_FOUND));
          
          // 2. 참여자 목록에 유저 추가
          GameParticipant participant = GameParticipant.builder()
              .gameRoom(room)
              .participant(user)
              .isHost(room.getHost().getId().equals(userId)) // 방장 여부 확인
              .isReady(false) // 방장도 초기에는 준비 상태 false
              .build();
          participantRepository.save(participant);

          // 3. 게임방 현재 인원 수 1 증가
          room.incrementParticipants();
          gameRoomRepository.save(room);
          
          // 4. 응답 데이터 생성 및 반환
          return JoinRoomResponse.of(room, participantResponses);
          
          // @Transactional에 의해, 위 모든 작업은 함께 성공하거나 함께 실패합니다.
      }
  }
  ```

-----

### 4\. 향후 유지보수 가이드

#### 4.1. OAuth 공급자 추가

* **상황**: 카카오 로그인 외에 'GOOGLE' 로그인을 추가하고 싶은 경우.
* **조치**: `ALTER TABLE` 구문을 사용하여 `user` 테이블의 `provider` 컬럼 정의를 직접 수정해야 합니다.
* **SQL 예시**:
  ```sql
  ALTER TABLE user MODIFY COLUMN provider ENUM('KAKAO', 'GOOGLE') NOT NULL;
  ```

#### 4.2. AI 모델 학습 상태 관리

* **상황**: 새로운 학습 상태를 추가하고 싶은 경우 (예: 'FAILED', 'REVIEWING').
* **조치**: `sign` 테이블의 `learning_status` ENUM 값을 확장합니다.
* **SQL 예시**:
  ```sql
  ALTER TABLE sign MODIFY COLUMN learning_status 
  ENUM('PENDING', 'IN_PROGRESS', 'COMPLETED', 'FAILED', 'REVIEWING') 
  NOT NULL DEFAULT 'PENDING';
  ```

#### 4.3. 성능 개선을 위한 인덱스(Index) 추가

* **상황**: 서비스가 성장하여 특정 데이터 조회(SELECT)가 느려지는 경우.
* **조치**: `WHERE` 절에 자주 사용되는 컬럼에 인덱스를 추가하여 검색 속도를 향상시킬 수 있습니다.
* **SQL 예시**:
  ```sql
  -- 대기중인 방 목록을 빠르게 조회하기 위해 status 컬럼에 인덱스 추가
  CREATE INDEX idx_gameroom_status ON game_room (status);

  -- 특정 유저가 참여했던 게임 기록을 빠르게 조회하기 위해 participant_id에 인덱스 추가
  CREATE INDEX idx_gamehistory_participant ON game_history (participant_id);

  -- 학습 완료된 수어 데이터를 빠르게 조회하기 위한 인덱스
   CREATE INDEX idx_learning_status ON sign (learning_status);

  -- OAuth 공급자별 사용자 조회를 위한 복합 인덱스
   CREATE UNIQUE INDEX uk_provider_id ON user (provider, provider_id);
  ```


## 변경 이력

| 버전     | 날짜         | 변경 내용     | 작성자 |
|--------|------------|-----------|-----|
| v1.0.0 | 2025.10.08 | 초기 문서 작성 | 고동현 |
| v1.1.0 | 2025.10.21 | 백엔드 엔티티 기반 DDL 업데이트, OAuth2 인증 시스템 반영, 데이터 파이프라인 설계 추가 | 고동현 |