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