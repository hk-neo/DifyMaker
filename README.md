# MPS Dify Assistant

> Dify 챗봇 DSL 프로젝트 — MPS(Mission-Performance-Strategy) 주간 Planning/Evaluation 마크다운 자동 생성

## 📦 앱 목록

| 앱 이름 | 설명 | 파일 |
|--------|------|------|
| MPS 주간 도우미 | 주간 MPS 마크다운을 자동 생성하는 Dify 챗봇 | [`dsl/MPS-주간-도우미.yml`](dsl/MPS-주간-도우미.yml) |

## 🚀 빠른 시작

1. [Dify](https://dify.ai)에서 워크스페이스 열기
2. **Studio** → **Import from DSL**
3. [`dsl/MPS-주간-도우미.yml`](dsl/MPS-주간-도우미.yml) 업로드
4. 앱 설정 → 모델 확인 (기본 `gpt-oss`)
5. **Publish** 클릭

자세한 사용법: [`dsl/README.md`](dsl/README.md)

## 📁 프로젝트 구조

```
DifyMaker/
├── dsl/
│   ├── MPS-주간-도우미.yml    # Dify 앱 DSL (Import 가능)
│   ├── README.md              # DSL 사용법 가이드
│   └── research/
│       ├── dify-dsl-schema.md # Dify DSL 스키마 조사
│       └── validation-report.md
└── docs/
    └── superpowers/
        ├── specs/             # 디자인 스펙
        └── plans/             # 구현 계획
```

## 🔗 관련 문서

- DSL 사용법: [`dsl/README.md`](dsl/README.md)
- 스펙: [`docs/superpowers/specs/2026-07-06-mps-dify-design.md`](docs/superpowers/specs/2026-07-06-mps-dify-design.md)
- 구현 계획: [`docs/superpowers/plans/2026-07-06-mps-dify-implementation.md`](docs/superpowers/plans/2026-07-06-mps-dify-implementation.md)