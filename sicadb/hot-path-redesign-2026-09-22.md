# 시장 데이터 hot path 재설계: 결정과 계획

작성: 2026-09-22 KST
상태: **결정 기록 + 계획. 구현 미착수.** §7 의 결정은 모두 끝났고, 착수 지시가 오면 §5 순서로 시작한다.
근거 문제: TODO 007 (프로젝트 내부 문서: todo/007-hot-path-apl-per-packet-overhead.md), TODO 008 (프로젝트 내부 문서: todo/008-recording-flush-policy-and-nas-coupling.md)

## 0. 요약

2026-09-22 전체 시장(546 endpoint) 실행에서 패킷당 처리 818µs, p99 0.5초, max 3.2초가 관측됐다. 원본 hft-oms_rust 는 같은 계산을 2.87µs(합성)·5.2µs(실캡처 replay) 에 한다. 원인은 저장소가 아니라 (1) 패킷마다 APL 인터프리터를 도는 조립 방식, (2) 포트당 스레드 546개, (3) 문자열 키 레코드와 문자열화된 숫자, (4) 매초 NAS fsync 와 RAM 회수의 NAS 결합이다.

결정은 다음과 같다. 공유메모리 기반은 유지하고 테이블 종류를 «단일 writer 열 벡터 링» 으로 바꾼다. 모든 쓰기는 append-only 다. OMS 는 전략(이론가)을 포함하지 않는 큰 DLL 로 두고 orders 는 사후 분석용 로그만 남긴다. 스레딩은 코어 수 이하의 샤드로 간다. 문자열은 interned Sym 으로 바꾼다. 계산 모델은 kdb 처럼 **벡터 기반** 이다. 참조 심볼이 바뀌면 의존 심볼 전부를 건너뛰지 않고(strict) 벡터 연산 한 번으로 재계산한다. 구현 순서는 **타입 열 계약(벡터 표현)을 맨 먼저 고정** 하고, 그 위에 shmem 링·APL 값 모델(타입 벡터, Sym, 슬롯 레코드, Rc)·compute 커널을 병렬로 세운 뒤, 실행기 → 샤드·fan-out → recording → OMS DLL 순이다. 패킷 경로의 네이티브 호출 순서표(A)는 10µs 를 위한 수단이지 필수 목표가 아니다.

## 1. 배경: 2026-09-22 실행에서 확인된 사실

### 1.1 실행 이력

| 실행 | 결과 |
|---|---|
| udp-btc-eth-20260922-030109 (8 워커) | 원본 record oracle 과 72/72 row exact 일치. key/value/NaN diff 0 |
| all-smoke-20260922-050811 (546 워커) | 546/546 ready, exit 0. warmup 5s + main 5s |
| all-main-20260922-051602 (546 워커, 60s) | port 55102 `Table(Backlog { batch_index: 0 })` 로 exit 1 |
| all-final-20260922-0553 (546 워커, 60s) | recorder `PermissionDenied (os 5)`, `manifest.txt.tmp` 2개 잔류로 exit 1 |

### 1.2 stage 별 p50 (같은 바이너리, 워커 수만 다름)

| stage | 8 워커 | 546 워커 |
|---|---:|---:|
| receive/decode | 1.0µs | 2.6µs |
| decode→source append | 13.5µs | 21.2µs |
| book event | 17.6µs | 32.5µs |
| native OFI DLL | 5.3µs | 7.1µs |
| native TT DLL | 2.8µs | 4.4µs |
| window aggregate | 47.1µs | 21.5µs |
| feature append/commit | 53.4µs | 111.4µs |
| **APL/host overhead** | **355.6µs** | **601.8µs** |
| total | 513.4µs | 818.1µs |

- 546 워커 실행의 수신 구간 11초 동안 프로세스가 16 논리 코어 중 14~15.6 코어를 계속 사용했다(`run-metadata.json` 의 `process_samples`). 209,508 패킷 / 약 10.5초 / 15 코어 ≈ 패킷당 750µs CPU.
- 7µs 짜리 native OFI 의 max 가 531ms, 4µs 짜리 native TT 가 523ms 등 전 stage max 가 520~590ms 로 균일하다. 코드가 아니라 스레드가 CPU 를 뺏긴 흔적이다. gate_wait max 2.33초는 같은 종목 mutex 를 쥔 스레드가 뺏겨 있는 동안의 대기다.
- 8 워커에서도 p99 6ms / max 37ms 는 전부 gate_wait(같은 종목의 trade·depth 포트가 다른 스레드) 다.
- 같이 돌린 원본 `record_run` 은 혼자 돌 때 udp_drop 0 이었으나 병행 시 25만~31만 개를 떨어뜨렸다.

### 1.3 원인 (코드 기준)

1. **패킷당 APL 실행.** `scripts/matching-first46.apl` 의 `f` 는 host 콜백 약 15회, `Runtime.Get` 약 22회, `Runtime.Dict` 6회를 패킷마다 돈다. 값은 `Value::Record(BTreeMap<String, Value>)`, 배열은 `Vec<Number>`(원소별 enum 태그), `Op::Load` 마다 `value.clone()`, 노드마다 quota tick. 숫자(`capture_id`, timestamp, sequence)는 `Value::Text(to_string())` 로 넘긴다. host 경계마다 레코드를 깊은 복제한다.
2. **포트당 OS 스레드 546개** (`port_workers.rs`) 가 16 코어를 두고 경쟁한다. 같은 종목이 두 포트로 오면 `execution_gate` mutex 로 직렬화된다.
3. **recording.** recorder 가 테이블당 매초 write + fsync 2회를 NAS(SMB) 에 보낸다(`log_flush_after_ms = 1000`). `buffer_seconds = 60` 은 ring 크기일 뿐이다. RAM 회수는 카탈로그가 NAS 에 publish 성공한 범위까지만 진행하므로 NAS 가 느리면 회수 0 → ring 포화 → `Backlog`. seal 은 13 테이블에 한꺼번에 가고 seal 마다 manifest 재작성 + 카탈로그가 13개 manifest 전체 재읽기를 해 SMB rename 오류 5 가 났다.
4. **공유메모리 테이블 append 15~35µs.** batch 하나 = 64KiB 블록 하나, 다중 프로세스·다중 writer 커밋 프로토콜(블록 예약 → 락 아래 index claim → 레코드 인코딩 → 락 아래 commit → named event). 후보 하나당 최대 14개 receipt 를 복사한다.

## 2. 목표

- 패킷당 처리 **10µs 이하** (테이블 게시 포함). 장기 목표는 원본 수준 2.87~5.2µs.
- **이벤트 즉시 처리.** 타이머 배치 없음. 이론가는 패킷마다 갱신되고, 참조하는 심볼로 전파된다. 전파는 중간 상태를 건너뛰지 않는다(strict).
- «컴파일 없이 조립» 은 유지한다. 패킷 경로에서 인터프리터를 빼는 것(A)은 필수 목표가 아니라 수단이다. 10µs 이하가 인터프리터를 남긴 채로 달성되면 그대로 둔다(2026-09-22 사용자 결정).
- OMS 의 다중 writer·락·스레드·상태 전이 의미는 DLL 안에서 그대로 보존한다.
- 저장은 **append-only**, NAS 에 대량 단위로 쓴다.
- hot path 에 문자열이 없다.

## 3. 결정 사항

각 항목은 «결정 / 이유 / 기각한 대안» 순이다.

**D1. 공유메모리 기반은 유지한다.** 같은 PC 의 다른 프로세스(recorder, 전략, GUI, 리서치)가 복사 없이 읽는 것은 kdb 에 없는 장점이고 «구독자는 별도 프로세스», «760GB 워크스테이션에 데이터 적재» 요구에 직접 필요하다. pool·mapping·generation·named event 코드는 문제의 원인이 아니다. 기각: kdb 식 IPC 전용(구독자마다 이벤트마다 복사).

**D2. 시장 데이터 테이블 = 단일 writer 열 벡터 링.** 원본 `Fs2d`(`arr[col*len + row]` col-major + `head`) 를 공유메모리 pool 위에 올리고 head 를 atomic 으로 게시한다. append 는 값 복사 + store 하나. 링 하나 = (종목, family) 하나, writer = 그 종목을 소유한 샤드 스레드. 1,653 종목 × 5 family ≈ 링 8천 개, 각 200KB 안팎. reader 는 [a, b) 를 읽은 뒤 `head − a ≤ capacity` 로 유효성을 검증한다. 락·pin 없음. writer 가 죽으면 head 가 멈출 뿐이다. S1 세그먼트의 `published_end: AtomicU64` 패턴은 참고하되, S1 도 append 하나 = 레코드 하나라 열이 연속이 아니어서 그대로 쓰지 않는다.

**D3. 다중 writer 커밋 프로토콜(S8-a `shared_table.rs`, `table_control.rs`) 과 batch-per-block(`table.rs`) 은 시장 데이터 경로에서 제외한다.** OMS keyed 상태처럼 여러 스레드가 갱신하는 작은 테이블에만 남긴다. 쓰기 패턴이 다르면 테이블 종류도 다르다.

**D4. append-only.** 메모리 링은 append 와 head 전진만. 디스크 세그먼트는 한 번 쓰면 불변. 수정이 필요하면 블록·세그먼트를 새 버전으로 통째로 쓰고 새 세대 포인터가 가리키며 옛것은 GC. `manifest.txt` 재작성은 세대 파일(`manifest-<n>`) 또는 디렉터리 목록 + footer 로 대체한다. 카탈로그 `rev-N.txt` 는 이미 불변이라 유지. 이로써 rename 폭주와 매초 fsync 가 사라지고 ADR0024 (프로젝트 내부 문서: docs/adr/0024-cross-process-metadata-locks-for-smb-publication.md) 의 파일 락은 폐기한다. 기각: bounded retry, `ReplaceFileW`, 전역 I/O 게이트(전부 NAS(SMB) 프로브에서 실패하거나 병목).

**D5. OMS 는 큰 DLL 이며 전략(이론가)을 포함하지 않는다.** «전략» = 이론가 생성(OMS 바깥). «OMS 전략» = 이론가를 받아 주문 생성·관리(DLL 안). 경계 계약은 (종목, 시각, 이론가, 필요 시 신뢰도·상태) 의 **벡터(배치)** 를 넘기는 호출 하나다. 참조 심볼 하나가 바뀌어 의존 심볼 100개의 이론가가 한꺼번에 갱신되면 DLL 호출도 100번이 아니라 한 번이고, «이론가가 임계 이상 움직인 것만» 골라내는 필터는 DLL 안에서 벡터로 한다(D7). OMS 내부 상태(`id_sets → order_map` 중첩 락, `Inventory`·`Wallet`·`IdMap` parking_lot mutex, reconcile·rate limiter 스레드)는 원본 그대로 DLL 안에 있다. orders/fills 는 사후 분석용 append-only 이벤트 로그로만 기록한다(DLL 안 로거 스레드 하나가 writer). 원본 판단 로직을 actor/큐로 옮기는 재작성은 하지 않는다. 주의: matching-engine-dll-apl-composition.md (프로젝트 내부 문서: docs/design/matching-engine-dll-apl-composition.md) 의 «DLL 은 작게» 는 범용 연산에 대한 원칙이고, OMS 는 도메인 상태기계라 예외라는 논리로 문서를 고쳐야 한다.

**D6. 스레딩 = 샤드.** 코어 수 이하(8~16)의 스레드가 각각 종목 묶음을 독점 소유한다. 같은 종목의 모든 포트는 같은 샤드. 샤드는 자기 포트 묶음을 `WSAPoll` 하나로 기다리고 그 자리에서 콜백 처리하며, 같은 루프에서 inbox(D7)를 비운다. 배정은 시작 시 feeding 통계의 포트별 패킷 비율로 정하고 실행 중 고정, 재분배는 재시작. 전체 OS 스레드는 샤드 + OMS DLL 스레드 + recorder + 제어를 합쳐 30개 안쪽. `execution_gate` 는 정의상 사라진다. 기각: 포트당 스레드(546), tokio 태스크(work-stealing 이 종목 친화성과 캐시 지역성을 깨고 reactor 홉이 하나 더 있음).

**D7. cross 전파 = strict + 벡터 fan-out (2026-09-22 결정).** 심볼 B 의 이론가는 B 의 소유 샤드만 쓴다. A 의 패킷이 오면 A 의 샤드는 A 를 참조하는 심볼들이 있는 샤드마다 inbox 에 «A 바뀜» 메시지 하나(MPSC, 수십 ns)를 넣는다. 받은 샤드는 메시지를 **순서대로 전부** 처리하며, 메시지 하나당 자기 의존 심볼 N 개의 이론가를 **벡터 연산 한 번** 으로 다시 쓴다. 의존 심볼은 열 지향 테이블의 행이고 이론가 재계산은 그 열들에 대한 연산이다(kdb 의 `update theo: f[...] from deps where ref=A` 와 같은 모양). A 의 최신값은 A 의 링을 lock-free 로 읽는다. 중간 상태를 합치거나 건너뛰지 않는다. 밀리면 kdb 의 slow subscriber 처럼 inbox 가 길어지고 지연이 늘 뿐 누락은 없다. 과부하 보호(큐 길이 감시, 상한 시 정책)는 별도 결정. 조건: 이론가 함수가 심볼 축으로 벡터화 가능해야 하고(심볼마다 다른 분기가 많으면 안 됨. 원본 cross 는 shape 17개로 수식 4,726개를 덮었다), OMS DLL 은 (심볼, 이론가) 벡터를 한 번에 받아야 한다(D5). 비용 추정: BTC 초당 5,000 패킷 × 샤드 12개 × 벡터 연산 수 µs ≈ 코어 0.3개. 기각: coalescing(최신값으로 한 번만 재계산. 중간 상태를 건너뛰어 사용자가 거부), 참조 심볼 샤드가 의존 심볼 이론가를 직접 쓰는 방식(단일 writer 위반).

**D12. 계산 모델 = 벡터 기반. 타입 열 표현이 첫 계약이다.** 피처·이론가 계산은 «심볼 하나에 스칼라 함수 호출» 이 아니라 «심볼 묶음에 열 연산» 으로 구성한다. 링 테이블(D2)의 열, APL 의 `Value::Array`, compute 커널의 입력이 **같은 타입 열 표현**(`f64`/`i64`/`u32` Sym/bool 버퍼, 정렬, validity, owned 와 링 view 의 구분) 을 공유해야 링을 복사 없이 읽고 fan-out(D7)을 열 연산으로 쓸 수 있다. 그래서 APL 의 타입 벡터화는 «측정 후 결정» 이 아니라 전제이며, 타입 열 계약이 구현 순서의 첫 항목이다(§5.1). `Vec<Number>`(원소별 태그) 를 남긴 채 다른 단계를 측정하면 링 read 마다 태그 붙여 복사하는 변환 비용에 가려진다.

**D8. 발행은 즉시.** append 가 head 를 올리는 순간 소비자가 본다. 타이머 없음. 깨우기는 소비자가 «잔다» 플래그를 세운 경우에만 `SetEvent` 를 부른다(따라오는 동안 syscall 0). 패킷마다 named event 를 쏘던 push 는 폐지.

**D9. 문자열 최소화.** `Value::Sym(u32)` 와 전역 interner(원본 core-types `Registry` 재사용 또는 동일 설계)를 도입한다. 심볼·거래소·컬럼·필드·함수 이름은 attach/compile 때 정수로 바뀌고 hot path 는 정수만 든다. 숫자를 `Text` 로 넘기는 것 금지. 문자열은 설정 로드, 로그, 파일명에만.

**D10. APL.** (a) 값 모델을 타입 열 계약(D12) 위에서 **한 번에** 바꾼다. 타입 벡터, Sym, 슬롯 레코드, Rc 복사 제거가 한 덩어리다. (b) 실행기(quota 분리, 평탄 tape) 는 그 뒤. 벡터화는 결정 사항이고 측정은 각 단계 뒤 확인용이다. (c) 패킷 경로는 attach 때 정한 **네이티브 호출 순서표** 를 직접 탄다(A). APL 은 «이 종목엔 이 함수들을 이 파라미터로 이 순서» 를 한 번 정할 뿐 패킷마다 돌지 않는다. (d) JIT/AOT(Cranelift, rustc 경유)는 보류. 값 모델이 먼저 있어야 코드생성 대상이 생기고, B 만으로 5µs 안팎이 나오며, 패킷마다 스크립트를 sub-µs 로 돌려야 하는 요구가 확정돼야 의미가 있다. 그 전 단계는 원본 tape VM 의 fused kernel 방식이다. (e) «안전하게 격리된 실행» 은 신뢰 스크립트의 목표가 아니다. 격리(quota, catch_unwind)는 외부 스크립트에만 켠다.

**D11. recording 은 TP 로그 방식으로 교체한다.** append-only 불변 파일에 몇 분치 또는 N MB 단위로 한번에 쓴다. rename 없음, 매초 fsync 없음. RAM 회수는 링 크기와 구독자 진행도로만 정하고 NAS 성공 여부와 분리한다. 상세 설계는 별도 문서. SSD 중간층 여부는 §7.

## 4. kdb 대비 (사실과 추정 구분)

| 항목 | kdb (문서 수준의 사실) | 이 설계 |
|---|---|---|
| 값 | 타입 벡터, interned sym, refcount 포인터 전달 | 동일하게 간다 (D9, D12) |
| 인메모리 테이블 쓰기 | 프로세스당 메인 스레드 하나만 쓴다. `peach`·멀티스레드 입력은 읽기 전용(`noupdate`). 락이 없는 게 아니라 동시성이 없다 | 테이블당 writer 하나 (D2). 다른 스레드는 요청을 보낸다 |
| 프로세스 간 | 공유메모리 없음. IPC 로 사본 전달 | shmem 링 직접 읽기 (D1) |
| cross | 필요한 심볼을 전부 구독해 로컬 사본으로 계산. 메시지는 순서대로 전부 처리(건너뛰기 없음). 의존 재계산은 테이블 한 번의 벡터 연산. 밀리면 송신 큐가 쌓임(slow subscriber) | 링 직접 읽기 + inbox, strict, 벡터 fan-out (D7) |
| 배치 | tickerplant `-t` 타이머, `-t 0` = zero-latency 모드 | 타이머 없음 (D8) |
| 인터프리터 | 인터프리터다(공개 범위에서 JIT 없음). 프리미티브가 C 벡터 루프라 해석 비용이 원소에 분산 | 값 모델은 kdb 와 같은 타입 벡터(D12), 실행기는 평탄 tape (D10) |
| 병렬 | 프로세스 여러 개 + 읽기 전용 스레드. «코어당 프로세스 하나» 는 관행이지 규칙이 아님 | 샤드 스레드 (D6) |

## 5. 계획

벡터 기반(D12)이 결정됐으므로 순서는 «타입 열 계약을 먼저 고정하고, 그 위에서 병렬로 세운다» 다. 이전 초안의 «값 모델 → 실행기 → 측정 후 벡터화» 는 폐기했다(2026-09-22).

### 5.0 벤치 하네스 (B0)

8-row fixture 로 첫 46 그래프 `f` 를 반복 실행해 패킷당 µs 와 할당 횟수를 기록한다. 기준선 355~600µs. 이후 모든 단계는 이 벤치로 전후를 비교한다. 대상: market-ingest 테스트.

### 5.1 타입 열 계약 (먼저, 작게)

shmem 링·APL 배열·compute 커널이 공유할 열 표현을 작은 공용 타입으로 고정한다. 자리는 `sicadb-core`(Arrow 독립 숫자 view 크레이트) 가 맞다. 내용: `f64`/`i64`/`u32`(Sym)/bool 열 버퍼, 정렬, validity, owned 버퍼와 링 view 의 구분, Sym interner(원본 core-types `Registry` 재사용 또는 동일 설계). 이 계약이 정해져야 5.2 의 세 작업이 서로 파일을 겹치지 않고 병렬로 갈 수 있다. 완료 조건: 타입·레이아웃 문서 + 단위 테스트, 세 소비자가 참조할 API 시그니처 고정.

### 5.2 병렬 셋 (같은 계약 위, 파일 겹침 없음)

| 작업 | 내용 | 대상 | 완료 조건 |
|---|---|---|---|
| (a) 링 테이블 (D2) | pool 위 col-major 열 + atomic head. writer API(append), reader API(구간 읽기 + generation 검증). 기존 S8-a 테이블은 그대로 둔다 | `sicadb-shmem` 새 모듈 (1~2천 줄) | 단일 writer append ≈ 100ns, reader 복사 0, lapped 검출 테스트 |
| (b) APL 값 모델 (D9, D10a, D12) | **한 번에**: `Value::Array` 를 타입 벡터로(`Vec<Number>` 폐지), `Value::Sym(u32)`, `Record` 를 shape + `Vec<Value>` 슬롯으로(`Runtime.Get`·`Runtime.Dict` 는 컴파일 시 슬롯), 배열·레코드 `Rc`, `Op::Load` 는 Rc 증가, host 콜백은 `&Value` 빌림, 콜백 내부 BTreeMap 생성 제거, 숫자 `Text` 제거, `numeric.rs` 프리미티브를 벡터 루프로(원소별 overflow·finite 검사 제거). 지점 수(2026-09-22): `Value::Record` 53, `Value::Text` 33, `Number::` 111, `Value::Array` 29, host 콜백 6. `Value` enum 모양을 바꾸면 모든 `match` 가 동시에 깨지므로 중간 상태 없이 끝낸다 | `sicadb-apl/src/{value,plan,structured,host,numeric}.rs`, `sicadb-compute-apl`, `market-ingest/.../{matching_apl_host,apl_ingest,apl_workers,matching_graph}.rs` | 회귀 게이트 통과, 패킷당 clone 0, 문자열 할당 0, 링 열을 view 로 읽음 |
| (c) compute 커널 | `sicadb-compute` window aggregate 와 fan-out 용 열 연산을 계약의 열 위에 직접 구현 | `sicadb-compute` | 링 view 입력으로 복사 없이 동작 |

### 5.3 측정 1

B0 벤치로 패킷당 µs·할당·링 read 복사 0 을 확인한다. 회귀 게이트(§6) 통과.

### 5.4 APL 실행기 (D10b)

quota·tick 를 신뢰 모드에서 no-op. 재귀 `eval` → 평탄 tape. 대상 `sicadb-apl/src/plan.rs`. 완료 조건: 회귀 게이트 통과, **패킷당 ≤ 10µs**(host 포함). 원본 tape VM(op 당 1.46ns) 참고.

### 5.5 패킷 경로 네이티브 호출 순서표 (A)

attach 때 APL 이 종목별로 «커널·window·파라미터·순서» 를 결정해 네이티브 노드 목록을 만들고, 패킷은 그 목록을 직접 호출한다. 5개 `Window.Aggregate` 의 config(`window_ref_*`, `duration_ns`, `ncols_use`, `aggr`) 는 종목당 상수로 고정. 5.4 뒤 측정에서 10µs 가 안 나올 때 쓰는 수단이며, 나오면 생략할 수 있다(§2).

### 5.6 샤드 + inbox + 벡터 fan-out (D6, D7, D12)

`port_workers.rs` 를 샤드 루프로 교체. 배정표 생성(feeding 통계 기반), 샤드당 소켓 묶음 `WSAPoll`, inbox MPSC, 코어 affinity. 샤드는 자기 의존 심볼을 열 지향 테이블(심볼 = 행)로 들고, «A 바뀜» 메시지 하나에 그 테이블의 A 의존 행 전체를 5.2(c) 의 열 연산으로 재계산한다. inbox 길이를 관측값으로 남긴다. `matching_*` 모듈은 링 위에서 다시 쓴다(현재 18,000줄 중 상당수가 사라진다). 5.5 를 하는 경우 여기와 한 묶음이다.

### 5.7 recording 교체 (D11)

별도 설계 문서. 이 문서에서는 원칙(append-only, 대량 단위, rename 없음, 회수와 NAS 분리)만 고정한다. 5.6 과 독립.

### 5.8 OMS DLL 경계 (D5)

입력: (종목, 시각, 이론가, 부가 상태) 의 벡터. 출력: 주문·체결 이벤트 로그(append-only 링). DLL 은 자기 스레드·락을 가진다. ABI 는 기존 `sicadb-plugin-abi` 규칙(`#[repr(C)]`, `struct_size`, 버전, host 소유 출력 버퍼, panic 차단) 을 따른다. 5.6 과 독립.

### 5.9 순서

| 순서 | 단계 | 비고 |
|---|---|---|
| 0 | 5.0 B0 벤치 | 기준선 |
| 1 | 5.1 타입 열 계약 | 작고 빠르게. 병렬의 전제 |
| 2 | 5.2 (a)(b)(c) 병렬 | executor 3개, 파일 겹침 없음. 끝나면 build-fixer 한 명이 통합 |
| 3 | 5.3 측정 1 | |
| 4 | 5.4 실행기 | |
| 5 | 측정 2 → 10µs 면 5.5 생략, 아니면 5.5 | |
| 6 | 5.6 샤드 + fan-out (+5.5) | |
| 7 | 5.7 recording, 5.8 OMS DLL | 서로 독립, 6 과도 독립 |

이 순서에서 «측정 후 결정» 게이트는 없다. 측정은 각 단계의 확인이다.

## 6. 회귀 게이트와 측정

- APL 스크립트 문법과 host 함수 이름은 유지한다. `matching-first46.apl` 은 안 바뀐다.
- 기존 테스트 95개(`sicadb-apl` + `sicadb-live-ingest`), replay fixture(`matching-replay-btc-4.json`, `matching-replay-richer-btc-8.json`, `apl_ingest_packets.json`).
- BTC/ETH 실 UDP 8,192 패킷 → 원본 record oracle 72-row exact 비교(`matching-record-oracle`). 덩어리마다 재실행.
- 성능: B0 벤치(패킷당 µs, 할당 수, 링 read 복사 수), capture-off 5+30초 실행의 stage 표, `run-metadata.json` 의 프로세스 CPU 샘플(코어 사용률). 숫자는 심볼·워커 수와 함께 기록하고 평균으로 뭉개지 않는다.

## 7. 착수 전 결정 (2026-09-22, 모두 결정됨)

1. **Codex 미커밋 트리 커밋.** 결정: 커밋한다. 단일 커밋이 아니라 논리 단위로 나눠 커밋한다(shmem / compute·APL bridge / recording 락 실험 / 문서 등). 실험 로그·python 스크립트·백업 파일은 커밋에서 제외한다.
2. **서브에이전트.** 결정: 구현에 Sonnet(Luna) 서브에이전트를 써도 된다. 단, 문제 해결(원인 분석·설계 판단·막힌 곳 뚫기)은 서브에이전트에 맡기면 느리고 토큰을 많이 쓰므로 메인 모델이 직접 더 많이 개입한다.
3. **cross 전파.** 결정: coalescing 하지 않는다. kdb 처럼 메시지를 전부 순서대로 처리하고, 의존 심볼 재계산을 벡터 연산 한 번으로 만들어 비용을 없앤다(D7, D12). 과부하 시 정책(큐 길이 상한·감시)은 별도 결정.
4. **SSD 중간층.** 결정: 중간층 없이 간다. NAS 직접 쓰기를 유지하고 쓰기 패턴(append-only, 대량 단위, rename 없음)만 바꾼다. NAS 쓰기 단위(시간·바이트)는 recording 상세 설계에서 정한다.
5. **이론가 갱신 정책 상세.** 결정: 패킷마다 갱신, 참조 심볼 변화는 의존 심볼 전부에 strict 전파, OMS 에는 벡터로 전달(D5, D7). 남은 것은 이론가 함수를 심볼 축으로 벡터화할 수 있는지의 확인이며, 원본 cross shape 17개를 기준으로 검토한다.
6. **«작은 DLL» 원칙의 OMS 예외.** 결정: 명시한다. matching-engine-dll-apl-composition.md (프로젝트 내부 문서: docs/design/matching-engine-dll-apl-composition.md) 에 반영.

## 8. 관련

- TODO 005 (프로젝트 내부 문서: todo/005-apl-matching-one-second-parity.md): 문제가 드러난 실행
- TODO 007 (프로젝트 내부 문서: todo/007-hot-path-apl-per-packet-overhead.md), TODO 008 (프로젝트 내부 문서: todo/008-recording-flush-policy-and-nas-coupling.md)
- market-ingest 성능 보고서 (프로젝트 내부 문서: market-ingest/docs/reports/matching-live-performance-2026-09-22.md), 검토 보고서 (프로젝트 내부 문서: market-ingest/docs/reports/matching-live-review-2026-09-22.md)
- ADR0022 (프로젝트 내부 문서: docs/adr/0022-generic-window-compute-and-apl-operator.md), ADR0023 (프로젝트 내부 문서: docs/adr/0023-first46-feature-table-storage-and-apl-graph.md), ADR0024 (프로젝트 내부 문서: docs/adr/0024-cross-process-metadata-locks-for-smb-publication.md) (폐기 예정)
- 원본: `hft-oms_rust/crates/feature-engine/src/fs2d.rs`(Fs2d), `hft-oms_rust/crates/core-types/src/lib.rs`(Registry interner), `hft-oms_rust/crates/oms/src/concurrent.rs`(OMS 락 구조), `hft-oms_rust/README.md`(2.87µs / 5.2µs)
