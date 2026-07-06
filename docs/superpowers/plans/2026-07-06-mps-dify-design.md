# MPS Dify Assistant Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Dify Chatbot that generates weekly MPS (Mission-Performance-Strategy) Planning and Evaluation markdown documents from free-text input, supports editing existing documents, and exports as a portable DSL yml.

**Architecture:** Dify Chatbot with If/Else branching → two LLM nodes (one per mode) → single Answer. Inputs include mode, content, previous_mps, edit_request, plus optional author/due_date/mission_label. System prompts directly encode MPS rules. Conversation memory ON.

**Tech Stack:** Dify (running instance, default LLM = gpt-oss), Markdown, Git.

## Global Constraints

- App type: **Chatbot** (not Workflow, not Agent)
- 7 input variables with exact names from spec §입력 변수 명세
- 2 LLM nodes with separate system prompts (Planning / Evaluation)
- Mode selection: **Select variable, no LLM auto-detect**
- Output: pure markdown, no code fences, no greetings, no commentary
- v1 scope only — NO monthly/annual, NO validation pass, NO external integrations, NO knowledge base
- Reference spec: [`../specs/2026-07-06-mps-dify-design.md`](../specs/2026-07-06-mps-dify-design.md)
- Reference framework: [`../../MyMPS/가이드/MPS 작성 가이드.md`](../../MyMPS/가이드/MPS%20작성%20가이드.md)
- All markdown output must satisfy MPS 양식 (h1/h2/h3 levels, PO 그룹 구조, 고정/변동변수 1:1 매핑)
- File names: English for code paths; Korean allowed for user-facing content and Dify app name

---

## File Structure

```
DifyMaker/
├── docs/superpowers/
│   ├── specs/2026-07-06-mps-dify-design.md       [exists]
│   └── plans/2026-07-06-mps-dify-design.md       [this file]
├── dsl/
│   ├── MPS-주간-도우미.yml                        [Task 11]
│   └── README.md                                  [Task 12]
├── README.md                                       [Task 1]
├── CLAUDE.md                                       [Task 1]
└── .gitignore                                      [Task 1]
```

---

## Task 1: Project Scaffolding

**Files:**
- Create: `README.md`
- Create: `CLAUDE.md`
- Create: `.gitignore`

**Produces:** A clean repo with README explaining purpose, CLAUDE.md for future agents, .gitignore excluding Dify export artifacts.

- [ ] **Step 1: Create `.gitignore`**

```gitignore
# macOS
.DS_Store
*.swp
*~

# Editor
.vscode/
.idea/

# Dify local export drafts (only the curated dsl/ folder is tracked)
dify-draft-*.yml

# Node (if any UI prototype added later)
node_modules/
.env
```

- [ ] **Step 2: Create `README.md`**

```markdown
# DifyMaker

Dify 앱 DSL(yaml) 라이브러리. 각 서브폴더는 독립적인 Dify 앱.

## 📦 앱 목록

| 앱 | 용도 | DSL |
|---|---|---|
| MPS 주간 도우미 | 주간 MPS Planning/Evaluation 마크다운 생성 | [`dsl/MPS-주간-도우미.yml`](dsl/MPS-주간-도우미.yml) |

## 🚀 사용법

1. Dify 워크스페이스 로그인
2. Studio → `Import from DSL`
3. `dsl/<앱명>.yml` 업로드
4. 앱 설정에서 모델/권한 확인
5. Publish

## 📚 문서

- 디자인 스펙: [`docs/superpowers/specs/`](docs/superpowers/specs/)
- 구현 계획: [`docs/superpowers/plans/`](docs/superpowers/plans/)

---

<p align="center"><em>Built with Dify + Claude</em></p>
```

- [ ] **Step 3: Create `CLAUDE.md`**

```markdown
# CLAUDE.md

## 프로젝트 개요

Dify 앱 DSL 라이브러리. `dsl/` 폴더의 각 `.yml`은 독립적인 Dify 앱.

## 작업 규칙

- 새 앱 추가: `docs/superpowers/specs/` 에 디자인 스펙 먼저 → 구현 → DSL 내보내기 → `dsl/` 에 커밋
- DSL 파일은 한글 이름 OK (Dify가 지원)
- Dify 내보내기 시 `app_mode`, `workflow`, `features` 등 메타 필드는 그대로 보존
- 시스템 프롬프트는 인라인 문자열로 유지 (지식베이스 미사용)

## 파일 명명

- Dify 앱 이름: 한글 가능 (사용자에게 보이는 이름)
- DSL 파일명: `<한글 이름>.yml`
- 변수명: 영어 (mode, content, previous_mps, edit_request, author, due_date, mission_label)
```

- [ ] **Step 4: Commit scaffolding**

```bash
cd /Users/fotogrammer/Projects/DifyMaker
git add .gitignore README.md CLAUDE.md
git commit -m "chore: scaffold project (README, CLAUDE.md, .gitignore)"
```

---

## Task 2: Verify Dify Access and Create Empty Chatbot App

**Files:**
- Modify: Dify workspace (UI only, no file change)
- Create: Dify app named `MPS 주간 도우미` (type: Chatbot)

**Produces:** An empty Chatbot app in Dify, ready for node configuration.

- [ ] **Step 1: Verify Dify is accessible**

Open the Dify URL in browser. Log in with your credentials. Confirm you can see the Studio page.

Expected: Studio page loads, showing "Create from Blank" or app list.

If Dify is not running, start it locally (Docker) or get the URL from your admin. Do not proceed without access.

- [ ] **Step 2: Create new Chatbot app**

In Studio, click **"Create from Blank"** → choose **"Chatbot"** (NOT "Agent", NOT "Workflow", NOT "Chatflow"). Name it exactly: **`MPS 주간 도우미`**. Add description: "주간 MPS Planning/Evaluation 마크다운 생성. 자유 텍스트로 작업 내용을 입력하면 MPS 양식을 엄격히 따르는 마크다운을 출력한다." Choose language: Korean. Click Create.

Expected: A blank Chatbot editor opens. Default nodes visible: Start → LLM (placeholder) → Answer.

- [ ] **Step 3: Delete the default LLM node**

The default Chatbot has a built-in LLM. We will replace it with our own branching structure. Click the default LLM node and press Delete.

Expected: Canvas now shows only Start → Answer.

- [ ] **Step 4: Verify the app exists**

Click the app name in top-left to confirm. The app ID will be needed for later export.

---

## Task 3: Configure Start Node Inputs

**Files:**
- Modify: Dify Start node (UI only)

**Produces:** 7 input variables ready for downstream nodes.

- [ ] **Step 1: Open Start node**

Click the Start node on the canvas.

- [ ] **Step 2: Add `mode` (Select, required)**

Click **"Add Input Field"** → type: `mode`, label: `모드`, type: **Select**, required: ON. Options:
- `Planning`
- `Evaluation`

Expected: `mode` appears with two radio options.

- [ ] **Step 3: Add `content` (Paragraph, conditional required)**

Add Input Field → name: `content`, label: `내용 (자유 텍스트)`, type: **Paragraph**, required: OFF (we'll enforce in prompt logic).

- [ ] **Step 4: Add `previous_mps` (Paragraph, optional)**

Add Input Field → name: `previous_mps`, label: `기존 MPS (수정 시 붙여넣기)`, type: **Paragraph**, required: OFF.

- [ ] **Step 5: Add `edit_request` (Paragraph, optional)**

Add Input Field → name: `edit_request`, label: `수정 요청`, type: **Paragraph**, required: OFF.

- [ ] **Step 6: Add `author` (Text, optional, default "본인")**

Add Input Field → name: `author`, label: `성과창출자`, type: **Text**, required: OFF, default: `본인`.

- [ ] **Step 7: Add `due_date` (Text, optional)**

Add Input Field → name: `due_date`, label: `완료예상일자 (YYYY-MM-DD)`, type: **Text**, required: OFF.

- [ ] **Step 8: Add `mission_label` (Text, optional)**

Add Input Field → name: `mission_label`, label: `상위 미션 라벨`, type: **Text**, required: OFF.

- [ ] **Step 9: Verify all 7 inputs**

Count the input fields. You should have exactly 7: mode, content, previous_mps, edit_request, author, due_date, mission_label. Names match exactly (snake_case for previous_mps and edit_request).

- [ ] **Step 10: Save and confirm**

Click outside the node or press Tab. Inputs are auto-saved.

---

## Task 4: Add If/Else Branching Node

**Files:**
- Modify: Dify canvas (UI only)

**Produces:** A branching structure that splits based on `mode` value.

- [ ] **Step 1: Add If/Else node**

On the canvas, click the **`+`** button between Start and Answer. Search for "If/Else" and add it.

Expected: New If/Else node appears on the canvas.

- [ ] **Step 2: Connect Start → If/Else**

Drag from Start's output port to If/Else's input port.

- [ ] **Step 3: Configure IF condition (Planning branch)**

In the If/Else node, set:
- Variable: `{{#start.mode#}}`  (this is Dify's syntax for referencing the start input `mode`)
- Operator: **equal to**
- Value: `Planning`
- Branch name: `Planning`

Expected: One branch created with condition "mode == Planning".

- [ ] **Step 4: Configure ELSE branch (Evaluation)**

Add an `else if` (or `else`) condition:
- Variable: `{{#start.mode#}}`
- Operator: **equal to**
- Value: `Evaluation`
- Branch name: `Evaluation`

If the node only supports one condition + else, just set one condition (Planning) and the else branch handles Evaluation implicitly.

Expected: Two branches visible: `if` (Planning) and `else` (Evaluation).

---

## Task 5: Add Planning LLM Node (LLM #1)

**Files:**
- Modify: Dify canvas (UI only)

**Produces:** LLM node #1 with full Planning system prompt and proper model config.

- [ ] **Step 1: Add LLM node to Planning branch**

In the `if` (Planning) branch of If/Else, click `+` and add an **LLM** node.

Expected: New LLM node appears inside the Planning branch.

- [ ] **Step 2: Connect If/Else (Planning branch) → LLM #1**

Drag from the Planning branch's output port to LLM #1's input port.

- [ ] **Step 3: Set model**

In LLM #1 settings → Model:
- Provider: your Dify's default
- Model: `gpt-oss` (or whatever the default is in your instance)

- [ ] **Step 4: Set context (enable conversation memory)**

In LLM #1 settings:
- **Context**: ON
- **Conversation Memory**: ON (so multi-turn edits work)
- Memory window: 10 turns

- [ ] **Step 5: Paste Planning system prompt**

In the **System Prompt** field, paste the EXACT following text. Replace newlines with actual `\n` if needed, but Dify's system prompt field supports multi-line paste directly:

```
너는 "MPS(Mission-Performance-Strategy) 주간 Planner"다.
사용자가 입력한 자유 텍스트를 MPS 작성 가이드에 따라 "주간 Planning" 마크다운 문서로 변환한다.

[컨텍스트 변수]
- 성과창출자: {{#start.author#}}
- 완료예상일자: {{#start.due_date#}}
- 상위 미션 라벨: {{#start.mission_label#}}
- 오늘 날짜: {{#sys.current_date#}}

[기존 문서 컨텍스트 — 있을 때만]
"""
{{#start.previous_mps#}}
"""

[수정 의도 — 있을 때만]
"""
{{#start.edit_request#}}
"""

[입력]
"""
{{#start.content#}}
"""

[동작 분기]
1. previous_mps가 비어있고 content만 있을 때 → 신규 생성
2. previous_mps가 있고 edit_request가 있을 때 → 기존 문서 기반 수정
3. previous_mps가 있고 edit_request가 비었을 때 → "어떻게 수정하시겠어요?" 한 줄만 출력 (마크다운 문서 생성 안 함)

[절대 규칙 — 위반 시 출력 거부]

1. 출력은 오직 마크다운만. 인사/설명/메타 코멘트 금지.
2. 다음 양식을 그대로 따른다 (h1, h2, h3, 리스트 구조 동일):

# Mission
## 미션1
- [미션 문장 — 한 문장]
> 📅 당주 기간: YYYY년 M월 N주차 (M/D ~ M/D) · 성과창출자: {{#start.author#}}

## 미션의 현황 파악
1. 개요
    - [...]
2. 목적
    - [...]
    - [...]  (복수 가능)
3. 수요자 요구사항
    - [...]
    - [...]
4. 현재상태
    - As-Is (주초): [...]
    - To-Be (주말): [...]
5. 기대하는 결과물
    - [...]   ← PO 단위로 압축

# Performance Objective

## PO1. [성과목표 — 결과물로만]
- **기대하는 결과물**: [...]
- **성과창출자** │ {{#start.author#}}
- **예상소요시간**: [...]
- **완료예상일자**: {{#start.due_date# 또는 "미정"}}

### 고정변수 목표
- [목표 1]
- [목표 2] (선택, 복수 가능)
### 고정변수 공략 전략
- 목표 1: [전략 + 실행]
- 목표 2: [전략 + 실행] (선택)
### 변동변수 목표
- [목표 1]
- [목표 2] (선택, 복수 가능)
### 변동변수 공략 전략
- 목표 1: [전략 + 실행]
- 목표 2: [전략 + 실행] (선택)

(PO2, PO3는 입력에 있을 때만)

# Risk
## 외부환경 대응방안
- 리스크: [...]
- 대응: [...]
## 내부역량 대응방안
- 리스크: [...]
- 대응: [...]

3. 성과목표(PO)는 반드시 결과물로 작성. 행위/조건/상태 금지.
   - ❌ "X 시스템 구축" (행위)
   - ❌ "사용자 수용성 확보" (상태)
   - ✅ "X 시스템" / "사용자 수용성 보고서"

4. 고정/변동변수 목표는 1개 이상 복수 가능. 목표 N개 → 전략 N개, 1:1 매핑.

5. 0개 섹션은 통째로 생략. "(해당 없음)" 같은 표기 금지.

6. Planning에는 진척률/상태/commit 수/이미 도달한 수치/도구명 직접 표기 금지.

7. 현재상태는 As-Is / To-Be 두 줄 모두 작성.

8. 5번 기대하는 결과물은 PO 단위로 압축 — PO 안에 이미 결과물 필드 있으므로 페이지명/보고서명 일일이 나열 금지.

[수정 모드 추가 규칙]
- 기존 문서(previous_mps)가 있으면 그 양식(헤딩, 메타데이터 위치, 들여쓰기)을 그대로 유지
- 추가/변경된 부분만 반영, 불필요한 부분은 임의 삭제 금지

[입력 부족 시 처리]
- 사용자가 PO1만 언급 → PO1만 생성, PO2/PO3 생략
- 사용자가 고정변수만 언급 → 변동변수 섹션 통째 생략
- 미션 문장이 모호하면 주간 범위에서 합리적으로 추정 (추정 표시 없이 자연스럽게)
- content와 previous_mps가 모두 비었으면 "작성을 시작하려면 미션/PO 내용을 적어주시거나, 수정할 기존 MPS를 붙여주세요." 한 줄만 출력
- previous_mps만 있고 edit_request가 비었으면 "기존 MPS를 확인했습니다. 어떤 부분을 수정/추가할까요?" 한 줄만 출력

[출력]
- 위 양식의 마크다운만. 코드블록(```)으로 감싸지 않음.
- 앞뒤 공백·설명 문장 금지.
- 첫 줄은 "# Mission"으로 시작.
```

- [ ] **Step 6: Set User prompt**

User Prompt field → leave blank OR set to `{{#sys.query#}}` (Dify captures the user message automatically).

- [ ] **Step 7: Set Temperature**

Temperature: **0.3** (low for format consistency).

- [ ] **Step 8: Save**

LLM #1 is auto-saved. Verify by clicking away and re-opening.

---

## Task 6: Add Evaluation LLM Node (LLM #2)

**Files:**
- Modify: Dify canvas (UI only)

**Produces:** LLM node #2 with full Evaluation system prompt.

- [ ] **Step 1: Add LLM node to Evaluation branch**

In the `else` (Evaluation) branch of If/Else, click `+` and add an **LLM** node.

- [ ] **Step 2: Connect If/Else (Evaluation branch) → LLM #2**

Drag from the Evaluation branch's output port to LLM #2's input port.

- [ ] **Step 3: Set model (same as LLM #1)**

Same provider and `gpt-oss` model.

- [ ] **Step 4: Set context (same as LLM #1)**

Context ON, Conversation Memory ON, 10 turns.

- [ ] **Step 5: Paste Evaluation system prompt**

Paste the EXACT following text:

```
너는 "MPS 주간 Evaluator"다.
사용자가 입력한 자유 텍스트(이번 주에 한 일, 실제 행위, 수치 등)를 "주간 Evaluation" 마크다운 문서로 변환한다.

[컨텍스트 변수]
- 성과창출자: {{#start.author#}}
- 완료예상일자: {{#start.due_date#}}
- 상위 미션 라벨: {{#start.mission_label#}}
- 오늘 날짜: {{#sys.current_date#}}

[기존 문서 컨텍스트 — 있을 때만]
"""
{{#start.previous_mps#}}
"""

[수정 의도 — 있을 때만]
"""
{{#start.edit_request#}}
"""

[입력]
"""
{{#start.content#}}
"""

[동작 분기]
1. previous_mps가 비어있고 content만 있을 때 → 신규 생성
2. previous_mps가 있고 edit_request가 있을 때 → 기존 문서 기반 수정
3. previous_mps가 있고 edit_request가 비었을 때 → "어떻게 수정하시겠어요?" 한 줄만 출력 (마크다운 문서 생성 안 함)

[절대 규칙 — 위반 시 출력 거부]

1. 출력은 오직 마크다운만. 인사/설명/메타 코멘트 금지.
2. 다음 양식을 그대로 따른다 (Planning과 동일 골격 + 행위/GAP 섹션):

# Mission
## 미션1
- [해당 주간의 미션 문장]
> 📅 당주 기간: ... · 성과창출자: {{#start.author#}}

## 미션의 현황 파악
1. 개요 — 한 일의 개요
2. 목적
    - [...]
3. 수요자 요구사항
    - [...]
4. 현재상태
    - As-Is (주말): [...]   ← Evaluation은 주말 시점
    - To-Be (다음 주 초): [...]  (선택)
5. 기대하는 결과물
    - [이미 달성했거나 부분 달성한 결과]

# Performance Objective

## PO1. [성과목표]
- **기대하는 결과물**: [...]
- **성과창출자** │ {{#start.author#}}
- **예상소요시간**: [...]
- **완료예상일자**: {{#start.due_date# 또는 "미정"}}
- **진척률**: N%  ← 필수

### 행위 (1:1 매핑)
- 목표 1 행위: [실제 수행한 행위 + 사용 도구 명시]
- 목표 2 행위: [...] (선택)

### 창출성과
- [목표 1 결과/산출물 — 실제 commit 수, 도달 수치 등 OK]
- [목표 2 결과] (선택)

### GAP
- [목표 vs 실제의 차이 — 미달/초과/방향 변화 이유]

### 고정변수 목표
- [목표 1]
- [목표 2] (선택, 복수 가능)
### 고정변수 공략 전략
- 목표 1: [전략 + 실행 결과]
### 변동변수 목표
- [목표 1]
- [목표 2] (선택, 복수 가능)
### 변동변수 공략 전략
- 목표 1: [전략 + 실행 결과]

# Risk
## 외부환경 대응방안
- 리스크: [...]
- 대응: [...]
- 실행 결과: [...]  ← Evaluation에서는 결과도 함께
## 내부역량 대응방안
- 리스크: [...]
- 대응: [...]
- 실행 결과: [...]

3. Evaluation은 행위·진척·창출성과·GAP 모두 포함.
   - ✅ commit 수 / 사용 도구 명시 / 도달 수치 / 진척률
   - ✅ "구축 완료", "테스트 진행 중" 같은 행위 표현
   - ✅ GAP 필수 (목표 vs 실제 차이)

4. 행위 섹션은 목표와 1:1 매핑 — 목표 N개 → 행위 N개.

5. 사용 도구명 직접 표기 OK (시스템 아키텍처 외 도구도 OK).

6. 진척률 100% = 완료, 0% = 미착수. 부분 진행은 N%.

7. 현재상태는 As-Is / To-Be 두 줄 모두 작성 (주말 시점 As-Is).

8. 5번 기대하는 결과물은 이미 산출되었거나 부분 산출된 결과 위주.

[수정 모드 추가 규칙]
- 기존 문서(previous_mps)가 있으면 그 양식(헤딩, 메타데이터 위치, 들여쓰기)을 그대로 유지
- 추가/변경된 부분만 반영, 불필요한 부분은 임의 삭제 금지
- previous_mps가 Planning이고 mode가 Evaluation이면
  → 같은 양식 골격에 Evaluation 전용 필드(행위/창출성과/GAP/진척률)만 채움
  → Planning에 있던 진척률·행위·도구명은 Evaluation에 자연스럽게 흡수

[입력 부족 시 처리]
- 진척률/GAP이 없으면 추정해서 채움 (Planning과 동일 톤)
- 한 일/도구/수치가 전혀 없으면 "한 일 정리해줘" 라고 정중히 되묻기
- content와 previous_mps가 모두 비었으면 "작성을 시작하려면 미션/PO 내용을 적어주시거나, 수정할 기존 MPS를 붙여주세요." 한 줄만 출력
- previous_mps만 있고 edit_request가 비었으면 "기존 MPS를 확인했습니다. 어떤 부분을 수정/추가할까요?" 한 줄만 출력

[출력]
- 위 양식의 마크다운만. 코드블록(```)으로 감싸지 않음.
- 앞뒤 공백·설명 문장 금지.
- 첫 줄은 "# Mission"으로 시작.
```

- [ ] **Step 6: Set User prompt**

Same as LLM #1: blank or `{{#sys.query#}}`.

- [ ] **Step 7: Set Temperature**

Temperature: **0.3**.

---

## Task 7: Wire End-to-End Flow with Answer Node

**Files:**
- Modify: Dify canvas (UI only)

**Produces:** Complete flow: Start → If/Else → [LLM #1 | LLM #2] → Answer.

- [ ] **Step 1: Verify both LLM nodes are inside their branches**

LLM #1 should be inside the `if` (Planning) branch. LLM #2 should be inside the `else` (Evaluation) branch.

- [ ] **Step 2: Connect LLM #1 → Answer**

Drag from LLM #1's output port to the Answer node's input port.

- [ ] **Step 3: Connect LLM #2 → Answer**

Drag from LLM #2's output port to the Answer node's input port.

If the Answer node only accepts one input, Dify will merge them. The active branch's LLM output flows to Answer.

Expected: Canvas shows full graph: Start → If/Else → {LLM #1, LLM #2} → Answer.

- [ ] **Step 4: Verify Answer node references LLM text**

Click the Answer node. In the "Answer Content" field, it should reference `{{#llm1.text#}}` or Dify's auto-merged text variable. If Dify shows two separate variables for LLM #1 and LLM #2, set:
- Variable: merge both branches with conditional, OR
- Just use `{{#llm1.text#}}` if Dify handles branch merging automatically (typical behavior).

If unsure, test with T1 in Task 8 — if output works, the wiring is correct.

- [ ] **Step 5: Save and verify**

App is auto-saved. The full flow should now be operational.

---

## Task 8: Test Scenario T1 (New Planning)

**Files:**
- Modify: Dify app debug panel (UI only)

**Verifies:** LLM #1 generates valid Planning markdown from free-text input.

- [ ] **Step 1: Open debug panel**

Click the "Debug" or "Run" button in the top-right of the Dify editor.

- [ ] **Step 2: Fill inputs for T1**

Set inputs:
- `mode`: Planning
- `content`: `이번 주 미션: X 시스템 설계. PO1: 설계 문서 1건. PO2: 기술 검토 1회. 고정변수 목표: 설계 문서 1건. 변동변수 목표: 외부 API 사양 확정.`
- `author`: (leave default "본인")
- Other fields: leave empty

- [ ] **Step 3: Run**

Click Run.

- [ ] **Step 4: Verify output starts with `# Mission`**

Expected: Output begins with `# Mission` (no greeting, no code fence).

- [ ] **Step 5: Verify PO group structure**

Expected: Contains `# Performance Objective` followed by `## PO1.` and `## PO2.` Each PO has `### 고정변수 목표`, `### 고정변수 공략 전략`, `### 변동변수 목표`, `### 변동변수 공략 전략`.

- [ ] **Step 6: Verify 1:1 strategy mapping**

Expected: Fixed goals N → fixed strategies N. Variable goals N → variable strategies N.

- [ ] **Step 7: Verify Planning purity**

Expected: No 진척률 (%), no commit 수, no 도구명 like "Notion" or "GitHub" in Planning output.

- [ ] **Step 8: Save T1 output**

Copy the successful T1 output to a scratch file (e.g., `/tmp/t1_output.md`). Will be used as `previous_mps` in T3.

- [ ] **Step 9: If any check fails, refine prompt**

If output fails any check, edit LLM #1 system prompt to tighten the failing rule. Re-run. Repeat until all 7 checks pass.

---

## Task 9: Test Scenario T2 (New Evaluation)

**Files:**
- Modify: Dify app debug panel (UI only)

**Verifies:** LLM #2 generates valid Evaluation markdown with 행동/창출성과/GAP/진척률.

- [ ] **Step 1: Fill inputs for T2**

Set inputs:
- `mode`: Evaluation
- `content`: `PO1 완료, 설계 문서 v1.0 커밋 12건. PO2 부분 진행, API 사양 80% 확정. 진척률 70%. 도구: Notion, GitHub.`
- Other fields: default/empty

- [ ] **Step 2: Run**

- [ ] **Step 3: Verify output starts with `# Mission`**

- [ ] **Step 4: Verify 진척률 is present**

Expected: Each PO has `**진척률**: N%` line.

- [ ] **Step 5: Verify 행위 + 창출성과 + GAP sections**

Expected: Each PO contains `### 행위`, `### 창출성과`, `### GAP`.

- [ ] **Step 6: Verify 도구명 is allowed**

Expected: Output mentions "Notion", "GitHub" without auto-removal.

- [ ] **Step 7: Verify commit 수 is allowed**

Expected: "12건" or similar appears in 창출성과.

- [ ] **Step 8: If any check fails, refine prompt**

Edit LLM #2 system prompt to fix. Re-run. Repeat until all 6 checks pass.

---

## Task 10: Test Scenario T3 (Modify Existing Planning)

**Files:**
- Modify: Dify app debug panel (UI only)

**Verifies:** Conversation memory + previous_mps + edit_request add a new PO correctly.

- [ ] **Step 1: Paste T1 output into previous_mps**

In the debug panel, set:
- `previous_mps`: paste the T1 output from `/tmp/t1_output.md`
- `edit_request`: `PO3 추가: 테스트 자동화 보고서. 고정변수: 단위 테스트 커버리지 80% 달성.`
- `mode`: Planning
- `content`: (leave empty — previous_mps provides context)
- Other fields: default/empty

- [ ] **Step 2: Run**

- [ ] **Step 3: Verify PO3 is added**

Expected: Output contains `## PO3. 테스트 자동화 보고서` (or similar result-oriented title).

- [ ] **Step 4: Verify PO1 and PO2 unchanged**

Expected: PO1 and PO2 sections are byte-identical to T1 output. Only PO3 is new.

- [ ] **Step 5: Verify PO3 has all sub-sections**

Expected: PO3 has 고정변수 목표, 고정변수 공략 전략, and 1:1 mapping.

- [ ] **Step 6: Test edge case — previous_mps without edit_request**

Set `previous_mps` to T1 output, leave `edit_request` empty, leave `content` empty. Run.

Expected: Output is the single line `"기존 MPS를 확인했습니다. 어떤 부분을 수정/추가할까요?"` (NOT a full MPS document).

- [ ] **Step 7: Test edge case — both content and previous_mps empty**

Leave both empty. Run.

Expected: Output is `"작성을 시작하려면 미션/PO 내용을 적어주시거나, 수정할 기존 MPS를 붙여주세요."`

- [ ] **Step 8: If any check fails, refine prompt**

Edit LLM #1 system prompt's `[동작 분기]` section. Re-run. Repeat until all 7 checks pass.

---

## Task 11: Export DSL and Save to Repo

**Files:**
- Create: `dsl/MPS-주간-도우미.yml`

**Produces:** Portable Dify DSL file in the repo.

- [ ] **Step 1: Export from Dify**

In the app's top-right menu (three dots `⋮`), click **"Export DSL"**. A `.yml` file downloads.

- [ ] **Step 2: Rename and save to repo**

Move/rename the downloaded file to `/Users/fotogrammer/Projects/DifyMaker/dsl/MPS-주간-도우미.yml`.

- [ ] **Step 3: Sanity-check the YAML**

```bash
cd /Users/fotogrammer/Projects/DifyMaker
head -30 dsl/MPS-주간-도우미.yml
```

Expected: Starts with `app:` or `version:` etc. (Dify DSL standard). Contains `workflow:` section with nodes.

- [ ] **Step 4: Verify prompts are embedded**

```bash
grep -c "MPS" dsl/MPS-주간-도우미.yml
```

Expected: At least 2 (Planning prompt mentions MPS multiple times, Evaluation prompt too).

- [ ] **Step 5: Verify input variables are in DSL**

```bash
grep -E "mode|content|previous_mps|edit_request|author|due_date|mission_label" dsl/MPS-주간-도우미.yml | head -10
```

Expected: All 7 variable names appear.

- [ ] **Step 6: Commit**

```bash
git add dsl/MPS-주간-도우미.yml
git commit -m "feat: MPS 주간 도우미 DSL v1 export"
```

---

## Task 12: Write DSL README and Final Commit

**Files:**
- Create: `dsl/README.md`

**Produces:** User-facing documentation for the DSL.

- [ ] **Step 1: Create `dsl/README.md`**

```markdown
# MPS 주간 도우미

> 주간 MPS(Mission-Performance-Strategy) Planning / Evaluation 마크다운을 자동 생성하는 Dify 챗봇.

## 설치

1. Dify 워크스페이스 로그인
2. **Studio** → **Import from DSL**
3. `MPS-주간-도우미.yml` 업로드
4. 앱 설정 → 모델이 `gpt-oss` (또는 Dify 기본 모델) 로 설정되어 있는지 확인
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

### 사용 예시

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

순수 마크다운 (코드블록 감싸지 않음). MyMPS 양식을 따름:

```markdown
# Mission
## 미션1
...
# Performance Objective
## PO1. ...
# Risk
...
```

자세한 양식: [`../../MyMPS/가이드/MPS 작성 가이드.md`](../../MyMPS/가이드/MPS%20작성%20가이드.md)

## 제약

- 주간 단위만 지원 (월간/연간은 v2)
- v1 범위: 생성 + 수정. 자동 검증, 외부 연동 없음

## 디자인 문서

- 스펙: [`../docs/superpowers/specs/2026-07-06-mps-dify-design.md`](../docs/superpowers/specs/2026-07-06-mps-dify-design.md)
- 구현 계획: [`../docs/superpowers/plans/2026-07-06-mps-dify-design.md`](../docs/superpowers/plans/2026-07-06-mps-dify-design.md)

---

<p align="center"><em>MPS · Dify · v1</em></p>
```

- [ ] **Step 2: Commit**

```bash
cd /Users/fotogrammer/Projects/DifyMaker
git add dsl/README.md
git commit -m "docs: DSL 사용법 README 추가"
```

- [ ] **Step 3: Verify final state**

```bash
git log --oneline
ls -la dsl/
cat README.md
```

Expected: 4+ commits (scaffolding, spec, plan, DSL export, README). `dsl/` contains `MPS-주간-도우미.yml` and `README.md`. Top `README.md` mentions `dsl/MPS-주간-도우미.yml`.

- [ ] **Step 4: Done 🎉**

The MPS 주간 도우미 DSL is now portable. Anyone with a Dify instance can import it and use it.

---

## Self-Review Notes (kept for transparency)

- **Spec coverage**: Each spec section maps to a task:
  - Architecture (start → if/else → 2 LLM → answer) → Tasks 2-7
  - 7 input variables → Task 3
  - Planning system prompt → Task 5
  - Evaluation system prompt → Task 6
  - Conversation memory ON → Tasks 5, 6
  - Mode select (not auto-detect) → Task 3 (Select type), Task 4 (If/Else)
  - Test scenarios T1/T2/T3 → Tasks 8/9/10
  - DSL export → Task 11
  - File structure → Tasks 1, 11, 12
- **Placeholder scan**: No TBD/TODO. All code blocks complete. All shell commands exact.
- **Type consistency**: Variable names (`mode`, `content`, `previous_mps`, `edit_request`, `author`, `due_date`, `mission_label`) match spec §입력 변수 명세. Dify template syntax `{{#start.X#}}` used consistently.
- **Ambiguity**: Each task has clear pass/fail criteria via verification steps. Tests T1/T2/T3 give concrete inputs and expected checks.