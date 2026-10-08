# OMS DLL — 목적·계획·스펙·검증·테스트 (검토용 초안, 2026-10-08)

상태: **초안. 사용자 검토 대기.** 구현은 시작하지 않았다.
선행 결정: 2026-09-27 «OMS DLL을 sicadb 위에 붙이기» 사용자 결정(아래 §2.3 요약).
코드 기준: hft-oms_rust `master`(2026-10-08), sicadb `0249d68`, market-ingest `main`.
경로는 모두 저장소 기준 상대 경로다.

---

## 0. 한 장 요약

- **무엇**: hft-oms_rust의 «대상 심볼 경로»(호가창 → 큐 추적 → 체결 모델/거래소 연결 → 전략 → 주문 라우터 → 주문·재고·지갑)를 **DLL 하나**로 떼어 낸다. sicadb의 «여러 스레드가 함께 부르는 DLL» 규약(ADR0050)으로 싣고 부른다.
- **왜**: ① 실전 구성 ②(한 프로세스: 매칭엔진 → 피처 → 이론가 → OMS)를 만들기 위해. ② 백테스트(새 APBT)와 실전이 **같은 OMS 코드**를 쓰게 하기 위해. ③ 이론가는 APL로, 전략 선택·파라미터는 설정으로 넘겨 **재컴파일 없이** 실험하기 위해.
- **어떻게**: P0 규약 확정 → P1 runner에서 OMS 코어 분리(동작 불변) → P2 DLL·페이퍼 모드 → P3 sicadb 기반 새 APBT 호스트 → P4 실전(승인 후).
- **검증 핵심**: 단계마다 «바뀌기 전과 바이트 단위로 같다»를 오라클로 증명한다. 리팩터 전후 APBT 체결 동일(V1), DLL과 runner 동일(V2), sicadb 패킷과 캡처 패킷 대조(V3), DLL 수명·장애 경로 발동 증명(V4), testnet → 소액 실전(V5).

---

## 1. 목적

### 1.1 풀려는 문제
1. **실전 구성**: 지금 OMS는 runner 프로세스 하나가 수신·피처·이론가·주문을 모두 한다. 2026-09-27 결정(구성 ②)은 «피드 핸들러는 데이터만, 한 프로세스가 매칭엔진 → 피처 → 이론가 → 같은 프로세스의 OMS DLL 호출»이다. sicadb 기반 수신·매칭(market-ingest)은 있지만 OMS가 그 프로세스에 들어갈 방법이 없다.
2. **백테스트 데이터 단절**: 기존 APBT는 패킷 캡처(pkt_record)를 runner에 넣어 돌린다. 캡처는 2026-10-06 249 은퇴로 멈췄다. 앞으로 데이터는 sicadb 아카이브(market.orderbook, market.trades, …)에만 쌓인다.
3. **재컴파일 없는 실험**: 리서치가 이론가 계산과 주문 방식을 바꿀 때마다 재컴파일하지 않고, 에이전트가 CLI 인자(APL 프로그램, 설정)로 돌릴 수 있어야 한다(사용자 2026-10-08).

### 1.2 OMS가 «통째로» 필요한 이유
- 전략 11종이 모두 대상 심볼의 **L3 호가창 전체**를 읽는다(`hft-oms_rust/crates/oms/src/strategy.rs`, `best_queue.rs`).
- 주문을 만들려면 OMS 내부 상태가 필요하다: 살아 있는 내 주문(가격·수량·상태), 내 앞 큐 잔량(QT), 재고·지갑, 접수 대기 중 주문, rate limit.
- 페이퍼 체결(TOM, `oms/src/tom.rs`)은 패킷·L3 이벤트를 매번 받아야 한다.
- 그래서 «이론가만 받는 얇은 함수»로는 안 되고, **호가창·체결 모델·전략·주문 상태가 한 몸인 OMS**가 데이터 경로 옆에 있어야 한다.

### 1.3 범위
- 포함: OMS 코어 분리, DLL, 페이퍼 모드, sicadb 기반 백테스트 호스트(새 APBT), 실전 모드(커넥터·거래내역)와 market-ingest 호스트.
- 제외(이번 계획 밖): 이론가 모델 자체(리서치 몫), 피처 엔진 이전, 여러 심볼을 한 인스턴스에서 묶는 일(1단계는 인스턴스 하나 = 대상 하나), GUI 재설계.

---

## 2. 현재 상태 (코드 사실)

### 2.1 runner의 대상 경로 (`hft-oms_rust/crates/runner/src/trading_runner.rs`)
```
UDP 수신 → TradingRunner::on_packet(패킷, ts)
  ├ Engine::process: 전 심볼 호가창·피처 갱신
  ├ (대상 심볼이 아니면 끝)
  ├ QT.on_packet           — 내 주문 큐 위치 추적(관찰용)
  ├ TOM.on_tick + apply_l3 — 페이퍼 체결
  ├ tp = 모델 선형식(bp) → tp_price = WAP × (1 + tp×1e-4)
  ├ call_strategy(...)     — 전략 → FlipOrder(New/Cancel) 목록
  └ OrderRouter::do_order  — SharedOms 등록 → 페이퍼면 TOM, 실전이면 HTTP 전송
```
- 전략은 닫힌 enum `StrategyParams`(variant 11개)와 `oms_*` 함수들이다. `call_strategy`는 private이라 바깥에서 전략을 끼울 수 없다.
- 실전 커넥터: binance-futures, bithumb(`crates/connector/src/`). HTTP 풀, private WS, reconcile worker, 주문 로그 worker가 `SharedOms`를 각자 잠가 갱신한다.

### 2.2 sicadb·market-ingest 쪽에 있는 것
| 필요 | 상태 | 위치 |
|---|---|---|
| 여러 스레드가 함께 부르는 DLL 규약·호스트 | 있음, **테스트 DLL로만 검증** | sicadb ADR0050, `sicadb-native-kernel-abi/src/shared.rs`, `sicadb-native-kernel-host/src/shared.rs` |
| sicadb 원천 행 → 패킷 재조립 | 있음 | market-ingest `sicadb-live-ingest/src/matching_pipeline.rs` `PacketAssembler` |
| 원천 행 수신 순서 재생(과거) | 있음 | sicadb `archive_query::ReplayMerge` |
| APL 프로그램 재생(이론가 등, 미래참조 방지 시계·캐시 포함) | 있음 | sicadb `replay_apl` |
| 매칭엔진·1초 피처를 수신 프로세스 안에서 | 있음 | market-ingest `market-producer-inline` |
| 이론가 계산과 OMS 호출 | **없음** | — |
| OMS DLL 본체 | **없음** | — |

### 2.3 2026-09-27 사용자 결정 요약
- 구성 ②: 한 프로세스에서 매칭엔진 → 피처 → 이론가 → 같은 프로세스의 OMS DLL 호출.
- 호출 계기: 대상 심볼의 매칭 경로 패킷마다(지금 runner와 같음).
- 인자: 같은 프로세스이므로 이론가 bp + WAP를 주면 나머지는 OMS가 한다.
- 동시 호출·중복 식별·실패 처리·종료·복구·거래내역 기록은 **모두 DLL 내부 책임**. sicadb는 호출만 한다.
- 다른 경로(data-service 등) 데이터는 이론가 계산 시점에 sicadb 최신값으로 읽는다(기다리지 않음).
- GUI는 OMS 프로세스에 붙인다. OMS 상태 사본 테이블은 만들지 않는다.

---

## 3. 구조

```
                      ┌──────────────────────── 호스트 프로세스 ────────────────────────┐
 sicadb 원천 행  ──▶  │ 패킷 재조립 ─▶ (매칭엔진·피처) ─▶ 이론가(APL) ─┐                 │
 (실전: 공유메모리,    │                                                 ▼                 │
  백테: 아카이브 재생) │                                   OMS DLL  call(PACKET, …)       │
                      │                                    ├ 호가창(book) · QT           │
                      │                                    ├ 체결: TOM(페이퍼) / 커넥터(실전)│
                      │                                    ├ 전략(설정으로 선택·파라미터)  │
                      │                                    ├ 주문 라우터 · SharedOms      │
                      │                                    └ 거래내역 → sicadb 테이블     │
                      └────────────────────────────────────────────────────────────────┘
```
- **OMS 코어(`OmsCore`)**: runner에서 떼어 낸 대상 경로. 피처 엔진은 포함하지 않는다(이론가는 호출 인자로 받는다).
- **DLL(`oms-dll`, cdylib)**: OmsCore를 sicadb 공유 DLL 규약으로 감싼 것.
- **호스트 두 개**
  - 백테스트 호스트(새 APBT): sicadb 아카이브 재생 → 패킷 재조립 → 이론가 APL → DLL(페이퍼).
  - 실전 호스트: market-ingest 수신 프로세스가 DLL을 싣고 대상 패킷마다 부른다.
- **runner**: 리팩터 뒤에는 OmsCore를 직접 쓴다. 실전 runner의 동작은 바뀌지 않아야 한다(§6 V1).

---

## 4. 스펙

### 4.1 DLL 규약 (sicadb ADR0050, 이미 정해진 부분)
```
sicadb_native_shared_describe_v1() -> *const SharedDescriptor   (flags = SHARED_FLAG_CONCURRENT)
create(config: *const u8, config_len: u64, instance: *mut *mut c_void) -> i32
start(instance) -> i32
call(instance, op: u32, input: *const u8, input_len: u64,
     output: *mut u8, output_cap: u64, output_len: *mut u64) -> i32
stop_join(instance) -> i32
destroy(instance)
```
- `create`는 스레드를 띄우지 않는다. 실패하면 스스로 정리한다.
- `call`은 어느 스레드에서든 동시에 올 수 있다. 상태 코드는 호스트가 해석하지 않고 호출자에게 그대로 간다.
- unwind가 C ABI를 넘지 않는다(경계에서 `catch_unwind`).
- `stop_join`이 실패하면 호스트는 destroy·언로드를 하지 않는다.

### 4.2 설정 (`create`의 config 바이트)
- 지금 runner TOML의 절을 그대로 쓴다: `[strategy]`(종류·파라미터), `[tom]`(지연·join_frac·cross_fill·수수료), `[oms]`(limit_qty 등), `[runtime]`(order_warmup_sec 등), 대상 심볼·가격/수량 정밀도·틱 규격, 모드(`paper`/`live`), 실전이면 커넥터 절(키는 파일 경로 참조만, 값은 넣지 않음), 거래내역 sicadb 출력 위치.
- 파싱·검증은 `create`에서 끝낸다. 잘못되면 `create`가 실패한다(조용한 기본값 금지).

### 4.3 op 목록
| op | 입력 | 출력 | 부르는 곳 |
|---|---|---|---|
| `PACKET` | 패킷 바이트(`MarketDataPacket`), 수신 시각 ts_ns, 이론가 tp_bp(NaN 허용), WAP, 성분 tp1·tp2·tp12·ema, 외부 신호 값들 | 상태, 이번 호출에서 낸 신규·취소 수, 체결 수 | 대상 심볼 패킷마다 |
| `MARKET` | 대상이 아닌 심볼 패킷(교차 체결·틱 규격에 필요한 경우만) | 상태 | 필요 시 |
| `CONTROL` | Pause / Resume / CancelAll / KillSwitch / SetLimitQty / SetStrategyParams / ResetOms | 상태 | GUI·운영 |
| `QUERY` | 무엇을(주문 목록·재고·지갑·통계) | 직렬화한 스냅샷 | GUI·리포트·백테 끝 |
| `EXTERNAL_QUOTES` | (선택, §4.6) 원하는 호가 목록 | 상태 | External 전략일 때 |

- 입력·출력은 `#[repr(C)]` 구조체 + 가변 길이 꼬리(패킷 바이트, 목록)로 고정한다. 앱과 DLL이 같은 크레이트(`oms-dll-abi`)를 공유해 정의 한 벌만 둔다.
- 출력 버퍼 부족은 **주문·ID 발급 같은 부작용보다 먼저** 거부한다(재호출해도 중복 주문이 안 나게).

### 4.4 시간·결정성
- 페이퍼 모드의 시계는 **패킷 수신 시각**이다(벽시계를 읽지 않는다). 같은 입력이면 같은 출력이 나와야 한다(백테 재현성).
- 실전 모드는 커넥터가 벽시계를 쓴다. 주문 판단 시점은 `PACKET` 호출 시각이다.
- 주문 판단 계기는 지금과 같다: 대상 심볼 패킷마다(결정 2a).

### 4.5 동시성·실패
- 인스턴스 하나 = 대상 심볼 하나(1단계). 같은 대상은 늘 같은 스레드가 부른다(market-ingest의 심볼 샤딩).
- 인스턴스 안의 상태 보호는 지금 `SharedOms` 잠금을 그대로 쓴다(커넥터 스레드와 공유).
- 상태 코드: 0 성공 / 입력 오류 / 버퍼 부족(부작용 전) / 부분·불명(부작용 뒤 실패 — 자동 재호출 금지, reconcile로 확정) / poison(panic 뒤 — 더 부르지 않음).
- 발동 카운터: 거부·불명·poison·reconcile 발동을 `QUERY`로 볼 수 있게 센다(회복 경로 «발동» 증명용).

### 4.6 전략을 재컴파일 없이 바꾸는 범위
- **이론가**: 호출 인자. 백테·실전 모두 APL 프로그램(sicadb `replay_apl`)이 계산해 넘긴다.
- **전략 선택·파라미터**: 설정(`[strategy]`)과 `CONTROL SetStrategyParams`. 지금 전략 11종을 그대로 쓴다.
- **새 전략 모양**: `External` 전략 종류를 하나 추가한다(제안). 호출마다 APL이 낸 «원하는 호가 목록(방향·가격·수량)»을 받고, OMS는 지금 걸린 주문과 비교해 신규·취소를 낸다(재고 한도·GTX·rate limit 같은 OMS 규칙은 그대로 적용). 리서치에서 모양을 정한 뒤 성능이 필요하면 OMS 코드 전략으로 옮긴다.
  - 한계: APL이 OMS 내부 상태(내 주문·재고·큐)를 보려면 호출 출력(`QUERY`)을 다음 계산에 넣는 경로가 필요하다. 1단계 External은 «신호 → 원하는 호가»까지만 하고, 재고 반영은 OMS 규칙(limit_qty 등)에 맡긴다.

### 4.7 거래내역
- 지금 `OrderLogger`(QuestDB)의 두 구조체(`SubmitEvent`, `OrderEventRow`) 그대로 sicadb 테이블 `oms.orders`·`oms.order_events`에 쓴다(DLL이 sicadb producer API를 직접 사용, 드문 행은 ADR0049로 20분 안에 파일).
- 백테 호스트는 같은 내용을 기존 APBT 덤프 형식(fills CSV, tp TSV, events TSV)으로도 낸다 → `apbt_report`·`maker_stats`를 수정 없이 쓴다.

---

## 5. 단계 계획

| 단계 | 할 일 | 산출물 | 완료 기준 | 운영 영향 |
|---|---|---|---|---|
| P0 | §4 스펙 확정: op·구조체·상태 코드·설정 | `oms-dll-abi` 크레이트(타입만) + 결정 기록 | 사용자 승인 | 없음 |
| P1 | runner에서 `OmsCore` 분리. runner는 OmsCore를 쓰게 바꾸고 동작은 그대로 | hft-oms_rust 리팩터 | V1 통과 | 프로덕션 코드 변경(동작 불변 증명 후 반영) |
| P2 | `oms-dll`(cdylib): 규약 구현, 페이퍼 모드, `QUERY`·`CONTROL`, 발동 카운터 | DLL + 테스트 | V2·V4 통과 | 없음 |
| P3 | 새 APBT 호스트: sicadb 재생 → 패킷 재조립 → 이론가 APL → DLL(페이퍼) → 기존 형식 덤프. CLI 인자만으로 실행 | 도구(hft_research) | V3 통과, 10-06 이후 날짜 실행 | 없음 |
| P4 | 실전 모드: 커넥터·거래내역 sicadb·market-ingest 호스트·GUI 연결 | DLL 실전 경로 | V5 통과 | 실주문 — 단계마다 사용자 승인 |

---

## 6. 검증 (오라클)

원칙: 경계 데이터는 실환경 바이트로(합성 픽스처 금지), mock은 전송층에만, 회복 경로는 «발동»을 증명, 비교는 «전부 같다» 또는 «다른 이유를 설명».

| # | 무엇과 무엇 | 데이터 | 통과 기준 |
|---|---|---|---|
| **V1** 리팩터 동등성 | 리팩터 전 runner ↔ 후 runner (APBT `analyzer --batch`) | pkt_record 캡처, 여러 심볼(bithumb·binance-futures)·여러 날짜, 전략 여러 종 | fills·events·tp 파일 **바이트 동일** |
| **V2** DLL ↔ runner | 같은 패킷 스트림, 이론가는 runner가 덤프한 값을 고정 주입(OMS만 비교) | V1과 같은 캡처 | 주문 제출·체결·이벤트 **바이트 동일** |
| **V3** 패킷 소스 | sicadb에서 재조립한 패킷 ↔ pkt_record 캡처 | 겹치는 날짜(≤ 2026-10-05) | 패킷 수·패킷마다 best 호가·깊이 대조. 기록 PC가 달라(252/249) 완전 일치는 기대하지 않음 → 차이율 측정·원인 설명. 이어서 같은 설정으로 기존 APBT ↔ 새 APBT 체결·손익 비교 |
| **V4** 규약·장애 경로 | 실제 DLL을 sicadb 호스트로 싣고 시험 | 장애 주입 | create 실패 정리 / start 실패 시 stop_join→destroy / 여러 스레드 동시 호출 / 버퍼 부족이 부작용보다 먼저 / panic → poison / stop_join이 모든 스레드 join — **각 경로의 발동 카운터 > 0** |
| **V5** 실전 | 커넥터 경로 | 실제 거래소 응답 캡처(오류 응답 포함) → testnet(binance-futures) → bithumb 소액 | 응답 처리·reconcile·거래내역이 거래소 조회와 일치. 사용자 승인 후 단계별 |

성능: 릴리스 빌드로 `PACKET` 호출 1회 지연(중앙·p99)과 runner 직접 호출 대비 오버헤드를 잰다. 기준은 P0에서 정한다.

---

## 7. 테스트 방식

- **dev 빌드 `cargo test`**(빠른 반복): OmsCore 단위 동작, ABI 수명주기·장애 주입(V4), 패킷 재조립·호가창 대조(실데이터에서 잘라 만든 작은 픽스처). 릴리스 빌드는 성능 측정과 실데이터 오라클 실행에만 쓴다.
- **오라클 실행**: V1·V2·V3는 예제 바이너리로 실데이터 날짜를 돌려 비교하고, 결과(일치 여부·차이)를 sk에 남긴다.
- **회귀 고정**: V1·V2의 기준 출력(fills·events 해시)을 기록해 두고, 이후 변경마다 다시 대조한다.
- **독립 리뷰**: 단계마다 Opus 독립 리뷰(fresh context)를 받고, 지적은 수정·테스트로 닫는다.
- **실전 전 점검표**: 키 파일 경로·rate limit·KillSwitch 발동·CancelAll 발동을 testnet에서 실제로 일으켜 확인한다.

---

## 8. 위험·미결정

| 항목 | 내용 | 대응 |
|---|---|---|
| 프로덕션 리팩터 | P1은 실전 runner 코드를 바꾼다 | V1 바이트 동일 증명 전에는 실전 반영 안 함. 반영은 사용자 승인 |
| 단일 심볼 OMS | Inventory·Wallet·pending 가격이 단일 심볼용 | 1단계 인스턴스=대상 하나. 같은 계정 지갑 공유는 후속 과제 |
| 이론가 시점 | 실전 구성 ②는 이론가가 그 패킷을 반영한 직후 호출 → runner와 같은 의미. 백테에서도 같은 순서를 지켜야 함 | 백테 호스트는 패킷 반영 → APL 이론가 → DLL 순서를 같은 스레드에서 지킨다 |
| sicadb 패킷 ≠ 캡처 패킷 | 249 캡처와 252 sicadb 기록은 다른 PC | V3에서 차이를 재고 설명. 10-06 이후는 sicadb가 유일한 원천 |
| External 전략의 재고 피드백 | APL이 OMS 상태를 못 봄(1단계) | OMS 규칙(limit_qty)으로 막고, 피드백 경로는 2단계에서 결정 |
| TOM 판정 시점 | 체결 판정이 대상 패킷 시점으로 양자화 | 지금 APBT와 같음(유지). 바꾸면 V1이 깨지므로 별도 결정 |

---

## 9. 검토 요청 (피드백 받을 것)

1. 목적(§1)과 범위(§1.3)가 맞는가.
2. **P1 리팩터 허용 여부**: runner에서 OmsCore를 떼어 runner도 그것을 쓰게 할지(추천, 코드 한 벌), 아니면 runner는 두고 DLL에 복제할지.
3. **1단계 목표**: 페이퍼(새 APBT)까지(P0~P3)로 할지(추천), 실전(P4)까지 한 번에 갈지.
4. **External 전략**(APL이 원하는 호가를 넘김) 추가 여부(추천).
5. op 목록·설정 형식(§4.2~4.3)에 빠진 것.
6. 검증 기준(§6) — 특히 V3의 허용 차이를 어떻게 볼지.
7. 성능 기준(호출 1회 지연 목표).
