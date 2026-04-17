## 1. 전역 가드레일의 한계
`CLAUDE.md`를 프로젝트 루트에 두는 것은 훌륭한 시작점이지만, 애플리케이션 규모가 커질수록 치명적인 단점이 생김

* **컨텍스트 낭비:** 백엔드 코드를 수정할 때 프론트엔드의 CSS 규칙까지 읽어야 하므로 토큰 비용 낭비
* **지시어 충돌:** "모든 에러는 JSON 형태로 반환해라"(백엔드 규칙)와 "모든 에러는 Toast UI로 띄워라"(프론트엔드 규칙)가 충돌하여 에이전트가 혼란(환각)에 빠짐

이를 해결하기 위해 폴더(모듈) 단위로 규칙을 분리하는 **지역 가드레일(Local Guardrails)** 개념이 필요

## 2. 지역 가드레일 구축 방법: 라우팅(Routing)과 위임(Delegation)
AI 에이전트에게 지역 규칙을 학습시키는 가장 확실한 방법은 전역 `CLAUDE.md`를 **목차**로 활용하고, 상세 규칙은 각 폴더의 로컬 문서로 **위임**

### 1단계: 전역 `CLAUDE.md`를 라우터로 축소하기
루트의 `CLAUDE.md`에는 프로젝트의 핵심 목표와 '어느 폴더에서 무슨 규칙을 찾아야 하는지' 길잡이 역할만 남김

>**루트파일:** `/CLAUDE.md`
```text
# 전역 프로젝트 가드레일
이 레포지토리는 프론트엔드와 백엔드가 결합된 모노레포입니다.
작업하는 디렉토리에 따라 아래의 지역 가드레일 문서를 반드시 먼저 읽고 지시를 따르세요.

- **Frontend 작업 시:** `frontend/LOCAL_RULES.md` 참조
- **Backend 작업 시:** ``backend/LOCAL_RULES.md` 참조
- **테스트 코드 작성 시:** `e2e_tests/TEST_RULES.md` 참조
```

### 2단계: 폴더 단위의 지역 규칙(`LOCAL_RULES.md`) 작성
각 폴더 안에 그 생태계에만 적용되는 구체적이고 엄격한 룰을 작성합니다.
(파일명은 `README.md`, `LOCAL_RULES.md`등 자유롭게 설정 가능)

**프론트 엔드 지역 파일:** `/frontend/LOCAL_RULES.md`
```text
# Frontend 지역 가드레일
- **Framework:** React, TailwindCSS
- **상태 관리:** 컴포넌트 내부 상태는 `useState`만 사용하고, 전역 상태는 가급적 피할 것
- **실행 명령:** 작업 완료 후 반드시 `npm run lint`와 `npm run build`를 실행하여 자체적으로 검증할 것
```

**백엔드 지역 파일:** `/backend/LOCAL_RULES.md`
```text
# Backend 지역 가드레일
- **Framework:** Kotlin, Spring Boot
- **
```
