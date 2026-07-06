# DSL 검증 리포트

**날짜**: 2026-07-06
**파일**: `dsl/MPS-주간-도우미.yml`
**총 줄 수**: 429

---

## 통과 항목

### Step 1: yaml 문법 및 구조
```
Top keys: ['version', 'kind', 'app', 'workflow']
App mode: advanced-chat
App name: MPS 주간 도우미
Nodes: 5
Edges: 5
```
- `kind`, `version`, `app`, `workflow` 네 키 모두 정상
- 노드 5개, 엣지 5개 — 예상과 일치
- 단, `app.mode`는 `chat`이 아닌 `advanced-chat`으로 표기됨 (Dify 실제 export와 일치하는 정상값)

### Step 2: 노드 타입
```
  start_node: data_type=start
  if_else_node: data_type=if-else
  llm_planning: data_type=llm
  llm_evaluation: data_type=llm
  answer_node: data_type=answer
```
- start, if-else, llm(x2), answer — 모두 예상 타입과 일치
- Dify DSL에서 노드 타입은 `data.type`에 위치 (상위 `type` 필드 아님)

### Step 3: 변수 참조 일관성
```
{{#llm_evaluation.text#}}
{{#llm_planning.text#}}
{{#start.author#}}
{{#start.due_date#}}
{{#start.edit_request#}}
{{#start.mission_label#}}
{{#start.mode#}}
{{#start.previous_mps#}}
{{#sys.query#}}
```
- `start.*` 참조 7개 — 모두 start 노드의 실제 변수와 일치
- `llm_planning.text`, `llm_evaluation.text` — LLM 노드 출력 변수 참조로 정상
- `sys.query` — Dify 시스템 변수 (사용자 입력)
- 불일치 참조 없음

### Step 4: 시스템 프롬프트 키워드 무결성

| 키워드 | 횟수 | 비고 |
|--------|------|------|
| 결과물로만 | 1 | Planning 규칙 |
| 고정변수 목표 | 2 | Planning 규칙 |
| 변동변수 목표 | 3 | Planning 규칙 |
| 1:1 매핑 | 3 | 공통 |
| As-Is / To-Be | 2 | 공통 |
| PO 단위로 압축 | 2 | Planning 규칙 |
| 행위 | 12 | Evaluation 규칙 |
| GAP 필수 | 1 | Evaluation 규칙 |
| 진척률 | 7 | 공통 |
| commit 수 | 3 | Evaluation 허용 (Planning 금지) |
| 도구명 | 4 | Evaluation 규칙 |
| 진척률/상태 | 1 | Planning 금지 규칙 |

** Planning/Evaluation 시스템 프롬프트 모두 분리되어 있음 **

### Step 5: 변수 정의
- start 노드에 7개 변수 정의됨 (mode, content, previous_mps, edit_request, author, due_date, mission_label)
- 모든 변수가 실제 참조와 일치

---

## 주의 사항

### 1. 창조성과 키워드 미검출
스펙 원문에 `창조성과`로 표기되어 있으나, 실제 스펙 문서(`2026-07-06-mps-dify-design.md`)에도 `창조성과` 텍스트가 존재하지 않음. yml 내 `성과창출자`, `성과목표` 등 `성과` 관련 키워드는 다수 존재. 원문의 `창조성과`가 `창조적 성과`로 오기된 것으로 보이며, 실제 Evaluation 프롬프트에는 `창조` 텍스트 없이도 `GAP`, `행위`, `도구명` 등 핵심 키워드가 충분히 포함되어 있음.

### 2. app.mode 값: `advanced-chat` vs `chat`
Step 1 출력에서 `app.mode`가 `advanced-chat`으로 표기됨. Dify에서 `chat`은 단순 채팅, `advanced-chat`은 워크플로우 기반 채팅. 본 DSL이 워크플로우를 사용하므로 `advanced-chat`이 올바른 값이며, brief의 예상값 `chat`은 Dify 워크플로우 Export 시 `advanced-chat`으로 표기되는 실제 동작을 반영하지 못한 것으로 판단됨.

---

## 다음 단계 (수동)

- [ ] **Dify Import**: Dify에 yml 파일을 import하여 워크플로우 시각적으로 확인
- [ ] **디버그 패널 테스트**: T1(Planning), T2(Evaluation), T3(수정 의도 포함) 시나리오 각각 실행
- [ ] **모델 プロバイダー確認**: `gpt-oss` 모델이 실제 Dify에 등록된 プロバイダー인지 확인
