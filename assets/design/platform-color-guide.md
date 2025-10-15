# SignBell 플랫폼 컬러 가이드 (JSX & SCSS 최적화)

---

## 문서 정보
- **문서명**: SignBell 플랫폼 컬러 가이드
- **버전**: v1.0.0
- **작성일**: 2025.10.16
- **작성자**: [신동준](https://github.com/sdj3959)
- **최종 수정일**: 2025.10.16

---

## 프로젝트 정보
- **팀명**: SynergySign
- **프로젝트명**: SignBell
- **플랫폼명**: SignBell
- **컨셉**: AI 기반 실시간 수어 학습 플랫폼
- **서비스 개요**: AI 기술을 통해 배우고, 게임을 통해 함께 즐기는 실시간 양방향 수어 학습 플랫폼

---

## 컬러 팔레트 (SCSS 변수 및 CSS 커스텀 속성)

```scss
// SCSS Variables for easier management and calculations
$primary-color-value: #BBDEFB; // 라이트 블루
$secondary-color-value: #90CAF9; // 스카이 블루
$accent-color-value: #FFCC80; // 소프트 오렌지
$text-color-value: #212121; // 다크 그레이
$background-color-value: #F8F8F8; // 울트라 라이트 그레이

$success-color-value: #66BB6A; // 성공 (그린)
$warning-color-value: #FFCC80; // 경고/주의 (소프트 오렌지)
$error-color-value: #EF5350; // 에러/실패 (레드)
$info-color-value: #29B6F6; // 정보 (라이트 스카이 블루)

// CSS Custom Properties (Variables) for runtime access and theming
:root {
  /* Primary Colors */
  --primary-color: #{$primary-color-value};
  --secondary-color: #{$secondary-color-value};

  /* Supporting Colors */
  --text-color: #{$text-color-value};
  --background-color: #{$background-color-value};
  --accent-color: #{$accent-color-value};

  /* Semantic Colors */
  --success-color: #{$success-color-value};
  --warning-color: #{$warning-color-value};
  --error-color: #{$error-color-value};
  --info-color: #{$info-color-value};

  /* Typography */
  --font-family: 'SF Pro Display', 'Apple SD Gothic Neo', sans-serif;

  /* Interaction States (using SCSS functions for dynamic values) */
  --state-default: var(--secondary-color);
  --state-hover: #{rgba($secondary-color-value, 0.8)}; // 20% 투명도
  --state-active: var(--accent-color);
  --state-disabled: #{rgba($secondary-color-value, 0.4)}; // 60% 투명도
  --state-focus: var(--primary-color);
}
```

---

## 컬러 선정 이유

### Primary Color: #BBDEFB (라이트 블루)
**선정 이유:**
- **포용성과 신뢰**: 하늘을 연상시키는 연한 파란색은 SignBell의 핵심 가치인 '소통의 장벽 해소'와 '포용적 사회 기여'를 시각적으로 표현합니다. 사용자에게 안정감과 신뢰감을 줍니다.
- **고요함과 집중**: 학습 플랫폼으로서 사용자가 수어 학습에 집중할 수 있도록 차분하고 고요한 분위기를 조성합니다.
- **iOS 트렌드 반영**: iOS의 미니멀하고 깨끗한 디자인 언어와 잘 어울리며, 부드러운 그라데이션 및 블러 효과와 조화롭게 사용될 수 있습니다.
- **희망의 종소리**: 'SignBell'이라는 이름처럼, 새로운 시작과 희망을 상징하는 색상으로, 수어 학습의 긍정적인 경험을 강조합니다.

### Secondary Color: #90CAF9 (스카이 블루)
**선정 이유:**
- **활기찬 소통**: Primary Color보다 약간 더 진한 스카이 블루는 실시간 퀴즈, 게이미피케이션 등 '함께 즐기는' SignBell의 활기찬 소통 요소를 표현합니다.
- **동기 부여**: 학습에 대한 동기를 부여하고, 긍정적인 인터랙션을 유도하는 데 효과적입니다.
- **시각적 계층**: Primary Color와 함께 사용하여 UI에 시각적 계층을 부여하고, 중요한 정보나 액션 포인트를 부드럽게 강조합니다.
- **브랜드 일관성**: Primary Color와의 조화를 통해 SignBell의 전체적인 브랜드 아이덴티티를 강화하면서도 다채로운 느낌을 줍니다.

### Accent Color: #FFCC80 (소프트 오렌지)
**선정 이유:**
- **즐거움과 성취**: AI 기반 피드백, 퀴즈 정답, 미션 완료 등 '즐거움을 통한 학습'과 '성취감'을 상징하는 따뜻하고 밝은 색상입니다.
- **행동 유도**: CTA(Call To Action) 버튼, 중요 알림, 긍정적인 피드백 메시지 등 사용자의 시선을 즉각적으로 끌고 행동을 유도하는 데 사용됩니다.
- **시각적 대비**: 차분한 블루 계열의 주 색상과 대비되어, 중요한 요소들을 더욱 돋보이게 하며 활력을 더합니다.
- **iOS 알림 스타일**: iOS에서 긍정적인 알림이나 강조에 사용되는 색상과 유사하여 사용자에게 친숙하고 직관적인 경험을 제공합니다.

### Text Color: #212121 (다크 그레이)
**선정 이유:**
- **최적화된 가독성**: 순수한 검정(#000000)보다 부드러우면서도 충분한 명도 대비를 확보하여 장시간 학습 시에도 눈의 피로를 최소화합니다.
- **현대적 세련됨**: iOS의 텍스트 스타일에 부합하는 모던하고 세련된 느낌을 주며, 전체적인 디자인 톤과 조화를 이룹니다.
- **정보 위계**: 다양한 텍스트 크기와 굵기를 통해 정보의 위계를 명확하게 전달하는 데 용이합니다.

### Background Color: #F8F8F8 (울트라 라이트 그레이)
**선정 이유:**
- **미니멀리즘**: iOS 디자인의 핵심인 미니멀하고 깔끔한 배경을 제공하여 콘텐츠(수어 영상, 퀴즈)에 집중할 수 있도록 돕습니다.
- **눈의 편안함**: 순백색보다 부드러워 눈의 피로를 줄이고, 장시간 학습 환경에 적합합니다.
- **콘텐츠 강조**: AI 기반 거울 모드의 캠 화면, 퀴즈 UI 등 핵심 콘텐츠를 돋보이게 하는 중립적인 배경 역할을 합니다.
- **다크 모드 확장성**: 라이트 모드에서 최적의 경험을 제공하며, 향후 다크 모드 구현 시에도 자연스러운 전환이 가능하도록 설계되었습니다.

---

## 60-30-10 컬러 비율 가이드

### 60% - Primary Color (#BBDEFB) 영역
```
- [AUTH-000] 랜딩 페이지: 배경, 서비스 로고/이미지 배경
- [AUTH-002] 약관 동의 페이지: 전체 배경, 약관 내용 스크롤 영역 배경
- [MAIN-001] 메인 페이지: 전체 배경, 헤더 배경, 사용자 프로필 카드 배경
- [STUDY-001] 개인 학습 사이드바: 전체 배경, 단어 리스트 배경
- [STUDY-002] 단어 상세 모달: 모달 배경, 수어 영상 플레이어 배경
- [STUDY-003] 수어 연습 영상 데이터 제공 페이지 (1단계): 전체 배경, 수어 영상 플레이어 배경
- [STUDY-004] 수어 연습 영상 데이터 제공 페이지 (2단계): 전체 배경, 웹캠 영상 영역 배경
- [QUIZ-001] 실시간 퀴즈 사이드바: 전체 배경, 방 목록 배경
- [QUIZ-002] 방 만들기 모달: 모달 배경
- [QUIZ-003] 퀴즈 대기방: 전체 배경, 참여자 정보 카드 배경 (빈 자리 포함)
- [QUIZ-004] 퀴즈 진행 페이지 (문제 제시): 전체 배경, 플레이어 카드 배경
- [QUIZ-005] 퀴즈 진행 페이지 (도전자 차례): 전체 배경, 관전자 카드 배경
- [QUIZ-006] 게임 결과 모달: 모달 배경
- [COMMON-001] 나가기 확인 모달: 모달 배경
```

### 30% - Secondary Color (#90CAF9) 영역
```
- [AUTH-002] 약관 동의 페이지: 체크박스 활성화 상태, 제출 버튼 (비활성화 시)
- [MAIN-001] 메인 페이지: 개인 학습/실시간 퀴즈 버튼 배경, 마이페이지/로그아웃 버튼 아이콘
- [STUDY-001] 개인 학습 사이드바: 단어 검색 입력 칸 테두리, 정렬 조건 콤보박스 테두리, 개별 단어 카드 테두리 (호버 시)
- [STUDY-002] 단어 상세 모달: 모달 닫기 버튼, 거울모드로 연습하기 펼치기 바 배경
- [STUDY-003] 수어 연습 영상 데이터 제공 페이지: 방 정보 섹션 배경, 준비/시작 버튼 (비활성화 시), 캠 상태 점 아이콘 (ON)
- [QUIZ-001] 실시간 퀴즈 사이드바: 방 번호 검색 입력 칸 테두리, 방 만들기 버튼 (비활성화 시), 개별 방 카드 테두리 (호버 시)
- [QUIZ-002] 방 만들기 모달: 생성하기 버튼 (비활성화 시)
- [QUIZ-003] 퀴즈 대기방: 준비/시작 버튼 (비활성화 시), 캠 상태 점 아이콘 (ON)
- [QUIZ-004] 퀴즈 진행 페이지 (문제 제시): 문제 도전하기 버튼 (비활성화 시)
- [QUIZ-005] 퀴즈 진행 페이지 (도전자 차례): 도전자 메인 카드 테두리
- [COMMON-001] 나가기 확인 모달: '취소' 버튼
```

### 10% - Accent Color (#FFCC80) 영역
```
- [AUTH-000] 랜딩 페이지: 소셜 계정으로 시작하기 버튼 배경
- [AUTH-002] 약관 동의 페이지: 제출 버튼 (활성화 시)
- [MAIN-001] 메인 페이지: 사용자 점수 텍스트, 닉네임 수정 아이콘
- [STUDY-002] 단어 상세 모달: 영상 제공하고 보상받으러 가기 버튼
- [STUDY-003] 수어 연습 영상 데이터 제공 페이지 (1단계): Next 버튼
- [STUDY-004] 수어 연습 영상 데이터 제공 페이지 (2단계): 녹화 상태 메시지 (긍정적 메시지), Prev 버튼 (호버 시)
- [STUDY-005] 영상 촬영 데이터 제공 안내 모달: 메인으로 버튼
- [QUIZ-001] 실시간 퀴즈 사이드바: 방 만들기 버튼 (활성화 시)
- [QUIZ-002] 방 만들기 모달: 생성하기 버튼 (활성화 시)
- [QUIZ-003] 퀴즈 대기방: 준비/시작 버튼 (활성화 시), 계정 통합 점수 텍스트
- [QUIZ-004] 퀴즈 진행 페이지 (문제 제시): 문제 도전하기 버튼 (활성화 시), 게임 내 점수 텍스트
- [QUIZ-005] 퀴즈 진행 페이지 (도전자 차례): 게임 내 점수 텍스트, 모션 인식/정답 안내 (긍정적 메시지)
- [QUIZ-006] 게임 결과 모달: 1~3등 아이콘, 대기실로 돌아가기 버튼
- [COMMON-001] 나가기 확인 모달: '나가기' 버튼
```

---

## 브랜드 철학과 컬러의 연결

**"SynergySign → SignBell"**의 브랜드 스토리가 컬러에 구현된 방식:

-   **라이트 블루**: 'SynergySign'의 'Synergy'처럼 기술과 수어가 만나 시너지를 창출하고, 'SignBell'의 'Bell'처럼 수어 학습의 시작을 알리는 '포용성'과 '희망'을 상징합니다.
-   **스카이 블루**: 'Sign'처럼 수어 소통의 활기찬 에너지를 표현하며, 'Synergy'를 통해 팀원 간의 협력과 소통의 즐거움을 강조합니다.
-   **소프트 오렌지**: AI 기반 학습과 게이미피케이션을 통해 얻는 '즐거움'과 '성취감'을 직관적으로 전달하며, '희망의 종소리'가 울려 퍼지는 긍정적인 경험을 나타냅니다.

이 삼색 조합을 통해 **"포용적이고 신뢰할 수 있으며, 활기찬 소통과 즐거운 학습 경험을 제공하는"** SignBell만의 독창적인 브랜드 아이덴티티를 완성했습니다.

---

## 기술적 고려사항

### 접근성 (Accessibility)
-   **명도 대비 준수**: 모든 텍스트와 UI 요소는 WCAG 2.1 AA 기준(4.5:1 이상)을 충족하여 시각적 불편을 최소화합니다.
-   **색상 외 정보 제공**: 퀴즈 정답/오답, 학습 진행 상태 등은 색상뿐만 아니라 아이콘, 텍스트 라벨, 진동 피드백 등을 함께 제공하여 색맹/색약 사용자도 정보를 명확히 인지할 수 있도록 합니다.
-   **다크 모드 지원**: 향후 다크 모드 구현 시, 색상 팔레트를 유연하게 전환할 수 있도록 컬러 변수를 체계적으로 관리합니다.

### iOS 디자인 시스템 통합
-   **SF Pro Font**: iOS 기본 폰트인 'SF Pro Display'를 우선적으로 사용하여 시스템과의 일관성을 유지하고 뛰어난 가독성을 제공합니다.
-   **블러 및 투명도**: iOS의 Glassmorphism(글래스모피즘) 디자인 트렌드를 반영하여, 배경 블러 및 투명도 효과를 활용한 UI 요소를 설계할 수 있도록 색상 팔레트를 구성합니다.
-   **시스템 아이콘 활용**: iOS 시스템 아이콘과 컬러 팔레트가 조화롭게 어우러지도록 설계하여 사용자에게 익숙하고 직관적인 경험을 제공합니다.

### 브랜드 일관성
-   **크로스 플랫폼**: 웹과 iOS 앱에서 동일한 브랜드 컬러 경험을 제공하여 일관된 아이덴티티를 유지합니다.
-   **마케팅 자료**: SignBell의 모든 마케팅 자료(웹사이트, 소셜 미디어, 홍보물)에서도 이 컬러 가이드를 준수하여 통일된 브랜드 이미지를 구축합니다.
-   **협력사 연동**: 잠재적 협력사(예: 교육 기관, 농인 단체)와의 협업 시에도 SignBell 고유의 색상 체계를 유지하여 브랜드 정체성을 확고히 합니다.

---

## 인터랙션 상태 컬러 가이드 (SCSS 함수 활용)

```scss
// SCSS Variables for easier management and calculations (재사용을 위해 위에 정의된 변수들을 활용)
$secondary-color-for-states: #90CAF9; // 스카이 블루
$accent-color-for-states: #FFCC80; // 소프트 오렌지
$primary-color-for-states: #BBDEFB; // 라이트 블루

// CSS Custom Properties (Variables) for runtime access and theming
:root {
  /* Interaction States */
  --state-default: var(--secondary-color); // 기본 버튼/링크 색상
  --state-hover: #{rgba($secondary-color-for-states, 0.8)}; // 호버 시 20% 투명도
  --state-active: var(--accent-color); // 클릭/활성화 시 액센트 컬러
  --state-disabled: #{rgba($secondary-color-for-states, 0.4)}; // 비활성화 시 60% 투명도
  --state-focus: var(--primary-color); // 포커스 시 Primary Color
}