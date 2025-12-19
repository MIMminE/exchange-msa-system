# 🚀 exchange-msa-system

## 1. 📝 프로젝트 개요 (Overview)
*   **목표:** 대용량 트래픽과 동시성 이슈가 빈번한 가상화폐 거래소를 **MSA(Microservices Architecture)**로 구현하여, **안정성(Reliability)**과 **확장성(Scalability)**을 갖춘 시스템을 구축함.
*   **핵심 가치:**
    *   **High Performance:** Go 언어 기반 매칭 엔진과 Redis 캐싱을 통한 초고속 체결 및 조회.
    *   **Data Consistency:** 분산 환경(Saga, Outbox Pattern)에서의 데이터 무결성 보장.
    *   **Observability:** ELK Stack과 Zipkin을 활용한 모니터링 및 트러블 슈팅 환경 구축.

---

## 2. 🏗️ 시스템 아키텍처 (System Architecture)

```mermaid
graph TD
    User[Client (Web/App)] -->|HTTP/REST| Gateway[API Gateway (Spring Cloud)]
    
    subgraph "Frontend Facing Services"
        Gateway -->|Auth| AuthSvc[Identity Service]
        Gateway -->|Order| OrderSvc[Order Service]
        Gateway -->|Asset| WalletSvc[Wallet Service]
        Gateway -->|View| MarketSvc[Market Service]
    end

    subgraph "Core Engine & Async Processing"
        OrderSvc -->|1. Order Created| Kafka{Kafka Cluster}
        Kafka -->|2. Consume| MatchEng[⚡ Matching Engine (Go)]
        MatchEng -->|3. Trade Executed| Kafka
    end

    subgraph "Data Consistency & View"
        Kafka -->|4. Update Balance| WalletSvc
        Kafka -->|4. Update Chart/Ticker| MarketSvc
    end

    subgraph "Infrastructure"
        Redis[(Redis Cluster)]
        MySQL[(MySQL DBs)]
        ELK[ELK Stack & Zipkin]
    end

    AuthSvc -.-> Redis
    MarketSvc -.-> Redis
    WalletSvc -.-> MySQL
    OrderSvc -.-> MySQL
```

---

## 3. 🛠️ 기술 스택 (Tech Stack)

| 분류 | 기술 (Technology) | 선정 이유 및 용도 |
| :--- | :--- | :--- |
| **Backend (Main)** | **Java 17+, Spring Boot 3.x** | 안정적인 엔터프라이즈급 서버 구축 |
| **Backend (Core)** | **Go (Golang)** | 초고속 거래 체결 엔진 (Matching Engine) 구현 |
| **Architecture** | **Spring Cloud (Gateway, Eureka)** | MSA 환경의 라우팅 및 서비스 디스커버리 |
| **Database** | **MySQL 8.0** | 트랜잭션 보장이 필요한 주문/자산 데이터 저장 |
| **NoSQL / Cache** | **Redis** | 실시간 시세 조회(ZSet), 분산 락, 세션 관리 |
| **Messaging** | **Apache Kafka** | 서비스 간 비동기 통신 및 대용량 트래픽 버퍼링 |
| **Testing** | **JUnit5, Mockito, k6** | 단위/통합 테스트 및 부하 테스트(Load Testing) |
| **DevOps / Tool** | **Docker, ELK, Zipkin** | 컨테이너 배포 및 로그/트레이싱 모니터링 |

---

## 4. 🧩 서비스별 상세 역할 (Service Breakdown)

### ① Identity Service (인증/인가)
*   **역할:** 회원가입, 로그인, JWT 발급 및 검증.
*   **기술:** Spring Security, JWT, Redis(Blacklist).
*   **특징:** Stateless 인증 구조.

### ② Wallet Service (자산 관리)
*   **역할:** 입출금, 자산 조회, 자산 동결(주문 시).
*   **기술:** MySQL, Redis Distributed Lock (Redisson).
*   **핵심:** **동시성 제어.** 따닥(Double Spending) 방지를 위한 락 적용.

### ③ Order Service (주문 관리)
*   **역할:** 매수/매도 주문 접수, 주문 취소, 주문 내역 조회.
*   **기술:** MySQL, Kafka Producer.
*   **핵심:** **Transactional Outbox Pattern.** DB 저장과 Kafka 메시지 발행의 원자성 보장.

### ④ Matching Engine (체결 엔진) - ⭐ 핵심(Go)
*   **역할:** 매수/매도 주문을 가격/시간 우선순위로 매칭.
*   **기술:** **Go (Goroutine)**, In-memory Orderbook.
*   **특징:** GC Pause를 최소화하여 극한의 처리량(TPS) 확보.

### ⑤ Market Service (시세 조회)
*   **역할:** 실시간 현재가, 호가창, 차트(Candle) 데이터 제공.
*   **기술:** **Redis (ZSet, Hash)**, WebSocket (실시간 푸시).
*   **핵심:** **CQRS 패턴.** 체결 이벤트를 소비(Consume)하여 읽기 전용 뷰(View) 생성.

---

## 5. 💡 주요 기술적 해결 과제 (Problem Solving)

이 프로젝트에서 **"어떤 문제를 어떻게 해결했는지"**가 포트폴리오의 핵심입니다.

1.  **동시성 이슈 해결 (Concurrency)**
    *   *문제:* 사용자가 동시에 여러 번 출금 요청 시 잔고가 마이너스가 되는 현상.
    *   *해결:* **Redisson 분산 락(Distributed Lock)**을 도입하여 자원 접근을 순차적으로 제어.
2.  **데이터 정합성 보장 (Consistency)**
    *   *문제:* 주문은 성공했는데, Kafka 메시지 전송 실패로 체결이 안 되는 현상.
    *   *해결:* **Transactional Outbox Pattern**을 적용하여 이벤트 발행을 보장.
3.  **대용량 트래픽 처리 (High Traffic)**
    *   *문제:* 시세 조회 요청 폭주로 DB 부하 발생.
    *   *해결:* **Redis Caching (Look-aside & Write-through)** 및 **Local Cache(Caffeine)** 계층형 캐시 적용.
4.  **분산 트랜잭션 관리 (Distributed Tx)**
    *   *문제:* 출금 중 외부 은행 API 오류 시 내부 잔고 롤백 필요.
    *   *해결:* **Saga Pattern (Orchestration)**을 통해 보상 트랜잭션 구현.

---

## 6. 📅 개발 로드맵 (Roadmap)

*   **Phase 1 (MVP):** Java로 모놀리식 혹은 간단한 MSA 구성. 기본 주문/체결 로직 구현.
*   **Phase 2 (MSA & Async):** Kafka 도입, 서비스 분리, Redis 캐싱 적용.
*   **Phase 3 (Optimization):** **Go 매칭 엔진** 교체, 부하 테스트(k6), 동시성 테스트 코드 작성.
*   **Phase 4 (Ops):** ELK, Zipkin 모니터링 환경 구축 및 Docker Compose 배포.

---
