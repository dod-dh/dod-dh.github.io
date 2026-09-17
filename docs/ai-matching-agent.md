AI 기반 상품 매칭 Agent --- 아키텍처 및 기술 상세

1. 프로젝트 개요

주문 파일에 입력된 비정형 상품명과 사내 상품 마스터 데이터를 자동으로
매칭하는 AI Agent 시스템입니다.

상품명은 판매처와 주문 데이터에 따라 표현 방식이 달라 단순 문자열
검색만으로 정확한 상품을 찾기 어려울 수 있습니다. 이를 해결하기 위해 DB
기반 후보 검색과 문자열 유사도 필터링을 선행하고, 최종 판단은 LLM 기반
Agent가 수행하도록 단계적인 매칭 Pipeline을 설계했습니다.

또한 실제 업무 환경에서 사용할 수 있도록 대량 상품 병렬 처리, Agent 실행
이력, LLM/MCP 호출량 모니터링, 멀티테넌트 관리 및 AWS 기반 배포 환경까지
함께 구축했습니다.

2. 전체 아키텍처

                         사용자 / 업무 시스템
                                │
                                ▼
                         ┌──────────────┐
                         │     ALB      │
                         └──────┬───────┘
                                │
                                ▼
                    ┌────────────────────────┐
                    │      ECS Fargate       │
                    │                        │
                    │       FastAPI          │
                    │          │             │
                    │   AI Matching Agent    │
                    └──────────┬─────────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
              ▼                ▼                ▼
         ┌─────────┐     ┌────────────┐   ┌────────────┐
         │  MySQL  │     │ MCP Server │   │  Bedrock   │
         │ Product │     │ Tool Layer │   │   Claude   │
         │ Master  │     └─────┬──────┘   └────────────┘
         └─────────┘           │
                               ▼
                         사내 업무 시스템

        ┌─────────────────────────────────────────┐
        │              운영 / 관측                │
        │                                         │
        │ Phoenix │ RDS PostgreSQL │ Admin       │
        │ Trace   │ 실행이력/통계   │ Dashboard   │
        └─────────────────────────────────────────┘

Deployment:
GitHub Actions → ECR → ECS Fargate → ALB

3. AI Matching Pipeline

주문 상품
   │
   ▼
상품명 정규화
   │
   ▼
DB 후보 검색
   │
   ▼
RapidFuzz 유사도 필터링
   │
   ▼
후보 상품 구성
   │
   ▼
LLM Agent 판단
   │
   ├── 매칭 가능 → 결과 반환
   │
   └── 후보 부족 / 불확실
              │
              ▼
        검색 조건 보완
              │
              ▼
          추가 후보 검색
              │
              └── 재판단

모든 상품을 LLM에게 직접 전달하지 않고, 먼저 DB 검색과 문자열 유사도
필터링을 수행하여 LLM이 판단해야 하는 후보 범위를 줄였습니다.

이 구조를 통해 검색 단계와 AI 판단 단계를 분리하고, 대량의 상품 마스터
데이터에서도 효율적으로 매칭할 수 있도록 설계했습니다.

4. Candidate Search

상품 매칭 요청이 들어오면 먼저 사내 상품 Master DB에서 후보 상품을
검색합니다.

후보 검색 단계에서는 상품명에 포함된 검색 조건을 활용하여 검색 범위를
줄이고, 이후 RapidFuzz 기반 문자열 유사도를 계산하여 LLM에게 전달할
후보를 추가로 필터링합니다.

Order Product
      │
      ▼
   DB Search
      │
      ▼
Candidate Products
      │
      ▼
RapidFuzz Similarity
      │
      ▼
 Top Candidates

LLM이 전체 상품 Master를 직접 검색하지 않도록 검색 가능한 후보군을 먼저
구성하여 불필요한 LLM 입력과 호출을 줄였습니다.

5. LLM Agent

후보 상품이 구성되면 LLM Agent가 주문 상품과 후보 상품의 정보를 비교하여
최종 매칭 여부를 판단합니다.

단일 LLM 호출에 의존하지 않고, 검색 결과가 충분하지 않거나 판단에 필요한
정보가 부족한 경우 추가 검색을 수행할 수 있도록 Agent Loop를
구성했습니다.

Candidate Search
      │
      ▼
   LLM 판단
      │
      ├── Match / Review
      │
      └── 후보 부족
              │
              ▼
       검색 조건 보완
              │
              ▼
          추가 검색
              │
              ▼
          재판단

반복 검색에는 종료 조건을 적용하여 Agent가 불필요하게 계속 검색하거나
LLM을 반복 호출하지 않도록 했습니다.

또한 기존 후보 검색 결과를 활용하고 필요한 경우에만 추가 검색을
수행하도록 Loop를 최적화하여 호출 횟수와 처리 지연을 줄였습니다.

6. 대량 상품 병렬 처리

주문 파일에는 여러 상품이 포함될 수 있기 때문에 상품을 순차적으로
처리하면 전체 처리 시간이 증가할 수 있습니다.

이를 해결하기 위해 상품 단위로 작업을 분리하고 Background 기반
비동기·병렬 처리 구조를 적용했습니다.

Job
 │
 ├── Item 1 ──► Matching Agent
 ├── Item 2 ──► Matching Agent
 ├── Item 3 ──► Matching Agent
 ├── Item 4 ──► Matching Agent
 └── Item N ──► Matching Agent

각 Item의 매칭 작업을 독립적으로 처리할 수 있도록 구성하여 대량 상품
요청에서도 전체 처리 시간을 줄일 수 있도록 설계했습니다.

7. MCP 연계

AI Agent가 사내 데이터와 업무 기능을 직접 구현 방식에 의존하지 않고
사용할 수 있도록 MCP Server를 Tool Layer로 활용했습니다.

AI Matching Agent
       │
       ▼
   MCP Server
       │
       ├── DB Tool
       ├── Search Tool
       └── 업무 기능 Tool
              │
              ▼
       사내 데이터 / 시스템

MCP Server를 통해 Agent와 데이터 계층을 분리하고, 공통 업무 기능을 Tool
형태로 추상화했습니다.

이를 통해 향후 다른 Agent에서도 동일한 업무 Tool을 재사용할 수 있도록
구성했습니다.

8. 멀티테넌트 구조

여러 테넌트가 동일한 AI 시스템을 사용할 수 있도록 테넌트 정보를 기반으로
요청과 데이터 접근 범위를 분리했습니다.

Request
   │
   ▼
Tenant Authentication
   │
   ▼
Tenant Routing
   │
   ├── Tenant A → Data / Usage
   ├── Tenant B → Data / Usage
   └── Tenant C → Data / Usage

Agent 및 Tool 호출 과정에서 테넌트 컨텍스트를 유지하도록 구성하여
테넌트별 데이터와 사용량을 분리해서 관리할 수 있도록 했습니다.

9. Observability

AI Agent는 하나의 요청 안에서 여러 LLM 호출과 Tool Call이 발생할 수 있기
때문에 실행 과정을 추적할 수 있는 구조가 필요합니다.

Arize Phoenix를 활용하여 다음 실행 정보를 추적할 수 있도록 구성했습니다.

LLM 호출

Agent 실행 과정

MCP Tool Call

각 단계별 처리 시간

Agent 실행 이력

호출 결과

Job
 │
 └── Item
      │
      ├── Agent
      │    ├── LLM Call
      │    ├── MCP Tool
      │    └── LLM Call
      │
      └── Matching Result

최종 결과뿐 아니라 Agent가 어떤 과정을 거쳐 결과를 생성했는지 확인하고
병목 구간을 분석할 수 있도록 구성했습니다.

10. 관리자 시스템

운영 환경에서 AI 사용량과 처리 상태를 확인할 수 있도록 관리자
Dashboard를 구축했습니다.

주요 관리 항목

테넌트별 Job / Item 처리량

LLM 호출량

MCP Tool 호출량

매칭 결과 통계

평균 처리 시간

Job 및 Item 실행 이력

Agent 실행 상태 및 오류 확인

관리자 화면에서는 테넌트 단위로 사용량을 조회할 수 있으며, Job 상세
화면에서는 개별 Item의 처리 결과와 실행 정보를 확인할 수 있도록
구성했습니다.

11. AWS 인프라

서비스는 컨테이너 기반으로 구성하고 AWS 환경에서 배포·운영했습니다.

GitHub
   │
   ▼
GitHub Actions
   │
   │ Docker Build
   ▼
AWS ECR
   │
   ▼
ECS Fargate
   │
   ▼
ALB
   │
   ▼
FastAPI / AI Matching Agent

              ┌─────────────────┐
              │ RDS PostgreSQL  │
              │ 실행 이력 / 통계 │
              └─────────────────┘

구성 요소

구성 요소        역할

GitHub Actions   CI/CD 및 이미지 빌드·배포 자동화
ECR              Docker 이미지 저장소
ECS Fargate      컨테이너 실행 환경
ALB              외부 요청 분산 및 서비스 진입점
RDS PostgreSQL   실행 이력 및 통계 데이터 저장
AWS Bedrock      Claude 기반 LLM 호출
Arize Phoenix    Agent / LLM / Tool 실행 추적

12. 주요 설계 포인트

검색과 판단의 분리

DB 검색과 문자열 유사도 필터링을 먼저 수행하고 LLM은 최종 판단과 필요한
추가 검색에 집중하도록 구성했습니다.

LLM 호출 최소화

후보군을 사전에 축소하고 Agent Loop에 종료 조건을 적용하여 불필요한 반복
호출과 검색을 줄였습니다.

대량 처리

상품 단위 Background·병렬 처리 구조를 적용하여 여러 Item을 독립적으로
처리할 수 있도록 했습니다.

업무 기능의 Tool화

MCP를 통해 사내 기능을 공통 Tool로 추상화하여 Agent와 데이터 계층의
결합도를 낮췄습니다.

운영 가능성

Agent / LLM / MCP 실행 과정을 추적하고 테넌트별 사용량 및 실행 이력을
관리자 Dashboard에서 확인할 수 있도록 구성했습니다.

배포 자동화

GitHub Actions를 기반으로 Docker 이미지 빌드부터 ECR 등록, ECS Fargate
배포까지 CI/CD Pipeline을 구성했습니다.

13. 프로젝트에서 담당한 범위

본 프로젝트에서 AI Matching Agent의 설계 및 개발뿐 아니라 API 서버,
Agent Pipeline, MCP 연계, 대량 처리 구조, Observability, 관리자 시스템
및 AWS 배포 인프라까지 전반적인 시스템을 1인 개발했습니다.
