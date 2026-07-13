# DifyMaker — Progress Ledger

Repository: `git@github.com-neo:hk-neo/DifyMaker.git`
Branch: main

## Apps

### 1. MPS 주간 도우미 ✅
- **File**: `dsl/MPS-주간-도우미.yml`
- **Status**: v1 완료, Dify Import 통과, 런타임 동작 확인
- **Spec**: `docs/superpowers/specs/2026-07-06-mps-dify-design.md`
- **Dify 디버깅 누적 (8 픽스)**:
  1. if-else `logical_operator` 필수
  2. 변수 `hint` 필수 (프론트엔드 UI)
  3. if-else 필드명: `variable`→`variable_selector`, `operator`→`comparison_operator`
  4. `variable_selector` array 포맷 (string 아님)
  5. `prompt_template` array 포맷 (object 아님 — flatMap 요구)
  6. `comparison_operator: 'is'` (enum, `equals` 아님)
  7. `context.enabled: true` 필수
  8. `{{#start_node.X#}}` prefix (노드 ID 일치)
  + answer block scalar 내부 주석 제거 (출력 노출)

### 2. 점심 뭐먹지 ✅
- **File**: `dsl/점심뭐먹지.yml`
- **Status**: v2 완료 (실데이터 40개), Dify Import 통과
- **데이터**: 대륭포스트 7차 반경 500m, 네이버/구글/다음맵 기반, 카페 제외
- **픽스**: hint 값에 콜론(`예:`) → 따옴표 처리 (YAML 매핑 오류)

## Key Learnings (Dify DSL)
- 한국어 hint/label에 콜론(`:`) → 항상 따옴표
- 노드 ID = 변수 참조 prefix (`{{#NODE_ID.var#}}`)
- Dify 버전별 스키마 미세 차이 → 실제 에러 메시지 우선
- block scalar(`|`) 내부의 `#`는 주석 아님 → 출력에 노출됨

## Repository Hygiene
- `.gitignore` 생성 (2026-07-07): `.DS_Store`, SDD scratch diff/report/brief/json