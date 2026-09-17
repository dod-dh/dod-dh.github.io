# Multi-Tenant MCP Server
## 사내 AI Agent를 위한 공통 Tool Layer

> **수천 개 테넌트의 업무 데이터와 사내 기능을 AI Agent가 안전하게 사용할 수 있도록 구축한 공통 MCP 서버**
>
> 초기에는 LLM 챗봇의 고객사별 데이터 조회를 위한 MCP 서버로 시작했지만,
> 현재는 챗봇에 한정하지 않고 **사내 모든 AI Agent가 공통으로 Tool을 호출할 수 있는
> Multi-Tenant MCP Layer**로 확장하여 운영 환경에 배포했다.

---

## 1. 프로젝트 개요

| 항목 | 내용 |
|---|---|
| 기간 | 2026.08 ~ 2026.09 |
| 역할 | 단독 설계·개발 |
| 주요 기술 | Python, MCP Python SDK, PyMySQL, PyJWT, Paramiko |
| AI 연계 | AWS Bedrock (Claude) |
| 데이터 | MySQL / 고객사별 개별 DB |
| 배포 | Docker 기반 AWS EC2 운영 |
| 시스템 형태 | 사내 공통 AI Agent 인프라 |

### 프로젝트 목적

사내 AI 서비스가 개별적으로 DB 접근 및 업무 API를 구현하지 않고,
**하나의 공통 MCP 서버를 통해 필요한 데이터와 업무 기능을 Tool 형태로 사용할 수 있도록
공통 접근 계층을 구축**했다.

특히 고객사마다 별도의 DB를 사용하는 Multi-Tenant 환경에서
AI Agent가 잘못된 테넌트의 데이터에 접근하지 못하도록
**인증 → 테넌트 확정 → 데이터 라우팅 → Tool 실행**을 하나의 보안 경계로 설계했다.

---

## 2. 기존 구조의 문제

초기 요구사항은 LLM 고객지원 챗봇에서 다음과 같은 질문에 답할 수 있도록 하는 것이었다.

- "우리 업체의 재고 설정은 어떻게 되어 있나요?"
- "이 주문의 처리 상태를 알려주세요."
- "우리 업체에서 특정 화면이 다르게 동작하나요?"

문서 기반 RAG만으로는 이런 질문에 답할 수 없었다.

### 데이터가 고객사별로 분리된 환경

고객사마다 서로 다른 DB를 사용하고 있었기 때문에,

```text
AI Agent
   │
   ├── 고객 A 요청 → 고객 A DB
   ├── 고객 B 요청 → 고객 B DB
   └── 고객 C 요청 → 고객 C DB
```

와 같은 **테넌트 단위 데이터 라우팅**이 필요했다.

문제는 MCP Tool을 호출하는 주체가 사람이 아니라 **LLM/AI Agent**라는 점이었다.

따라서 다음과 같은 구조는 배제했다.

```python
get_order_status(
    domain="customer-a",
    order_id="..."
)
```

`domain`이 Tool Schema에 노출되면 모델이 대화 내용을 기반으로 해당 값을 직접 생성할 수 있기 때문이다.

---

# 3. 설계 핵심

## "테넌트는 인증의 산물이지, 대화의 내용물이 아니다."

테넌트 정보는 AI Agent가 Tool 인자로 선택하는 값이 아니라
**인증된 요청에서 서버가 확정하는 값**으로 정의했다.

```text
AI Agent
   │
   │ Authorization: Bearer JWT
   ▼
┌────────────────────────────────────┐
│          MCP Server                │
│                                    │
│  Tenant Middleware                 │
│    ├─ JWT 검증                     │
│    ├─ tenant claim 검증            │
│    ├─ Request Context 주입         │
│    └─ Audit Log                    │
│             │                      │
│             ▼                      │
│       MCP Tools                    │
│       ├─ 설정 조회                 │
│       ├─ 주문 조회                 │
│       ├─ CS 조회                   │
│       ├─ 하드코딩 조회             │
│       └─ 기타 업무 Tool            │
│             │                      │
│             ▼                      │
│       Tenant Resolver              │
│             │                      │
│       Registry + TTL Cache         │
│             │                      │
│             ▼                      │
│        Tenant DB                  │
└────────────────────────────────────┘
```

### 이 구조의 핵심

**Tool에는 `domain` 파라미터가 존재하지 않는다.**

```text
Tool 호출 인자
    ↓
업무 데이터만 전달
    ↓
테넌트 정보는 Request Context에서 조회
    ↓
해당 테넌트 DB로 자동 라우팅
```

이를 통해 Tool 개발자가 테넌트 처리 로직을 반복해서 구현하지 않아도
동일한 테넌트 격리 구조를 유지할 수 있도록 했다.

---

# 4. 공통 AI Agent Tool Layer로 확장

초기에는 **고객지원 챗봇 전용 MCP 서버**를 목표로 설계했다.

하지만 MCP의 역할을 특정 AI 서비스에 종속시키지 않고
**사내 AI Agent와 내부 시스템 사이의 공통 Tool Layer**로 분리했다.

```text
                         ┌───────────────┐
                         │   AI Chatbot  │
                         └───────┬───────┘
                                 │
                         ┌───────▼───────┐
                         │ AI Matching   │
                         │    Agent      │
                         └───────┬───────┘
                                 │
                    ┌────────────▼────────────┐
                    │                         │
                    │   Multi-Tenant MCP      │
                    │      Tool Layer         │
                    │                         │
                    └────────────┬────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
          고객사 DB          사내 시스템          업무 기능
```

### 역할 분리

AI Agent는 **판단과 오케스트레이션**을 담당하고,

MCP 서버는 **사내 데이터 및 기능에 안전하게 접근할 수 있는 실행 계층**을 담당한다.

```text
AI Agent
 ├─ 사용자 의도 파악
 ├─ 필요한 Tool 선택
 ├─ Tool 결과 해석
 └─ 여러 Tool 조합

        ↓ MCP

MCP Server
 ├─ 인증
 ├─ 테넌트 확정
 ├─ 데이터 라우팅
 ├─ 권한/입력 검증
 ├─ Tool 실행
 └─ 감사 로그
```

따라서 새로운 AI Agent가 추가되더라도
각 Agent가 고객사 DB 접근 로직을 새로 구현하지 않고
**기존 MCP Tool을 재사용할 수 있는 구조**를 만들었다.

---

# 5. 요청 처리 흐름

하나의 `tools/call` 요청은 다음 순서로 처리된다.

```text
1. AI Agent
   │
   │ MCP tools/call
   │ Authorization: Bearer <JWT>
   ▼
2. Tenant Middleware
   │
   ├─ JWT Signature 검증
   ├─ exp 검증
   ├─ tenant claim 추출
   ├─ tenant 형식 검증
   └─ Request Context에 tenant 저장
   │
   ▼
3. MCP Tool
   │
   └─ tenant_cursor()
          │
          ├─ 현재 tenant 확인
          ├─ Registry 조회
          ├─ TTL Cache 확인
          ├─ DB Connection Pool 획득
          └─ Query 실행
   │
   ▼
4. Tool Result
   │
   └─ AI Agent에 결과 반환
   │
   ▼
5. Request 종료
   │
   ├─ ContextVar reset
   └─ DB transaction rollback / connection 반환
```

요청이 종료되면 요청 스코프의 테넌트 정보를 반드시 제거하도록 구현했다.
워커가 재사용되는 환경에서 이전 요청의 테넌트가 남는 것은
곧 **타사 데이터 접근 문제로 이어질 수 있기 때문**이다.

---

# 6. Multi-Tenant Routing

## Tenant Registry

테넌트의 DB 접속 정보는 별도의 Registry에서 관리한다.

```text
JWT
 │
 │ tenant claim
 ▼
Tenant Resolver
 │
 ├─ TTL Cache Hit
 │      └─ DB 접근 생략
 │
 └─ Cache Miss
        │
        ▼
     Registry
        │
        ▼
   DB Connection Info
```

### TTL Cache

Tool 호출마다 Registry를 조회하면 시스템 DB에 불필요한 부하가 발생한다.

반대로 무기한 캐시하면 접속 정보 변경이나 테넌트 상태 변경이 즉시 반영되지 않는다.

따라서 **짧은 TTL 기반 캐시 + 명시적 무효화** 구조를 적용했다.

---

# 7. 고객사 DB Connection Pool

고객사 DB가 원격 서버에 존재하기 때문에
Tool 호출마다 새로운 DB 연결을 생성하는 방식은 불필요한 네트워크 비용을 발생시킨다.

이를 해결하기 위해 테넌트 DB Connection Pool을 구현했다.

```text
Tool Call
   │
   ▼
Connection Pool
   │
   ├─ Idle Connection 확인
   ├─ ping(reconnect=True)
   ├─ 필요 시 신규 연결
   └─ Query 실행
   │
   ▼
rollback()
   │
   ▼
Pool 반환
```

### Connection 안정성

장시간 유휴 상태의 Connection은 DB 서버에 의해 종료될 수 있기 때문에
Connection checkout 시 `ping(reconnect=True)`로 연결 상태를 확인한다.

또한 Connection 반환 전에 `rollback()`을 수행하여
이전 요청의 트랜잭션 상태가 다음 요청으로 넘어가지 않도록 했다.

---

# 8. 보안 설계

## 8.1 JWT 기반 Tenant 인증

모든 데이터 접근 요청은 JWT를 통해 테넌트를 확정한다.

검증 항목:

- JWT Signature
- 고정된 알고리즘
- `exp` 필수
- `iss` / `aud` 검증
- tenant claim 형식 검증
- 시계 오차 보정

```text
initialize / tools/list
        │
        └─ Tool Schema 조회 목적 → 허용

tools/call
resources/read
prompts/get
        │
        └─ 데이터 접근 → Tenant 인증 필수
```

---

## 8.2 Tool Schema에서 Tenant 제거

가장 중요한 보안 설계다.

```text
잘못된 구조

get_order_status(
    domain,
    order_id
)

        ↓

LLM이 domain을 선택할 수 있음
```

대신:

```text
안전한 구조

get_order_status(
    order_id
)

        ↓

TenantMiddleware
        ↓
JWT claim
        ↓
Request Context
        ↓
Tenant DB
```

**모델이 볼 수 있는 정보와 모델이 선택할 수 있는 정보를 최소화했다.**

---

## 8.3 범용 SQL Tool 배제

다음과 같은 Tool은 의도적으로 제공하지 않았다.

```python
run_query(sql: str)
```

범용 SQL 실행은 다음과 같은 문제를 만든다.

- 다른 DB/스키마 접근 가능성
- SQL Injection 및 잘못된 Query 위험
- 전체 스키마 노출
- 모델이 비효율적인 Query를 생성할 가능성
- 응답 크기 및 실행 비용 예측 어려움

대신 목적별 **명시적 Tool + 화이트리스트** 구조를 사용했다.

```text
read_config
get_order_status
get_order_cs
check_hardcoding
...
```

---

# 9. 민감정보 보호

고객사 설정 테이블에는 기능 설정과 함께
외부 연동 비밀번호, Token, 카드 관련 정보, DB 접속 정보 등
민감한 컬럼이 존재할 수 있다.

따라서 설정 조회 역시 전체 컬럼을 그대로 노출하지 않고
**화이트리스트 기반으로 조회 대상 자체를 제한**했다.

```text
사용자/Agent 입력
       │
       ▼
   라벨 검색
       │
       ▼
Whitelist
       │
       ├─ 허용 컬럼 → 조회
       │
       └─ 민감 컬럼 → 제외
```

추가로 컬럼명 기반 민감정보 패턴 검사를 적용해
화이트리스트 설정 실수에 대한 2차 방어도 구성했다.

---

# 10. Source Code 기반 업무 정보 조회

모든 업무 정보가 DB에 존재하는 것은 아니었다.

레거시 시스템에는 고객사별 하드코딩이 존재하기 때문에
특정 화면이 왜 다르게 동작하는지 판단하려면 소스 코드 확인이 필요했다.

이를 위해 MCP Tool에서 개발 서버의 소스를
**읽기 전용 SSH 기반으로 조회**하도록 구성했다.

```text
AI Agent
   │
   ▼
check_hardcoding(page_code)
   │
   ▼
MCP Server
   │
   └─ SSH Read Only
          │
          ├─ 화면 Template
          ├─ 화면 Class
          └─ 공통 Class
```

소스 원문 자체를 AI Agent에 반환하지 않고
필요한 판정 결과만 반환하도록 설계했다.

---

# 11. Tool 설계

현재 제공하는 대표 Tool은 다음과 같다.

| Tool | 역할 |
|---|---|
| `read_config` | 고객사 기능/환경 설정 조회 |
| `check_hardcoding` | 특정 화면의 고객사 전용 하드코딩 확인 |
| `get_order_cs` | 주문 CS 접수/처리 내역 조회 |
| `get_order_status` | 주문 진행 상태 조회 |
| `count_tables` | DB 연결 상태 확인 |
| `whoami` | 현재 연결된 Tenant 확인 |

모든 Tool에는 **Tenant 파라미터가 없다.**

Tool은 업무에 필요한 입력만 받고
테넌트 정보는 공통 인증 계층에서 가져온다.

---

# 12. Tool 응답 설계

AI가 직접 호출하는 API에서는 단순히 개발자에게 편한 응답보다
**모델 컨텍스트를 거쳐 사용자에게 전달되는 결과라는 점**을 고려해야 한다.

따라서 Tool 결과를 고객에게 전달 가능한 문장 중심으로 설계했다.

```text
check_hardcoding(page_code="A100")

        ↓

"A100 화면에 고객사 전용 로직이
 존재하는 것으로 확인되었습니다.
 자세한 내용은 고객센터에 문의해 주세요."
```

예외가 발생한 경우에도 DB Host, 계정명 등의 내부 정보가 포함된
원문 예외를 그대로 반환하지 않도록 공통 예외 처리 계층을 적용했다.

---

# 13. 테넌트별 Schema 차이 대응

고객사마다 시스템 업데이트 시점이 달라
동일한 설정 테이블이라도 실제 컬럼 구성이 다를 수 있었다.

```text
Tenant A
 └─ 약 570개 컬럼

Tenant B
 └─ 약 580개 컬럼
```

화이트리스트 전체를 그대로 SELECT하면
존재하지 않는 컬럼 하나 때문에 전체 조회가 실패할 수 있다.

따라서 매 호출 시 `information_schema`를 이용해
실제 컬럼과 조회 대상의 교집합을 구하도록 처리했다.

```text
Whitelist
   ∩
Tenant 실제 Schema
   ↓
실제 존재하는 컬럼만 조회
```

---

# 14. Stateless HTTP와 확장성

MCP Streamable HTTP는 **stateless 방식**으로 구성했다.

```text
              ┌───────────────┐
              │ Load Balancer │
              └───────┬───────┘
                      │
            ┌─────────┴─────────┐
            ▼                   ▼
       MCP Instance 1      MCP Instance 2
            │                   │
            └─────────┬─────────┘
                      ▼
                Tenant DB
```

서버 세션에 Tenant 상태를 저장하지 않고
매 요청마다 JWT를 기준으로 Tenant를 확정하기 때문에
특정 인스턴스에 세션을 고정하는 Sticky Session이 필요하지 않다.

따라서 향후 트래픽 증가 시 수평 확장을 고려할 수 있는 구조로 설계했다.

---

# 15. 운영 및 배포

개발 환경에서만 동작하는 MCP 서버가 아니라
실제 사내 AI 서비스에서 공통으로 사용할 수 있도록
**Docker 기반으로 AWS EC2 환경에 배포하여 운영**하고 있다.

```text
AI Agents
    │
    │ MCP / Streamable HTTP
    ▼
┌──────────────────────┐
│      AWS EC2         │
│                      │
│   Docker Container   │
│   ┌──────────────┐   │
│   │ MCP Server   │   │
│   └──────────────┘   │
└──────────┬───────────┘
           │
     ┌─────┴─────┐
     ▼           ▼
  System DB   Tenant DBs
```

운영 환경에서는 공통 MCP 서버 하나를 중심으로
여러 AI Agent가 동일한 Tool Layer를 사용할 수 있도록 구성했다.

---

# 16. Observability / 감사 로그

AI Agent가 호출하는 시스템에서는
단순한 서버 로그보다 **누가 어떤 Tool을 호출했는지 추적할 수 있는 구조**가 중요하다.

요청 단위로 다음 정보를 기록한다.

```text
tenant
method
tool
request_id
```

특히 내부 정보가 포함될 수 있는 Source 조회 결과는
사용자 응답에서는 제거하고 서버 로그에서만 추적할 수 있도록 했다.

이를 통해 장애나 보안 이슈 발생 시
**어떤 Tenant가 어떤 Tool을 호출했는지 재구성할 수 있도록** 설계했다.

---

# 17. 검증

구현 후 정상 동작뿐 아니라
**테넌트 경계를 실제로 우회할 수 없는지**를 중심으로 검증했다.

### 인증

| 시나리오 | 결과 |
|---|---|
| 정상 Tenant JWT | 해당 Tenant DB 조회 |
| 다른 Tenant JWT | 해당 Tenant DB로 라우팅 |
| Token 없음 | 차단 |
| 위조 Token | 차단 |
| 만료 Token | 차단 |
| `exp` 없는 Token | 차단 |
| Tenant claim 누락 | 차단 |

### Prompt Injection / Tenant Injection

```text
Tenant A Token
+
domain="Tenant B"를 포함한 악의적인 Tool 호출
```

→ Tool Schema에 Tenant 인자가 존재하지 않기 때문에
요청의 Tenant는 **JWT에서 확정된 Tenant A를 유지**한다.

즉,

```text
"모델이 어떤 Tenant를 말했는가"
          ≠
"서버가 어떤 Tenant에 접근하는가"
```

가 되도록 설계했다.

---

# 18. 성능 최적화

실측 결과 Tool 자체의 DB/SSH 처리 시간보다
AI Model 왕복 및 응답 컨텍스트 크기가 더 큰 영향을 주는 것으로 확인했다.

대표적인 최적화:

| 영역 | 적용 |
|---|---|
| Tenant Registry | TTL Cache |
| DB | Connection Pool |
| DB Connection | `ping(reconnect=True)` |
| Transaction | Connection 반환 전 rollback |
| SSH | Connection 재사용 |
| HTTP | Stateless |
| 설정 조회 | 필요한 컬럼만 조회 |
| Tool 응답 | 불필요한 대량 응답 제한 |

특히 전체 설정 조회는 응답 크기가 커질수록
**LLM 비용·지연·컨텍스트 사용량**에 직접 영향을 주기 때문에
Tool Description과 호출 규칙을 통해 필요한 항목만 조회하도록 유도했다.

---

# 19. 아키텍처의 핵심 책임 분리

```text
┌───────────────────────────────────────────┐
│               AI Agent Layer              │
│                                           │
│  의도 파악 / Tool 선택 / 결과 조합 / 응답  │
└──────────────────────┬────────────────────┘
                       │ MCP
                       ▼
┌───────────────────────────────────────────┐
│              MCP Tool Layer               │
│                                           │
│  인증 / Tenant Routing / Tool Execution   │
│  Input Validation / Data Access / Logging │
└──────────────────────┬────────────────────┘
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
       Customer DBs          Source Server
```

이 구조를 통해 **AI Agent와 내부 시스템의 결합도를 낮추고**
사내 여러 AI 서비스에서 동일한 데이터 접근 계층을 재사용할 수 있도록 했다.

---

# 20. 프로젝트에서 담당한 범위

단독으로 다음 영역을 설계하고 구현했다.

- Multi-Tenant MCP Server 아키텍처 설계
- MCP Python SDK 기반 Tool Server 구현
- JWT 인증 및 Tenant Middleware
- Tenant Resolver / Registry / TTL Cache
- Tenant별 DB Connection Pool
- MCP Tool 설계 및 구현
- 고객사 설정 조회
- 주문 / CS 데이터 조회
- Source Code 기반 하드코딩 조회
- 민감정보 화이트리스트 및 2중 필터링
- 입력값 검증 및 공통 예외 처리
- Stateless HTTP 구성
- CLI 기반 인증/격리 검증
- AI Agent 연동 구조 설계
- Docker 기반 AWS EC2 배포 및 운영

---

# 21. 주요 설계 포인트 요약

### ① AI Agent가 Tenant를 선택하지 못하게 설계

Tenant를 Tool Parameter가 아닌
**인증된 Request Context의 값**으로 관리했다.

### ② 특정 챗봇이 아닌 공통 AI 인프라로 설계

초기 챗봇용 요구사항에서 출발했지만
**사내 모든 AI Agent가 재사용할 수 있는 Tool Layer**로 역할을 확장했다.

### ③ 범용 SQL 대신 목적별 Tool

모델의 자유도를 제한하는 대신
데이터 접근 범위를 명확하게 통제했다.

### ④ 안전한 기본 동작

인증/테넌트 정보가 없으면 실행되는 것이 아니라
**실패하도록 설계하는 Fail-Closed 원칙**을 적용했다.

### ⑤ 운영 환경까지 연결

단순 PoC가 아니라 Docker 기반 AWS EC2에 배포하여
실제 사내 AI 서비스가 사용할 수 있는 공통 인프라로 운영하고 있다.

---

# 22. 회고

이 프로젝트에서 가장 중요했던 부분은 MCP 자체를 구현하는 것보다
**LLM이 호출하는 Tool을 기존 API와 다르게 설계하는 것**이었다.

기존 API에서는 인자를 늘리면 유연해질 수 있지만,
LLM이 호출하는 API에서는 인자가 늘어날수록
모델이 잘못 선택할 수 있는 범위도 함께 늘어난다.

결국 다음 원칙으로 설계를 정리할 수 있었다.

> **모델이 볼 수 있는 것은 모델이 선택할 수 있다.**
>
> 따라서 보안과 데이터 격리에 중요한 값은
> 모델에게 보여주지 않는 것이 가장 강한 방어가 된다.

이를 바탕으로 MCP 서버를 특정 AI 서비스의 부속 기능이 아니라
**사내 AI Agent가 내부 데이터와 기능을 사용하는 공통 실행 계층**으로 분리했다.

---

## 공개 범위

본 프로젝트는 사내 시스템으로 실제 소스 코드와 운영 정보는 공개하지 않는다.

포트폴리오에서는 내부 식별 정보와 고객 데이터를 제외하고
아키텍처, 설계 원칙, 문제 해결 과정, 기술적 의사결정을 중심으로 정리했다.
