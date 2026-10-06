# Kiwoom AI Scanner MASTER SPEC V2.0

## 0. 프로젝트 목적

기존 V1.0, V1.1, V1.2, V1.3 요구사항을 하나의 Master Specification으로 통합한다.

이 프로젝트는 단순 종목검색기가 아니다.

최종 목표는 다음 전체 흐름을 하나의 시스템으로 연결하는 것이다.

```text
증권사 리포트
+
시장 데이터
+
수급 데이터
+
조건검색
↓
주도주 Scanner
↓
14:30 / 14:50 / 15:10 / 15:20 추적
↓
TOP5 선정
↓
매수 전 위험검사
↓
Position Size
↓
실제 체결
↓
자동 매매일지
↓
실시간 보유종목 위험관리
↓
청산
↓
실제 손익
↓
MFE / MAE / R-Multiple
↓
Daily / Weekly / Monthly Review
↓
전략 통계
↓
실패 원인
↓
전략 개선
```

가장 중요한 원칙은:

**기능을 많이 만드는 것보다 데이터가 틀리지 않는 것을 우선한다.**

---

## 1. 개발 절대 원칙

다음 순서를 반드시 지킨다.

```text
기능 구현
↓
데이터 검증
↓
페이지 검증
↓
계산식 검산
↓
Integration Test
↓
Regression Test
↓
오류 수정
↓
Stable 버전 생성
↓
다음 버전 진행
```

V1.0~V1.3 전체를 한 번에 무작정 구현한 후 마지막에 테스트하지 않는다.

---

## 2. 버전 개발 순서

반드시 순서대로 진행한다.

```text
PHASE 0
Foundation

PHASE 1
V1.0

PHASE 1-QA
V1.0 페이지별 검증

PHASE 2
V1.1

PHASE 2-QA
V1.1 페이지별 검증

PHASE 3
V1.2

PHASE 3-QA
V1.2 페이지별 검증

PHASE 4
V1.3

PHASE 4-QA
V1.3 페이지별 검증

PHASE 5
전체 Integration Test

PHASE 6
전체 Regression Test

PHASE 7
Release Candidate

PHASE 8
Stable Release
```

---

## 3. 가장 중요한 구조

Frontend가 Broker API를 직접 호출하거나 계산하지 않는다.

반드시 다음 구조를 사용한다.

```text
Kiwoom REST API
↓
Broker Adapter
↓
Normalized Data Model
↓
Database
↓
Domain Engine
↓
Backend API
↓
Frontend
```

---

## 4. Single Source of Truth

각 데이터의 계산 위치는 하나만 존재해야 한다.

### 시장 데이터

`Market Data Service`

담당:
- 현재가
- 시가
- 고가
- 저가
- 거래량
- 거래대금
- 일봉

### 수급

`Investor Service`

담당:
- 외국인 순매수
- 기관 순매수
- 누적 수급
- 수급 Momentum

### 리포트

`Report Service`

담당:
- 증권사 리포트
- 목표주가
- 실적 전망
- 리포트 변화

### Scanner

`Scanner Engine`

담당:
- AI Score
- Closing Bet Score
- Risk Score
- Ranking

### 실제 매매

`Trade Engine`

담당:
- 평균체결가
- 실현손익
- 평가손익
- Trade Session

### 위험관리

`Risk Engine`

담당:
- Position Size
- R-Multiple
- Portfolio Heat
- Daily Loss
- Stop
- Target

Frontend는 이 값을 다시 계산하지 않는다.

---

## 5. 프로젝트 구조

```text
kiwoom-ai-scanner/

backend/
    app/

        api/
            routes/

        brokers/
            kiwoom/

        market/
        investors/
        reports/
        scanner/
        trades/
        risk/
        reviews/
        strategy/

        database/
        services/
        core/

    tests/

frontend/
    src/

        pages/
        components/
        hooks/
        services/
        types/
        utils/

database/

    migrations/
    backups/

docs/

logs/

scripts/

MASTER_SPEC.md

README.md

.env.example

.gitignore
```

---

## 6. PHASE 0 — Foundation

기능 개발 전에 기반을 먼저 만든다.

### 6-1. Environment

```env
KIWOOM_MODE=mock

KIWOOM_APP_KEY=
KIWOOM_APP_SECRET=

ENABLE_AUTO_ORDER=false
ENABLE_AUTO_STOP_ORDER=false
ENABLE_AUTO_EXIT=false

DEMO_MODE=true
```

---

## 7. 실전 / 모의 / 데모 분리

세 가지 Mode:

```text
DEMO
MOCK
REAL
```

화면 상단에 항상 표시한다.

예:

```text
🔵 DEMO
🟢 MOCK
🔴 REAL
```

혼동되면 안 된다.

---

## 8. 키움 API 원칙

절대로 다음을 추측하지 않는다.

```text
Endpoint
API ID
TR ID
Header
Request Field
Response Field
Real-time Field
```

Codex는 실제 구현 전에 공식 Spec을 검사해야 한다.

---

## 9. API Adapter

```python
class BrokerProvider:

    authenticate()

    get_stock_info()

    get_quote()

    get_daily_prices()

    get_investor_flow()

    get_condition_list()

    run_condition()

    subscribe_realtime()

    get_account_balance()

    get_positions()

    get_orders()

    get_fills()

    get_transactions()

    get_realized_profit()
```

Scanner와 UI는 Broker 구현을 직접 알지 못하게 한다.

---

## 10. Data Normalization

키움 원본 필드를 그대로 전체 시스템에 퍼뜨리지 않는다.

```text
Kiwoom Raw
↓
Normalized

stock_code
stock_name

price
open_price
high_price
low_price
close_price

volume
turnover

foreign_net_amount
institution_net_amount
```

이 구조를 기준으로 나머지 시스템을 작성한다.

---

## 11. PHASE 1 — V1.0

목표:

**좋은 후보를 정확히 찾는 검색기**

구현:

- 시장데이터
- 조건검색
- 리포트 분석
- 거래량
- 거래대금
- 외국인
- 기관
- 이동평균
- AI Score
- Closing Bet Score
- Risk Score
- TOP10
- 종목 상세
- Watchlist

---

## 12. V1.0 Scanner 기본 필터

```text
현재가 >= 2,000

시가총액 >= 1,000억원

등락률
+1% ~ +12%

거래대금
>= 500억원

현재가 > MA20

MA5 >= MA20

거래량 >= 20일 평균 × 1.5
```

설정 변경 가능.

---

## 13. AI Score

0~100점.

대표 구성:

```text
