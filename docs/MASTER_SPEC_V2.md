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
리포트       25
시장주도력   25
수급         20
기술적강도   20
모멘텀       10
```

위험요인은 별도 감점.

---

## 14. 점수 근거 저장

Score 값만 저장하면 안 된다.

반드시 `왜 87점인가?` 를 추적 가능하게 한다.

예:

```text
거래대금 +8
외국인 +5
기관 +5
CloseStrength +5
목표가 상향 +5
```

---

## 15. V1.0 페이지

완료 후 반드시 페이지별 점검.

- 대시보드
- 오늘의 주도주
- 종가배팅 후보
- 증권사 리포트
- 조건검색
- 관심종목
- 종목 상세
- 설정
- API 상태

---

## 16. V1.0 QA

각 페이지에서 다음 4단계를 비교한다.

```text
1. API 원본
2. DB 저장값
3. Backend 계산값
4. Frontend 표시값
```

네 값의 의미가 일치해야 한다.

---

## 17. V1.0 계산 검산

별도 Fixture 생성 후 아래 항목을 검산한다.

- MA5
- MA20
- 거래량 배수
- 거래대금
- CloseStrength
- Upper Tail
- AI Score
- Closing Score

---

## 18. V1.0 Stable 조건

다음 테스트가 모두 PASS되어야 한다.

- Backend Unit Test
- Frontend Build
- Demo Mode
- Mock API
- Scanner
- Report Import
- Score
- Ranking
- Filters
- Page QA

그 후 Git Tag:

`v1.0-stable`

---

## 19. PHASE 2 — V1.1

목표:

**후보의 실제 성과를 자동 검증**

추가:

- 14:30 Scanner
- 14:50 재평가
- 15:10 TOP5
- 15:20 Risk Check
- Score Momentum
- Turnover Momentum
- 수급 Momentum
- VWAP
- Market Score
- 다음날 추적
- 09:10
- 09:30
- 고가
- 저가
- 종가
- 3일
- 5일
- MFE
- MAE

---

## 20. Snapshot 구조

시간별 상태는 덮어쓰지 않는다.

반드시 저장:

```text
14:30
14:50
15:10
15:20
```

---

## 21. 후보 Lock

15:10 TOP5는 이후 결과가 나빠져도 삭제하지 않는다.

`rank_at_1510` 유지.

---

## 22. 다음날 성과

저장:

```text
next_open_return
next_0910_return
next_0930_return
next_high_return
next_low_return
next_close_return
return_3d
return_5d
```

---

## 23. False Positive

예:

```text
AI >= 85
AND
09:30 return < 0
```

자동 분석.

---

## 24. False Negative

예:

```text
AI < 70
AND
Next High >= +5%
```

분석.

---

## 25. V1.1 QA

특히 다음 오류를 검사한다.

- 날짜 잘못 연결
- 휴장일 오류
- 15:10 값이 14:30 값 덮어씀
- 다음날이 잘못 연결됨
- 3일/5일 거래일 계산 오류
- MFE/MAE 계산 오류

---

## 26. V1.1 페이지 QA

- 종가배팅 TOP5
- 오늘의 복기
- 전략 연구소
- 성과 Dashboard
- Score 구간 분석
- Factor 분석

---

## 27. V1.1 Stable 조건

```text
V1.0 Tests
+
V1.1 Tests
```

모두 PASS 후 Git Tag:

`v1.1-stable`

---

## 28. PHASE 3 — V1.2

목표:

**실제 키움 매매와 Scanner 연결**

추가:

- 주문
- 체결
- 계좌
- 잔고
- 실현손익
- Trade Session
- 자동 매매일지
- Scanner ↔ Trade 매칭
- Entry Evaluation
- Exit Evaluation
- Execution Score
- Trading Score
- 손절 Simulation
- 익절 Simulation

---

## 29. 주문 / 체결 구분

반드시 `Order` 와 `Fill` 을 분리한다.

부분체결 대응.

---

## 30. 평균 체결가

```python
average_fill_price = (
    sum(fill_price * qty)
    /
    sum(qty)
)
```

---

## 31. Trade Session

동일 종목 포지션의 생성부터 종료까지 묶는다.

지원:

- 단일매수
- 분할매수
- 부분체결
- 분할매도
- 당일매매
- Overnight
- Swing

---

## 32. 실제 손익

Backend에서 계산.

- 매수금액
- 매도금액
- 수수료
- 세금
- 총손익
- 순손익
- 수익률

Frontend에서 다시 계산 금지.

---

## 33. Scanner ↔ 실제매매 비교

구분:

```text
Scanner 후보 거래
Scanner 외 거래
```

각 성과 별도 분석.

---

## 34. Selection vs Execution

반드시 분리:

```text
Selection Score
Execution Score
```

좋은 종목을 골랐지만 매매를 못한 것과 종목 자체가 나빴던 것을 구분한다.

---

## 35. Entry Evaluation

비교:

- Scanner 기준가격
- 15:10 가격
- VWAP
- 실제 평균매수가
- 고가

---

## 36. Exit Evaluation

비교:

- 실제 매도가
- 계획 목표가
- 계획 손절가
- 매도 후 고가
- 매도 후 저가

---

## 37. 실제 체결 기준 MFE / MAE

Scanner 기준뿐 아니라 실제 매수가 기준도 별도 계산.

---

## 38. 손절/익절 Simulation

지원:

- Fixed %
- R Based
- ATR
- VWAP
- Trailing Stop

---

## 39. Look-Ahead Bias 금지

14:30 전략에서 15:30 정보를 사용하면 실패다.

백테스트 시점별로 그 당시 알 수 있는 데이터만 사용한다.

---

## 40. V1.2 QA 우선순위

이 단계는 가장 엄격하게 검증한다.

특히:

- 평균체결가
- 분할매수
- 분할매도
- 실현손익
- 수수료
- 세금
- 중복체결
- Trade Session
- 보유수량

---

## 41. V1.2 페이지 QA

- 실전 매매일지
- Trade Detail
- 오늘의 복기
- 월간 성과
- 계좌현황
- 청산 전략 연구소

---

## 42. V1.2 Stable

```text
V1.0 Tests
+
V1.1 Tests
+
V1.2 Tests
```

모두 실행 후 Git Tag:

`v1.2-stable`

---

## 43. PHASE 4 — V1.3

목표:

**실전 위험관리**

추가:

- 매수 전 체크
- Position Sizing
- R-Multiple
- Portfolio Heat
- Daily Loss Limit
- Stop Monitor
- Target Monitor
- Trailing Stop
- Market Risk
- Risk Alerts
- Overtrading
- Consecutive Loss
- Risk Dashboard

---

## 44. Position Sizing

```python
risk_amount = account_equity * risk_pct
```

```python
risk_per_share = entry_price - stop_price
```

```python
qty = floor(risk_amount / risk_per_share)
```

---

## 45. 최종수량

```python
final_qty = min(
    risk_based_qty,
    capital_limit_qty,
    cash_limit_qty
)
```

---

## 46. R-Multiple

```python
r_multiple = net_profit / initial_risk_amount
```

---

## 47. Portfolio Heat

```python
portfolio_heat = (
    sum(open_position_risk)
    /
    account_equity
    * 100
)
```

---

## 48. 일일 손실한도

지원:

```text
Percentage
R
```

예:

```text
-1.5%
또는
-3R
```

---

## 49. Pretrade Gate

검사:

- 시장상태
- AI Score
- Closing Score
- Risk Score
- 손익비
- Position Size
- Portfolio Heat
- Daily Loss
- 연속손실
- 오늘 거래수

---

## 50. Pretrade 결과

```text
PASS
CAUTION
BLOCKED_BY_RISK_RULE
```

자동 주문 실행과는 별개.

---

## 51. Risk Alert

지원:

```text
STOP_NEAR
STOP_TOUCH
TARGET_NEAR
TARGET_TOUCH
VWAP_BREAK
FOREIGN_SELL_REVERSAL
INSTITUTION_SELL_REVERSAL
TURNOVER_SLOWDOWN
MARKET_RISK_OFF
PORTFOLIO_HEAT_HIGH
DAILY_LOSS_LIMIT
CONSECUTIVE_LOSS
OVERTRADING
```

---

## 52. 실시간 구독 우선순위

```text
P1 보유종목
P2 미체결
P3 TOP5
P4 관심종목
P5 Scanner 후보
```

---

## 53. 오래된 실시간 데이터 방지

연결이 끊기면 이전 가격을 현재가처럼 표시하지 않는다.

반드시:

```text
⚠ 업데이트 지연
마지막 업데이트 HH:MM:SS
```

표시.

---

## 54. V1.3 QA

특히 다음 항목을 수동 검산한다.

- Position Size
- R
- Portfolio Heat
- Daily Loss
- Stop Distance
- Target Distance
- Current R
- Trailing Stop

---

## 55. V1.3 페이지 QA

- 매수 전 점검
- 실시간 위험관리
- 보유종목
- Risk Dashboard
- Alert Center
- Risk Rule 연구소

---

## 56. V1.3 Stable

다음 전부 테스트.

```text
V1.0
V1.1
V1.2
V1.3
```

모두 PASS 후 Git Tag:

`v1.3-stable`

---

## 57. 페이지별 QA 표준

모든 페이지마다 동일한 QA 프로세스를 사용한다.

1. 페이지 Route가 정상 접근되는가?
2. Backend Endpoint가 정상인가?
3. DB 실제값과 일치하는가?
4. 계산식이 Backend 단일 위치에서 계산되는가?
5. Frontend가 같은 값을 다시 계산하고 있지 않은가?
6. 정렬 정상?
7. 검색 정상?
8. 필터 정상?
9. 새로고침 정상?
10. Loading 상태 정상?
11. Empty 상태 정상?
12. API Error 상태 정상?
13. None/NaN 없음?
14. Mobile/PC 표시 정상?

---

## 58. 페이지 QA Report

각 페이지 점검 후 반드시 생성:

```text
Page:
종가배팅 TOP5

API:
PASS

DB:
PASS

Calculation:
PASS

UI:
PASS

Filter:
PASS

Sorting:
PASS

Null Handling:
PASS

Responsive:
PASS

Known Issues:
0
```

---

## 59. 데이터 정확도 QA

화면 숫자는 반드시 역추적 가능해야 한다.

예:

```text
화면
AI Score 92
```

클릭하면:

```text
Report 20
Market 24
Investor 18
Technical 20
Momentum 10
Risk 0
Total 92
```

---

## 60. 금액 단위 QA

특히 주의:

- 원
- 천원
- 백만원
- 억원

변환 오류 검사.

예:

```text
50000000000
```

→

```text
500억
```

정확히 표시.

---

## 61. 퍼센트 QA

`0.05`가 데이터상 `5%`인지 `0.05%`인지 반드시 데이터 계약 정의.

---

## 62. 날짜 QA

모든 날짜는:

```text
trade_date
```

와:

```text
calendar_date
```

를 구분한다.

휴장일 고려.

---

## 63. Null 정책

DB Null을 억지로 0으로 변환하지 않는다.

```text
0 = 실제 0
NULL = 데이터 없음
```

UI:

```text
-
미수집
계산불가
```

중 적절히 표시.

---

## 64. 자동 테스트 계층

반드시 다음 계층으로 운영.

```text
Unit Test
Service Test
Repository Test
API Test
Integration Test
Frontend Test
Regression Test
```

---

## 65. Golden Test Data

고정 검산 데이터를 만든다.

```text
fixtures/golden/
```

여기에:

```text
market.json
investor.json
reports.json
fills.json
trades.json
risk_cases.json
```

저장.

---

## 66. Golden Test 목적

코드를 리팩터링해도 동일 입력에 동일 결과가 나와야 한다.

예:

```text
AI Score 92
Average Fill 10,250
Net Profit 84,000
R Multiple +1.42R
```

같은 기대값을 고정.

---

## 67. Regression Test

새 버전을 추가할 때 이전 기능을 반드시 다시 검사한다.

Codex는 신규 기능 Test만 통과했다고 완료 선언하면 안 된다.

---

## 68. Regression Matrix

| 기능 | V1.0 | V1.1 | V1.2 | V1.3 |
|---|---|---|---|---|
| Scanner | PASS | PASS | PASS | PASS |
| Reports | PASS | PASS | PASS | PASS |
| TOP5 | - | PASS | PASS | PASS |
| Trade Journal | - | - | PASS | PASS |
| Risk | - | - | - | PASS |

자동으로 생성.

---

## 69. Database Migration 원칙

DB 삭제 후 재생성 금지.

각 버전:

```text
backup
migration
validation
```

순서.

---

## 70. Migration 실패

즉시 `rollback` 후 원인을 수정한다.

---

## 71. Git 전략

하나의 Repository 사용.

기본:

```text
main
```

기능 개발:

```text
feature/v1.0
feature/v1.1
feature/v1.2
feature/v1.3
```

---

## 72. Stable Tags

각 버전 통과 후:

```text
v1.0-stable
v1.1-stable
v1.2-stable
v1.3-stable
```

---

## 73. Commit 규칙

기능 단위 Commit.

예:

```text
feat(scanner): add AI score engine
test(scanner): add golden score fixtures
fix(trades): correct partial-fill average price
feat(risk): add portfolio heat calculation
```

---

## 74. 대형 Commit 금지

프로젝트 전체를 한 번에 수정하는 거대한 Commit은 피한다.

---

## 75. Codex 작업단위

Master Spec은 전체 설계다.

하지만 실제 작업은 작게 나눈다.

한 작업은 가급적:

```text
3~7개 파일
1개의 명확한 목표
1개의 테스트 범위
```

로 제한.

---

## 76. Codex Task Template

각 작업에서 다음 형식 사용.

```text
CURRENT PHASE:

현재 버전:

현재 목표:

관련 파일:

변경 가능:

변경 금지:

필수 테스트:

완료 기준:
```

---

## 77. 예시

```text
CURRENT PHASE:
V1.2 STEP 7

목표:
키움 체결데이터 Import

변경 가능:
broker_fills
kiwoom account provider
fill repository

변경 금지:
scanner scoring
report engine
risk engine

필수 테스트:
partial fill
duplicate fill
average fill price

완료 기준:
모든 테스트 PASS
```

---

## 78. 오류 수정 규칙

오류 발생 시 바로 임시 Patch만 하지 않는다.

순서:

```text
원인 분석
↓
최소 수정
↓
관련 테스트
↓
Regression Test
```

---

## 79. API 오류

다음 상태를 구분한다.

```text
Authentication Error
Rate Limit
Timeout
Server Error
Invalid Request
Missing Data
Stale Data
```

---

## 80. 사용자 화면 오류

화면이 죽지 않도록 Error Boundary 적용.

---

## 81. Loading 상태

데이터가 오기 전 `0`을 보여주지 않는다.

Skeleton 또는:

```text
불러오는 중
```

표시.

---

## 82. Demo Data

Demo 데이터와 실제 데이터 혼합 금지.

DEMO Mode에서는:

```text
DEMO DATA
```

고정 Banner 표시.

---

## 83. Mock Mode

Mock도 실전 데이터처럼 표현하지 않는다.

---

## 84. 자동주문

MASTER SPEC V2.0의 기본 방향은:

```text
검색
분석
리스크
복기
```

이다.

기본:

```text
ENABLE_AUTO_ORDER=false
```

유지.

---

## 85. 실제 돈 관련 기능의 추가 검증

다음은 2중 검산.

- 평균체결가
- 실현손익
- 수수료
- 세금
- Position Size
- R-Multiple
- Portfolio Heat
- Daily Loss

---

## 86. Manual Verification

자동 테스트 외에 Golden Sample을 이용해 사람이 계산한 값과 비교.

Codex가 스스로 계산표 생성.

---

## 87. 최종 메뉴

- 대시보드
- 오늘의 주도주
- 종가배팅 TOP5
- 매수 전 점검
- 실시간 위험관리
- 보유종목
- 알림센터
- 증권사 리포트
- 조건검색
- 관심종목
- 실전 매매일지
- 오늘의 복기
- 전략 연구소
- 청산 전략 연구소
- 리스크 규칙 연구소
- 월간 성과
- 계좌 현황
- API 상태
- 설정

---

## 88. 대시보드 첫 화면

5초 안에 파악 가능:

- 시장 상태
- 오늘 TOP5
- 오늘 실현손익
- 평가손익
- 현재 R
- Portfolio Heat
- Daily Risk
- 현재 보유종목
- 위험경고
- 전일 TOP5 결과
- 최근 전략 성과

---

## 89. Dashboard 숫자 클릭

가능한 경우 상세 근거 페이지로 이동.

예:

```text
Portfolio Heat 1.6%
```

클릭:

```text
삼성ABC 0.4%
XYZ 0.6%
DEF 0.6%
```

---

## 90. Review System

Daily:

`오늘의 복기`

Weekly:

`주간복기`

Monthly:

`월간복기`

---

## 91. 전략 통계

기간:

```text
20거래일
60거래일
120거래일
전체
```

---

## 92. 성과분석

반드시 표본수를 함께 표시.

```text
승률 68%
n=47
```

---

## 93. 표본부족

`n < 10` 이면:

```text
⚠ 표본 부족
```

---

## 94. 과최적화 방지

특정 설정이 단기간 성과가 높다고 자동 적용하지 않는다.

---

## 95. 전략 Versioning

가중치 변경 시 새 버전.

예:

```text
Strategy V1.3-A
Strategy V1.3-B
```

과거 데이터는 원래 전략 버전을 유지.

---

## 96. 추천 설정 적용

프로그램이 `권장 가중치` 를 제시할 수는 있다.

하지만 실제 적용은 사용자가 해야 한다.

---

## 97. Data Quality Score

각 종목:

`Data Quality`

계산 가능.

예:

```text
Market data ✓
Investor data ✓
Report data ✕
Technical data ✓
```

---

## 98. 데이터 부족

AI Score가 높아도 데이터가 부족하면:

```text
⚠ 데이터 부족
```

표시.

---

## 99. Logging

로그 분리:

```text
app.log
api.log
scanner.log
trade.log
risk.log
scheduler.log
error.log
```

---

## 100. 로그 개인정보 보호

금지:

- App Secret
- Access Token
- 전체 계좌번호
- 주민번호
- 비밀번호

---

## 101. Performance

동일 데이터 중복 요청 금지.

Cache 적용.

---

## 102. API 호출 중앙관리

모든 Broker 요청은 `RateLimiter` 통과.

---

## 103. Scheduler

스케줄은 중앙 관리.

`Scheduler Service`

페이지 코드가 직접 Timer를 만들지 않는다.

---

## 104. 최종 Integration Flow Test

반드시 아래 전체 흐름을 테스트한다.

```text
Login/Auth
↓
Market Data
↓
Condition Search
↓
Scanner
↓
TOP5
↓
Pretrade
↓
Position Size
↓
Mock Fill
↓
Trade Session
↓
Position Monitor
↓
Risk Alert
↓
Exit
↓
Profit Calculation
↓
Journal
↓
Daily Review
↓
Performance Statistics
```

---

## 105. 장애 Scenario

반드시 테스트:

- API Disconnect
- Token Expired
- Rate Limit
- Missing Investor Data
- Missing Report
- DB Locked
- Duplicate Fill
- Frontend Refresh
- Backend Restart

---

## 106. 재시작 복구

앱이 종료됐다 다시 실행되어도:

- 오늘 후보
- Trade Session
- Risk Plan
- 보유종목
- 알림상태

를 DB에서 복구한다.

---

## 107. Release Candidate

모든 테스트가 완료되면:

`v1.3-rc1`

생성.

실사용 전 Demo/Mock 테스트.

---

## 108. 최종 Stable

Critical Bug가 없으면:

`v1.3-stable`

생성.

---

## 109. 완료 선언 조건

Codex는 다음 문구만으로 완료하면 안 된다.

```text
구현했습니다.
```

반드시 테스트와 QA 증거를 보여준다.

---

## 110. 최종 보고서

형식:

```text
KIWOOM AI SCANNER
MASTER SPEC V2.0

1. 구현상태
2. V1.0 결과
3. V1.0 QA
4. V1.1 결과
5. V1.1 QA
6. V1.2 결과
7. V1.2 QA
8. V1.3 결과
9. V1.3 QA
10. DB Migration
11. API Tests
12. Unit Tests
13. Integration Tests
14. Regression Tests
15. Frontend Build
16. Demo Mode
17. Mock Mode
18. 페이지별 QA Matrix
19. Known Issues
20. Stable Release 상태
```

---

## 111. Page QA Matrix

최종 보고서에 반드시 포함.

| Page | API | DB | Calc | UI | Null | Filter | Regression |
|---|---|---|---|---|---|---|---|
| Dashboard | PASS | PASS | PASS | PASS | PASS | PASS | PASS |
| TOP5 | PASS | PASS | PASS | PASS | PASS | PASS | PASS |
| Trade Journal | PASS | PASS | PASS | PASS | PASS | PASS | PASS |
| Risk | PASS | PASS | PASS | PASS | PASS | PASS | PASS |

---

## 112. 데이터 QA Matrix

| Data | API | DB | Engine | UI |
|---|---|---|---|---|
| 현재가 | PASS | PASS | PASS | PASS |
| 거래대금 | PASS | PASS | PASS | PASS |
| 외국인 | PASS | PASS | PASS | PASS |
| 평균매수가 | PASS | PASS | PASS | PASS |
| 손익 | PASS | PASS | PASS | PASS |
| R | PASS | PASS | PASS | PASS |

---

## 113. Critical Bug 기준

다음은 Stable Release 금지.

- 잘못된 손익
- 잘못된 평균체결가
- 잘못된 Position Size
- 잘못된 날짜 매칭
- 중복체결
- 잘못된 R
- 실전/모의 혼동
- 오래된 가격을 현재가로 표시
- 자동주문 의도치 않은 실행

---

## 114. Major Bug

예:

- 필터 오류
- 정렬 오류
- 통계 오류
- 차트 오류
- 일부 페이지 API 실패

가능하면 Stable 전에 수정.

---

## 115. Minor Bug

예:

- 문구
- 정렬 간격
- Responsive UI
- Tooltip

Known Issues로 관리 가능.

---

## 116. 절대 금지

- API를 추측해서 구현
- Mock 데이터를 Real처럼 표시
- Frontend에서 중요한 금융 계산
- DB 값 덮어쓰기
- 과거 Snapshot 삭제
- Migration 없이 Schema 변경
- 테스트 없는 Release
- None을 0으로 처리
- AI Score를 상승확률로 표현
- 수익 보장
- 자동주문 기본 ON

---

## 117. Codex 최종 작업 명령

지금부터 이 MASTER SPEC을 전체 프로젝트의 기준 문서로 사용한다.

먼저 기존 Repository가 존재한다면 전체 구조와 구현 상태를 분석하라.

바로 대규모 수정하지 않는다.

먼저 다음 파일을 생성한다.

```text
docs/MASTER_SPEC_V2.md
docs/IMPLEMENTATION_STATUS.md
docs/QA_MATRIX.md
docs/DATA_CONTRACT.md
```

---

## 118. DATA_CONTRACT.md

가장 먼저 정의한다.

각 필드마다:

```text
Name
Meaning
Source
Type
Unit
Nullable
Calculation
Example
```

작성.

예:

```text
turnover

Meaning:
거래대금

Unit:
KRW

Type:
integer

Source:
Market Data Service

Nullable:
false
```

---

## 119. 구현 상태 점검

기존 코드가 있다면 각 요구사항을:

```text
DONE
PARTIAL
MISSING
BROKEN
```

으로 분류.

---

## 120. Gap Report

구현 전 다음을 정리한다.

- 현재 구현
- 부족한 기능
- 오류 기능
- 중복 코드
- 위험한 코드

---

## 121. 그 다음 구현

다음 순서대로 진행.

```text
PHASE 0
Foundation
↓
V1.0
↓
V1.0 QA
↓
v1.0-stable
↓
V1.1
↓
V1.1 QA
↓
v1.1-stable
↓
V1.2
↓
V1.2 QA
↓
v1.2-stable
↓
V1.3
↓
V1.3 QA
↓
v1.3-stable
```

---

## 122. 각 Phase 종료 보고

반드시:

- 완료 기능
- 수정 파일
- 신규 파일
- DB 변경
- 테스트
- Page QA
- Regression
- 남은 문제

작성.

---

## 123. 사용자 승인 없이 진행

개발 도중 사소한 사항마다 사용자에게 질문하지 않는다.

명백한 요구사항은 MASTER SPEC을 기준으로 스스로 판단하고 진행한다.

---

## 124. 단 불확실한 API는 추측 금지

키움 API 명세가 확인되지 않으면 해당 기능을 가짜로 구현하지 않는다.

상태:

```text
BLOCKED: OFFICIAL API SPEC REQUIRED
```

로 기록한다.

---

## 125. 최종 목표

최종 프로그램은 단순히 `좋은 종목` 을 찾는 프로그램이 아니다.

사용자가 다음 질문에 데이터로 답할 수 있어야 한다.

- 어떤 종목을 골랐는가?
- 왜 골랐는가?
- 몇 점이었는가?
- 장 마감까지 더 강해졌는가?
- 다음날 실제로 올랐는가?
- 실제로 내가 매수했는가?
- 좋은 가격에 들어갔는가?
- 계획대로 손절했는가?
- 너무 일찍 팔았는가?
- 한 거래에서 몇 R을 얻었는가?
- 한 거래에서 얼마를 위험에 노출했는가?
- 현재 계좌 위험은 어느 정도인가?
- 어떤 조건이 실제로 가장 잘 작동하는가?
- 내 반복적인 실수는 무엇인가?

---

## 126. 최종 성공 구조

```text
DATA
↓
NORMALIZATION
↓
DATABASE
↓
SCANNER
↓
DECISION SUPPORT
↓
TRADE
↓
RISK
↓
REVIEW
↓
STATISTICS
↓
IMPROVEMENT
```

이 전체 흐름이 실제 데이터로 연결되고 검증되어야 한다.

---

## 127. 지금 실행

다음 순서로 지금 바로 시작한다.

```text
1. Repository 분석
2. MASTER_SPEC_V2.md 생성
3. DATA_CONTRACT.md 생성
4. 현재 구현상태 분석
5. Gap Report 작성
6. 현재 전체 Test 실행
7. 문제가 있는 기존 기능 우선 수정
8. PHASE 0 시작
9. V1.0 구현 및 QA
10. V1.1 구현 및 QA
11. V1.2 구현 및 QA
12. V1.3 구현 및 QA
13. 전체 Regression
14. Release Candidate
15. Stable Release
```

중간에 코드만 생성하고 완료하지 않는다.

반드시 실제 테스트와 데이터 검산을 수행한다.

최종 완료 시:

```text
MASTER SPEC V2.0
IMPLEMENTATION COMPLETE
```

라고 표시하되 모든 필수 Test와 QA가 PASS한 경우에만 사용한다.
