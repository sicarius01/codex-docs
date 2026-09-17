# sicadb — Rust 코어 성능·정확성 계약

작성·갱신: 2026-09-17. **상태: 구현에 적용하는 공개 명세 v0.1. Rust 계산 코어·Arrow/mmap·TCP/worker prototype과 성능 개선을 Windows 11에서 실행했다.** 최신 합격 범위·미해결 회귀는 전체 성능 비교 (프로젝트 내부 문서: performance-sweep-results-2026-09-16.md), 첫 구현 근거는 K0/K1 결과 (프로젝트 내부 문서: k0-k1-results-2026-09-16.md), 작업 순서는 [컴포넌트 README](README.md)를 따른다. 이 문서는 2026-09-17 기준 공개본(v0.1)이며 이전 v0는 Git 이력에 보존한다.

## 1. 결정과 적용 범위

**sicadb 구현 언어는 Rust로 확정한다.** APL은 사용자가 함수를 표현하고 LLM이 프로그램을 작성하는 언어의 첫 검증 후보다. 구현 언어와 사용자 언어는 별개다.

성능은 코어의 완료 조건이다. 저수준 계산 함수(kernel)는 최초 구현부터 데이터 배치, SIMD, 할당, 메모리 읽기·쓰기, 동시성 비용을 검토하고 측정해야 한다. 동작하는 scalar 코드만 작성한 뒤 최적화를 무기한 후속 과제로 넘기지 않는다. 자동 벡터화로 목표를 충족한다면 그것도 최적화된 구현으로 인정하되 생성 코드와 측정 근거가 있어야 한다.

범위는 범용 값·배열·함수 실행, Arrow/mmap, snapshot, 작업 실행·프로세스 통신이다. HFT와 추가 데이터를 같은 기반에서 처리한다. RAM보다 큰 데이터도 정상 입력이다. 모든 데이터를 RAM에 올리는 전제는 없다. 상위 데이터 schema와 수집 책임은 [실행 계획](implementation-plan.md)의 경계를 따른다.

이 문서의 **필수**는 구현·기능 완료 판정의 조건이다. **실측 선택**은 후보를 비교하여 채택할 조건이며, 전부 켜는 것이 요구사항은 아니다. **미검증**은 완료가 아니다. 아래 성능 목표·시험 규칙은 sicadb의 설계 결정이며, 외부 제품이 보장해 주는 성질로 해석하지 않는다.

## 2. 확인한 환경과 아직 확인할 환경

| 항목 | 현재 근거 | 구현·측정 전 필요한 확인 |
|---|---|---|
| 개발 환경 | Windows 11 Pro 10.0.26200, i9-9900K, `rustc 1.93.1`, LLVM 21.1.8, `x86_64-pc-windows-msvc`, `cargo 1.93.1` | 첫 workspace의 toolchain·의존성·빌드 profile 고정 및 실행. 변경 시 관련 correctness/performance 증거 갱신 |
| 주 연구 머신 | `config/infra.toml`의 241, Dell Precision 7920 Tower. 기존 변환 실측 보고서는 Xeon Platinum 8173M × 2, 56물리/112논리 CPU, RAM 약 768GiB로 기록 | 현재 topology·cache·ISA·메모리 채널·SSD·OS build·processor group을 K0 전에 다시 확인. 기존 기록을 새 sicadb 실측으로 표시하지 않음 |
| 영속 입력 | HFT Arrow의 여러 schema와 별도 변환기의 48컬럼 IPC FILE, data-service의 IPC FILE 출력이 존재 | 파일/스키마별 alignment·압축·batch·validity·정렬·실제 보유 범위. writer별 실물은 §12.3에서 구분 |
| 운영 실행 환경 | 사용자 확정: 메인 PC·241은 Windows 11, feeding·일일 batch PC는 Windows 10 | Windows 11을 주대상으로 검증하고 Windows 10은 별도 컴파일을 허용. 실제 Windows 10 실행·mmap·IPC·복구 검증 전 호환 완료라고 표시하지 않음 |

이번 문서 작성에서는 241 전원·설정 변경, NAS 대량 읽기, 하드웨어 실측을 하지 않았다. 하드웨어 수치는 [기존 NAS 변환 실측 보고서](https://github.com/sicarius01/codex-docs/blob/main/quant-research/nas-arrow-conversion-benchmark-2026-09-16.md)의 기록이다. Windows의 많은 논리 CPU는 processor group과 affinity API 동작까지 확인한다. 초기 target은 x86_64/MSVC이며 CPU 기능은 별도 검출한다. Windows 11 전용 API가 필요하면 Windows 10 대체 경로나 기능별 미지원 상태를 명시한다. Linux는 초기 운영 합격 대상에 넣지 않되 OS 경계를 분리한다. [Microsoft processor groups](https://learn.microsoft.com/en-us/windows/win32/procthread/processor-groups)

## 3. 코어의 필수 불변조건

| ID | 계약 | 합격 증거 |
|---|---|---|
| C01 | 정수·시각·null·오류 의미를 최적화가 바꾸지 않는다 | 독립 정답 및 scalar/최적화 경로 비교 |
| C02 | 지원하는 비압축 primitive Arrow 읽기에서 입력 buffer를 통째로 복제하지 않는다 | backing buffer 참조와 수명 검증, 입력 복사 bytes 계측 |
| C03 | 준비 완료 후 순수 kernel의 inner loop는 heap 할당·lock·시스템 호출·행별 reference count·문자열 로그가 0이다 | 할당 계측·코드 점검·프로파일. 준비 비용은 별도 포함 보고 |
| C04 | 출력과 scratch의 크기·소유자를 호출 전에 설명할 수 있다 | caller 제공 buffer 또는 사전 reserve, 예산 초과 시험 |
| C05 | 작업·queue·cache·상태·spill에 유한한 자원 예산이 있다 | 과부하·높은 cardinality·느린 consumer·취소 시험 |
| C06 | SIMD 적용 가능한 핵심 kernel은 첫 완료 시 벡터화 후보를 검증한다 | dispatch 표·생성 코드·경계 시험·성능 비교 |
| C07 | `unsafe`는 좁은 내부 경계에 모으고 safe API가 전제조건을 보장한다 | 각 unsafe의 safety 근거·독립 검토·적합한 동적 검사 |
| C08 | mmap backing file과 view 수명이 일치하고 읽는 파일은 불변이다 | 동시 reader·회수·재시작·잘못된 입력 시험 |
| C09 | 접수·live 공개·내구 저장·영속 snapshot 공개를 구분한다 | 장애 지점별 복구·중복 재전송·회복 경로 발동 증거 |
| C10 | 성능 주장은 동일 의미·명시된 환경·재현 가능한 입력에서 측정한다 | 원시 측정값·버전·설정·비교 기준·변동성 보고 |
| C11 | 오류·취소로 끝난 부분 출력을 정상 결과로 공개하지 않는다 | 중간 실패·초기화 범위·안전한 drop·후속 buffer 재사용 검사 |
| C12 | 단위·scale·epoch·해상도를 검증하고 연산 의미에 포함한다 | 서로 다른 단위·scale·시각 기준 입력의 거부 또는 명시적 변환 시험 |
| C13 | 외부 경계의 실물 입력과 장애 경로 발동을 검증한다 | writer별 실물 fixture·최하층 장애 주입·경로 trace/counter·최종 상태 |
| C14 | 작업과 회복 상태의 진행을 유한한 시간 안에 판정한다 | 시나리오별 deadline·진행 감시·실패/격리/알림 전이 시험 |
| C15 | 유효한 최신 live 데이터의 공개가 디스크 영속화 완료를 기다리지 않는다 | disk 지연·sync 실패·recording queue 포화 중 live 반응 및 기록 지연/공백 표시·허위 durable ACK 없음 |

`C03`은 순수 계산 kernel 계약이다. 파일 열기·schema 검증·결과 준비·새 그룹 삽입·작업 등록까지 모두 무할당이라고 주장하지 않는다. 이 비용은 숨기지 않고 query 전체 측정에 포함한다. `C02`도 압축 해제·형변환·필터 결과 생성까지 무복사라는 뜻은 아니다.

아래 필수 조항의 검증 결과는 이 ID와 해당 절에 연결한다. §4·§9는 C02/C04/C07/C08, §5는 C01/C11/C12, §6~§8·§13은 C03~C06/C10, §10~§12는 C05/C08/C09/C11/C13~C15의 증거를 남긴다. 실측으로 선택할 후보와 아직 구현하지 않은 기능은 각각 채택 근거와 검증 누락으로 기록한다. 모든 세부 문장에 별도 ID를 늘리는 대신 kernel·기능 완료표에서 적용 ID와 제외 사유를 추적한다.

## 4. 값·배열 표현과 소유권

### 4.1 행마다 객체를 만들지 않는 실행

- 수치 열은 연속 primitive buffer와 별도 validity bitmap을 기본으로 한다. 일반 함수의 값 표현과 수백만 행의 실제 저장 표현을 분리한다.
- runtime에서 타입·연산·실행 경로를 한 번 결정하고 typed kernel에 batch를 전달한다. hot loop 안에서 행마다 동적 타입 판정·가상 함수 호출을 반복하지 않는다.
- 배열 view는 backing owner, offset, length, 실제 alignment, validity bit offset을 보존한다. chunked 배열을 지원하고 전체 입력의 연속화를 요구하지 않는다. 주소가 인접하더라도 별개 allocation/mapping들을 하나의 Rust slice로 합치지 않는다. [Rust slice 생성 조건](https://doc.rust-lang.org/std/slice/fn.from_raw_parts.html)
- 함수는 동일한 의미로 owned 배열과 mapped 배열을 받는다. 배열을 언어 heap으로 모두 복사해야만 호출 가능한 연결은 이 계약에 미달한다.
- symbol/string은 dictionary/interning을 검토하되 dictionary ID의 범위를 snapshot·dictionary version과 묶는다. 서로 다른 dictionary의 같은 정수 ID를 같은 값으로 취급하지 않는다.
- 실행 중 자주 읽는 metadata와 드문 오류·디버깅 정보를 분리한다. packed struct나 작은 값 표현을 채택할 때 alignment와 분기 비용을 함께 확인한다.

### 4.2 소유권과 쓰기

- mapped 입력은 불변이다. 결과는 호출자가 제공한 출력 또는 작업 소유 buffer에 쓴다. in-place 연산은 유일한 소유권과 alias 조건이 입증된 별도 경로다.
- view를 유지하는 동안 mapping·파일 세대를 회수할 수 없다. 행마다 `Arc`를 clone하지 않고 배열·batch·작업 경계에서 owner를 유지한다.
- slice로 작은 부분만 남겨도 큰 mapping 전체를 붙잡을 수 있다. retained backing bytes와 실제 view bytes를 따로 계측하고, 오래 유지할 작은 결과는 예산 내 명시적 복사를 허용한다.
- 프로세스 사이에는 Rust 포인터·`Vec` 내부 구조·기본 Rust ABI를 전달하지 않는다. version이 있는 descriptor와 segment/snapshot 식별자·검증된 offset을 전달한다.
- 새 원시 buffer는 초기화 완료한 범위만 읽거나 공개한다. capacity를 length로 바꾸는 최적화는 부분 실패·panic·취소 시의 초기화 상태까지 검증한다.

Rust의 alias·alignment·초기화·data race 규칙은 `unsafe` 안에서도 지켜야 한다. 성능 측정에서 빨랐다는 사실이 안전성 근거를 대신하지 않는다. [Rust undefined behavior](https://doc.rust-lang.org/reference/behavior-considered-undefined.html)

## 5. 수치 의미를 먼저 고정

각 연산은 구현 전에 입력/출력 타입, promotion, overflow, null/NaN, 빈 입력, 오류, 정렬 의미를 정한다. Arrow나 APL이 제공하는 이름만으로 이 의미가 자동 합의되지는 않는다.

| 영역 | v0.1 원칙과 시험 |
|---|---|
| Int64·timestamp·scaled integer | float로 우회하지 않는다. elementwise 산술은 명시적 checked 의미가 기본이다. wrapping/saturating은 이름과 계약이 다른 연산으로만 제공 |
| scaled 곱셈·rescale | i64 입력 곱셈은 i128 중간값을 사용한다. 결과 단위·출력 scale·rescale 순서·중간/최종 범위 검사를 연산별로 고정한다. 추가 배율 계산도 checked로 처리. 기본 exact 변환은 나누어떨어지지 않으면 오류이며 반올림 연산은 별도 rounding mode를 명시 |
| 정수 reduce | 반환 타입과 누산 타입, 중간/최종 overflow 시점을 연산별로 고정. 넓은 누산기 등으로 재배열 후 동일 의미를 지킬 수 있어야 SIMD/병렬화 허용 |
| 오류 시 출력 | kernel은 연산 전체에 `Err`를 반환하고 부분 결과를 정상 배열로 공개하지 않는다. caller buffer는 일부 수정될 수 있으나 초기화 범위·소유권·drop은 항상 안전해야 한다. 원자적 출력이 필요한 wrapper는 scratch에 계산 후 commit하며 추가 bytes/복사 비용을 보고 |
| validity | null과 NaN, 실제 0을 구분. null lane이 유효 결과·오류·관측 가능한 FP 의미에 영향을 주지 않아야 함. 초기화된 lane의 안전한 비트 연산/산술 후 mask 적용은 허용하되 분모·인덱스·trap 가능 연산은 실행 전에 안전하게 처리 |
| FP elementwise | NaN·무한대·부호 있는 0·subnormal·나눗셈을 명시. FMA, reciprocal 근사, FTZ/DAZ, fast-math는 결과를 바꿀 수 있는 선택으로 취급 |
| FP reduce | 단순 직렬 합을 무조건 유일한 정답으로 삼지 않는다. 기본 재현 모드는 고정된 논리적 분할·누산·결합 규칙을 정해 I/O chunk·worker 수·ISA 선택이 결과 순서를 임의 변경하지 못하게 함 |
| FP 검증 | 고정 환경의 재현성과 수학적 오차를 따로 검사. 독립 고정밀 정답과 연산별 절대/상대/ULP 기준을 사전 결정. NaN 분류와 signed zero 규칙도 별도 검사 |
| order·group·join | 동일 key/time tie, hash 순회 순서, 정렬 안정성, 경계 포함 여부를 명시. dictionary 순서가 사용자 정렬 순서를 대신하지 않음 |
| 상태·시간 | window warm-up/close, late/correction, event/available time과 코드 버전을 고정. 같은 함수 이름만으로 live/replay 동일성을 주장하지 않음 |

단위·scale·epoch·시간 해상도는 검증된 열 metadata와 실행 계획의 일부다. 동적 Arrow schema를 컴파일 시점 타입 검사만으로 모두 검증했다고 주장하지 않는다. 경계에서 검증한 descriptor를 typed kernel로 전달하고, 가격·수량·OI라는 이름만으로 scale을 추정하지 않는다. 기존 HFT OI의 ×1e4와 현재 data-service 일부 OI의 ×1e8은 각각의 dataset 계약을 따른다. 다른 scale의 산술은 명시적 변환을 거치며 필요한 정밀도를 Int64로 표현할 수 없으면 넓은 타입을 사용하거나 오류로 거부한다. 원천 Float64를 이미 정확한 scaled integer였던 것처럼 바꾸지 않는다. 시각은 단위·epoch가 정해진 UTC 정수를 기본으로 하고, KST 표시·파티션 규칙은 별도다. 원천 해상도를 높여 의미상 정밀도가 생긴 것처럼 표시하지 않으며 윤초 등 지원하지 않는 의미는 거부/미지원으로 명시한다.

checked 의미와 SIMD 구현 방식은 구분한다. checked loop의 자동 벡터화 가능 여부는 고정 compiler의 생성 코드로 판단한다. overflow mask를 모아 batch 끝에서 오류를 반환하는 후보도 허용하지만 null lane의 가짜 overflow를 제외하고 C11을 지켜야 한다. 최초 오류 위치를 반환하는 연산은 그 순서까지 보존해야 하며 단순 `Err` 계약과 섞지 않는다.

전 플랫폼·전 compiler·전 수학 라이브러리의 FP 비트 동일성을 처음부터 약속하지 않는다. 지원·검증한 범위를 기록한다. 처리량 우선의 재배열 모드가 필요하면 별도 opt-in 계약과 결과 metadata를 마련하고 기본 의미를 조용히 바꾸지 않는다. APL 프로필을 정할 때 이 경계와 일치 여부를 검증한다.

FP 수치 프로필에는 **backend끼리의 재현성 비교자**와 **고정밀 oracle 대비 정확도 기준**을 별도 필드로 둔다. 기본 산술·재현 reduce는 명시된 지원 환경 안에서 결과 비트 동일성을 기본으로 하되, NaN payload 비교 제외 여부·signed-zero 규칙·FMA·FTZ/DAZ 정책을 프로필 버전과 함께 고정한다. 비트 재현을 제공하지 않는 함수는 별도 동등성/오차 규칙을 명시하고 그 보장 범위로만 표시한다. 측정 결과를 본 뒤 비교자를 완화하지 않는다.

정수의 `MIN / -1`, 부호 반전, shift 범위, scale·epoch 변환도 경계 시험에 포함한다. `exp`·`sin` 같은 초월함수는 기본 산술/고정 reduce와 별도 정확도 계약이 필요하다. 표준 라이브러리도 일부 함수의 정밀도를 고정하지 않으므로 모든 FP 함수를 비트 동일성 하나로 채점하지 않는다. [Rust 정수 overflow](https://doc.rust-lang.org/reference/expressions/operator-expr.html#overflow), [Rust f64 수학 함수](https://doc.rust-lang.org/std/primitive.f64.html)

## 6. SIMD와 CPU 실행

### 6.1 필수 경로

1. 독립 정답과 비교할 명료한 reference 구현을 둔다.
2. 연속 배열·명확한 alias 조건·loop 범위로 compiler의 자동 벡터화를 우선 활용한다.
3. 핵심 kernel의 assembly/최적화 결과를 확인한다. SIMD 명령이 있다는 사실만으로 개선을 인정하지 않는다.
4. compiler 경로가 불충분하면 `std::arch` 등의 대상별 intrinsic으로 개선한다. scalar/baseline fallback은 유지한다.
5. CPU/OS에서 실제로 사용할 수 있는 기능을 검사한 뒤 해당 함수를 선택한다. dispatch는 행별이 아닌 연산/batch 경계에서 수행한다.
6. 입력 길이·null 밀도·선택률·캐시 상태에 따른 이득을 측정하고 작은 입력 경로도 둔다.

지원하지 않는 `target_feature` 코드 실행은 허용하지 않는다. 테스트의 강제 backend 선택도 검출을 우회하지 않는다. unsupported 경로는 거부/skip 상태와 실제 검증 누락을 표시한다. baseline 배포 빌드에 전역 `target-cpu=native`를 넣고 이식 가능하다고 보고하지 않는다. [Rust SIMD와 동적 기능 검출](https://doc.rust-lang.org/std/arch/index.html)

reference 소스가 scalar 형태여도 compiler가 벡터화할 수 있다. 성능 비교에서 실제 scalar 기계어라고 표시하려면 그 빌드의 생성 코드를 확인한다. 참조 의미 검증과 scalar 성능 비교는 별개다.

### 6.2 반드시 검사할 경계

- 길이 0/1, SIMD lane 수의 앞뒤, 여러 lane 폭의 배수와 tail.
- 자연 정렬된 입력, offset slice, SIMD 경계 비정렬, validity의 모든 bit offset.
- cache line/page/file 마지막 경계. 다른 page가 접근 가능할 것이라 가정한 과잉 읽기 금지.
- all-valid/all-null/혼합 bitmap, 선택률 0/희소/절반/전부, 무작위·군집 분포.
- int min/max, overflow, NaN·infinity·signed zero·subnormal.
- 서로 겹치는 입력과 출력의 허용/금지 조건, 출력 뒤 canary, 미초기화 영역 읽기.

Arrow가 권장하는 정렬과 padding은 임의 파일·slice에 대한 무조건적인 SIMD load 허가가 아니다. 실제 buffer 경계·정렬을 검증하고 안전한 unaligned/tail 경로를 제공한다. 압축된 buffer는 먼저 해제해야 한다. [Arrow 물리 배치](https://arrow.apache.org/docs/format/Columnar.html)

### 6.3 실측으로 선택할 것

AVX2/AVX-512 등 ISA 폭, unroll, 여러 accumulator, gather/scatter, mask/branch, prefetch, non-temporal store를 비교한다. 가장 넓은 SIMD가 늘 가장 빠르다고 가정하지 않는다. 작은 batch의 dispatch 비용, 메모리 대역폭, 주파수·동시 worker 영향까지 포함한다. 읽기 전용 scan과 큰 결과를 쓰는 연산의 최적 경로는 다를 수 있다.

현재 공식 문서의 `std::simd`는 nightly 실험 API다. 초기 운영 코어는 고정 stable Rust와 대상별 경로를 기본 후보로 두고, nightly 의존을 암묵적으로 추가하지 않는다. 사용 시 toolchain·장애 대응·대체 경로를 별도로 기록한다. [Portable SIMD 상태](https://doc.rust-lang.org/std/simd/index.html)

## 7. cache line·메모리 대역폭·NUMA

### 필수 설계

- 필요한 열·기간을 먼저 줄인다. scan한 bytes와 실제 결과에 필요한 bytes를 구분한다.
- 연속 scan과 cache에 맞는 block 처리를 기본으로 검토한다. 필터/산술/집계의 불필요한 중간 배열을 줄이되 fusion으로 오류·FP·null 의미를 바꾸지 않는다.
- worker별 출력 영역·누산기·자주 쓰는 counter를 분리한다. 공유 cache line을 번갈아 수정하는 false sharing을 측정한다.
- 공용 hot counter의 행별 atomic 갱신을 피하고 local 합산 후 batch 단위로 전달한다.
- 한 작업의 모든 worker가 전체 입력을 다시 읽지 않게 partition한다. scheduler와 라이브러리 thread pool의 중첩으로 core 수가 폭증하지 않게 한다.
- 평균 처리량과 함께 working set, 읽기·쓰기량, LLC/TLB miss, branch miss, page fault, 실제 memory bandwidth를 가능한 계측 도구로 기록한다. 하드웨어 counter가 없으면 미측정이라고 쓴다.

### 하드웨어별 선택

| 후보 | 도입 조건과 비교 |
|---|---|
| cache line padding/alignment | 실제 대상 layout과 경합을 확인. 모든 객체에 64/128바이트 padding을 넣어 working set을 키우지 않음 |
| NUMA별 partition·scratch·worker | 실제 node topology와 메모리 위치 확인. local/remote 접근·한 socket/여러 socket·first-touch 비교 |
| affinity·SMT | 처리량·p99 지연·다른 역할 간 간섭을 함께 측정. 논리 core가 늘수록 빠르다고 가정하지 않음 |
| huge/large pages | TLB 이득과 할당·회수·권한·메모리 압박 비용 비교. 일반 파일 mmap과 같은 방식으로 된다고 가정하지 않음 |
| software prefetch | 접근 패턴·거리·worker 수별 이득 확인. cold storage 지연을 사라지게 하는 기능으로 보지 않음 |
| non-temporal store | 큰 연속 출력의 후속 재사용·정렬·가시성까지 검토. 즉시 읽을 작은 결과에는 기본 적용하지 않음 |
| code specialization | dtype/null/ISA별 전문화와 I-cache·binary 크기·시작 시간의 균형. 과도한 monomorphization·inline 제한 |

읽기 대역폭이나 SSD I/O가 병목이면 명령어 수 감소가 전체 시간을 줄이지 못할 수 있다. kernel 자체의 계산 상한과 실제 end-to-end 병목을 따로 보고한다.

NUMA의 preferred node는 물리 배치의 절대 보장이 아니므로 실제 위치·원격 접근을 확인한다. Windows `SEC_LARGE_PAGES`를 일반 Arrow 데이터 파일의 mmap 옵션으로 붙일 수 있다고 가정하지 않는다. [Windows NUMA](https://learn.microsoft.com/en-us/windows/win32/procthread/numa-support), [CreateFileMapping의 large-page 조건](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-createfilemappinga)

## 8. 할당·예산·cache 정책

### 8.1 할당 계약

- 순수 kernel은 caller가 제공한 출력/scratch를 사용한다. resize·reserve·성장·drop 비용은 명시된 wrapper 경계로 옮긴다.
- arena/pool은 query 또는 worker 수명과 상한을 갖는다. arena를 썼다는 이유로 자원 해제·고수위 유지·원격 thread free 문제를 무시하지 않는다.
- 대체 allocator는 기본 allocator와 실제 작은 객체·큰 buffer·다중 thread·해제 패턴에서 비교한다. 처음부터 전역 교체를 정답으로 삼지 않는다.
- 메모리 예산에는 입력 복사, 출력, 임시 배열, group/hash table, 정렬 상태, queue payload, cache pin을 구분해 반영한다. 공유 backing을 중복 합산한 RSS와 실제 private 사용량을 구별한다.
- 예산은 query 단위뿐 아니라 모든 worker·동시 작업의 합에도 적용한다. admission 단계에서 여유 자원을 예약하고, query 하나가 통과했다는 이유로 동일 크기 작업을 무제한 병렬 접수하지 않는다. pool·allocator 잔류량과 thread stack 같은 런타임 비용도 계측한다.
- 지원하는 out-of-core 연산은 예산에 맞게 chunk/spill하고, 아직 지원하지 않는 연산은 입력 전체 할당을 시도하기 전에 명시적으로 거부한다. 예상치보다 커지는 고유 key 수와 skew를 시험한다.
- spill도 disk bytes·file 수·동시 I/O 상한을 가진다. disk full·취소·재시작 시 임시 데이터 회수 조건을 정한다.

### 8.2 cache는 세 층으로 구분

| 층 | 역할 | 필수 정책 |
|---|---|---|
| CPU cache | 실행 중 데이터·명령 접근 | layout·block·false sharing·측정으로 대응 |
| OS page cache와 로컬 파일 staging | 영속 파일 재사용 | 동일 데이터를 불필요하게 또 heap에 cache하지 않음. SSD staging도 용량·원본 버전·무결성·퇴거 조건 보유 |
| 실행 계획·중간값·결과 cache | 반복 계산 절약 | snapshot, schema/dictionary, 함수/코드, 인수, 수치 모드 등 실제 의존성을 key에 포함. byte 상한·eviction·동시 miss 중복 실행 억제 |

요청한 snapshot·정정 세대와 다른 결과를 최신 결과인 것처럼 돌려주거나, 아직 준비되지 않은 결과를 공개하면 cache 성능 개선은 무효다. 사용자가 명시한 과거 snapshot의 결과와 §10.4의 마지막 완료 추론값은 각각의 snapshot·입력 시각·나이를 표시해 제공할 수 있다. 오래 유지한 snapshot pin은 명시적으로 제한/거부/취소 정책을 적용하며, 참조 중인 파일을 몰래 회수하지 않는다. 실행 시간·임의성·외부 입출력에 의존하는 함수를 순수 함수처럼 결과 cache하지 않는다.

kernel 비교에서는 결과 cache를 끄고, 사용자 반복 작업에서는 cache 효과와 비용을 별도로 측정한다.

알고리즘 선택에도 비용 모델을 둔다. 예를 들어 길이 `n`의 packed selection bitmap은 약 `ceil(n/8)` bytes, 64비트 행 인덱스는 선택 행 수 `k`에 대해 약 `8k` bytes이며 owner·정렬 padding 등은 별도다. 희소/밀집 선택, 정렬된 key의 merge와 hash, 사전 집계와 전체 materialization을 비교한다. hash 충돌·skew·rehash가 입력 복사와 예산 폭증으로 이어지지 않도록 검사한다. SIMD가 비효율적인 알고리즘 선택을 만회할 것이라고 가정하지 않는다.

## 9. mmap·영속 파일·대용량 I/O

현재 기본은 불변 FILE과 필요 범위의 RAM 준비이며 NAS→shared RAM과 선택 HDD cache를 구분한다. SSD 필수는 아니다. 기존 local SSD 측정은 역사적 결과로만 보존한다.



- 초기 mmap 지원 대상은 로컬의 검증된 불변 sealed segment이며 backing device는 HDD 또는 SSD로 선택한다. NAS/SMB 자료는 필요한 범위를 RAM으로 준비해 사용한다. NAS 직접 mmap은 초기 지원 범위 밖이며, 운영체제에서 불가능하다는 주장은 아니다.
- mmap은 주소 공간에 파일을 연결하는 방식이다. 파일 open, metadata 검증, page fault, storage read, 계산, 결과 전달을 별도로 측정한다.
- read-only mapping만으로 다른 프로세스의 파일 수정·truncate가 차단되는 것은 아니다. sicadb가 관리하는 불변 segment의 수명·쓰기 권한을 보장한다. 외부 원본은 불변성이 보장된 등록 또는 관리 영역의 검증된 import로 연결한다.
- Windows에서는 writer·reader·회수자별 open access/share mode와 유지할 handle 수명을 고정하고 실제로 충돌 open·write·truncate·rename·delete를 시험한다. 쓰기/삭제 공유를 허용하지 않는 reader open은 OS 보호 수단으로 검토하되 snapshot pin·세대별 참조·재시작 복구를 대체하지 않는다. 배타 open 실패만으로 reader 생존이나 모든 접근 종료를 단정하지 않는다.
- 검사 시점의 checksum만으로 이후 불변성을 증명하지 않는다. writer→seal→publish 이후에는 같은 파일을 덮어쓰지 않는다. reader lease와 세대별 회수 조건을 구현한다.
- lease/heartbeat 만료는 실행 중인 reader의 접근 종료 증거가 아니다. 일시 정지한 reader가 다시 실행될 수 있는 동안 mapping·backing file·공유 영역을 변경하거나 재사용하지 않는다. 종료/접근 차단이 확인된 뒤 회수한다.
- map 전체를 미리 touch하거나 `mlock`/working-set 고정으로 전 데이터 상주를 요구하지 않는다. map window·readahead·prefetch·I/O queue는 접근 패턴과 운영체제별 실측 선택이다.
- OS의 map offset 단위와 Arrow/SIMD buffer alignment를 구분한다. Windows allocation granularity를 확인해 mapping 시작 offset과 실제 view offset을 각각 검증한다. 라이브러리가 offset을 보정해도 view의 범위·정렬 검증을 생략하지 않는다.
- 파일 수·open handle·mapped region·page table·metadata cache도 자원이다. 날짜×종목 단순 분할로 소파일을 대량 생성하기 전에 scan·pruning·동시 조회를 비교한다.
- column projection이 항상 물리 디스크의 정확히 그 bytes만 읽는다는 뜻은 아니다. page granularity·IPC batch 배치로 인한 read amplification을 계측한다.
- 압축 IPC/차가운 저장 계층은 공간·I/O 절감과 decode/복사 비용을 별도로 측정한다. 비압축 직접 참조 경로와 혼동하지 않는다.
- NAS에서 필요한 범위를 RAM으로 준비하는 시간, 첫 읽기, 반복 읽기를 각각 표시한다. backing device가 HDD인지 SSD인지도 구분한다. 대량 이동은 프로젝트의 batch 전송 규칙 (프로젝트 내부 문서: nas_sync.md)을 따른다. mmap이나 SIMD가 NAS 대역폭 제한을 없애지는 않는다.

file-backed mmap의 안전성은 파일이 외부에서 변경될 수 있다는 조건까지 다룬다. `memmap2`도 이 때문에 file-backed 생성 API의 unsafe 조건을 설명한다. 구체 crate/version과 Arrow buffer 연결 API는 구현 시 고정·검증한다. [memmap2 safety](https://docs.rs/memmap2/latest/memmap2/struct.MmapOptions.html)

Windows 11과 Windows 10에서 위 수명·공유 모드·offset·재시작 시험을 각각 실행한다. 파일 mapping 생성자와 원본 파일 handle의 수명이 항상 같다고 가정하지 말고 선택한 crate의 실제 handle 유지 방식까지 확인한다. mmap의 쓰기 가시성과 별도 파일 I/O의 일관성도 무조건적인 즉시 반영으로 가정하지 않는다.

**작업 메모리 예산보다 큰 입력 시험과, 실제 물리 RAM/파일 cache를 초과하는 시험은 별개다.** 첫 시험은 제한된 scratch로 계산하는지 보여 준다. RAM 초과 성능은 별도 격리된 큰 입력/제약 환경에서 page fault·storage bytes·시스템 전체 메모리를 관찰해야 한다.

## 10. 공유 메모리·동시성·다중 프로세스·비동기 실행

### 10.1 기본 구조

일반 live writer 문구는 시세·feature·1초 append 경로에만 적용한다. OMS는 다중 writer와 lock 상태 갱신을 유지한다.

- 시세·feature·1초 적재의 main process writer와 불변 snapshot reader를 기본으로 한다. 연구 worker와 실시간 경로의 실행·메모리·queue 예산을 분리한다. OMS는 다중 writer와 lock 상태 갱신을 유지한다.
- mutable state는 가능한 한 owner 안에 둔다. 여러 thread의 결과는 local 집계 후 명시된 순서로 결합한다.
- async I/O와 CPU 작업을 분리한다. 긴 kernel이나 page fault 가능성이 큰 분석을 실시간 event loop에서 직접 실행하지 않는다.
- queue는 항목 수와 byte 수에 상한을 둔다. queue full일 때 대기·거부·재시도 중 무엇을 하는지 정하고, 상태를 조용히 버리거나 순서를 바꾸지 않는다.
- 배열 실행의 취소는 bounded chunk/batch 경계에서 확인한다. **취소 요청 → 접근 종료 확인 또는 worker 프로세스 종료 확인 → 출력·scratch·snapshot pin 회수** 순서를 지킨다. deadline 초과/취소 요청을 실제 실행 종료와 혼동하지 않으며, 이미 durable해진 작업의 ack 의미를 유지한다.
- 범용 evaluator의 사용자 반복·재귀·함수 호출에는 별도의 실행 예산·stack 한도·취소 확인 지점을 둔다. 오래 걸리는 I/O에는 timeout/취소 정책을 두고, 협조하지 않는 native 호출은 지원 제한 또는 격리 worker 종료로 처리한다. 배열 chunk만으로 모든 사용자 프로그램의 취소가 가능하다고 주장하지 않는다.
- lock-free는 기본 의무가 아니다. mutex/channel/owner 분리가 요구를 만족하면 채택한다. 경합 때문에 변경할 때만 atomic ordering·ABA·재활용·안전한 reclamation을 함께 명세화한다.

### 10.2 shared memory와 IPC

같은 호스트 live 경로의 기본은 shared memory다. 아래 TCP application protocol 설명은 원격 호스트와 선택적 제어·호환 경로에만 적용한다. direct read와 push는 같은 table/query 의미를 사용하며, push는 payload 복제 없이 sequence 범위와 segment offset/generation을 알린다. OMS의 다중 writer·lock 상태 갱신은 이 일반 IPC 원칙의 예외로 보존한다.

같은 호스트의 live 통신은 공유 메모리가 기본이다. direct read와 push 구독을 같은 table/query API로 제공한다. push는 payload 복제 대신 row sequence 범위, segment offset, generation을 알리고 소비자 executor가 callback을 수행한다. TCP는 원격 호스트와 선택적 제어·호환 경로이며 frame을 직렬화한다. 각 프로세스의 가상 주소가 같을 필요는 없다. 전체 DB가 하나의 물리 RAM이나 중앙 CPU라는 뜻은 아니다.

공유 ring/segment는 column별 독립 회전 없이 공통 row sequence의 columnar batch로 구성한다. `published_end`(exclusive), reader done, active pin, time retention, durable position은 별도다. slot 초기화→모든 column/validity/dictionary/variable 영역 쓰기→Release publish→Acquire observe 순서를 지킨다. sequence wrap, generation, offset 길이와 resize를 검사하며 고정 N을 바꾸어 modulo를 재해석하지 않는다. Rust reference·Vec·std mutex·함수 포인터 callback은 shared layout에 저장하지 않는다. callback은 소비자 executor가 수행하고 cross-process 완료는 done/ACK로 알린다. inter-process atomic/layout 정렬은 실제 Windows OS·ABI·구현 보장을 확인한다.

공유 pool은 고정 page/segment 할당·반환과 bounded 예산을 갖는다. 초기 mapping·commit·touch를 별도로 계측하되 물리 pinned 보장은 주장하지 않는다. logical table block을 추가해 전체 복사 없이 증설하고 OS page와 sicadb block을 구분한다. `syncReadGuard`는 Drop으로 반환하고 async DLL 입력은 실제 사용 완료까지 pin한다. timeout/heartbeat만으로 사용 중 공간을 회수하지 않으며 종료 또는 접근 종료 증거가 필요하다. 새 generation 공개→새 reader 전환→옛 reader 종료→옛 공간 반환 순서를 지키고 설정된 예산 안에서는 live writer가 old reader를 기다리지 않으며, pool 소진은 명시적으로 처리하고 사용 중인 block을 덮어쓰지 않는다.

TCP application protocol은 원격 호스트와 선택적 제어·호환 경로다. 같은 호스트 live 경로는 shared memory descriptor를 기본으로 하며 direct read와 push를 같은 API로 제공한다. q wire 호환은 요구하지 않는다. 작은 제어 frame과 Arrow payload는 TCP 경로에서 구분하고, 각 프로세스의 가상 주소가 같을 필요는 없다.

frame은 protocol version·메시지 종류·길이 상한·request/job/stream 식별자와 성공/오류/종료/취소 의미를 가진다. 부분 read/write, frame 분할·결합, 잘린 메시지, 과대 크기, 모르는 버전, 재접속·중복 재전송·늦게 도착한 이전 세대 결과를 검증한다. queue에는 frame 수뿐 아니라 bytes 상한과 backpressure 정책을 둔다. 상세 wire layout은 별도 구현 명세에서 고정한다.

큰 결과 전송이 제어 메시지나 live 경로를 막지 않도록 connection/queue·처리 예산을 분리하고 혼합 부하 지연을 측정한다. 원격 전송의 직렬화·복사·network 비용과 로컬 descriptor+mmap의 비용은 따로 보고한다. Arrow Flight 등 비교 대상보다 자체 protocol이 빠르다고 측정 없이 단정하지 않는다.

공유 메모리의 상세 layout·publish·pin·reclaim·resize 계약은 [공유 메모리 런타임 계약](shared-memory-runtime-contract.md)을 단일 원본으로 삼는다. 이 문서는 해당 계약의 성능·정확성 gate와 검증 결과만 참조한다.

작업 ID는 접수·실행·완료·실패·취소를 구별한다. 재시도 가능한 분석과 외부 부작용이 있는 action은 다르게 취급하며, 모든 action의 exactly-once를 일반적으로 보장한다고 선언하지 않는다.

### 10.3 지연 계약

측정은 queue 대기, 실제 실행, page fault/I/O, 결과 직렬화·전달을 구분한다. 평균뿐 아니라 p50/p95/p99, deadline 초과, 최대 관측값, sample 수를 기록한다. 무제한 batch로 처리량을 높여 취소·실시간 지연을 악화시키지 않는다. 아직 목표 지연을 측정·고정하지 않은 경로를 실거래 준비 완료라고 표시하지 않는다.

### 10.4 느린 계산·DL과 최신 상태

작업의 trigger 정책을 명시한다. 모든 사건을 순서대로 처리하는 `all-events`, 일정 주기의 snapshot을 계산하는 `periodic`, 실행 중 추가 trigger를 최신 상태로 합치는 `latest-only`를 구분한다. 일반 질의나 외부 부작용 action의 요청을 자동으로 합치거나 버리지 않는다.

독립 window/snapshot 기반의 느린 추론은 **model generation × instrument/cohort별 실행 중 1개 + 최신 대기 상태 1개 + 마지막 유효 완료 결과**를 기본 후보로 한다. 실행 중에는 다음 추론 trigger만 합치며, raw 수집·호가 delta 적용·필요한 feature/state 업데이트는 계속 처리한다. 연산 완료와 새 업데이트가 동시에 일어날 때 다음 실행이 누락되지 않도록 소유권과 상태 전이를 검증한다.

입력은 일관된 snapshot으로 고정한다. CPU/GPU가 참조하는 입력·출력 buffer는 실제 완료 확인 전 재사용하지 않는다. GPU 비동기 enqueue 성공이나 취소 요청을 계산 완료로 취급하지 않는다. 실행이 끝나면 최신 대기 상태로 다음 계산을 시작하되 cohort의 다종목 입력 시각·완전성 조건을 함께 기록한다.

결과는 model generation, input snapshot/as-of, 시작·완료·공개 시각, `valid/stale/unavailable/error` 상태를 포함한다. 나이는 완료 시점 대신 입력 시점에서 계산하며 허용 나이·실패 fallback은 전략 정책으로 정한다. 아직 첫 결과가 없으면 unavailable이다. 계산 중 새 tick이 도착했다는 이유만으로 모든 완료 결과를 폐기해 영원히 결과가 없는 상태를 만들지 않는다. 반대로 이전 model generation이나 이미 공개한 결과보다 오래된 입력의 늦은 완료로 현재값을 덮어쓰지 않는다. 엄격한 최신값이 필요한 호출자는 허용 나이를 검사하고 거부할 수 있다.

호출 간 hidden state를 누적하는 모델은 snapshot 모델과 계약이 다르다. 필수 입력 순서·누락 처리·catch-up·hidden-state commit을 먼저 정하며 취소가 이미 바뀐 상태를 자동 rollback한다고 가정하지 않는다. latest-only로 안전하게 바꿀 수 없다면 ordered 처리나 분리된 state 갱신을 사용한다.

다종목·다모델 전체의 GPU/CPU·batch·대기 bytes·pin 예산과 공정성을 제한한다. 한 종목의 연속 업데이트가 다른 종목의 실행을 영구 지연시키지 않아야 한다. replay는 실제 입력 가용 시각·trigger 합치기·추론 지연·결과 공개 시각을 재현 가능한 범위에서 반영하며, event time에 결과가 즉시 존재했던 것처럼 백테스트하지 않는다.

## 11. 내구성과 장애 복구도 성능 계약

단일 owner 표현은 저장 게시 metadata와 시세·feature·1초 append 경로에 한정한다. OMS 상태 갱신에는 적용하지 않는다.

sicadb 코어의 저장·동시성에는 SQLite를 필수 의존성으로 넣지 않는다. log/manifest/checkpoint와 단일 owner·불변 reader의 전이 및 복구를 직접 명세·검증한다. 별도 data-service의 현재 SQLite source/outbox 구현은 그 컴포넌트의 현 상태이며 이 결정만으로 제거하거나 이전 완료라고 표시하지 않는다.

**유효한 최신 live 데이터는 디스크 저장 완료 전에 반드시 사용할 수 있어야 한다.** live 공개를 fsync, stream/file seal, 영속 manifest 게시 완료에 종속시키지 않는다. live와 recording의 queue·worker·자원 예산을 분리해 disk 지연/실패가 live 경로의 동기식 대기가 되지 않게 한다. 이것은 입력 검증·일관된 상태 갱신까지 생략한다는 뜻이 아니다.

파일을 생성한 속도와 durable하게 확정한 속도는 다르다. `accepted`는 명시된 수신 경계의 접수, `live-visible`은 실시간 소비 가능, `durable`은 정해진 장애 모델에서 복구 가능한 저장 확정, `snapshot-published`는 영속 snapshot을 reader가 조회 가능하다는 뜻이다. live와 durable 상태는 별도 축이며 모든 입력이 하나의 일렬 상태 사슬을 밟는다고 가정하지 않는다. ACK와 지연 측정에는 어느 상태인지 표시한다. source fixture의 `fixture_durable`은 시험 consumer 전용이며 sicadb durable ACK가 아니다.

`.arrow.stream`은 진행 중 append/전달 경로, `.arrow`는 sealed IPC FILE 경로로 사용한다. 크기·시간·batch 조건에 따라 seal하고 하루 경계만 강제하지 않는다. Arrow stream 바이트만으로 durable WAL이 완성되는 것은 아니다. 완료된 기록의 범위·sequence·checksum·commit/checkpoint·sync 순서와 재전송 식별을 함께 정해 미완 tail을 판별하고 안전한 복구 위치를 찾는다.

다음 순서는 **영속 이력과 sealed snapshot의 게시 경로**이며 live 공개의 선행조건이 아니다.

1. 임시 segment 작성·구조 검증.
2. 필요한 data/metadata 동기화와 sealed 상태 확정.
3. snapshot/manifest를 복구 가능한 순서로 게시.
4. 규약에 맞는 ack 및 checkpoint 전진.
5. 참조 없는 이전 세대·임시 파일 회수.

각 단계 전후의 프로세스 종료, 일부 쓰기, disk full, sync 실패, ack 유실·재전송, 임시 파일 flush·rename/교체 중단, reader가 이전 snapshot을 계속 사용하는 상황을 시험한다. 정상 재시작 후 **durable ACK한 데이터**의 누락·이중 반영·미완성 파일 공개가 없어야 한다. rename의 이름 변경 완료를 전원 손실 내구성 전체와 동일시하지 않으며 선택한 Windows API·파일 시스템·장치의 보장 범위를 기록한다.

recording queue도 유한하다. disk 정지·포화 시 기록 지연·누락 범위·저장 불가 상태를 명시하고 운영 감시로 전달한다. live queue와 공유한 backpressure로 디스크 복구를 기다리게 만들거나, 보관하지 못한 입력에 durable ACK를 보내지 않는다. 원천 재전송으로 복구 가능한 범위와 복구 불가능한 volatile 구간을 구분한다. live 자체의 처리 용량 초과도 별도의 과부하 상태이며 무한 무손실 queue를 약속하지 않는다. 장애 전에 전략이 본 비영속 데이터는 재시작 후 사라질 수 있으므로 그 구간의 strict replay 일치를 보장하지 않는다.

Windows의 `FlushViewOfFile`만으로 모든 metadata·장치 cache·네트워크 서버의 영속성까지 보장한다고 가정하지 않는다. `FlushFileBuffers`를 포함한 플랫폼별 절차와 파일 시스템/장치의 보장 범위를 확인한다. 프로세스 kill 시험 통과와 실제 전원 차단 내구성 입증은 구분한다. [Microsoft FlushViewOfFile](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-flushviewoffile)

## 12. 검증 체계

### 12.1 같은 버그를 공유하지 않는 정답

| 검증 | 필수 범위 |
|---|---|
| 독립 oracle | 손계산 fixture·넓은 정수/고정밀 계산·독립 알고리즘. reference와 SIMD가 같은 helper를 호출해 동시에 틀리는 경우 방지 |
| 경로 차등 | reference / compiler 최적화 / 각 지원 ISA / owned / mapped / slice / 여러 chunk·worker 구성 |
| 속성 시험 | split/merge·slice·빈 입력·순서 변경의 허용 조건. FP에서 일반적 결합법칙이 성립한다고 가정하지 않음 |
| 오류/경계 | overflow·invalid offset·불일치 length·깨진 bitmap/schema/footer·과대 크기·미완 파일·지원하지 않는 dtype |
| 소유권 | 결과보다 먼저 원본 handle 해제, snapshot pin, reader 동시 유지, 취소·panic·오류 후 회수 |
| concurrency | publish/read/reclaim, queue full·재시도·취소 경합, 느린 consumer, worker 죽음, reader 정지→lease 만료→재개, 무한 반복/재귀·비협조적 호출의 종료 정책 |
| 영속/재현 | 장애 지점별 복구, 동일 snapshot·코드·인수, live/replay 시간·정정 의미 |
| live와 recording | disk 정지·sync 실패·recording queue 포화 중 live 공개 지속, 기록 지연/공백 상태와 durable ACK의 정확성 |
| 느린 추론 | trigger 합치기·완료 경합·마지막 완료값의 나이·obsolete generation 거부·입력 pin·다종목 공정성·stateful 순서 |

오류 처리는 정해진 오류를 반환하는 것이 정상 동작이다. 미구현 기능이 실패하는 것을 성공처럼 박제하지 않는다. 지원하지 않는 기능·플랫폼은 지원표와 검증 누락에 기록한다.

### 12.2 검사 도구와 한계

| 층 | 사용할 수단 | 완료 판정 시 주의 |
|---|---|---|
| 기본 | fmt, clippy, debug/release 테스트, 고정 seed 속성 시험 | release에서도 overflow/오류 의미가 같아야 함 |
| unsafe·수명 | Miri, 적합한 sanitizer, guard page/canary/native 경계 시험 | 하나의 도구 통과는 soundness 증명이 아님. SIMD·mmap·OS 지원 범위를 구분 |
| 동시성 모델 | Loom 등으로 작은 publish/queue/reclamation 모델 탐색 | 외부 crate·std primitive·프로세스·OS 전체를 자동 검증하지 않음 |
| 입력 견고성 | fuzzing: Arrow/descriptor/parser/offset·length·오류 상태 | 크기·시간 예산으로 fuzz 자체 OOM을 구분. seed와 재현 corpus 보존 |
| 실행 경로 | 실제 지원 CPU·실제 mapping·다중 프로세스 native 시험 | emulation/Miri만으로 특정 SIMD 하드웨어의 동작·성능을 주장하지 않음 |
| 성능 | release 벤치마크·allocator 계측·OS/CPU profiler·assembly 점검 | sanitizer/계측 빌드의 시간과 배포 빌드 시간을 섞지 않음 |

Windows에서 Miri·fuzz·sanitizer의 모든 조합이 된다고 가정하지 않는다. 선정 toolchain/target마다 smoke test를 하고 지원되는 별도 환경이 필요하면 명시한다. Miri의 intrinsic 지원을 ISA 이름만으로 일괄 판정하지 않고 사용하는 연산·버전별로 확인한다. owned buffer·offset/length·validity는 해석기 검증 후보이며 OS mapping·프로세스·하드웨어 실행의 빈틈은 native 시험으로 채운다. 도구를 실행할 수 없었던 부분은 통과가 아니라 검증 누락으로 남긴다.

Miri의 Linux target 교차 해석은 Windows native 동작 검증을 대신하지 않는다. Loom 밖의 동기화는 모델에 보이지 않을 수 있으며 모든 relaxed 재배치를 탐색하는 것도 아니다. 지원 범위는 [Miri](https://github.com/rust-lang/miri), [Loom 제약](https://docs.rs/loom/latest/loom/#limitations-and-caveats), [Rust sanitizer target](https://doc.rust-lang.org/unstable-book/compiler-flags/sanitizer.html)을 기준으로 확인한다.

작성 시점에는 Windows fuzz 지원에 대해 [Fuzz Book](https://rust-fuzz.github.io/book/cargo-fuzz/setup.html)과 [cargo-fuzz README](https://github.com/rust-fuzz/cargo-fuzz/blob/main/README.md)의 설명이 일치하지 않는다. 따라서 특정 Windows/MSVC 조합을 이미 지원·검증했다고 기록하지 않고 고정 버전 smoke test로 판단한다. 운영 stable toolchain과 별도로 검증용 nightly를 고정할 수 있다.

### 12.3 실제 writer 경계와 장애 발동 증거

프로젝트의 경계 계약 검증 규율 (프로젝트 내부 문서: BOUNDARY_TESTING.md)을 sicadb의 저장·실행 경계에 적용한다. 아래는 현재 확인한 writer 계보이며 sicadb reader가 이미 통과했다는 지원표가 아니다.

| 입력 계보 | 현재 확인한 표현 | 확보·검증할 실제 fixture |
|---|---|---|
| HFT Arrow.jl 기록 | 기존 42/46 피처 벡터를 담은 `.arrow.stream` 계열 | 원본 writer가 기록한 작은 정상 파일, 실제 schema/버전/validity·batch 경계. 정확한 writer 버전은 해당 표본 provenance에서 확인 |
| HFT cache-builder | arrow-rs 55 (프로젝트 내부 문서: Cargo.toml), StreamWriter (프로젝트 내부 문서: writer.rs)의 derived feature stream. 파일명 `.arrow`만으로 FILE이라고 판정하지 않음 | timestamp Int64·feature Float64를 포함한 실제 산출물. 아래 48컬럼 변환물과 별도 계보 |
| 별도 HFT 변환기 | [변환 실측 보고서](https://github.com/sicarius01/codex-docs/blob/main/quant-research/nas-arrow-conversion-benchmark-2026-09-16.md)의 arrow-rs 59.3.0, timestamp+46 feature+원래 폭의 48컬럼 무압축 IPC FILE | 기존 검증 변환물의 작은 표본과 원본 대비 값·null·순서 정답. 전체 이관 완료는 전제하지 않음 |
| data-service producer | 현재 producer 계약 (프로젝트 내부 문서: producer-contract-v0.md)의 arrow-rs 60.0.0 FILE, Int64/Utf8/Boolean 및 일부 Decimal128 | 실제 수집 표본을 실제 exporter에 통과시킨 출력과 당시 schema/metadata. 기존 합성 demo만으로 원천→producer 경계를 검증했다고 표시하지 않음 |

각 fixture에는 hash·writer/의존성 버전·schema·수집/생성 경로와 기대 결과의 근거를 남긴다. 버전 숫자를 영구 가정으로 박지 않고 지원하려는 계보별로 갱신한다. 첫 K0/K1은 실제 HFT 변환 FILE과 source FILE을 먼저 다루며, stream 지원 완료 전에는 해당 계보를 미검증으로 표시한다. kernel의 합성 경계값·속성 시험·malformed parser fuzz는 추가로 사용하되 실물 경계 검증을 대체하지 않는다. 잘린 파일 시험도 writer 강제 종료로 얻은 실제 미완 파일을 포함한다.

회복 시험은 가능한 최하층 storage/transport에서 장애를 주입하고 그 위 parser·상태 전이·복구 코드를 실제로 통과시킨다. spill, queue full, 취소, owner/worker 종료, reader 정지, dedup, partial write, sync 실패 각각에 **어느 주입이 어느 경로를 발동시켰는지** counter/trace와 최종 상태로 증명한다. dedup·trigger coalescing처럼 여러 입력이 한 전이로 합쳐지는 경우에는 시나리오에 맞는 기대 횟수를 정한다. 모든 counter가 무조건 주입 횟수 이상이어야 한다는 보편 규칙으로 바꾸지 않는다.

### 12.4 진행성과 타입 경계

작업 접수·취소·복구·snapshot 게시 대기마다 시나리오별 시간 한도 T와 진행 지표를 정한다. T 안에 정상 완료할 수 없는 경우 실패·격리·재시도 대기·운영 경고 중 정의한 상태로 전이하고 원인을 남겨야 한다. disk 고장에도 queue가 무조건 비워진다는 불가능한 약속을 하지 않는다. 취소 deadline 초과는 입력 접근 종료를 의미하지 않으며 §10.1의 회수 순서를 유지한다.

코어는 진행성 위반을 구조화된 상태/이벤트로 즉시 노출하고 호스트 운영 adapter가 알림을 담당한다. 특정 메신저를 코어 의존성으로 넣지 않는다. 시험은 정상 종료뿐 아니라 deadline 초과 경로·감시 발동·격리·알림 adapter 전달을 검증한다.

성공값과 오류를 하나의 payload를 해석하는 관례로 구분하지 않는다. Rust 경계는 `Result`와 구분된 오류/상태 타입을 기본으로 한다. validated schema·dictionary generation·snapshot/segment 식별·수치 단위 정보를 명시적인 타입/descriptor로 전달하고 신뢰할 수 없는 wire/file metadata는 런타임에서 검증한다. 오류 타입을 만들었다는 사실만으로 검사되지 않은 외부 값이 안전해지는 것은 아니다.

## 13. 벤치마크 명세

### 13.1 세 규모를 모두 본다

| 수준 | 입력·목적 | 주요 지표 |
|---|---|---|
| kernel | L1/L2/LLC 근처와 그 이상, 짧은 배열, offset·null·선택률 분포 | ns/element, cycles/element(가능할 때), 유효 GB/s, 명시적 input copy, allocation count/bytes |
| runtime/operator | 함수 dispatch·조합·filter/reduce·group/state | 시작 시간, 유휴 private memory, binary 크기, scratch/output, dispatch·함수 호출 비용 |
| 전체 경로 | HFT+추가 데이터, cold/warm, 1/여러 worker, 읽기/쓰기 혼합 | wall time, storage/network bytes, faults, shared/private resident memory, queue·tail latency, spill |

유효 GB/s는 어떤 입력/출력 bytes를 분자에 넣었는지 표시한다. 논리적으로 처리한 bytes와 memory controller/디스크가 실제 이동한 bytes를 같은 값으로 취급하지 않는다.

### 13.2 cache와 입력 조건

- `hot CPU cache`, `page cache warm`, `storage cold`, `NAS staging 포함`을 구분한다.
- 프로세스 재시작만으로 page cache가 비워졌다고 보고하지 않는다. cold 조건의 준비 방법과 검증 근거를 기록한다.
- 코어 benchmark는 생성·설정·검증 시간을 분리하지만, 통합 benchmark는 그 비용을 포함한 수치도 낸다. 산출값을 소비해 compiler가 계산을 제거하지 못하게 한다.
- 실데이터 표본과 결정적 합성 입력을 함께 사용한다. 1,500 instruments × 수개월은 최종 통합 검증 축이며 처음부터 NAS 전체를 반복 scan하지 않는다.
- 균일 데이터뿐 아니라 skew·고유 key 과다·결측 군집·작은 batch 다수·큰 batch·늦은 수정·slow reader를 포함한다.
- 사전 checksum·oracle 계산·fixture 검증이 입력을 미리 읽어 page cache를 데웠는지도 기록한다. 무결성 검증을 생략해 속도를 높인 값과 검증 포함 처리 시간을 분리한다.
- worker 수를 늘려도 무조건 선형 개선을 요구하지 않는다. 대역폭 포화 시점과 실시간 역할에 미치는 간섭을 보고한다.

### 13.3 재현과 회귀 판정

측정마다 코드/입력 hash, toolchain/LLVM, dependency lock, build flags, CPU/ISA/OS, topology/affinity, 전원·주파수 조건, worker 수, memory budget, snapshot, backend를 기록한다. warm-up·측정 길이·반복·실행 순서를 사전에 고정한다. 전후 버전은 가능한 한 교차 실행하고 원시 sample을 보존한다.

Windows에서는 OS build·processor group·전원 계획·core parking·timer 조건·백신 실시간 검사/경로 제외 여부를 함께 기록한다. 이 설정을 benchmark 도중 임의로 바꾸지 않는다. cold 준비와 하드웨어 counter 수집의 도구·권한·지원 범위를 K0에서 확인하며, 권한이나 도구가 없어 측정하지 못한 지표는 미측정으로 표시한다.

**정확성·안전성·할당/복사 계약 위반은 1건도 기능 완료로 인정하지 않는다.** 시간 회귀는 잡음과 구분한다. 고정 성능 runner의 초기 noise 측정 후 benchmark별 허용 변화·통계 기준·tail latency 한도를 결과를 보기 전에 고정한다. 기준이 없는 상태를 성능 gate 통과로 표시하지 않는다.

새 최적화는 정확성·자원 계약을 지키면서 대표 workload에서 유의미한 이득을 보여야 기본 경로가 된다. 특정 크기/분포에서만 이득이면 그 조건의 dispatch로 제한한다. 좋은 평균으로 작은 입력·tail·동시성의 큰 회귀를 상쇄하지 않는다.

공용 CI의 noisy wall time으로 merge를 임의로 흔들지 않는다. 그곳에서는 정확성·할당·실행 가능성을 확인하고, 비교 가능한 runner에서 성능 gate를 별도로 수행한다. p99/p99.9는 충분한 요청 수·관측 구간·부하 생성 방식·신뢰도를 함께 표시한다. timeout·실패·거부 요청도 결과에 포함하고 완료된 요청만으로 지연을 보고하지 않는다. 느려지면 요청 발생도 멈추는 부하 도구로 실제 queue 지연을 숨기지 않는다. 통계적으로 구분할 수 없으면 통과/회귀를 억지로 선택하지 않고 판정 불가로 둔다.

절대 처리량·최대 지연 수치는 아직 **미확정**이다. 241 및 실거래 대상의 baseline 후 용도별 수치를 고정한다. 임의의 GB/s·ns 목표를 이미 입증한 요구 충족으로 표시하지 않는다.

## 14. Rust 빌드·의존성·unsafe 관리

- 최초 workspace에서 Rust toolchain과 의존성을 고정하고 자동 업데이트로 benchmark 환경이 바뀌지 않게 한다.
- safe public API / 검증된 layout·소유권 / 작은 unsafe kernel 순서로 경계를 둔다. pointer arithmetic·length 계산은 overflow와 byte 범위를 먼저 검사한다.
- unchecked indexing, 수동 prefetch, SIMD, 원시 buffer 초기화는 safety 근거와 측정 근거가 있는 위치에만 허용한다. 운영 입력 검증을 debug assertion에만 맡기지 않는다.
- release의 산술 동작은 계약으로 정한다. Cargo profile의 overflow 설정만으로 모든 수치 의미를 해결했다고 보지 않는다.
- LTO, codegen units, PGO, inlining, allocator는 재현 가능한 비교 후 선택한다. PGO 학습 입력과 평가 입력을 구분한다.
- 프로파일링 가능한 빌드와 실제 배포 빌드를 구분한다. 성능을 위해 오류·관측 가능성을 모두 제거하지 않는다. panic/unwind/abort와 worker 복구 정책을 함께 정한다.
- Arrow·mmap·queue·allocator 같은 dependency도 코어 성능과 safety 경계에 포함한다. 버전 교체 시 §12.3의 지원 writer 계보별 fixture와 관련 차등·성능·native 시험을 반복한다.
- 문서의 최신 Rust API 설명과 현재 설치 toolchain의 지원 여부를 혼동하지 않는다. 초기 배포 target마다 실제 compile/run으로 확인한다.

빌드 옵션과 profile 동작의 근거: [rustc codegen](https://doc.rust-lang.org/rustc/codegen-options/index.html), [Cargo profiles](https://doc.rust-lang.org/cargo/reference/profiles.html).

## 15. kernel마다 남겨야 하는 짧은 명세

새 kernel의 구현과 검토에는 아래 기록을 함께 둔다. 값은 측정 후 채우며 빈칸을 합격으로 처리하지 않는다.

| 항목 | 기록할 내용 |
|---|---|
| 의미 | 이름·입출력 타입·shape·unit/scale·epoch/해상도·null/NaN·overflow·오류 시 출력 상태·정렬·수치 모드 |
| 메모리 | 입력/출력 alias, alignment, offset, 읽고 쓰는 범위, backing 수명 |
| 비용 | 시간 복잡도, 논리 read/write bytes, scratch 식, 준비/steady-state 할당, spill 여부 |
| 실행 | scalar/reference, 자동 벡터화, ISA backend, dispatch 조건, chunk/cancel 경계 |
| 검증 | 적용 C01~C15와 증거/제외 사유, 독립 oracle, 실물 경계/속성/fuzz, unsafe 검사, 지원 CPU/OS 실행, 알려진 미검증 |
| 성능 | 입력 분포별 baseline·후보·채택 근거, 원시 결과·환경·회귀 한도 |

코드 작성자와 별도 검토자가 의미·unsafe·성능 증거를 확인한다. CI 녹색, 테스트 개수, 코드 줄 수만으로 완료를 판정하지 않는다.

## 16. 첫 구현 묶음과 단계별 합격선

### K0 — 코어와 측정 기반

Rust workspace, 지원 target/toolchain, 값·view 소유권, 고정 snapshot fixture, 독립 정답, 할당 계측과 release benchmark runner를 먼저 만든다. 새 runtime에서 일반 함수 두 개를 조합하는 예제를 포함한다. 제어 계층·kernel·저장/OS 계층의 의존 방향을 분리한다. K1에서 사용할 자원 예산과 성능 비교 기준을 후보 결과 판정 전에 고정한다. 구체 첫 작업은 작업 정의 (프로젝트 내부 문서: k0-k1-work-item.md), 현재 증거는 결과 보고서 (프로젝트 내부 문서: k0-k1-results-2026-09-16.md)를 따른다.

### K1 — 작은 kernel 집합의 엄격한 완료

첫 핵심 kernel은 bitmap/validity, primitive 비교·선택, count, 의미를 고정한 수치 reduce다. 실제 HFT 열과 추가 데이터 fixture를 owned/mapped 배열로 같은 함수에 전달한다.

**K1 합격:** 독립 정답·수치/단위 의미(C01/C12), 지원 SIMD 경로·fallback과 tail/offset/null/overflow 검사(C06/C07), pure loop 0 할당(C03), 지원 직접 참조 경로의 입력 복사 0과 수명(C02/C08), 작업·출력 예산(C04/C05), 오류 시 출력 안전성(C11), 실물 fixture·적용 오류 경로와 bounded 취소(C13/C14), chunk 변화에 대한 계약상 동일 결과와 baseline/최적 경로 비교(C01/C10)를 제시한다. 함수 조합 비용도 포함한다. C09/C15의 저장·ingest 검증은 후속 단계이며 첫 read-only kernel만으로 통과 처리하지 않는다. 처음부터 전체 join·sort·APL evaluator를 한 번에 완성할 필요는 없다.

### K2 — 같은 실행을 worker로 이동

고정 snapshot을 지정한 비동기 요청·결과·취소·제한 queue를 구현한다. mapping 수명·동시 reader·worker 죽음·재실행과 연구/실시간 역할 간 간섭을 검증한다. 초기 단일 프로세스 코어부터 job budget과 chunk 취소 지점을 받아 이 단계에서 핵심 loop를 다시 설계하지 않게 한다.

### K3 — 대용량과 상태·영속성

group/rolling/as-of·외부 sort/join, snapshot 공개·정정·복구를 기능별로 추가한다. 각 연산은 이 문서의 동일 gate를 통과해야 한다. live 비대기 공개와 별도 durable ACK를 실제 저장 경로에서 검증한다(C09/C15). 작은 작업 예산 시험 이후 실제 RAM 초과·SSD/NAS·혼합 부하와 전체 instrument 범위로 확대한다. 보존된 입력·가용 시각 범위의 strict live/replay와 실거래 latency 합격 전 운영 전환을 완료로 표시하지 않는다. 비영속 입력 손실 구간을 재현 가능 범위로 포장하지 않는다.

APL 작성성 검증은 K0/K1과 병행한다. 그 결과가 아직 없다고 mmap·kernel 검증을 멈추지 않으며, Rust로 구현한다는 결정이 APL 선호를 폐기한다는 뜻도 아니다.

## 17. 현재 완료·미완료

- **구현·실행:** Rust workspace, borrowed view·작은 builtin 호출/조합, reference/Compiler/AVX2 kernel, 고정 수치 의미, 실물 fixture/oracle, Arrow/mmap 수명·검증, TCP/worker pool·예산·실제 취소/강제 종료/부모 사망/재시작과 페이지 공유. 조건 결합·bitmap·bounded metadata 재사용·알림 기반 대기·Wait까지 구현했다. 성능 개선 당시 debug/release 각 99개, core Miri 22개와 성능 근거는 전체 비교 (프로젝트 내부 문서: performance-sweep-results-2026-09-16.md)에 있으며 K0/K1 (프로젝트 내부 문서: k0-k1-results-2026-09-16.md)·K2 (프로젝트 내부 문서: k2-results-2026-09-16.md)는 당시 기록으로 보존한다.
- **미완료:** 전체 범용 언어·APL 평가, 영속 snapshot·live/내구 저장, Windows 10 실기, 241 topology·baseline, 운영 지연 gate, 장기 fuzz/sanitizer·실제 RAM 초과/대용량/장기 혼합 부하. native 종료 확인 실패 경로는 소스 검토이며 실제 OS 실패 주입은 남았다. 작은 read-only slice로 C09/C15 또는 전체 플랫폼 지원을 통과 처리하지 않는다.
- **지연 재검증과 다음 작업:** 최초 예정 시각 기준 p99 약 25% 증가를 보존했다. 후속 진단 (프로젝트 내부 문서: pacing-diagnosis-results-2026-09-16.md)의 같은 바이너리 재실행과 별도 observer 비교에서는 악화가 재현되지 않았다. sleep 안의 지연은 관찰했지만 OS 내부 원인은 실제 ETW 접근 거부로 미확정이며 운영 지연 목표 합격은 보류한다. 진단 도구까지 포함한 최신 debug/release는 각 106개 통과다. [다음 작업](next-work.md)은 저장·snapshot/live의 최소 완성본이며 대규모 I/O·운영 지연 검증으로 이어간다. 현재 공개 저장소에 게시하는 버전으로 갱신했다.

성능·정확성·복구는 서로 대체할 수 없다. 빠르지만 값이 틀리거나, 작은 입력에서만 빠르거나, 장애 뒤 같은 데이터를 설명할 수 없는 코어는 sicadb의 기초로 채택하지 않는다.
