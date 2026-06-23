# 청킹 파이프라인 TODO List & 스펙

---

## 1. 프로젝트 셋업

- [ ] Poetry 프로젝트 초기화
- [ ] 의존성 정의 (pyproject.toml)
- [ ] 폴더 구조 생성
- [ ] 환경변수 설정 (.env)
- [ ] 로깅 설정

### 의존성 스펙

```toml
[tool.poetry.dependencies]
python = "^3.11"

# 파이프라인
langgraph = "^0.2"
langchain-anthropic = "*"

# 문서 파싱
pymupdf = "*"
docling = "*"

# 청킹
chonkie = "*"

# 평가
ragas = "*"

# 벡터 DB
qdrant-client = "*"

# 유틸
python-dotenv = "*"
pydantic = "^2.0"
loguru = "*"
```

### 폴더 구조

```

├── pyproject.toml
├── .env
├── main.py                  ← 진입점
├── src/
│   ├── state.py             ← AgentState 정의
│   ├── graph.py             ← LangGraph 그래프 조립
│   ├── nodes/
│   │   ├── analyze.py
│   │   ├── plan.py
│   │   ├── ocr.py
│   │   ├── vlm.py
│   │   ├── parse.py
│   │   ├── preprocess.py
│   │   ├── chunk.py
│   │   ├── postprocess.py
│   │   ├── evaluate.py
│   │   └── store.py
│   ├── routers/
│   │   └── conditions.py    ← 조건부 엣지 함수
│   └── utils/
│       ├── pdf_utils.py
│       └── chunk_utils.py
└── tests/
    ├── test_nodes.py
    └── test_graph.py
```

---

## 2. State 정의

- [ ] AgentState TypedDict 정의
- [ ] 각 필드 타입 및 기본값 명세
- [ ] Chunk 데이터 클래스 정의
- [ ] ChunkingPlan 데이터 클래스 정의

### 스펙

```python
# src/state.py

class AnalysisResult(TypedDict):
    has_text_layer: bool
    image_ratio: float      # 0~100
    header_count: int
    table_detected: bool
    page_count: int
    needs_ocr: bool
    needs_vlm: bool
    domain: str             # "financial" | "legal" | "general"

class ChunkingPlan(TypedDict):
    strategy: str           # "recursive" | "hierarchical" | "hybrid" | "page"
    chunk_size: int         # 256 ~ 1024
    overlap: int            # 0 ~ 200
    special_handlers: list  # ["table", "ocr", "vlm"]

class Chunk(TypedDict):
    text: str
    type: str               # "text" | "table" | "image"
    metadata: dict          # page, doc_name, header_path, strategy
    score: float            # stickiness 점수

class EvalResult(TypedDict):
    stickiness: float
    boundary_clarity: float
    passed: bool
    reason: str

class AgentState(TypedDict):
    # 입력
    file_path: str
    config: dict

    # 단계별 결과
    analysis: AnalysisResult
    plan: ChunkingPlan
    content: dict           # raw, tables, text_blocks
    chunks: list[Chunk]
    eval_result: EvalResult

    # 메타
    retry_count: int
    errors: list[str]
    status: str             # "running" | "done" | "failed"
```

---

## 3. 노드 구현

### 3-1. analyze 노드

- [ ] PyMuPDF로 PDF 열기
- [ ] 첫 페이지 텍스트 추출 → has_text_layer 판단 (기준: 100자 이상)
- [ ] 이미지 수 기반 image_ratio 계산
- [ ] 헤더(#) 카운트
- [ ] 표 존재 여부 (`|` 문자 기반)
- [ ] needs_ocr, needs_vlm 플래그 설정
- [ ] 에러 처리 (파일 없음, 손상된 PDF)

**판단 기준 스펙**

```
needs_ocr  = has_text_layer == False
needs_vlm  = image_ratio > 40
domain     = "financial" if 특정 키워드 포함 else "general"
```

---

### 3-2. plan 노드

- [ ] analysis 결과 기반 strategy 결정
- [ ] retry 시 chunk_size 조정 로직
- [ ] special_handlers 리스트 구성

**전략 결정 스펙**

```
needs_ocr  → strategy: "page",         chunk_size: 512, overlap: 50
needs_vlm  → strategy: "page",         chunk_size: 512, overlap: 50
header > 5 → strategy: "hierarchical", chunk_size: 512, overlap: 100
table      → strategy: "hybrid",       chunk_size: 600, overlap: 50
else       → strategy: "recursive",    chunk_size: 512, overlap: 100

retry 시:
  chunk_size = int(current_chunk_size * 0.8)
  overlap    = int(current_overlap * 1.2)
```

---

### 3-3. ocr 노드

- [ ] Mistral OCR API 연동
- [ ] 페이지별 텍스트 추출
- [ ] 실패 시 PyMuPDF fallback
- [ ] 결과를 content.raw에 저장

**스펙**

```
입력:    file_path
출력:    content.raw (str)
fallback: PyMuPDF get_text()
timeout: 30초
```

---

### 3-4. vlm 노드

- [ ] PDF 페이지 → 이미지 변환 (PyMuPDF)
- [ ] 이미지 base64 인코딩
- [ ] Claude API 호출 (표 구조화 프롬프트)
- [ ] 마크다운 표 형식으로 반환
- [ ] content.tables에 저장

**프롬프트 스펙**

```
"이 표를 마크다운으로 변환하세요.
 - 헤더를 각 행에 명시
 - 단위 포함
 - | 구분자 사용"
```

---

### 3-5. parse 노드

- [ ] Docling DocumentConverter 실행
- [ ] 마크다운으로 export
- [ ] 표/텍스트 블록 분리
- [ ] content.raw, content.tables, content.text_blocks 구성

**스펙**

```
입력: file_path
출력:
  content.raw: str          (전체 마크다운)
  content.tables: list[str] (표 블록)
  content.text_blocks: list[str] (텍스트 블록)
```

---

### 3-6. preprocess 노드

- [ ] 헤더 경로 추출 (# → ## → ### 계층)
- [ ] 표 블록과 텍스트 블록 분리
- [ ] 노이즈 제거 (페이지 번호, 빈 줄 연속 등)
- [ ] 각 블록에 헤더 경로 태깅

---

### 3-7. chunk 노드

- [ ] plan.strategy 기반 분기
- [ ] 표 블록 → 행 단위 청킹
- [ ] 텍스트 블록 → Chonkie 청킹
- [ ] 전략별 Chonkie 청커 선택

**청커 선택 스펙**

```
"recursive"    → RecursiveChunker
"hierarchical" → SemanticChunker
"hybrid"       → 표: 행 단위 / 텍스트: RecursiveChunker
"page"         → 페이지 단위 그대로
```

---

### 3-8. postprocess 노드

- [ ] 각 청크에 메타데이터 주입
- [ ] 청크 크기 검증 (min: 50, max: chunk_size * 1.5)
- [ ] 너무 작은 청크 → 인접 청크와 병합
- [ ] 너무 큰 청크 → 재분할
- [ ] 중복 청크 제거

**메타데이터 스펙**

```python
chunk.metadata = {
    "doc_name":    str,   # 파일명
    "page":        int,   # 페이지 번호
    "header_path": str,   # "섹션1 > 섹션1.1"
    "type":        str,   # "text" | "table"
    "strategy":    str,   # 사용된 청킹 전략
    "chunk_index": int,   # 청크 순서
}
```

---

### 3-9. evaluate 노드

- [ ] Chunk Stickiness 계산 (문장 간 임베딩 유사도)
- [ ] Boundary Clarity 계산 (경계 전후 유사도 차이)
- [ ] 기준치 미달 시 passed=False
- [ ] 재시도 횟수 체크

**평가 기준 스펙**

```
stickiness       > 0.6  → 통과
boundary_clarity > 0.3  → 통과
둘 다 통과해야 passed=True

임베딩 모델: all-MiniLM-L6-v2
```

---

### 3-10. store 노드

- [ ] Qdrant 클라이언트 연결
- [ ] 청크 텍스트 임베딩 생성
- [ ] 벡터 + 메타데이터 upsert
- [ ] 컬렉션 없으면 자동 생성
- [ ] 저장 결과 로깅

**Qdrant 스펙**

```
collection_name: "shinhan_chunks"
vector_size:     384        # all-MiniLM-L6-v2
distance:        Cosine
payload:         Chunk.metadata 전체
```

---

## 4. 조건부 엣지

- [ ] route_after_plan() 구현
- [ ] route_after_evaluate() 구현

### 스펙

```python
def route_after_plan(state) -> str:
    if state["analysis"]["needs_ocr"]: return "ocr"
    if state["analysis"]["needs_vlm"]: return "vlm"
    return "parse"

def route_after_evaluate(state) -> str:
    if state["eval_result"]["passed"]: return "store"
    if state["retry_count"] < 2:      return "plan"   # chunk_size 조정 후 재시도
    return "store"                                     # fallback
```

---

## 5. 그래프 조립

- [ ] StateGraph 인스턴스 생성
- [ ] 노드 등록 (10개)
- [ ] 엣지 연결
- [ ] 조건부 엣지 연결
- [ ] compile()
- [ ] 배치 처리 (Send API) 구현

### 그래프 구조 스펙

```python
graph.add_edge(START,         "analyze")
graph.add_edge("analyze",     "plan")
graph.add_conditional_edges("plan", route_after_plan,
    {"ocr": "ocr", "vlm": "vlm", "parse": "parse"})
graph.add_edge("ocr",         "preprocess")
graph.add_edge("vlm",         "preprocess")
graph.add_edge("parse",       "preprocess")
graph.add_edge("preprocess",  "chunk")
graph.add_edge("chunk",       "postprocess")
graph.add_edge("postprocess", "evaluate")
graph.add_conditional_edges("evaluate", route_after_evaluate,
    {"store": "store", "plan": "plan"})
graph.add_edge("store",       END)
```

---

## 6. 테스트

- [ ] 노드 단위 테스트
  - [ ] analyze: 텍스트 PDF, 스캔 PDF 각각
  - [ ] plan: 각 분기 조건별
  - [ ] chunk: 전략별 출력 검증
  - [ ] evaluate: pass/fail 경계값
- [ ] 통합 테스트
  - [ ] 일반 PDF end-to-end
  - [ ] 스캔 PDF end-to-end
  - [ ] 표 포함 PDF end-to-end
  - [ ] 재시도 루프 동작 확인

---

## 7. 우선순위

### P0 — 필수 (1주차)

- [ ] State 정의
- [ ] analyze 노드
- [ ] plan 노드
- [ ] parse 노드
- [ ] chunk 노드
- [ ] 그래프 조립 (기본 흐름)

### P1 — 핵심 기능 (2주차)

- [ ] preprocess 노드
- [ ] postprocess 노드
- [ ] evaluate 노드
- [ ] store 노드

### P2 — 고도화 (3주차)

- [ ] ocr 노드
- [ ] vlm 노드
- [ ] 배치 처리 (Send API)
- [ ] 재시도 루프
