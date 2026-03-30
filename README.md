# OpsAgent — Ops Decision Copilot

도메인 적응형 내부 AI 코파일럿 플랫폼

> 문서·CSV·지식 그래프를 결합해 운영 의사결정을 지원하는 실용적 AI 시스템.
> 어떤 업종이든 파일을 올리면 즉시 작동한다.

[![Streamlit App](https://static.streamlit.io/badges/streamlit_badge_black_white.svg)](https://ops-decision-copilot.streamlit.app)
[![Python 3.10+](https://img.shields.io/badge/Python-3.10+-blue.svg)](https://python.org)
[![Claude API](https://img.shields.io/badge/LLM-Claude%20Sonnet%204.6-orange.svg)](https://anthropic.com)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

---

## 목차

1. [배경 및 목적](#1-배경-및-목적)
2. [시스템 아키텍처](#2-시스템-아키텍처)
3. [핵심 기능 상세](#3-핵심-기능-상세)
4. [도메인 적응 구조](#4-도메인-적응-구조)
5. [채팅 라우터 & 질문 유형 분류](#5-채팅-라우터--질문-유형-분류)
6. [쿼리 플래너](#6-쿼리-플래너)
7. [지식 그래프 빌드 파이프라인](#7-지식-그래프-빌드-파이프라인)
8. [보안 설계](#8-보안-설계)
9. [데모 시나리오](#9-데모-시나리오)
10. [실행 방법](#10-실행-방법)
11. [기술 스택](#11-기술-스택)
12. [프로젝트 구조](#12-프로젝트-구조)
13. [한계 및 향후 계획](#13-한계-및-향후-계획)

---

## 1. 배경 및 목적

기업 내부에는 수많은 운영 데이터와 문서가 있지만, 담당자가 빠르게 조회하고 판단을 내리기 어렵다. ERP 조회·엑셀 분석·회의록 검색을 따로 해야 하고, 그 결과를 머릿속에서 통합해야 한다.

> "재고 위험 상품이 뭔지, 회의에서 어떤 결정을 했는지, 다음에 뭘 해야 하는지 — 한 번에 알 수 없을까?"

이 프로젝트는 그 질문에서 출발했다. 단순 RAG 챗봇을 넘어, **문서·데이터·지식 그래프를 결합**해 실무 의사결정 흐름을 AI가 직접 보조하는 내부 코파일럿을 구현했다.

**핵심 설계 원칙**

| 원칙 | 구현 |
|------|------|
| **도메인 무관** | 업종별 프리셋 7개 + Claude 동적 분석으로 어떤 도메인도 즉시 적응 |
| **증거 기반 답변** | 모든 답변에 사용 CSV·문서·KG 노드 수 badge 표시 |
| **판단 보조** | 데이터 조회 후 항상 "오늘/이번 주/이번 달 할 일" 자동 제시 |
| **모듈 독립성** | 분석 모듈이 Streamlit에 의존하지 않아 단독 테스트·재사용 가능 |
| **3-Layer 라우팅** | 질문 → data / doc / combined 자동 분류 후 전문 엔진 위임 |
| **스트리밍** | `st.write_stream` 기반 실시간 Claude 응답 스트리밍 |
| **보안** | 프롬프트 인젝션 방어 · XSS 방지 · 경로 순회 방어 |

---

## 2. 시스템 아키텍처

### 전체 흐름

```
사용자
  │
  ▼
Step 1: 도메인 설정
  │  domains/ 프리셋 선택 또는 직접 입력 → Claude가 entity_types·테마 자동 생성
  ▼
Step 2: 파일 업로드 (PDF / DOCX / TXT / CSV / JSON)
  │
  ├─→ document_parser  ─→ RAGEngine (ChromaDB)      ← 시맨틱 검색
  ├─→ extract_csv_schema ─→ KnowledgeGraph (NetworkX)
  └─→ CSVRegistry ─→ data_analyst / data_chat_engine
  ▼
Step 3: 결과 대시보드
  │
  ├─→ 지식 그래프 (3가지 뷰: Node / ERD / Data Flow)
  ├─→ AI 분석 탭 (요약 / 액션아이템 / 원인분석 / 보고서)
  ├─→ 일일 브리핑 4카드
  └─→ 데이터 채팅 패널 (스트리밍)
           │
           ▼
      chat_copilot (라우터)
           │
     ┌─────┼─────┐
     ▼     ▼     ▼
   data   doc  combined
     │     │     │
     ▼     ▼     ▼
  data_  RAG+  RAG+KG+
  chat_  KG    data
  engine
```

### 모듈 구성

```
┌───────────────────────────────────────────────────────────┐
│                    app.py  (UI Orchestration)              │
│   Step 1 → Step 2 → Step 3  (Streamlit session_state)     │
└────────────┬──────────────────┬───────────────────────────┘
             │                  │
  ┌──────────▼──────────┐  ┌────▼───────────────────────┐
  │  Document Pipeline  │  │     Analytics Engine        │
  │                     │  │                             │
  │  RAGEngine          │  │  data_analyst               │
  │  (ChromaDB)         │  │  data_chat_engine           │
  │                     │  │  query_planner              │
  │  KnowledgeGraph     │  │                             │
  │  (NetworkX+pyvis)   │  │  5종 질문 유형 자동 분류      │
  │  · Node Graph       │  │  CHART / RANKING /          │
  │  · ERD Table View   │  │  COMPARISON / RISK /        │
  │  · Data Flow View   │  │  DESCRIPTION                │
  │                     │  │                             │
  │  document_parser    │  │  ThreadPoolExecutor         │
  └─────────────────────┘  │  (KG 병렬 추출)              │
                            └────────────────────────────┘
                                          │
                    ┌─────────────────────┼──────────────────┐
                    │                     │                  │
          ┌─────────▼──────┐  ┌───────────▼──┐  ┌──────────▼──────┐
          │  Claude API    │  │  domains/    │  │  prompts/       │
          │  (Anthropic)   │  │  presets     │  │  templates      │
          │                │  │              │  │                 │
          │  claude_client │  │  beauty      │  │  system_base    │
          │  · generate()  │  │  supply_chain│  │  summarize      │
          │  · stream()    │  │  energy      │  │  action_items   │
          │  · retry 3회   │  │  finance     │  │  root_cause     │
          └────────────────┘  │  logistics   │  │  report_draft   │
                              │  manufactur. │  │  rag_query      │
                              │  generic     │  └─────────────────┘
                              └──────────────┘
```

---

## 3. 핵심 기능 상세

### 일일 브리핑 (Daily Briefing)

4개 카드 자동 생성. "📋 브리핑 생성" 버튼 한 번으로 오늘의 운영 현황을 즉시 파악한다.

| 카드 | 내용 | 핵심 지표 |
|------|------|-----------|
| 🚨 위험 항목 현황 | CRITICAL/WARNING 재고 | 위험 상품 수, 최저 커버리지 |
| 📈 채널·경로별 TOP3 | 최근 월 채널별 판매 순위 | 채널명, 판매량 |
| 📦 처리 필요 항목 | 발주 없는 위험 재고 상품 | 상품명, 재고 커버리지일 |
| ⚡ 이상 변화 감지 | 최근 2개월 vs 직전 2개월 ±50% 변동 | 상품명, 변화율 |

각 카드: **한 줄 요약** + **핵심 수치 3개** + **지금 해야 할 일** + **채팅으로 이어서 질문** 버튼

### 지식 그래프 (3가지 뷰)

| 뷰 | 설명 |
|----|------|
| 🕸️ Node Graph | pyvis 인터랙티브 HTML (드릴다운, 노드 클릭 → 상세 정보) |
| 📋 ERD Table View | 카드형 테이블 스키마 — 컬럼·PK·FK·관계 표시 |
| 🔀 Data Flow View | MST → FACT 방향 흐름 + JOIN 키 표시 |

### 문서 분석 (Document Analysis)

업로드한 문서를 Claude가 4가지 관점으로 분석한다.

| 탭 | 내용 |
|----|------|
| 📋 요약 | 문서 핵심 내용 3~5줄 요약 |
| ✅ 액션 아이템 | 실행 항목 추출 + **연관 데이터셋 badge 자동 연결** |
| 🔍 원인 분석 | 이슈·리스크 원인 구조 분석 |
| 📝 보고서 초안 | 보고용 문서 자동 생성 |

### 스트리밍 채팅

`ClaudeClient.stream()` → `st.write_stream()` 파이프라인으로 토큰 단위 실시간 출력. 429/529 응답 시 exponential backoff(1s→2s→4s) 최대 3회 재시도.

---

## 4. 도메인 적응 구조

### 프리셋 도메인

`domains/` 패키지에 7개 도메인 프리셋이 정의되어 있다.

| 도메인 | 샘플 데이터 | 키 엔티티 |
|--------|------------|-----------|
| 뷰티·이커머스 | `data/beauty/` | product, channel, warehouse |
| 공급망·재고 | `data/supply_chain/` | part, supplier, warehouse |
| 에너지 | `data/energy/` | plant, equipment, grid |
| 금융·핀테크 | `data/finance/` | merchant, transaction, fraud |
| 물류·배송 | `data/logistics/` | hub, route, vehicle |
| 제조·생산 | `data/manufacturing/` | line, equipment, product |
| 범용 비즈니스 | - | strategy, metric, risk |

### 신규 도메인 추가

```python
# domains/healthcare.py
PRESET = {
    "entity_types": {
        "patient":   "#2196F3",
        "doctor":    "#4CAF50",
        "facility":  "#FF9800",
        "issue":     "#F44336",
        "decision":  "#4CAF50",
        "metric":    "#607D8B",
        "default":   "#9E9E9E",
    },
    "terminology":       ["EMR", "DRG", "처방전", "진단코드"],
    "document_patterns": ["진료기록", "원무보고서", "감염관리보고서"],
    "analysis_focus":    ["환자 안전", "진료 효율", "비용 최적화"],
    "theme_color":       "#0284c7",
    "app_icon":          "🏥",
}

# domains/__init__.py에 한 줄 추가
from domains.healthcare import PRESET as _healthcare
ALL_PRESETS["의료"] = _healthcare
```

**프리셋 없이 텍스트만 입력해도** Claude가 즉시 맞춤 entity_types·용어·테마색상을 생성한다.

---

## 5. 채팅 라우터 & 질문 유형 분류

### 3단계 라우팅

`chat_copilot.py`가 질문을 분석해 최적 엔진으로 위임한다.

```
질문 입력
    │
    ▼
sanitize_input()  ← 프롬프트 인젝션 필터
    │
    ▼
detect_route()
    │
    ├─ data 키워드 감지 (판매/재고/발주/차트/제품코드)  → data_chat_engine
    ├─ doc 키워드 감지 (회의록/보고서/정책/의사결정)   → RAG + KG
    └─ 둘 다 / 둘 다 없음                            → combined (병렬 처리)
```

### 5종 질문 유형 (data 경로)

`data_chat_engine.py`가 data/combined 경로 질문을 세부 분류한다.

| 유형 | 트리거 예시 | 반환 |
|------|------------|------|
| CHART | `제품코드 차트 그려줘`, `채널별 그래프` | Plotly 라인/바 차트 |
| RANKING | `잘 팔리는 상품 TOP5`, `여름 시즌 판매 순위` | 수평 바 차트 + 순위 |
| COMPARISON | `작년 대비 비교`, `YoY 분석` | 연도별 바 차트 |
| RISK | `재고 위험 상품`, `품절 위기 목록` | 커버리지 차트 + 발주 현황 |
| DESCRIPTION | 위 외 모든 질문 | RAG + KG 결합 텍스트 답변 |

**모든 답변 구조**: 요약 1줄 → 핵심 수치 3개(metrics badge) → 차트 → 해석 → 다음 할 일(오늘/이번 주/이번 달)

---

## 6. 쿼리 플래너

업무 문장 한 줄을 입력하면 필요한 데이터셋을 자동 추천한다.

```
"여름 성수기 대비 재고 부족 상품을 파악하고 싶다"
        │
        ▼
Pass 1  키워드 매칭
        │  스키마 레지스트리 + 업로드 파일 컬럼 동적 보강
        │  → FACT_INVENTORY (0.9), FACT_MONTHLY_SALES (0.8) 매칭
        ▼
Pass 2a FK 체인 확장
        │  → MST_PRODUCT (FK: product_id, confidence 0.5 추가)
        ▼
Pass 2b Claude 이유 정제
        │  → 추천 이유 + 지금 확인할 질문 3개 + 다음 액션 제시
        ▼
Pass 3  RAG 문서 추천
        │  → 관련 문서(회의록, 보고서) 함께 반환
        ▼
결과 카드 표시 (데이터셋 + 문서 + 확인 질문 + 다음 액션)
```

---

## 7. 지식 그래프 빌드 파이프라인

파일 유형에 따라 4가지 빌드 경로가 있다.

| 파일 유형 | 빌더 | 방식 |
|-----------|------|------|
| PDF / DOCX / TXT | `build_from_claude_extraction()` | Claude가 엔티티·관계를 JSON으로 추출 (ThreadPoolExecutor 병렬) |
| SCHEMA_DEFINITION.json | `build_from_schema_json()` | 테이블 → 노드, FK → 엣지 직접 변환 |
| CSV | `build_from_csv_schema()` | 2-pass: 노드 추가 → FK 엣지 연결 |
| Python | `build_from_python_graph_data()` | AST: 클래스/함수/임포트 관계 추출 |

결과물: pyvis 인터랙티브 HTML (드릴다운, 노드 클릭 → 상세 정보) + ERD/Data Flow 정적 뷰

---

## 8. 보안 설계

| 항목 | 구현 |
|------|------|
| 프롬프트 인젝션 방어 | `sanitize_input()` — 역할 덮어쓰기 패턴 정규식 필터 (`ignore previous instructions` 등 14개 패턴) |
| XSS 방지 | `ensure_ascii=True` — JS 임베딩 시 특수문자 이스케이프 |
| 경로 순회 방어 | `pathlib` 기반 `safe_csv_path()` — `../` 상위 경로 접근 차단 |
| 데모 사용량 제한 | 세션당 API 호출 20회 제한, 소진 시 입력·버튼 비활성화 |

---

## 9. 데모 시나리오

### 시나리오 1: 공급망 재고 위험 분석

1. Step 1 → "🚀 샘플 데이터로 바로 시작" 클릭
2. "📋 브리핑 생성" → 재고 위험·채널 판매·발주 현황 4카드 확인
3. 채팅: `재고 CRITICAL 상품 발주 우선순위 알려줘` → 위험 목록 + 다음 액션
4. 채팅: `최근 3개월 채널별 판매 추이 그래프` → Plotly 채널 차트

### 시나리오 2: 회의록 → 데이터 연결

1. 회의록 TXT 파일 업로드
2. AI 분석 탭 → "✅ 액션 아이템 추출" → 연관 데이터셋 badge 자동 표시
3. "🔍 데이터 추천 자세히 보기" → 추천 이유·확인 질문·다음 액션 확인

### 시나리오 3: 신규 도메인 즉시 적응

1. Step 1에서 "금융 사기 탐지" 직접 입력
2. Claude가 금융 도메인 entity_types·용어·테마 자동 생성
3. `data/finance/` 샘플 데이터 로드 또는 직접 CSV 업로드
4. 지식 그래프·채팅 즉시 사용 가능

### 시나리오 4: CSV 직접 업로드 (어떤 파일이든)

1. 자사 판매 데이터 CSV 업로드
2. CSVRegistry가 컬럼 구조를 보고 역할(SALES/INVENTORY 등) 자동 분류
3. 기존 분석 함수가 그대로 작동 — 파일명·컬럼명 무관

---

## 10. 실행 방법

```bash
git clone https://github.com/SSEUNGSSEUNGWOO/OpsAgent.git
cd OpsAgent
pip install -r requirements.txt
```

API 키 설정 (둘 중 하나):

```bash
# 방법 1: .env 파일
echo "ANTHROPIC_API_KEY=sk-ant-..." > .env

# 방법 2: Streamlit secrets (클라우드 배포 시)
mkdir -p .streamlit
echo '[secrets]\nANTHROPIC_API_KEY = "sk-ant-..."' > .streamlit/secrets.toml
```

```bash
streamlit run app.py
```

### 샘플 데이터 생성 (선택)

```bash
python scripts/generate_demo_data.py
python scripts/generate_supply_chain_demo.py
python scripts/generate_finance.py
python scripts/generate_logistics.py
python scripts/generate_energy.py
python scripts/generate_manufacturing.py
```

### Streamlit Cloud 배포

1. GitHub 레포를 Streamlit Cloud에 연결
2. Secrets에 `ANTHROPIC_API_KEY` 추가
3. Main file: `app.py`

---

## 11. 기술 스택

| 레이어 | 기술 | 선택 이유 |
|--------|------|-----------|
| LLM | Claude Sonnet 4.6 (Anthropic) | 한국어 처리 최적, 긴 컨텍스트, 구조화 출력 |
| 벡터 DB | ChromaDB | 경량 임베딩 DB, 로컬·클라우드 동일 동작 |
| 임베딩 | paraphrase-multilingual-MiniLM-L12-v2 | 한국어 시맨틱 검색 최적화 |
| UI | Streamlit | 빠른 프로토타입, 인터랙티브 컴포넌트 |
| 그래프 | NetworkX + pyvis | 직관적 KG 구축 + HTML 인터랙티브 렌더링 |
| 데이터 분석 | pandas + plotly | 경량 수치 분석, 인터랙티브 차트 |
| 문서 파싱 | PyPDF2 + python-docx | PDF/DOCX 텍스트 추출 |
| 도메인 적응 | DomainConfig dataclass | 설정 변경 없이 런타임 도메인 전환 |
| 보안 | re (정규식) + pathlib | 인젝션 방어, 경로 순회 차단 |

---

## 12. 프로젝트 구조

```
OpsAgent/
├── app.py                      # UI 오케스트레이션 (Streamlit 3-step flow)
├── config.py                   # 전역 설정 (API 키, 경로, 색상 상수)
│
├── domains/                    # 도메인 프리셋 패키지
│   ├── __init__.py             # ALL_PRESETS, get_preset()
│   ├── beauty.py               # 뷰티·이커머스
│   ├── supply_chain.py         # 공급망·재고
│   ├── energy.py               # 에너지·발전
│   ├── manufacturing.py        # 제조·생산
│   ├── logistics.py            # 물류·배송
│   ├── finance.py              # 금융·핀테크
│   └── generic.py              # 범용 비즈니스
│
├── modules/                    # 핵심 비즈니스 로직
│   ├── claude_client.py        # [LLM]       Claude API 래퍼 (generate/stream/retry)
│   ├── rag_engine.py           # [RAG]       ChromaDB 벡터 검색
│   ├── knowledge_graph.py      # [KG]        엔티티 관계 그래프 (3-view)
│   ├── domain_adapter.py       # [Domain]    Claude 동적 도메인 분석
│   ├── document_parser.py      # [Docs]      PDF/DOCX/CSV 파서
│   ├── data_analyst.py         # [Analytics] pandas 데이터 분석 함수
│   ├── data_chat_engine.py     # [Analytics] 5종 질문 유형 분류·답변
│   ├── query_planner.py        # [Analytics] 업무 문장 → 데이터 추천 (3-pass)
│   ├── chat_copilot.py         # [Chat]      3-way 라우터 + 보안 필터
│   ├── prompt_loader.py        # [Prompts]   system_base 자동 주입 템플릿 로더
│   └── supabase_client.py      # [DB]        Supabase 연동 (선택)
│
├── prompts/                    # Claude 프롬프트 템플릿 ({placeholder} 방식)
│   ├── system_base.txt         # 공통 베이스 프롬프트 (역할·규칙·도메인 컨텍스트)
│   ├── summarize.txt
│   ├── action_items.txt
│   ├── root_cause.txt
│   ├── report_draft.txt
│   ├── rag_query.txt
│   └── chat_routing.txt
│
├── data/                       # 도메인별 샘플 데이터
│   ├── (root)                  # 공급망 기본 데모 데이터 (FACT_MONTHLY_SALES 등)
│   ├── energy/                 # 에너지·발전
│   ├── finance/                # 금융·핀테크
│   ├── logistics/              # 물류·배송
│   ├── manufacturing/          # 제조·생산
│   └── supply_chain/           # 공급망·재고
│
├── scripts/                    # 샘플 데이터 생성 스크립트
│   ├── generate_demo_data.py
│   ├── generate_supply_chain_demo.py
│   ├── generate_finance.py
│   ├── generate_logistics.py
│   ├── generate_energy.py
│   └── generate_manufacturing.py
│
├── tests/                      # 테스트
└── requirements.txt
```

---

## 13. 한계 및 향후 계획

### 현재 한계

| 항목 | 내용 |
|------|------|
| 데모 제한 | Streamlit Cloud 무료 배포: API 호출 20회 제한 |
| 임베딩 속도 | 초기 로드 시 sentence-transformer 모델 다운로드 (~수십 초) |
| 대용량 CSV | pandas 인메모리 처리 — 수백만 행 이상 파일은 느림 |
| 멀티 유저 | Streamlit session_state 기반 — 사용자별 완전 격리 미지원 |

### 향후 계획

- **멀티 유저 지원**: Supabase 기반 사용자별 독립 세션·데이터 격리
- **에이전트 루프**: 자율적으로 데이터 조회→분석→판단을 반복하는 ReAct 루프
- **슬랙 연동**: 브리핑 결과를 주기적으로 슬랙 채널에 전송
- **SQL 커넥터**: CSV 업로드 대신 DB 직접 연결 (PostgreSQL, MySQL)
- **대시보드 저장**: 분석 결과를 PDF/PPT로 내보내기

---

**라이브 데모**: [ops-decision-copilot.streamlit.app](https://ops-decision-copilot.streamlit.app) *(API 호출 20회 제한 적용)*
