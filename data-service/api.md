# Data Service M1 API

공개 열람본 · 함께 읽기: [데이터 조사·무료 범위](crypto-data-landscape-2026-09-15.md) · [실행 설계·다음 단계](implementation-plan.md)

작성: 2026-09-15, 구현 대조: 2026-09-16 KST. 이 문서는 신규 컴포넌트의 첫 HTTP 계약이다. 기존 `docs/protocols/`와 Feeding UDP 규약은 변경하지 않는다.

## 1. 지원 범위

- `reference.instrument`: 수집 시점의 상품 정보 snapshot.
- `market.ohlcv.1m`: 원천에서 확정 구간임을 확인한 1분 봉.
- 상품: `binance-futures`의 `BTC^USDT`, `ETH^USDT`, `perpetual`, settlement=`USDT`.
- 현재와 과거의 저장된 레코드를 같은 규약으로 조회한다. 첫 단계에서는 조회 요청이 원천 API를 호출하거나 백필 작업을 자동 생성하지 않는다.
- wall-clock `as_of`, `at`, OI, 펀딩, UDP, 구독, 자동 준비 작업은 M1에서 미지원이다. 지원하지 않는 요청 필드는 조용히 무시하지 않고 거부한다.

## 2. Endpoint

| 요청 | 의미 |
|---|---|
| `GET /` | 실제 catalog/query/health를 사용하는 로컬 검토 화면. 외부 CDN이나 별도 프론트 서버 없이 제공 |
| `GET /health` | 프로세스/수집 상태. 데이터 최신성을 일괄 보증하지 않음 |
| `GET /data/v1/catalog` | 지원 데이터·상품·단위·로컬 보유 범위·기능 |
| `POST /data/v1/query` | 최신 N개 또는 기간 조회, snapshot 고정 및 페이지 이동 |

기본 서버는 loopback의 임시 포트로 실행하고 실제 주소를 출력한다. 운영 포트·원격 접근·인증·배포는 후속 단계다.

## 3. 요청

```json
{
  "dataset": "market.ohlcv.1m",
  "instrument": {
    "venue": "binance-futures",
    "symbol": "BTC^USDT",
    "product": "perpetual",
    "settlement": "USDT"
  },
  "time": {"mode": "latest"},
  "limit": 5,
  "max_age_ms": 120000
}
```

`max_age_ms=120000`은 요청자의 허용 나이 예시이며 서비스 SLA가 아니다. 봉의 나이는 시작 시각이 아닌 종료 경계를 기준으로 계산한다. 오래된 값도 데이터와 stale 표시를 함께 반환하므로 소비자가 허용 여부를 결정해야 한다.

기간 조회는 `time`을 다음으로 바꾼다. 시간은 UTC Unix ms이며 `[start_ms, end_ms)`다. 반환 레코드의 event time을 기준으로 포함 여부를 판정한다. 1분 경계로 맞춘 요청을 권장한다.

```json
{"mode":"range","start_ms":1789483140000,"end_ms":1789483440000}
```

| 필드 | 제한/의미 |
|---|---|
| `dataset`, `instrument`, `time` | 필수. 지원하지 않는 조합은 입력 오류 |
| `limit` | 기본 500, 1~1,000. 최신 N개 또는 페이지 크기 |
| `view_seq` | 선택. 저장 snapshot의 revision 경계. 미래/음수 경계 거부 |
| `cursor` | 선택. 이전 응답의 opaque 페이지 토큰. 클라이언트가 내용을 만들거나 수정하지 않음 |
| `max_age_ms` | 선택. 비음수. 지정하지 않으면 fresh/stale 임계값을 임의로 가정하지 않음 |
| 기간 길이 | 한 요청 최대 31일. 더 긴 기간은 별도 구간 요청으로 분리 |

다음 페이지는 **같은 요청에 이전 `next_cursor`를 넣어** 호출한다. dataset/instrument/time/limit/max_age를 변경하지 않는다. 응답의 `view_seq`로 고정되며 페이지 사이에 들어온 정정/백필은 그 조회에 끼어들지 않는다.

## 4. 응답 레코드

상위 응답은 `records`, `view_seq`, `next_cursor`, `coverage`, `freshness`다. records는 event time 오름차순이다. latest는 최신 N개를 선택한 뒤 오름차순으로 반환하며 기간 pagination과 구분한다.

| 레코드 필드 | 의미 |
|---|---|
| `dataset`, `instrument` | 요청과 같은 데이터/상품 식별 |
| `event_time_ms` | 봉 시작 또는 상품 snapshot 관측 시각 |
| `record_id` | 같은 사건/봉의 정정 전후를 연결하는 ID |
| `revision` | 같은 record의 값이 바뀔 때 증가 |
| `seq` | 이 revision을 저장한 트랜잭션 경계. 같은 batch의 레코드는 같은 seq를 가지며 event time 순서와 다름 |
| `observed_at_ms` | 이 revision의 원천 응답을 실제 수신한 시각 |
| `recorded_at_ms` | 로컬 저장 처리 시각. 정확한 최초 공개 시각을 뜻하지 않음 |
| `raw_response_id` | 로컬 원본 응답 근거. 내부 파일 경로는 API로 노출하지 않음 |
| `payload` | 데이터별 타입 필드 |

봉 payload의 `open`, `high`, `low`, `close`, `volume`, `quote_volume`, `taker_buy_volume`, `taker_buy_quote_volume`는 **실제 값 ×100,000,000의 정수 문자열**이다. 예를 들어 가격 `123.45`는 `"12345000000"`이다. `scale=100000000`, `is_final=true`, `open_time_ms`, `close_time_ms`, 정수 `trades`를 함께 제공한다. ×1e8로 표현할 수 없는 유효숫자는 수집 오류다.

상품 payload는 원천 심볼·base/quote/settlement·거래 상태·계약 종류·상장/만기 시각·실제 tick/step을 담는다. tick/step도 ×1e8 정수 문자열이다. `pricePrecision`을 tick size로 사용하지 않는다. snapshot 관측 전의 계약 규칙을 소급해서 알 수 있다고 표시하지 않는다.

## 5. Coverage·freshness·PIT

- 봉 기간 조회는 요청 구간 전체의 관측 가능한 슬롯을 검사한다. 페이지에 일부 레코드만 반환됐다는 이유로 데이터 결측으로 판단하지 않는다.
- `complete`는 해당 요청 구간의 봉 슬롯이 로컬 snapshot에 모두 있다는 의미다. 원천 전체 이력을 모두 확보했다는 뜻은 아니다.
- `partial`은 요청 구간에 중간 누락을 포함한 결측이 있는 상태, `not_collected`는 요청 구간에 저장된 값이 없는 상태다. 구체적인 gap 구간을 반환한다.
- 상품 snapshot의 관측되지 않은 구간은 실제 변경 이력의 완전성을 알 수 없으므로 `unknown`으로 다룬다.
- fresh/stale은 조회 대상 범위에서 가장 최근 데이터의 대상 시각과 요청 임계값으로 계산한다. 기간 pagination에서는 현재 페이지의 모든 레코드가 fresh하다는 뜻이 아니라 전체 요청 범위의 최신 끝을 평가한다. 과거 조회의 stale 표시가 그 값의 역사적 유효성을 부정하는 것은 아니다.
- `view_seq`는 정정 전후 데이터셋을 재현하는 기능이다. 벽시계의 각 매매 판단 시점에 실제로 공개/이용 가능했는지 보장하는 PIT 기능은 아니다. 카탈로그에 PIT 미지원을 표시한다.
- 중복 재수집은 기존 revision을 유지한다. 값이 A→B→A로 정정되면 세 revision으로 보존한다.

## 6. 오류와 호출 책임

입력·지원 조합·cursor·범위 오류는 400 계열, 저장소 접근 실패는 503 계열로 처리한다. 빈 정상 결과와 실패 응답을 구분한다. 원천 장애는 수집 상태에 남기며 이미 저장된 조회 데이터는 해당 시각/품질과 함께 제공할 수 있다.

클라이언트는 `records`만 읽고 현재성·coverage를 버리지 않도록 한다. 분할 구간 전체를 한 버전으로 연구하려면 처음 받은 `view_seq`를 모든 후속 구간 요청에도 사용한다.

검증 결과·알려진 제한·다음 단계는 [실행 설계](implementation-plan.md)를 참조한다. 실제 구동 명령은 프로젝트 내부 README에서 관리하며, 실행용 파일·설정·로그는 이 공개본에 포함하지 않는다.
