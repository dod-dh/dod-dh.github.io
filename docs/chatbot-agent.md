# 🚀 Enterprise Multi-Tenant AI Agent System
> **고객사 지식(Knowledge Base) 기반의 AI 상담 및 업무 대행(Action Executing) 에이전트 서비스**

---

## 프로젝트 정보

| 항목 | 내용 |
|---|---|
| **기간** | 2025.05 ~ 2025.06 (약 2개월) |
| **팀 구성** | 1인 (이후 Phase 2 AWS 베드락 디벨롭은 타 팀 인원) |
| **본인 역할** | 단독 개발 (Phase 1까지 진행) |
| **개발 형태** | 신규 구축 |

## 📌 Executive Summary (핵심 요약)
* **결론 중심의 설계 원칙**:
  1. **조회는 AI가 자동화, 변경성 액션은 사람이 확인/승인 후 실행** (권한 및 책임 분리)
  2. **Hallucination Zero**: 모르면 지어내지 않고 거부하는 엄격한 Guardrail 파이프라인 구축
  3. **데이터 기반의 품질 검증**: 감이 아닌 정량적 메트릭을 통한 RAG 파이프라인 최적화

---

## 🏗️ System Evolution (초기 구현 vs 고도화 차이)

본 프로젝트는 초기 **Gemini 기반 End-to-End RAG 파이프라인(Phase 1)** 구축을 시작으로, 운영 효율성과 보안성, 검색 정밀도를 극대화한 **AWS Bedrock 기반 고도화 아키텍처(Phase 2)**로 진화했습니다.

### 🔄 Phase Comparison Matrix
| 구분 | Phase 1 (초기 구축 - 본인 담당 영역) | Phase 2 (AWS Bedrock 기반 고도화) |
| :--- | :--- | :--- |
| **코어 엔진** | Google Gemini (LLM + Embedding) | AWS Bedrock (Claude Haiku/Sonnet + Nova 2) |
| **지식 구축 (ETL)** | FAQ/문서 파싱 ➔ Q&A 추출 ➔ **자체 검증 Loop** | 다중 지식 타입 분리 (FAQ, QnA, Chunked Doc, Synonyms) |
| **질문 정제** | Gemini 기반 쿼리 정제 (맥락 보정) | Haiku 기반 쿼리 재작성 + 동의어 확장 |
| **검색 방식** | Gemini Embedding + **Top-K (K=5) Cosine Similarity** | 하이브리드 검색 (Vector + Keyword GIN) + **Cohere Rerank** |
| **검색 단위** | Top-5 단일 Q&A Chunk 전달 | Small-to-Large (작게 찾고 확장하여 문맥 복원) |
| **검증 파이프라인** | 지식 추출 시 **Human-in-the-Loop** 모호성 검증 | **제3자 LLM(Haiku) 대조 검증** 및 Fact-Check |
| **도구 실행** | 조회성 데이터 API 매핑 | **권한 격리형 Tool Call** (로그인 세션 주입 + OTP 승인) |

---

## ⚙️ Phase 1: Gemini 기반 End-to-End RAG 파이프라인 구축 (My Key Contribution)

초기 단계에서는 FAQ 데이터 정제(ETL)부터 사용자 질문을 받아 실시간으로 답변을 생성하는 RAG 파이프라인 전체를 Gemini 단일 생태계 기반으로 구현했습니다.

### 1. 지식 구축 ETL & 자체 검증 파이프라인 (Off-line)
[원천 데이터] ➔ [Gemini: Q&A 추출] ➔ [Gemini: 1차 검증] ➔ [애매함?] ➔ (YES) ➔ [Human-in-the-Loop (사람 승인)]
⬇ (NO)
[Vector DB 저장] ⬅ [Gemini Embedding] ⬅ [확인된 Q&A 쌍 색인]

* **자동화 추출 및 표준화**: 비구조화된 FAQ/고객센터 원문 데이터를 입력받아 LLM(Gemini)을 활용해 질문-답변(Q&A) 쌍으로 가공.
* **Self-Consistency & Verification (자체 검증)**: 
  * 추출된 Q&A가 원본 데이터와 일치하는지 Gemini에 재검증 프롬프트 실행.
  * 신뢰도가 부족하거나 모호한 지식은 **Human-in-the-Loop(사람 운영자 확인)** 큐로 이관하여 지식베이스 오염 방지.
* **Vector DB 색인**: 검증을 통과한 고품질 데이터만 Gemini Text Embedding 모델을 거쳐 Vector DB에 저장.

### 2. 실시간 질답 서빙 파이프라인 (On-line RAG)
[사용자 질문 입력] ➔ [Gemini: 쿼리 정제] ➔ [Gemini Embedding] ➔ [Vector DB Top-5 Similarity Search] ➔ [Gemini: Context 기반 답변 생성]

* **질문 정제 (Query Preprocessing)**: 대화 맥락이나 모호한 단어가 포함된 사용자 질문을 Gemini를 통해 검색에 최적화된 명확한 문장으로 정제.
* **유사도 기반 Top-K Retrieval**: 정제된 질문을 Gemini Embedding 모델로 벡터화한 후, Vector DB에서 코사인 유사도(Cosine Similarity) 기준 **상위 5개(Top-K = 5)** 후보 문서/Q&A를 추출.
* **Context-Aware Response Generation**: 검색된 Top-5 지식 조각을 Prompt Context로 주입하여 Gemini가 등록된 지식 범위 안에서 정확한 답변을 생성하도록 유도.

---

## 🚀 Phase 2: AWS Bedrock 기반 Enterprise RAG & Agent 고도화

이후 엔터프라이즈 환경에 맞춰 보안성(AWS VPC 내 격리), 답변 정밀도, 업무 처리 능력을 한 단계 높이기 위해 시스템을 개편했습니다.

### 1. 6-Step RAG Pipeline (지식 담당)
Hallucination을 제로 수준으로 낮추기 위해 단일 LLM 호출이 아닌 6단계의 파이프라인을 거칩니다.

[사용자 질문]
│
├── ① 질문 정제 (Haiku): 대화 맥락 반영 및 쿼리 재작성
├── ② 하이브리드 검색 (Nova + pgvector + GIN): 의미(Vector) + 키워드(BM25) 병렬 수집
├── ③ Re-ranking (Cohere): 후보 수십 개 중 관련도 순 재정렬 (Phase 1 Top-5 한계 극복)
├── ④ 판정 (Haiku): "자료로 답할 수 있는가?" (충분/재검색/되묻기/거절)
├── ⑤ 답변 생성 (Sonnet): 선별된 Context 기반 템플릿 작성
└── ⑥ 근거 검증 (Haiku): 생성된 답이 원본 Context에만 기반하는지 대조 후 발송


### 2. RAG 성능 향상을 위한 데이터 처리 전략 (Data Indexing)
* **Small-to-Large Retrieval**: 조각(1,500자) 단위로 정밀하게 검색하되, LLM 답변 생성 시에는 전후 Context(`±3 Chunk`)를 복원하여 문맥 단절 문제 해결.
* **Header-Aware Chunking**: HTML/문서의 Heading 계층구조를 보존하여 쪼개고, 조각마다 `[제목]`, `[섹션]` 메타데이터를 결합해 벡터 유사도 정밀도 대폭 향상.
* **별칭(Synonym) 전용 색인**: 동의어나 줄임말이 본문 검색 신호를 희석하지 않도록 별도의 별칭 벡터를 분리 색인하여 병렬 검색 실행.

### 3. Agent Tool Executing (도구 및 권한 담당)
* **Deterministic Flow + LLM Hybrid**:
  * "주문 취소", "환불" 등 상태 변경 액션은 LLM의 재량에 100% 맡기지 않고, 노드 그래프 기반의 **결정론적 워크플로우 엔진**이 순서를 강제.
  * LLM은 사용자의 의도 파악 및 필요한 파라미터 추출 역할만 수행.
* **Security & Permission Control**:
  * **계정 ID 격리**: LLM Tool Parameter에서 계정 ID를 제외하고, 백엔드 서버가 인증된 HTTP Session에서 직접 주입하여 **타인 데이터 조회 가능성 근본 차단**.
  * **Write Action Guardrail**: 데이터 변경성 요청 시 UI에 **Direct Action Button**만 제공하여 사용자가 직접 터치하게 유도하며, 필요 시 OTP 인증 절차 강제.

---

## 🛠 Tech Stack (최종 아키텍처 기준)

* **Backend**: Java 17, Spring Boot 3.x, Spring AI
* **Frontend**: React 18, TypeScript, Vite
* **AI & Machine Learning**:
  * **LLM**: AWS Bedrock (Claude 3.5 Sonnet / Claude 3 Haiku), Google Gemini (Phase 1)
  * **Embedding**: Amazon Nova 2 Multimodal Embeddings, Gemini Embedding (Phase 1)
  * **Reranker**: Cohere Rerank v3
* **Database & Search**:
  * PostgreSQL (RDS) + **pgvector** (HNSW Indexing)
  * PostgreSQL Native Text Search (`tsvector` + GIN Index)
  * **Lucene-Nori**: App-level 한국어 형태소 분석기 및 고객사 커스텀 사전 동적 적용
* **Infrastructure**: AWS ECS Fargate, ALB, CloudFront, S3

---

## 💡 Key Takeaways & Lessons Learned

1. **Top-K 벡터 검색의 한계와 Re-ranker의 필요성**: Phase 1에서 구현한 단순 유사도 기반 Top-5 추출 방식은 단어 하나에 편향되어 정답 문서가 아래로 묻히는 현상이 발생했습니다. 이를 통해 Phase 2에서 **하이브리드 검색(Keyword+Vector)과 Re-ranker 도입**의 필위성을 도출하고 시스템을 고도화했습니다.
2. **모델 성능보다 지식 구조화가 더 중요하다**: RAG의 품질 향상을 견인한 가장 큰 요소는 더 강력한 LLM으로 바꾼 것이 아니라, **"자료를 어떻게 잘라내고, 어떻게 메타데이터를 결합해 넣어주는가"**였습니다.
3. **AI 서비스의 책임 분리**: 기술적으로 AI가 직접 DB 조작 API를 호출하도록 다 뚫어놓는 것은 가상 판례(예: Air Canada 챗봇 사건)와 같이 커다란 비즈니스 리스크를 초래합니다. **"판단은 AI, 실행/권한은 시스템 구조"**라는 가이드라인을 세워 엔터프라이즈급 안정성을 확보했습니다.
