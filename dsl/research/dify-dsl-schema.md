# Dify Chatbot DSL Schema Reference

**Purpose**: Accurate Dify DSL `yml` structure for writing a valid Dify Chatbot DSL file.
**Based on**: Dify source code analysis at `github.com/langgenius/dify` (master branch, July 2026)
**Investigated by**: Task 1 of DifyMaker MPS project

---

## 1. Top-Level Structure

Every Dify DSL file has exactly four top-level keys:

```yaml
version: "0.1.8"        # DSL version string; auto-fixed to "0.1.0" if missing; kind is auto-fixed to "app"
kind: "app"             # always "app"
app:
  name: "My App"
  mode: "chat"          # "chat" | "completion" | "agent-chat" | "advanced-chat" | "workflow"
  icon: "🤖"
  icon_type: "emoji"    # "emoji" | "image" | "link"
  icon_background: "#FFFFFF"
  description: "..."
  use_icon_as_answer_icon: false
model_config: {...}     # present when mode is chat / agent-chat / completion
workflow: {...}         # present when mode is advanced-chat / workflow
dependencies: [...]      # optional; plugin dependencies list
```

**Mode classification** (from `api/core/app/apps/` + `models/model.py`):
- `chat`, `completion`, `agent-chat` = **EasyUI-based app** (no workflow graph; uses `model_config`)
- `advanced-chat`, `workflow` = **Workflow-based app** (uses `workflow` with graph)

For a Chatbot that uses a visual workflow (nodes + edges), use `mode: "advanced-chat"`.

---

## 2. Workflow App Structure (`advanced-chat` / `workflow` modes)

```yaml
version: "0.1.8"
kind: "app"
app:
  name: "MPS 주간 도우미"
  mode: "advanced-chat"
  icon: "📋"
  icon_type: "emoji"
  icon_background: "#FFE4B5"
  description: "MPS 주간 계획/평가 생성 도우미"
workflow:
  graph:
    nodes: [...]
    edges: [...]
  features: {...}
  environment_variables: [...]
  conversation_variables: [...]
dependencies: [...]
```

### 2.1 `workflow.graph.nodes[]`

Each node is a ReactFlow-style node with `id`, `data`, `position`, and optional `targetPosition`/`sourcePosition`:

```yaml
nodes:
  - id: "start"                        # unique string ID
    data:
      type: "start"                    # BlockEnum value (see below)
      ...data fields per type...
    position:
      x: 0
      y: 0
    targetPosition: "left"
    sourcePosition: "right"
```

#### Node `type` values (from `web/app/components/workflow/types.ts` `BlockEnum`):

| type | Description |
|------|-------------|
| `start` | Start node (entry point) |
| `end` | End node |
| `llm` | LLM call node |
| `answer` | Direct answer node |
| `if-else` | Conditional branch |
| `code` | Code execution |
| `template-transform` | Template transformation |
| `http-request` | HTTP request |
| `variable-assigner` | Variable assignment |
| `variable-aggregator` | Variable aggregation |
| `knowledge-retrieval` | Dataset retrieval |
| `question-classifier` | Question classification |
| `parameter-extractor` | Parameter extraction |
| `iteration` | Iteration loop |
| `tool` | Tool call |
| `agent` / `agent-v2` | Agent node |
| `human-input` | Human input node |
| `trigger-schedule` | Scheduled trigger |
| `trigger-webhook` | Webhook trigger |

#### 2.1.1 `start` Node

```yaml
- id: "start"
  data:
    type: "start"
    variables:                    # list of input variables
      - variable: "task_name"
        label: "과제명"
        type: "text-input"        # InputVarType enum
        required: true
        default: ""
      - variable: "task_description"
        label: "과제 설명"
        type: "paragraph"
        required: false
        default: ""
      - variable: "priority"
        label: "우선순위"
        type: "select"
        required: true
        options: ["높음", "중간", "낮음"]
        default: "중간"
      - variable: "estimated_hours"
        label: "예상 시간"
        type: "number"
        required: false
        default: 0
      - variable: "attachments"
        label: "첨부 파일"
        type: "files"
        required: false
      - variable: "context_docs"
        label: "참고 문서"
        type: "contexts"           # knowledge retrieval results
        required: false
    # System variables are injected automatically: query, files, conversation_id, user_id, etc.
    is_advanced: false
  position:
    x: 0
    y: 0
  targetPosition: "left"
  sourcePosition: "right"
```

**`InputVarType`** values (`web/app/components/workflow/types.ts`):
- `text-input` — single-line text
- `paragraph` — multi-line text
- `select` — dropdown select
- `number` — numeric input
- `url` — URL input
- `files` — file upload
- `json` — JSON object/array
- `json_object` — JSON object with schema
- `contexts` — knowledge retrieval contexts
- `iterator` — iteration input
- `singleFile` / `multiFiles` — file
- `checkbox` — boolean checkbox

#### 2.1.2 `llm` Node

```yaml
- id: "llm_node_1"
  data:
    type: "llm"
    model:
      provider: "openai"           # model provider ID
      name: "gpt-4o-mini"
      mode: "chat"                 # "chat" | "completion"
      completion_params:
        temperature: 0.7
        max_tokens: 2000
        top_p: 1.0
    prompt_template:
      system_prompt: |
        당신은 MPS(Method/Performance/Strategy) 주간 계획 및 평가를 지원하는 AI 어시스턴트입니다.
        사용자의 입력과 문맥을 바탕으로 명확하고 실행 가능한 계획을 세워주세요.
      user_prompt: |
        ## 과제 정보
        - 과제명: {{#start.task_name#}}
        - 설명: {{#start.task_description#}}
        - 우선순위: {{#start.priority#}}
        - 예상 시간: {{#start.estimated_hours#}}시간

        ## 참고 문서
        {{#start.context_docs#}}

        위 정보를 바탕으로 주간 계획을 세워주세요.
    context: {}                    # external context / dataset config
    memory:                        # conversation memory (optional)
      role_prefix:
        user: "user"
        assistant: "assistant"
      window:
        enabled: true
        size: 10
      query_prompt_template: "{{#sys.query#}}"
  position:
    x: 300
    y: 0
  targetPosition: "left"
  sourcePosition: "right"
```

**Prompt template structure**: `system_prompt` + `user_prompt`. Variables are referenced using the `{{#node_id.variable#}}` syntax. The `context` field can include dataset retrieval configuration.

**Memory config** (`memory.window` = `memory` type):
- `role_prefix.user` / `role_prefix.assistant` — prefix labels
- `window.enabled` — whether to use window memory
- `window.size` — number of turns to keep
- `query_prompt_template` — template for querying memory

#### 2.1.3 `if-else` Node

```yaml
- id: "if_else_1"
  data:
    type: "if-else"
    conditions_type: "and"         # "and" | "or"
    cases:
      - case_id: "true"            # branch when condition is true
        case_name: "높은 우선순위"
        conditions:
          - variable: "{{#start.priority#}}"
            condition: "contains"
            value: "높음"
    default_flow: "false"          # route when all conditions fail
  position:
    x: 600
    y: 0
  targetPosition: "left"
  sourcePosition: "right"
```

**Condition comparison operators** (`condition` field):
- `contains` — string contains
- `not_contains` — string does not contain
- `equals` — equals (string or value)
- `not_equals` — not equals
- `starts_with` / `ends_with`
- `is_empty` / `is_not_empty`
- `regex` / `not_regex`
- `>` / `<` / `>=` / `<=` — for numeric comparisons

#### 2.1.4 `answer` Node

```yaml
- id: "answer_1"
  data:
    type: "answer"
    answer: |
      ## 📋 주간 계획

      **{{#start.task_name#}}**

      우선순위: {{#start.priority#}}
      예상 시간: {{#start.estimated_hours#}}시간

      {{#llm_node_1.text#}}
  variables: []                    # referenced variables listed here
  status: "completed"
  error: null
  elapsed_time: 1.23
  position:
    x: 900
    y: 0
  targetPosition: "left"
  sourcePosition: "right"
```

The `answer` field is a template string. Variables are referenced using `{{#node_id.output_field#}}` syntax.

#### 2.1.5 `end` Node

```yaml
- id: "end"
  data:
    type: "end"
    inputs: []                     # outputs to pass to the end
  position:
    x: 1200
    y: 0
  targetPosition: "left"
```

---

### 2.2 `workflow.graph.edges[]`

```yaml
edges:
  - source: "start"                # source node ID
    target: "llm_node_1"          # target node ID
    sourceHandle: "true"           # which output port (for if-else branches)
    targetHandle: null             # which input port
    type: "custom"
  - source: "llm_node_1"
    target: "answer_1"
    sourceHandle: null
    targetHandle: null
    type: "custom"
  - source: "if_else_1"
    target: "llm_node_high"
    sourceHandle: "true"          # if-else "true" branch
    targetHandle: null
    type: "custom"
  - source: "if_else_1"
    target: "llm_node_normal"
    sourceHandle: "false"          # if-else "false" branch
    targetHandle: null
    type: "custom"
```

**Edge structure**:
- `source` / `target` — node IDs
- `sourceHandle` — output port identifier (`"true"`/`"false"` for if-else branches, port index for multi-output nodes)
- `targetHandle` — input port identifier
- `type` — always `"custom"` in the DSL

---

### 2.3 Variable Reference Syntax

Dify uses **`{{#selector#}}`** syntax for variable references:

| Reference | Meaning |
|-----------|---------|
| `{{#start.variable_name#}}` | Variable from `start` node (user input) |
| `{{#sys.query#}}` | System variable: current user query |
| `{{#sys.files#}}` | System variable: uploaded files |
| `{{#sys.conversation_id#}}` | System variable: conversation ID |
| `{{#sys.user_id#}}` | System variable: user ID |
| `{{#sys.dialogue_count#}}` | System variable: turn count |
| `{{#sys.timestamp#}}` | System variable: Unix timestamp |
| `{{#llm_node_N.text#}}` | LLM node output text |
| `{{#llm_node_N.usage#}}` | LLM node token usage |
| `{{#env.VAR_NAME#}}` | Environment variable |
| `{{#conversation.var_name#}}` | Conversation variable |

**Selector format**: `{{#node_id.variable_name#}}`

The **`sys.`** prefix comes from `SYSTEM_VARIABLE_NODE_ID = "sys"` in `api/core/workflow/variable_prefixes.py`. System variable keys include: `query`, `files`, `conversation_id`, `user_id`, `dialogue_count`, `app_id`, `workflow_id`, `workflow_run_id`, `timestamp`, `document_id`, `batch`, `dataset_id`, etc. (defined in `SystemVariableKey` enum in `api/core/workflow/system_variables.py`).

**IMPORTANT**: The brief mentions `{{#sys.X#}}` vs `{{X}}`. Based on source analysis:
- **Workflow DSL uses `{{#node_id.variable#}}`** — this is confirmed by `export_dsl()` which stores the graph as-is
- The `{{X}}` bare syntax may be used in the Dify UI prompt editor but is **not** the exported DSL format
- Always use the `{{#...#}}` selector syntax in DSL files

---

### 2.4 `workflow.features`

```yaml
features:
  file_upload:
    enabled: true
    allowed_file_upload_methods: ["local", "url"]
    allowed_file_types: ["image", "document", "audio", "video"]
    allowed_file_extensions: [".pdf", ".docx", ".png"]
    number_limits: 5
    max_length: 15728640           # 15MB in bytes
  image_upload:
    enabled: true
    detail: "high"                 # "high" | "low" | "auto"
  opening_statement: |
    안녕하세요! MPS 주간 도우미입니다.
    어떤 과제에 대한 주간 계획을 세워드릴까요?
  opening_questions:
    - "이번 주 핵심 과제는 무엇인가요?"
    - "성과 목표는 어떻게 되시나요?"
    - "전략적 우선순위를 알려주세요."
  suggested_questions_after_answer:
    enabled: true
  speech_to_text:
    enabled: false
  text_to_speech:
    enabled: false
  retrieval_resource:
    enabled: true
  annotation_reply:
    enabled: false
```

**Features keys** (from `api/core/app/app_config/features/`):
- `file_upload` — file attachment settings
- `image_upload` — image upload + vision detail setting
- `opening_statement` — greeting message
- `opening_questions` — suggested starter questions
- `suggested_questions_after_answer` — follow-up question suggestions
- `speech_to_text` — voice input
- `text_to_speech` — voice output
- `retrieval_resource` — dataset retrieval button
- `annotation_reply` — annotation/autoreply
- `more_like_this` — similar content suggestions
- `rate_limiting` — rate limit configuration

---

### 2.5 `workflow.environment_variables[]`

```yaml
environment_variables:
  - id: "env_api_key"
    name: "OPENAI_API_KEY"
    value: "sk-..."              # actual or encrypted value
    value_type: "secret"         # "string" | "number" | "secret"
    description: "OpenAI API Key for LLM calls"
```

---

### 2.6 `workflow.conversation_variables[]`

```yaml
conversation_variables:
  - id: "cv_1"
    name: "weekly_summary"
    value_type: "string"
    value: ""
    description: "주간 요약 저장소"
```

---

## 3. Chat App Structure (`chat` / `agent-chat` / `completion` modes)

For EasyUI-based chat apps (no workflow graph):

```yaml
version: "0.1.8"
kind: "app"
app:
  name: "Simple Chatbot"
  mode: "chat"
  icon: "💬"
  icon_type: "emoji"
  icon_background: "#FFFFFF"
  description: "Simple chatbot"
model_config:
  model:
    provider: "openai"
    name: "gpt-4o-mini"
    mode: "chat"
    completion_params:
      temperature: 0.7
      max_tokens: 2000
  prompt_template:
    system_prompt: "당신은 도움이 되는 AI 어시스턴트입니다."
    user_prompt: "{{#sys.query#}}"
  variables:                    # same structure as start node variables
    - variable: "query"
      label: "질문"
      type: "text-input"
      required: true
  dataset_configs: {}           # optional knowledge retrieval config
  file_upload:
    enabled: false
```

---

## 4. Node Data Field Summary

| Node Type | Key Data Fields |
|-----------|----------------|
| `start` | `variables[]`, `is_advanced` |
| `llm` | `model`, `prompt_template` (system_prompt + user_prompt), `context`, `memory`, `context_window_size`, `vision` |
| `if-else` | `conditions_type`, `cases[]` (case_id, case_name, conditions[]), `default_flow` |
| `answer` | `answer` (template string), `variables[]` |
| `end` | `inputs[]` |
| `code` | `code`, `language`, `outputs` |
| `template-transform` | `template`, `output_type` |
| `http-request` | `method`, `url`, `headers`, `params`, `body`, `authorization` |
| `variable-assigner` | `assign_variable[]`, `type` |
| `knowledge-retrieval` | `dataset_ids[]`, `retrieval_mode`, `multiple_retrieval_config` |
| `question-classifier` | `model`, `prompt_template`, `labels[]` |
| `parameter-extractor` | `model`, `prompt_template`, `parameters[]` |
| `tool` | `provider_id`, `tool_name`, `tool_parameters` |
| `iteration` | `max_iteration`, `parallel` |

---

## 5. DSL Version

The current DSL version is defined as `CURRENT_APP_DSL_VERSION` imported from `constants.dsl_version` in `api/services/app_dsl_service.py`.

The `check_version_compatibility()` function (`api/services/dsl_version.py`) uses semantic versioning:
- Imported version > current version → `ImportStatus.PENDING` (needs confirmation)
- Major version mismatch → `ImportStatus.PENDING`
- Minor version mismatch → `ImportStatus.COMPLETED_WITH_WARNINGS`
- Compatible → `ImportStatus.COMPLETED`

---

## 6. Known Uncertainties

The following items could not be fully verified from source code analysis and represent the closest approximation:

1. **`answer` node `variables` field**: The `answer` node's `variables` array may list referenced variables for the UI, but the actual output comes from the `answer` template string. This needs confirmation.

2. **`if-else` condition format**: The exact `condition` operator values (`"contains"`, `"equals"`, `"regex"`, etc.) are inferred from TypeScript type definitions and documentation. The exact string values should be verified against the if-else node implementation.

3. **Variable selector in `llm` context field**: The `context` field in the `llm` node accepts dataset configuration. Its exact nested structure (`dataset_ids[]`, `retrieval_mode`, etc.) is based on `KnowledgeRetrievalNodeData` and should be verified.

4. **Memory `query_prompt_template` default**: The default template for conversation memory queries may differ. The value `"{{#sys.query#}}"` is the most common pattern.

5. **`answer` node status fields**: The `status`, `error`, and `elapsed_time` fields in the `answer` node are runtime fields that may not be needed in the DSL input — they appear to be populated on export.

6. **`features` completeness**: The `features` block may contain additional keys not listed here. The listed keys are confirmed from the `features/` directory structure in `api/core/app/app_config/features/`.

---

## 7. Minimal Valid DSL Example

```yaml
version: "0.1.8"
kind: "app"
app:
  name: "Minimal Chatbot"
  mode: "advanced-chat"
  icon: "🤖"
  icon_type: "emoji"
  icon_background: "#FFFFFF"
  description: "A minimal chatbot example"
workflow:
  graph:
    nodes:
      - id: "start"
        data:
          type: "start"
          variables:
            - variable: "user_query"
              label: "질문"
              type: "text-input"
              required: true
              default: ""
          is_advanced: false
        position:
          x: 0
          y: 0
        targetPosition: "left"
        sourcePosition: "right"
      - id: "llm_1"
        data:
          type: "llm"
          model:
            provider: "openai"
            name: "gpt-4o-mini"
            mode: "chat"
            completion_params:
              temperature: 0.7
              max_tokens: 1000
          prompt_template:
            system_prompt: "당신은 유용한 어시스턴트입니다."
            user_prompt: "{{#start.user_query#}}"
          context: {}
        position:
          x: 300
          y: 0
        targetPosition: "left"
        sourcePosition: "right"
      - id: "answer_1"
        data:
          type: "answer"
          answer: "{{#llm_1.text#}}"
          variables: []
        position:
          x: 600
          y: 0
        targetPosition: "left"
        sourcePosition: "right"
      - id: "end"
        data:
          type: "end"
          inputs: []
        position:
          x: 900
          y: 0
        targetPosition: "left"
    edges:
      - source: "start"
        target: "llm_1"
        sourceHandle: null
        targetHandle: null
        type: "custom"
      - source: "llm_1"
        target: "answer_1"
        sourceHandle: null
        targetHandle: null
        type: "custom"
      - source: "answer_1"
        target: "end"
        sourceHandle: null
        targetHandle: null
        type: "custom"
  features:
    opening_statement: "안녕하세요! 무엇을 도와드릴까요?"
```

---

## 8. 분기 포함 미니멀 DSL 예시 (if-else Branching)

다음 예시는 `start` 노드에서 `mode` 선택 변수를 입력받아, `if-else` 노드로 Planning/Evaluation 분기를 처리하는 구조를 보여준다.

```yaml
version: "0.1.8"
kind: "app"
app:
  name: "MPS 분기 챗봇"
  mode: "advanced-chat"
  icon: "🤖"
  icon_type: "emoji"
  icon_background: "#FFFFFF"
  description: "Planning vs Evaluation 분기를 지원하는 MPS 챗봇"
workflow:
  graph:
    nodes:
      - id: "start"
        data:
          type: "start"
          variables:
            - variable: "mode"
              label: "작업 모드"
              type: "select"
              required: true
              options: ["Planning", "Evaluation"]
              default: "Planning"
          is_advanced: false
        position:
          x: 0
          y: 100
        targetPosition: "left"
        sourcePosition: "right"
      - id: "if_else_1"
        data:
          type: "if-else"
          conditions_type: "and"
          cases:
            - case_id: "true"
              case_name: "Planning 분기"
              conditions:
                - variable: "{{#start.mode#}}"
                  condition: "equals"
                  value: "Planning"
          default_flow: "false"
        position:
          x: 300
          y: 100
        targetPosition: "left"
        sourcePosition: "right"
      - id: "llm_planning"
        data:
          type: "llm"
          model:
            provider: "openai"
            name: "gpt-4o-mini"
            mode: "chat"
            completion_params:
              temperature: 0.7
              max_tokens: 1000
          prompt_template:
            system_prompt: "당신은 MPS Planning 어시스턴트입니다. 실행 가능한 주간 계획을 세워주세요."
            user_prompt: "{{#start.mode#}} 모드로 동작합니다."
          context: {}
        position:
          x: 600
          y: 0
        targetPosition: "left"
        sourcePosition: "right"
      - id: "llm_evaluation"
        data:
          type: "llm"
          model:
            provider: "openai"
            name: "gpt-4o-mini"
            mode: "chat"
            completion_params:
              temperature: 0.7
              max_tokens: 1000
          prompt_template:
            system_prompt: "당신은 MPS Evaluation 어시스턴트입니다. 계획의 실행 결과를 평가해주세요."
            user_prompt: "{{#start.mode#}} 모드로 동작합니다."
          context: {}
        position:
          x: 600
          y: 200
        targetPosition: "left"
        sourcePosition: "right"
      - id: "answer_1"
        data:
          type: "answer"
          answer: |
            ## 결과

            {{#llm_planning.text#}}{{#llm_evaluation.text#}}
          variables: []
        position:
          x: 900
          y: 100
        targetPosition: "left"
        sourcePosition: "right"
      - id: "end"
        data:
          type: "end"
          inputs: []
        position:
          x: 1200
          y: 100
        targetPosition: "left"
    edges:
      - source: "start"
        target: "if_else_1"
        sourceHandle: null
        targetHandle: null
        type: "custom"
      - source: "if_else_1"
        target: "llm_planning"
        sourceHandle: "true"
        targetHandle: null
        type: "custom"
      - source: "if_else_1"
        target: "llm_evaluation"
        sourceHandle: "false"
        targetHandle: null
        type: "custom"
      - source: "llm_planning"
        target: "answer_1"
        sourceHandle: null
        targetHandle: null
        type: "custom"
      - source: "llm_evaluation"
        target: "answer_1"
        sourceHandle: null
        targetHandle: null
        type: "custom"
      - source: "answer_1"
        target: "end"
        sourceHandle: null
        targetHandle: null
        type: "custom"
  features:
    opening_statement: "Planning 모드와 Evaluation 모드 중 선택해주세요."
```

**주요 포인트**:
- `if-else` 노드의 `sourceHandle: "true"`은 Planning 분기(Planning일 때), `sourceHandle: "false"`는 Evaluation 분기를 연결한다.
- 두 LLM 노드는 `start.mode` 변수의 값에 따라 서로 다른 시스템 프롬프트를 사용한다.
- 두 LLM 노드의 출력은同一个 `answer_1` 노드로汇聚한다.

---

## 9. Sources Investigated

| Source | What Was Found |
|--------|----------------|
| `api/services/app_dsl_service.py` | `export_dsl()`, `import_app()`, `_create_or_update_app()`, DSL top-level keys, version handling, mode-based branching |
| `api/services/dsl_version.py` | `check_version_compatibility()` semantic versioning logic |
| `api/core/workflow/system_variables.py` | `SystemVariableKey` enum, `system_variable_selector()`, variable prefix `sys` |
| `api/core/workflow/variable_prefixes.py` | `SYSTEM_VARIABLE_NODE_ID = "sys"`, `ENVIRONMENT_VARIABLE_NODE_ID = "env"`, `CONVERSATION_VARIABLE_NODE_ID = "conversation"` |
| `web/app/components/workflow/types.ts` | `BlockEnum` node types, `InputVarType`, `Memory`, `PromptItem`, `Edge` type, `WorkflowDataUpdater` |
| `api/core/app/apps/chat/app_config_manager.py` | Chat app config structure (EasyUI-based app) |
| Dify GitHub repo structure | `api/core/app/apps/`, `api/core/workflow/nodes/`, `api/services/` directories |

---

*이 정보는 2026-07-06 시점 Dify GitHub master 브랜치 버전을 기반으로 합니다.*
