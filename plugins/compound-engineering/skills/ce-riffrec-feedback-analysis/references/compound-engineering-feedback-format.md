# Compound Engineering 피드백 형식 (Compound Engineering Feedback Format)

Riffrec 증거를 내구성이 있는 브레인스토밍 또는 계획 입력값으로 변환할 때 이 형식을 사용하십시오.

## 발견 사항 (Finding)

```markdown
### F1. <문제 요약 제목>

- **Severity:** P0/P1/P2/P3
- **Observed:** <기록/이벤트/스크린샷에 근거하여 실제로 발생한 상황>
- **Expected:** <사용자가 기대한 것으로 보이는 동작 또는 제품의 정상 동작>
- **Evidence:** <Moment ID 및 스크린샷 링크>
- **Confidence:** High/Medium/Low 및 그 이유
- **Requirement candidates:** R1, R2
```

## 요구 사항 킥오프 (Requirements Kickoff)

```markdown
---
date: YYYY-MM-DD
topic: <주제>
---

# <주제 제목>

## 문제 프레임 (Problem Frame)

<누가 영향을 받는지, 무엇이 변하는지, 그리고 왜 중요한지 작성합니다.>

---

## 행위자 (Actors)

- A1. User: <녹화된 워크플로우에서의 역할>
- A2. Product surface: <테스트 대상 시스템>
- A3. Agent/assistant (해당되는 경우): <워크플로우에서의 역할>

---

## 주요 흐름 (Key Flows)

- F1. 녹화된 피드백 분류
  - **Trigger:** 리뷰 가능한 Riffrec zip 파일이 준비됨.
  - **Actors:** A1, A2
  - **Steps:** <녹화에서 확인된 3-7단계의 제품 동작 단계>
  - **Outcome:** <수정 후 달성되어야 하는 상태>
  - **Covered by:** R1, R2

---

## 요구 사항 (Requirements)

**관찰된 제품 동작**
- R1. <구체적인 제품 동작 요구 사항>

**피드백 증거 및 리뷰 가능성**
- R2. <문제를 관찰 가능하게 만들거나 재발을 방지하기 위한 요구 사항>

---

## 수락 예시 (Acceptance Examples)

- AE1. **R1 충족 여부 확인.** Given <상태>, When <동작>, <결과>.

---

## 성공 기준 (Success Criteria)

- <사용자 측면의 결과>
- <하류 에이전트 전달 품질>

---

## 범위 경계 (Scope Boundaries)

- <의도적인 비목표(non-goal)>

---

## 주요 결정 사항 (Key Decisions)

- <결정 사항>: <근거>

---

## 의존성 / 가정 (Dependencies / Assumptions)

- <중요한 의존성 또는 가정>

---

## 남은 질문 (Outstanding Questions)

### 계획 수립 전 해결 필요

- <계획 수립을 가로막는 제품 관련 질문만 작성>

### 계획 수립 단계로 이월

- [Technical] <코드베이스 탐색 중에 답변하는 것이 더 적절한 질문>

---

## 다음 단계 (Next Steps)

-> /ce-brainstorm 을 호출하여 계획 수립 전 수집된 요구 사항을 확인, 수정 및 재정리하십시오.
```

## 증거 규칙 (Evidence Rules)

- 서술형 주장보다는 Moment ID와 스크린샷 링크를 우선적으로 사용하십시오.
- 스크린샷이 의도를 명확히 증명하지 못하는 경우, 시각적 해석을 추론(inference)으로 표시하십시오.
- 요구 사항은 구현 세부 정보가 아닌 제품의 동작을 설명해야 합니다.
- CE 문서에 로컬 절대 경로를 포함하지 마십시오. 가능한 경우 리포지토리 상대 경로를 사용하십시오.
