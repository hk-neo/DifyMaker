# MPS Dify Assistant Implementation Plan (yml 직접 작성)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Dify 워크스페이스에서 Import 가능한 MPS 주간 도우미 DSL yml 파일을 직접 작성한다. UI 작업 없이 yml만으로 완결.

**Architecture:** 단일 Dify Chatbot DSL. Start (7 inputs) → If/Else (mode 분기) → {LLM Planning, LLM Evaluation} → Answer. 시스템 프롬프트에 MPS 규칙 직접 인코딩.

**Tech Stack:** Dify DSL (yaml format), Git.

## Global Constraints

- **산출물**: `dsl/MPS-주간-도우미.yml` — 사용자가 Dify에서 "Import from DSL"로 불러오면 즉시 동작해야 함
- **앱 유형**: Chatbot (`app.mode: chat`)
- **입력 7개** (정확한 이름): `mode`, `content`, `previous_mps`, `edit_request`, `author`, `due_date`, `mission_label`
- **2개 LLM 노드**: Planning / Evaluation 각각 전용 시스템 프롬프트
- **모드**: 시작 변수 `mode` (Select) — 자동감지 없음
- **출력**: 순수 마크다운, 코드블록 없음, 인사 없음
- **대화 메모리**: ON
- **v1 범위**: 생성 + 수정. 검증 패스·외부 연동 없음
- **참조 스펙**: [`../specs/2026-07-06-mps-dify-design.md`](../specs/2026-07-06-mps-dify-design.md)
- **참조 프레임워크**: [`../../../MyMPS/가이드/MPS 작성 가이드.md`](../../../MyMPS/가이드/MPS%20작성%20가이드.md)

## File Structure

```
DifyMaker/
├── docs/superpowers/
│   ├── specs/2026-07-06-mps-dify-design.md       [exists]
│   └── plans/2026-07-06-mps-dify-implementation.md  [this file]
├── dsl/
│   ├── MPS-주간-도우미.yml                        [Task 2]
│   └── README.md                                  [Task 4]
├── README.md                                       [Task 4 update]
├── CLAUDE.md                                       [exists]
├── .gitignore                                      [exists]
└── .superpowers/sdd/progress.md                   [exists]
```

---

## Task 1: Dify DSL 스키마 조사

**Files:**
- Create: `dsl/research/dify-dsl-schema.md` (조사 결과 문서)

**Produces:** Dify Chatbot DSL yml의 정확한 구조를 정리한 참조 문서. 이 문서를 기반으로 Task 2에서 실제 yml 작성.

**Why this task first:** Dify DSL은 공식 스펙이 아닌 역공학된 포맷. 정확한 노드 ID, edge 포맷, 변수 참조 문법 (`{{#sys.X#}}` vs `{{X}}`) 등을 알아야 잘못된 yml을 만들지 않음.

- [ ] **Step 1: Dify 공식 GitHub 예시 확인**

WebFetch: `https://github.com/langgenius/dify/blob/main/api/core/app/apps/chat_app.yaml` (예시 DSL 위치) 또는 공식 doc 사이트의 DSL 예시 페이지. 또는 Dify GitHub의 `api/core/app/` 디렉터리에서 DSL 스키마/예시 yml 검색.

특히 확인할 것:
- 최상위 키 (`app`, `kind`, `version`, `workflow`)
- `workflow.graph.nodes[]` 항목 구조 (id, type, data, position)
- 노드 타입별 `data` 필드 구조 (start, llm, if-else, answer)
- `workflow.graph.edges[]` 항목 구조
- 변수 참조 문법

- [ ] **Step 2: Chatbot 모드 DSL 샘플 확보**

Dify GitHub `api/services/` 또는 README에서 Chatbot DSL 예시 yaml 찾기. 또는 Dify의 "Export DSL" 기능으로 어떤 앱을 export한 샘플 yml 검색. 가능하면 웹에서 `dify chat app yaml example` 검색해 Dify 커뮤니티/블로그에서 실제 export된 yml 예시 확보.

- [ ] **Step 3: 노드별 필수 필드 확인**

각 노드 타입에서 필수 필드를 표로 정리:
- **start**: `variables[]` (각각 name, type, label, required, default, options)
- **if-else**: `cases[]` (각각 case_id, logical_operator, conditions)
- **llm**: `model`, `prompt_template`, `system_prompt`, `user_prompt`, `temperature`, `context`, `memory`
- **answer**: `answer` (template or variable reference)

- [ ] **Step 4: 조사 결과 문서 작성**

`dsl/research/dify-dsl-schema.md` 파일에 다음 섹션 포함:
- 최상위 구조 (예시 yml 스니펫)
- 각 노드 타입별 필수/선택 필드
- 변수 참조 문법
- 노드 ID/edge ID 네이밍 규칙
- 특수 필드 (예: `position`, `viewport`, `features` 등)

문서 마지막에 "이 정보는 2026-07-06 시점 Dify 버전을 기반으로 함" 명시.

- [ ] **Step 5: Commit**

```bash
git add dsl/research/dify-dsl-schema.md
git commit -m "docs: Dify DSL 스키마 조사 결과"
```

---

## Task 2: MPS 주간 도우미 yml 작성

**Files:**
- Create: `dsl/MPS-주간-도우미.yml`

**Produces:** 실제 Dify에 Import 가능한 완성된 DSL yml.

- [ ] **Step 1: 최상위 메타데이터 작성**

```yaml
app:
  description: '주간 MPS Planning/Evaluation 마크다운 자동 생성. 자유 텍스트 입력 → MPS 양식 엄격 준수.'
  icon: '📋'
  icon_background: '#FFEAD5'
  mode: chat
  name: 'MPS 주간 도우미'

kind: app
version: 0.1.0
```

- [ ] **Step 2: workflow.features 정의**

```yaml
workflow:
  conversation_variables: []
  environment_variables: []
  features:
    file_upload:
      allowed_file_extensions: []
      allowed_file_types: []
      enabled: false
      fileUploadConfig:
        audio_file_size_limit: 50
        batch_count_limit: 5
        file_size_limit: 15
        image_file_size_limit: 10
        video_file_size_limit: 100
        workflow_file_upload_limit: 10
    image_upload:
      enabled: false
      number_limits: 3
      transfer_methods:
        - local_file
        - remote_url
    opening_questions: []
    retriever_resource:
      enabled: true
    sensitive_word_avoidance:
      enabled: false
    speech_to_text:
      enabled: false
    suggested_questions: []
    suggested_questions_after_answer:
      enabled: false
    text_to_speech:
      enabled: false
```

- [ ] **Step 3: Start 노드 정의 (graph.nodes[0])**

`type: start` 노드. 변수 7개 포함:
- `mode`: select, required, options=[Planning, Evaluation]
- `content`: paragraph, optional
- `previous_mps`: paragraph, optional, label="기존 MPS (수정 시 붙여넣기)"
- `edit_request`: paragraph, optional
- `author`: text, optional, default="본인"
- `due_date`: text, optional
- `mission_label`: text, optional

Dify DSL의 실제 변수 정의 포맷은 Task 1 조사 결과 따라 작성. 일반적 형태:
```yaml
- id: start_node
  type: start
  data:
    variables:
      - type: select
        required: true
        options:
          - Planning
          - Evaluation
        label: 모드
        variable: mode
      - type: paragraph
        required: false
        label: 내용 (자유 텍스트)
        variable: content
      # ... 나머지 5개
  position:
    x: 80
    y: 200
```

- [ ] **Step 4: If/Else 노드 정의**

`type: if-else` 노드. 조건: `{{#start.mode#}} == Planning`. Else 브랜치는 Evaluation.

조사 결과에 따라 정확한 포맷 적용. 일반적 형태:
```yaml
- id: if_else_node
  type: if-else
  data:
    cases:
      - case_id: planning_case
        logical_operator: and
        conditions:
          - id: cond_mode_planning
            comparison_operator: equal
            value: Planning
            variable: mode  # 또는 {{#start.mode#}} 직접 참조
        # else case는 implicit
```

- [ ] **Step 5: LLM #1 (Planning) 노드 정의**

`type: llm`. 시스템 프롬프트는 스펙 §LLM #1의 내용 그대로 (마크다운 코드블록은 yml 내에서 multiline string 또는 `|-` 블록 사용).

```yaml
- id: llm_planning
  type: llm
  data:
    title: MPS Weekly Planner
    model:
      provider: openai  # 또는 Dify 기본 제공자에 맞게 조정
      name: gpt-oss
      mode: chat
      completion_params:
        temperature: 0.3
    prompt_template:
      - id: sys_prompt
        role: system
        text: |
          너는 "MPS(Mission-Performance-Strategy) 주간 Planner"다.
          [...시스템 프롬프트 전문...]
      - id: usr_prompt
        role: user
        text: '{{#sys.query#}}'
    context:
      enabled: true
    memory:
      role_prefix:
        assistant: ''
        user: ''
      window:
        enabled: true
        size: 10
  position:
    x: 400
    y: 100
```

- [ ] **Step 6: LLM #2 (Evaluation) 노드 정의**

Step 5와 동일 구조. 시스템 프롬프트는 스펙 §LLM #2의 내용. `position` 다르게.

- [ ] **Step 7: Answer 노드 정의**

```yaml
- id: answer_node
  type: answer
  data:
    title: Output
    answer: '{{#llm_planning.text#}}'  # 또는 LLM #2 — Dify가 분기 자동 머지
    position:
      x: 720
      y: 200
```

만약 Answer 노드가 단일 변수만 받는다면, 분기별로 별도 Answer가 필요할 수 있음. 조사 결과에 따라 조정.

- [ ] **Step 8: edges 정의 (노드 연결)**

```yaml
edges:
  - id: edge_start_ifelse
    source: start_node
    sourceHandle: source
    target: if_else_node
    targetHandle: target
    type: custom
  - id: edge_ifelse_llm1
    source: if_else_node
    sourceHandle: planning_case  # 또는 if 브랜치 핸들
    target: llm_planning
    targetHandle: target
    type: custom
  - id: edge_ifelse_llm2
    source: if_else_node
    sourceHandle: 'false'  # else 브랜치 핸들
    target: llm_evaluation
    targetHandle: target
    type: custom
  - id: edge_llm1_answer
    source: llm_planning
    sourceHandle: source
    target: answer_node
    targetHandle: target
    type: custom
  - id: edge_llm2_answer
    source: llm_evaluation
    sourceHandle: source
    target: answer_node
    targetHandle: target
    type: custom
```

- [ ] **Step 9: viewport 정의**

```yaml
    viewport:
      x: 0
      y: 0
      zoom: 1
```

- [ ] **Step 10: yml 파일 저장**

전체를 `/Users/fotogrammer/Projects/DifyMaker/dsl/MPS-주간-도우미.yml`에 저장. UTF-8 인코딩. 들여쓰기 일관되게 (스페이스 2칸).

- [ ] **Step 11: yaml 문법 검증**

```bash
python3 -c "import yaml; yaml.safe_load(open('/Users/fotogrammer/Projects/DifyMaker/dsl/MPS-주간-도우미.yml'))"
```

Expected: 에러 없이 통과.

- [ ] **Step 12: 스펙 대조 검증**

스펙 §입력 변수 명세의 7개 변수 (`mode`, `content`, `previous_mps`, `edit_request`, `author`, `due_date`, `mission_label`)가 모두 yml에 존재하는지 확인:

```bash
grep -E "^\s+(mode|content|previous_mps|edit_request|author|due_date|mission_label):" dsl/MPS-주간-도우미.yml | sort -u
```

Expected: 7개 모두 출력.

- [ ] **Step 13: 시스템 프롬프트 대조 검증**

두 시스템 프롬프트에 핵심 MPS 규칙이 모두 포함되어 있는지 확인:
- "고정변수 목표", "변동변수 목표", "1:1 매핑", "결과물로만" 등 Planning 키워드
- "행위", "창출성과", "GAP", "진척률" 등 Evaluation 키워드

```bash
grep -E "고정변수|변동변수|결과물|행위|창출성과|GAP|진척률" dsl/MPS-주간-도우미.yml | wc -l
```

Expected: 최소 10개 이상 매치 (각 키워드가 여러 번 등장).

- [ ] **Step 14: Commit**

```bash
git add dsl/MPS-주간-도우미.yml
git commit -m "feat: MPS 주간 도우미 DSL yml 작성"
```

---

## Task 3: 통합 검증

**Files:**
- Modify: 없음 (검증만)
- Create: `dsl/research/validation-report.md`

**Verifies:** 작성된 yml이 Dify의 실제 export 형식과 일치하는지, Import 시 문제가 없는지.

- [ ] **Step 1: yaml 파싱 확인**

```bash
python3 -c "
import yaml
with open('/Users/fotogrammer/Projects/DifyMaker/dsl/MPS-주간-도우미.yml') as f:
    data = yaml.safe_load(f)
print('Top keys:', list(data.keys()))
print('App mode:', data['app']['mode'])
print('App name:', data['app']['name'])
print('Nodes:', len(data['workflow']['graph']['nodes']))
print('Edges:', len(data['workflow']['graph']['edges']))
"
```

Expected:
- Top keys: `['app', 'kind', 'version', 'workflow']`
- App mode: `chat`
- App name: `MPS 주간 도우미`
- Nodes: 5 (start, if-else, llm_planning, llm_evaluation, answer)
- Edges: 5

- [ ] **Step 2: 노드 타입 확인**

```bash
python3 -c "
import yaml
with open('/Users/fotogrammer/Projects/DifyMaker/dsl/MPS-주간-도우미.yml') as f:
    data = yaml.safe_load(f)
for n in data['workflow']['graph']['nodes']:
    print(f\"  {n['id']}: type={n['type']}\")
"
```

Expected: 각 노드 ID와 타입이 정확. start, if-else, llm (x2), answer.

- [ ] **Step 3: 변수 참조 일관성 확인**

모든 `{{#...#}}` 참조가 실제 정의된 노드/변수를 가리키는지:

```bash
grep -oE '\{\{#[a-z_]+\.[a-z_]+#\}\}' dsl/MPS-주간-도우미.yml | sort -u
```

Expected: `{{#start.X#}}` 형식의 참조가 start 노드 변수와 일치.

- [ ] **Step 4: 시스템 프롬프트 무결성 확인**

Planning 프롬프트와 Evaluation 프롬프트가 분리되어 있고, 각각 스펙 §3의 절대 규칙 8개를 모두 포함하는지. yml 내 텍스트 검색으로 검증:

Planning 규칙 키워드:
- "결과물로만"
- "고정변수 목표"
- "변동변수 목표"
- "1:1 매핑"
- "진척률/상태/commit 수" (Planning 금지)
- "As-Is / To-Be"
- "PO 단위로 압축"

Evaluation 규칙 키워드:
- "행위"
- "창출성과"
- "GAP 필수"
- "진척률"
- "1:1 매핑"
- "commit 수" (허용)
- "도구명"

각 키워드가 yml에 최소 1번 이상 등장해야 함. 미충족 시 Task 2로 돌아가 수정.

- [ ] **Step 5: 검증 리포트 작성**

`dsl/research/validation-report.md`에 결과 기록:

```markdown
# DSL 검증 리포트

**날짜**: 2026-07-06
**파일**: `dsl/MPS-주간-도우미.yml`

## ✅ 통과
- yaml 문법 파싱
- 5개 노드 (start, if-else, llm_planning, llm_evaluation, answer)
- 5개 edges
- 7개 입력 변수
- Planning/Evaluation 시스템 프롬프트 분리
- 핵심 규칙 키워드 포함

## 📋 다음 단계 (수동)
- 사용자: Dify에 Import 후 시각적 확인
- 사용자: 디버그 패널에서 T1/T2/T3 시나리오 실행
```

- [ ] **Step 6: Commit**

```bash
git add dsl/research/validation-report.md
git commit -m "docs: DSL 검증 리포트"
```

---

## Task 4: DSL README 작성 + 최종 정리

**Files:**
- Create: `dsl/README.md`
- Modify: `README.md` (앱 목록 업데이트)

- [ ] **Step 1: `dsl/README.md` 작성**

```markdown
# MPS 주간 도우미

> 주간 MPS(Mission-Performance-Strategy) Planning / Evaluation 마크다운을 자동 생성하는 Dify 챗봇.

## 설치

1. Dify 워크스페이스 로그인
2. **Studio** → **Import from DSL**
3. `MPS-주간-도우미.yml` 업로드
4. 앱 설정 → 모델 확인 (기본 `gpt-oss`)
5. **Publish** 클릭

## 사용법

### 입력 변수

| 변수 | 필수 | 설명 |
|------|------|------|
| `mode` | ✅ | `Planning` 또는 `Evaluation` |
| `content` | ⚠️ | 신규 작성 시 자유 텍스트 |
| `previous_mps` | ❌ | 수정할 기존 MPS 마크다운 |
| `edit_request` | ❌ | 수정 의도 |
| `author` | ❌ | 성과창출자 (기본: 본인) |
| `due_date` | ❌ | YYYY-MM-DD |
| `mission_label` | ❌ | 상위 미션 라벨 |

### 예시

**T1 — 신규 Planning**:
```
mode: Planning
content: 이번 주 미션: X 시스템 설계. PO1: 설계 문서, PO2: 기술 검토.
```

**T2 — 신규 Evaluation**:
```
mode: Evaluation
content: PO1 완료, 커밋 12건. PO2 부분 진행 80%. 진척률 70%. 도구: Notion, GitHub.
```

**T3 — 기존 문서 수정**:
```
mode: Planning
previous_mps: <기존 MPS 마크다운>
edit_request: PO3 추가: 테스트 자동화 보고서
```

## 출력 형식

순수 마크다운. MyMPS 양식을 따름. 자세한 양식: [`../../MyMPS/가이드/MPS 작성 가이드.md`](../../MyMPS/가이드/MPS%20작성%20가이드.md)

## 디자인 문서

- 스펙: [`../docs/superpowers/specs/2026-07-06-mps-dify-design.md`](../docs/superpowers/specs/2026-07-06-mps-dify-design.md)
- 구현 계획: [`../docs/superpowers/plans/2026-07-06-mps-dify-implementation.md`](../docs/superpowers/plans/2026-07-06-mps-dify-implementation.md)

---

<p align="center"><em>MPS · Dify · v1</em></p>
```

- [ ] **Step 2: `README.md` 앱 목록 확인**

상위 README의 앱 목록이 정확히 1개 앱 (`MPS 주간 도우미`)만 가리키는지 확인. 필요시 업데이트.

- [ ] **Step 3: 최종 커밋**

```bash
git add dsl/README.md README.md
git commit -m "docs: DSL 사용법 README + 앱 목록 동기화"
```

- [ ] **Step 4: 최종 상태 확인**

```bash
git log --oneline
ls -la dsl/
```

Expected: 4+ 추가 커밋. `dsl/` 에 `MPS-주간-도우미.yml`, `README.md`, `research/` 폴더.

- [ ] **Step 5: Done 🎉**

`dsl/MPS-주간-도우미.yml` 파일이 Dify에 Import 가능한 상태. 사용자는 Dify 워크스페이스 → Import from DSL → 이 파일 업로드 → 즉시 사용 가능.