# ADR0001 — 봉·Binance 파생 지표·Vision 과거 데이터를 sicadb에 적재한다

상태: 초안 — 2026-09-27. 이관 방향(feeding ADR0011, «sicadb로 적재만 한다, 읽기는 읽는 쪽 몫»)은 사용자 결정이다. 아래 세부 설계는 원천 전수 조사 결과로 고친 뒤 확정한다. 구현은 sicadb 저장이 마무리되고 사용자가 지시한 뒤에 한다.

## 배경

- feeding은 WebSocket 실시간 시세만 남긴다. 봉(5m·1h·4h·1d)과 Binance OI·top trader·taker 비율 수집은 data-service가 넘겨받는다([feeding ADR0011](../../../feeding/docs/adr/0011-realtime-ws-only-scope.md)).
- bv_sync가 해 온 Binance Vision 과거 데이터 확보도 data-service가 흡수한다. 세부 구현은 가져오지 않고, 필요한 기능만 sicadb 기반으로 다시 만든다.
- 저장은 sicadb다. 실시간은 producer([usage.md 1절](../../../sicadb/docs/usage.md)), 과거는 `Backfill`([backfill.md](../../../sicadb/docs/backfill.md))로 같은 archive에 넣는다.

## 결정

### 1. 구성

- data-service 크레이트에 `src/sica/` 모듈과 두 실행 파일을 더한다. sicadb는 `sicadb-apl-bindings` path 의존으로 쓴다.
  - `ds-feed`: 실시간 feed handler. 원천 REST를 부르고 producer로 쓴다. 원천 기록(archive)을 켠다.
  - `ds-backfill`: 과거 적재. Vision 파일과 원천 REST 과거 구간을 `Backfill`로 넣는다.
- 데이터셋 정의(`DatasetDescriptor`)와 파티션 키는 `src/sica/schema.rs` 한 곳에서 만든다. 두 실행 파일이 같은 함수를 쓴다(backfill.md 2절의 «라이브와 같아야 하는 것»).
- 기존 SQLite·HTTP·UI·Arrow outbox 코드는 쓰지도 고치지도 않는다. 나중에 사용자가 지운다.

### 2. 키와 값

- **키**: (`exchange`, `symbol`) `FixedBinary(20)` 두 필드이고, 시각 열과 함께 키 descriptor를 만든다. 모양은 market-ingest `key_descriptor`와 같다. 그래서 같은 instrument는 시장 데이터와 같은 키 바이트를 가진다.
  - 이름 규약은 `symbol = BTC^USDT`, `exchange`는 공용 universe 이름(`binance-futures`, `binance`, `bybit-linear`, `okx-swap`, `okx-spot`)이다.
- **정확한 값은 고정 스케일 `I64`로만 담는다**(사용자 결정 2026-09-27: Decimal 금지, i64로 결정).
  - 원천의 소수 문자열을 그대로 정수로 옮긴다. 반올림·절삭은 하지 않는다.
  - 스케일에 맞지 않는 값(소수 자리 초과 또는 i64 초과)이 오면 그 **필드를 null로 두고 센다**. 행은 버리지 않고, 알림을 보낸다.
  - 파서는 지수 표기(`2.62e-05`, `0E-8`)를 정확히 정수로 옮기고, 빈 문자열은 null로 둔다.
- **스케일은 테이블마다 고정이고, 봉 테이블은 거래소마다 나눈다**(예: `binance-futures.kline.5m`). 한 열에 한 스케일만 쓰면 거래소마다 필요한 자릿수가 달라 잃는 것이 늘기 때문이다.
  - 실측 근거: 넘겨받는 1,133종목의 일봉 전 이력(2022~2026). Binance 선물은 1,500일, 나머지는 1,000일이다.
  - 스케일과 여유(«여유» = i64 한도 ÷ 관측 최대값×스케일, 일봉 기준):

    | 테이블(거래소) | 가격 | 거래량 | 거래대금 | taker 거래량 / 거래대금 | 여유가 가장 작은 곳 |
    |---|---|---|---|---|---|
    | binance-futures | ×10^10 | ×10^3 | ×10^8 | ×10^3 / ×10^8 | 거래대금 BTC 1d 6.9e10 → **1.3배** |
    | binance(현물) | ×10^10 | ×10^6 | ×10^8 | ×10^6 / ×10^8 | 거래량 1000SATS 2.0e12 → 4.5배 |
    | bybit-linear | ×10^10 | ×10^3 | ×10^7 | — | 넉넉함 |
    | okx-swap | ×10^10 | ×10^4 (기초자산, `volCcy`) | ×10^6 | — | 거래량 PEPE 4.5e14 → 2배 |
    | okx-spot | ×10^10 | ×10^8 | **담지 않음** | — | 거래량 PEPE·SHIB·BONK는 넘침 → null |

  - 가격 ×10^10: 전 종목 손실 0(가장 작은 틱 1e-9). 체결수는 `U64`, 시각은 `TimestampNs`다.
  - **i64로 잃는 것**(넘겨받는 범위 전체에서 이것뿐):
    - OKX 현물 거래대금: 소수 9~13자리이면서 BTC는 3e9라 한 스케일로 담을 수 없다. 전 종목 null로 둔다. 읽는 쪽은 거래량 × 가격으로 근사한다.
    - OKX 현물 거래량 중 PEPE·SHIB·BONK 3종목: null.
    - Binance OI 통계의 `CMCCirculatingSupply`(유통량): 12종목이 넘친다. 열을 두지 않는다.
  - 여유가 작은 곳(Binance 선물 BTC 1d 거래대금 1.3배, OKX PEPE 2배)은 거래가 폭증한 날 넘칠 수 있다. 그날의 그 필드는 null이 되고 알림이 간다. 5m·1h·4h는 여유가 크다.
- **Binance 5분 계열 스케일**(369종 30일 실측, 손실 0)
  - OI(`sumOpenInterest`, 현재 OI) ×10^4. 프로젝트 OI 관례와 같다.
  - OI 가치(`sumOpenInterestValue`) ×10^8.
  - taker `buyVol`·`sellVol` ×10^4.
  - 비율(L/S, taker 비율, basisRate)과 funding ×10^8.
  - basis 가격은 가격과 같은 ×10^10.
  - Vision metrics 2023년 비율 값 일부는 float 표현 잔재(16자리)라 ×10^8에 맞지 않는다. 이 값들을 null로 둘지, 잔재 자리만 잘라 받을지는 백필 시험 때 행 수를 세서 사용자가 정한다.
- 원천에 없는 필드는 nullable 열에 null로 둔다. 0으로 채우지 않는다.
- 원천에 없는 필드는 nullable 열에 null로 둔다. 0으로 채우지 않는다.

### 3. 테이블

시각 열 이름은 모두 `time`이다. 봉은 시작 시각이고, 그 밖의 계열은 원천이 준 시각이다.

| 테이블 | 채우는 경로 | 열 |
|---|---|---|
| `{거래소}.kline.5m` · `.1h` · `.4h` · `.1d` (거래소별 테이블, 스케일은 §2) | ds-feed(REST 폴링) + ds-backfill(Vision klines, REST 과거) | `time`, `close_time`, `open`, `high`, `low`, `close`, `volume`, `quote_volume`?, `taker_buy_volume`?, `taker_buy_quote_volume`?, `n_trades`? (?=nullable. 거래소마다 제공 필드가 다르다) |
| `derivatives.open_interest.snapshot` | ds-feed (`/fapi/v1/openInterest`, 5분 경계마다) | `time`(원천 `time`), `open_interest` |
| `derivatives.open_interest.5m` | ds-feed + ds-backfill (`/futures/data/openInterestHist`) | `time`, `sum_open_interest`, `sum_open_interest_value` (`CMCCirculatingSupply`는 i64를 넘쳐 두지 않는다) |
| `derivatives.top_trader_account_ratio.5m` | 〃 (`topLongShortAccountRatio`) | `time`, `long_account`, `short_account`, `long_short_ratio` |
| `derivatives.top_trader_position_ratio.5m` | 〃 (`topLongShortPositionRatio`) | 〃 |
| `derivatives.global_account_ratio.5m` | 〃 (`globalLongShortAccountRatio`) | 〃 |
| `derivatives.taker_ratio.5m` | 〃 (`takerlongshortRatio`) | `time`, `buy_sell_ratio`, `buy_vol`, `sell_vol` (절대량은 원천이 30일만 보존한다) |
| `derivatives.basis.5m` | ds-feed + ds-backfill (`/futures/data/basis`, PERPETUAL. 2019-12-24부터 전 기간) | `time`, `index_price`, `futures_price`, `basis`, `basis_rate`, `annualized_basis_rate`? |
| `derivatives.funding_rate.settled` (라이브) | ds-feed + ds-backfill (`/fapi/v1/fundingRate`. 2019-09-10부터, Vision보다 길다) | `time`(fundingTime), `funding_rate`, `mark_price`?, `rate_type` |
| `derivatives.binance_metrics.5m` | ds-backfill (Vision `metrics`) | Vision 헤더 그대로: `time`(create_time), `sum_open_interest`, `sum_open_interest_value`, `count_toptrader_long_short_ratio`, `sum_toptrader_long_short_ratio`, `count_long_short_ratio`, `sum_taker_long_short_vol_ratio` |
| `derivatives.funding_rate.vision` | ds-backfill (Vision `fundingRate`, 월별) | `time`(calc_time), `funding_interval_hours`, `last_funding_rate` |
| `market.mark_price_kline.1m` · `market.index_price_kline.1m` · `market.premium_index_kline.1m` | ds-backfill (Vision) | 봉과 같은 열(거래량 계열 null) |

- 원천이 다른 값은 **섞지 않고 테이블을 나눈다**. 예를 들어 Vision metrics와 REST 비율 계열은 필드 정의가 다르다. 경계를 맞추거나 병합하지 않는다(ADR0048: 순서·병합을 만들지 않는다).
- 원문 응답은 테이블에 넣지 않는다(sicadb ADR0054).

### 4. 실시간 수집 (`ds-feed`)

- **봉은 WebSocket이 아니라 경계 직후 REST 폴링으로 받는다.**
  - TF 경계 + 2초에 각 instrument의 최근 봉을 요청하고, `close_time < 원천 시각`인 확정 봉만 쓴다.
  - 같은 요청이 `frontier`(키마다 마지막으로 쓴 봉의 시작 시각) 이후를 모두 가져오므로, 빈 구간 보충과 실시간이 **같은 경로**다.
  - 이유: 연결·재구독·재연결 상태가 없고, 끊겼다 돌아와도 다음 요청이 빈 구간을 채운다. 부하는 5분마다 약 1,137요청(거래소별로 나뉨)이라 한도에 여유가 있다.
  - 대가: 확정 뒤 수 초 늦다. 봉 소비자(중기 전략)에는 충분하다고 보고, 지연 분포를 측정해 기록한다.
- **Binance 5분 비율·OI 계열**: `/futures/data/*`는 IP당 5분 1,000요청의 별도 한도다.
  - 369종 × 5계열 = 1,845 계열이라 5분마다 모두 부를 수 없다.
  - 그래서 계열마다 frontier 이후를 `limit`으로 한 번에 받는 **예산 기반 순환**으로 돈다(기본 5분 800요청). 각 계열은 약 12분마다 빈 행 없이 따라잡는다.
  - 현재 OI(`/fapi/v1/openInterest`, 일반 weight 1)는 5분 경계마다 전 종목을 부른다.
- **시작 시 frontier**
  - archive에서 키마다 마지막 행(`latest_by_key`, 최근 수신 40일 창)을 읽는다.
  - archive에 없는 키는 설정한 부트스트랩 깊이만큼 받는다(기본: 봉 TF마다 1,000개 이하, 비율 계열 원천 한도 30일).
  - 그보다 오래된 과거는 ds-backfill 몫이다.
- **쓰기**
  - producer 샤드 하나를 전용 OS 스레드가 가진다(`InputShard`는 `!Send`).
  - 원천 요청은 tokio task가 하고, 파싱한 행을 제한된 채널로 그 스레드에 넘긴다. 테이블마다 writer는 그 스레드 하나다.
  - `Held`는 다른 일을 한 뒤 다시 부른다. `Dropped`·거절은 세고 로그에 남긴다.
  - `received_at_ns`는 응답을 받은 시각이다. 한 응답의 행은 같은 `ingress_ordinal`을 공유한다.
- **universe**: `config/full_universe_shared.toml`의 `*_kline` 그룹(testnet 제외)을 읽는다. 비율 계열은 `bf_kline`의 binance-futures 목록을 쓴다.
- **거래소별 봉 계약**(2026-09-27 실측, [원천 조사](../원천조사_2026-09-27.md) §1.1)
  - Bybit `/v5/market/kline`
    - 응답은 내림차순이다.
    - 마지막 봉은 미확정이다.
    - 심볼 `launchTime` 이전 구간은 주지 않는다.
  - OKX
    - 6H 이상 봉은 UTC+8 경계가 기본이다. `4Hutc`·`1Dutc`로 요청한다.
    - `confirm="1"`만 쓴다.
  - taker 매수량과 체결수는 Binance만 준다. 다른 거래소는 그 열이 null이다.
- **호출 예산**
  - Binance fapi weight(분당 2,400)는 같은 IP의 다른 수집기(feeding 등)와 공유된다. 응답 헤더 `x-mbx-used-weight-1m`을 권위값으로 보고 속도를 맞춘다.
  - `/futures/data/*`는 사용량 헤더가 없어 직접 센다.
  - Bybit은 IP 5초당 600회이고, 넘으면 10분간 차단된다.

### 5. 과거 적재 (`ds-backfill`)

- **Vision**
  - `data.binance.vision`에서 `<raw>/data/...` 경로 그대로 받고, `.CHECKSUM`(SHA-256)을 검증한다. 원자적 쓰기(`.tmp` → rename)로 쓰고, 대역폭 상한은 설정으로 둔다.
  - 심볼마다 monthly를 먼저 보고, 없는 달은 daily로 채운다(bv_sync에서 가져오는 규칙).
  - 받은 zip은 지우지 않는다(검증 기준).
  - 형식 차이를 흡수한다(2026-09-27 실측).
    - 헤더: 같은 데이터셋 안에서도 파일마다 있거나 없다. 첫 줄로 판별한다.
    - 시각: spot은 2025-01-01부터 µs다. 자릿수로 판별한다. metrics 시각은 날짜 문자열이다.
    - 값: `True/False`의 대소문자, `0E-8`, 빈 문자열.
    - 중복·순서: metrics 2020-09~12는 모든 행이 중복이고, 일부 파일은 정렬돼 있지 않다. 중복 행은 같은 값이므로 (키, `time`)마다 하나만 넣는다. 정렬은 가정하지 않는다.
    - 잡파일: `part-*.zip`과 빈 interval 디렉터리는 건너뛴다.
  - Vision cm fundingRate는 2026-06에서 멈췄다. CM funding은 REST `/dapi/v1/fundingRate`로 받는다.
- **REST 과거**: bybit·okx 봉처럼 Vision에 없는 과거는 원천 REST history로 받아 같은 테이블에 넣는다.
- **Backfill 규칙**
  - 실행 하나 = (테이블, 한 달) 또는 (테이블, 하루)다. 실패한 단위만 다시 돌린다. 겹침 검사가 중복을 막는다.
  - 엔드포인트 이름: `import-binance-vision`, `import-rest-<거래소>`.
  - `received_at_ns = 0`(이벤트 시각을 수신 시각으로 쓴다).
- **라이브 테이블(`{거래소}.kline.*`, 5분 비율 계열)** 은 라이브 기록기의 시작 시각 **앞**까지만 넣을 수 있다(sicadb 규칙). 그래서 과거 적재를 ds-feed 운영 시작 전에 하거나, `--to`를 라이브 시작 앞으로 둔다.
  - ds-feed의 부트스트랩 구간과 Vision 구간이 겹치는 봉은 두 번 기록될 수 있다(수신 시각과 엔드포인트가 다르다). 읽는 쪽은 (키, `time`)마다 하나를 고른다.
- Vision 전용 테이블(metrics·funding·mark/index/premium)은 라이브 기록기가 없으므로, 매일 새 날짜를 이어서 넣을 수 있다.

### 6. 범위 밖

- 이 ADR은 넘겨받는 범위만 다룬다. 추가로 가져올 수 있는 데이터(청산·옵션 체인·호가 아카이브·교차거래소 집계·온체인·거시 등)는 [원천 조사](../원천조사_2026-09-27.md)에 있다. 채택 여부는 사용자가 정하고, 채택하면 별도 ADR로 설계한다.
- aggTrades(틱). bv_sync는 aggTrades로 봉을 만들었지만, Vision이 봉 파일(`klines`)을 직접 주므로 필요 없다. 틱이 필요하면 시장 데이터 경로(market-ingest)의 일이다.
- 조회 API·과도기 조회 경로·소비자 전환: 읽는 쪽이 sicadb로 읽는다.
- UDP 송신, QuestDB 적재.

## bv_sync 정리 방안

1. ds-backfill Vision 경로를 검증한다(행 수·표본 값을 Vision 원본과 대조).
2. 운영 archive에 과거 구간을 넣는다(사용자 확인 뒤).
3. bv_sync 스케줄 태스크를 멈춘다(QuestDB `bars_*`·`metrics_5m`·`funding` 적재도 함께 멈춘다). 읽는 쪽이 모두 sicadb로 옮긴 뒤 bv_sync 레포와 NAS의 bv_sync 원본 폴더를 정리한다. 지우는 것은 사용자가 결정한다.

## 이유

- 데이터셋 정의를 한 함수에서 만들어야 라이브와 백필이 한 테이블로 이어진다.
- 정확한 소수를 한 척도로 통일하면 넘침·절삭 걱정이 없고, 읽는 쪽이 열마다 척도를 몰라도 된다.
- 폴링 하나로 실시간과 빈 구간 보충을 함께 하면, 회복 경로가 따로 없어 «존재하지만 발동하지 않는 회복 코드»가 생기지 않는다(루트 CLAUDE.md 경계 계약 규율 3).
