# sicadb — Rust 코어 성능·정확성 계약

작성: 2026-09-16. **상태: 구현에 적용할 명세 v0. 런타임·벤치마크·검증 도구 실행은 아직 미착수.** 사용자 요청으로 게시한 공개 열람본이다. 내부 머신 식별과 비공개 문서 참조를 공개용으로 정리했다.

## 1. 결정과 적용 범위

**sicadb 구현 언어는 Rust로 확정한다.** APL은 사용자가 함수를 표현하고 LLM이 프로그램을 작성하는 언어의 첫 검증 후보다. 구현 언어와 사용자 언어는 별개다.

성능은 코어의 완료 조건이다. 저수준 계산 함수(kernel)는 최초 구현부터 데이터 배치, SIMD, 할당, 메모리 읽기·쓰기, 동시성 비용을 검토하고 측정해야 한다. 동작하는 scalar 코드만 작성한 뒤 최적화를 무기한 후속 과제로 넘기지 않는다. 자동 벡터화로 목표를 충족한다면 그것도 최적화된 구현으로 인정하되 생성 코드와 측정 근거가 있어야 한다.

범위는 범용 값·배열·함수 실행, Arrow/mmap, snapshot, 작업 실행·프로세스 통신이다. HFT와 추가 데이터를 같은 기반에서 처리한다. RAM보다 큰 데이터도 정상 입력이다. 모든 데이터를 RAM에 올리는 전제는 없다. 원천 수집·정규화·재전송은 별도 수집 컴포넌트가 맡고, sicadb는 공통 저장·snapshot·함수 실행·조회와 작업 처리를 맡는다.

이 문서의 **필수**는 구현·기능 완료 판정의 조건이다. **실측 선택**은 후보를 비교하여 채택할 조건이며, 전부 켜는 것이 요구사항은 아니다. **미검증**은 완료가 아니다. 아래 성능 목표·시험 규칙은 sicadb의 설계 결정이며, 외부 제품이 보장해 주는 성질로 해석하지 않는다.

## 2. 확인한 환경과 아직 확인할 환경

| 항목 | 현재 근거 | 구현·측정 전 필요한 확인 |
|---|---|---|
| 개발 환경 | Windows, 현재 `rustc 1.93.1`, LLVM 21.1.8, `x86_64-pc-windows-msvc`, `cargo 1.93.1` | 최초 workspace에서 toolchain·의존성·빌드 profile 고정. 현재 설치 버전이 최종 선택이라는 뜻은 아님 |
| 주 연구 머신 | 연구용 워크스테이션, Dell Precision 7920 Tower. RAM 규모는 사용자 설명 | 실제 CPU 모델·socket·NUMA node·물리/논리 core·cache·ISA·메모리 채널·SSD·OS build·processor group |
| 영속 입력 | HFT Arrow의 여러 schema, 추가 데이터 Arrow 출력 예정 | 파일/스키마별 alignment·압축·batch·validity·정렬·실제 보유 범위 |
| 운영 실행 환경 | 개발 머신과 연구용 워크스테이션을 구분 | 개발 PC 실측을 연구용 워크스테이션 성능으로 보고하지 않음. 실거래 호스트는 별도 검증 |

이번 문서 작성에서는 연구용 워크스테이션 전원·설정 변경, NAS 대량 읽기, 하드웨어 실측을 하지 않았다. CPU 모델이나 장착 socket 수는 chassis 이름으로 추정하지 않는다. Windows의 많은 논리 CPU는 processor group과 affinity API 동작까지 확인한다. [Microsoft processor groups](https://learn.microsoft.com/en-us/windows/win32/procthread/processor-groups)

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
| C09 | 수신·내구 저장·snapshot 공개를 구분한다 | 장애 지점별 복구 및 중복 재전송 검증 |
| C10 | 성능 주장은 동일 의미·명시된 환경·재현 가능한 입력에서 측정한다 | 원시 측정값·버전·설정·비교 기준·변동성 보고 |

`C03`은 순수 계산 kernel 계약이다. 파일 열기·schema 검증·결과 준비·새 그룹 삽입·작업 등록까지 모두 무할당이라고 주장하지 않는다. 이 비용은 숨기지 않고 query 전체 측정에 포함한다. `C02`도 압축 해제·형변환·필터 결과 생성까지 무복사라는 뜻은 아니다.

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

| 영역 | v0 원칙과 시험 |
|---|---|
| Int64·timestamp·scaled integer | float로 우회하지 않는다. elementwise 산술은 명시적 checked 의미가 기본이다. wrapping/saturating은 이름과 계약이 다른 연산으로만 제공 |
| 정수 reduce | 반환 타입과 누산 타입, 중간/최종 overflow 시점을 연산별로 고정. 넓은 누산기 등으로 재배열 후 동일 의미를 지킬 수 있어야 SIMD/병렬화 허용 |
| validity | null과 NaN, 실제 0을 구분. null lane이 유효 결과·오류·관측 가능한 FP 의미에 영향을 주지 않아야 함. 초기화된 lane의 안전한 비트 연산/산술 후 mask 적용은 허용하되 분모·인덱스·trap 가능 연산은 실행 전에 안전하게 처리 |
| FP elementwise | NaN·무한대·부호 있는 0·subnormal·나눗셈을 명시. FMA, reciprocal 근사, FTZ/DAZ, fast-math는 결과를 바꿀 수 있는 선택으로 취급 |
| FP reduce | 단순 직렬 합을 무조건 유일한 정답으로 삼지 않는다. 기본 재현 모드는 고정된 논리적 분할·누산·결합 규칙을 정해 I/O chunk·worker 수·ISA 선택이 결과 순서를 임의 변경하지 못하게 함 |
| FP 검증 | 고정 환경의 재현성과 수학적 오차를 따로 검사. 독립 고정밀 정답과 연산별 절대/상대/ULP 기준을 사전 결정. NaN 분류와 signed zero 규칙도 별도 검사 |
| order·group·join | 동일 key/time tie, hash 순회 순서, 정렬 안정성, 경계 포함 여부를 명시. dictionary 순서가 사용자 정렬 순서를 대신하지 않음 |
| 상태·시간 | window warm-up/close, late/correction, event/available time과 코드 버전을 고정. 같은 함수 이름만으로 live/replay 동일성을 주장하지 않음 |

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

정정 이후 이전 결과를 돌려주거나, 아직 준비되지 않은 결과를 공개하면 cache 성능 개선은 무효다. 오래 유지한 snapshot pin은 명시적으로 제한/거부/취소 정책을 적용하며, 참조 중인 파일을 몰래 회수하지 않는다. 실행 시간·임의성·외부 입출력에 의존하는 함수를 순수 함수처럼 결과 cache하지 않는다.

kernel 비교에서는 결과 cache를 끄고, 사용자 반복 작업에서는 cache 효과와 비용을 별도로 측정한다.

알고리즘 선택에도 비용 모델을 둔다. 예를 들어 길이 `n`의 packed selection bitmap은 약 `ceil(n/8)` bytes, 64비트 행 인덱스는 선택 행 수 `k`에 대해 약 `8k` bytes이며 owner·정렬 padding 등은 별도다. 희소/밀집 선택, 정렬된 key의 merge와 hash, 사전 집계와 전체 materialization을 비교한다. hash 충돌·skew·rehash가 입력 복사와 예산 폭증으로 이어지지 않도록 검사한다. SIMD가 비효율적인 알고리즘 선택을 만회할 것이라고 가정하지 않는다.

## 9. mmap·영속 파일·대용량 I/O

- mmap은 주소 공간에 파일을 연결하는 방식이다. 파일 open, metadata 검증, page fault, storage read, 계산, 결과 전달을 별도로 측정한다.
- read-only mapping만으로 다른 프로세스의 파일 수정·truncate가 차단되는 것은 아니다. sicadb가 관리하는 불변 segment의 수명·쓰기 권한을 보장한다. 외부 원본은 불변성이 보장된 등록 또는 관리 영역의 검증된 import로 연결한다.
- 검사 시점의 checksum만으로 이후 불변성을 증명하지 않는다. writer→seal→publish 이후에는 같은 파일을 덮어쓰지 않는다. reader lease와 세대별 회수 조건을 구현한다.
- lease/heartbeat 만료는 실행 중인 reader의 접근 종료 증거가 아니다. 일시 정지한 reader가 다시 실행될 수 있는 동안 mapping·backing file·공유 영역을 변경하거나 재사용하지 않는다. 종료/접근 차단이 확인된 뒤 회수한다.
- map 전체를 미리 touch하거나 `mlock`/working-set 고정으로 전 데이터 상주를 요구하지 않는다. map window·readahead·prefetch·I/O queue는 접근 패턴과 운영체제별 실측 선택이다.
- 파일 수·open handle·mapped region·page table·metadata cache도 자원이다. 날짜×종목 단순 분할로 소파일을 대량 생성하기 전에 scan·pruning·동시 조회를 비교한다.
- column projection이 항상 물리 디스크의 정확히 그 bytes만 읽는다는 뜻은 아니다. page granularity·IPC batch 배치로 인한 read amplification을 계측한다.
- 압축 IPC/차가운 저장 계층은 공간·I/O 절감과 decode/복사 비용을 별도로 측정한다. 비압축 직접 참조 경로와 혼동하지 않는다.
- NAS에서 필요한 범위를 로컬 SSD로 준비하는 시간, 첫 로컬 읽기, 반복 읽기를 각각 표시한다. 대량 이동은 파일별 연결·상태 확인을 반복하지 않고 묶음 전송으로 수행한다. mmap이나 SIMD가 NAS 대역폭 제한을 없애지는 않는다.

file-backed mmap의 안전성은 파일이 외부에서 변경될 수 있다는 조건까지 다룬다. `memmap2`도 이 때문에 file-backed 생성 API의 unsafe 조건을 설명한다. 구체 crate/version과 Arrow buffer 연결 API는 구현 시 고정·검증한다. [memmap2 safety](https://docs.rs/memmap2/latest/memmap2/struct.MmapOptions.html)

**작업 메모리 예산보다 큰 입력 시험과, 실제 물리 RAM/파일 cache를 초과하는 시험은 별개다.** 첫 시험은 제한된 scratch로 계산하는지 보여 준다. RAM 초과 성능은 별도 격리된 큰 입력/제약 환경에서 page fault·storage bytes·시스템 전체 메모리를 관찰해야 한다.

## 10. 동시성·다중 프로세스·비동기 실행

### 10.1 기본 구조

- 초기에는 상태의 단일 owner와 불변 snapshot reader를 기본으로 한다. 연구 worker와 실시간 owner의 실행·메모리·queue 예산을 분리한다.
- mutable state는 가능한 한 owner 안에 둔다. 여러 thread의 결과는 local 집계 후 명시된 순서로 결합한다.
- async I/O와 CPU 작업을 분리한다. 긴 kernel이나 page fault 가능성이 큰 분석을 실시간 event loop에서 직접 실행하지 않는다.
- queue는 항목 수와 byte 수에 상한을 둔다. queue full일 때 대기·거부·재시도 중 무엇을 하는지 정하고, 상태를 조용히 버리거나 순서를 바꾸지 않는다.
- 배열 실행의 취소는 bounded chunk/batch 경계에서 확인한다. **취소 요청 → 접근 종료 확인 또는 worker 프로세스 종료 확인 → 출력·scratch·snapshot pin 회수** 순서를 지킨다. deadline 초과/취소 요청을 실제 실행 종료와 혼동하지 않으며, 이미 durable해진 작업의 ack 의미를 유지한다.
- 범용 evaluator의 사용자 반복·재귀·함수 호출에는 별도의 실행 예산·stack 한도·취소 확인 지점을 둔다. 오래 걸리는 I/O에는 timeout/취소 정책을 두고, 협조하지 않는 native 호출은 지원 제한 또는 격리 worker 종료로 처리한다. 배열 chunk만으로 모든 사용자 프로그램의 취소가 가능하다고 주장하지 않는다.
- lock-free는 기본 의무가 아니다. mutex/channel/owner 분리가 요구를 만족하면 채택한다. 경합 때문에 변경할 때만 atomic ordering·ABA·재활용·안전한 reclamation을 함께 명세화한다.

### 10.2 shared memory와 IPC

우선 sealed 파일을 여러 프로세스가 각자 map하고 작은 descriptor를 교환한다. 각 프로세스의 가상 주소가 같을 필요는 없다. 공유 mutable ring은 latency/throughput 근거가 있을 때 별도로 구현한다.

공유 ring을 도입한다면 slot 초기화→내용 쓰기→publish 순서, acquire/release 관계, sequence wrap, false sharing, producer/consumer 중단, owner 사망, backpressure와 재접속을 명시한다. Rust의 일반 reference·프로세스 내부 mutex를 그대로 공유 메모리에 저장하지 않는다. inter-process atomic/동기화 지원은 실제 OS·ABI·구현의 보장을 확인해야 한다.

작업 ID는 접수·실행·완료·실패·취소를 구별한다. 재시도 가능한 분석과 외부 부작용이 있는 action은 다르게 취급하며, 모든 action의 exactly-once를 일반적으로 보장한다고 선언하지 않는다.

### 10.3 지연 계약

측정은 queue 대기, 실제 실행, page fault/I/O, 결과 직렬화·전달을 구분한다. 평균뿐 아니라 p50/p95/p99, deadline 초과, 최대 관측값, sample 수를 기록한다. 무제한 batch로 처리량을 높여 취소·실시간 지연을 악화시키지 않는다. 아직 목표 지연을 측정·고정하지 않은 경로를 실거래 준비 완료라고 표시하지 않는다.

## 11. 내구성과 장애 복구도 성능 계약

파일을 생성한 속도와 durable하게 확정한 속도는 다르다. accepted, durable, published의 ack를 분리하고 측정에 사용한 ack를 표시한다.

1. 임시 segment 작성·구조 검증.
2. 필요한 data/metadata 동기화와 sealed 상태 확정.
3. snapshot/manifest를 복구 가능한 순서로 게시.
4. 규약에 맞는 ack 및 checkpoint 전진.
5. 참조 없는 이전 세대·임시 파일 회수.

각 단계 전후의 프로세스 종료, 일부 쓰기, disk full, sync 실패, ack 유실·재전송, reader가 이전 snapshot을 계속 사용하는 상황을 시험한다. 정상 재시작 후 승인된 데이터의 누락·이중 반영·미완성 파일 공개가 없어야 한다.

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

Windows에서 Miri·fuzz·sanitizer의 모든 조합이 된다고 가정하지 않는다. 선정 toolchain/target마다 smoke test를 하고 지원되는 별도 환경이 필요하면 명시한다. 도구를 실행할 수 없었던 부분은 통과가 아니라 검증 누락으로 남긴다.

Miri의 Linux target 교차 해석은 Windows native 동작 검증을 대신하지 않는다. Loom 밖의 동기화는 모델에 보이지 않을 수 있으며 모든 relaxed 재배치를 탐색하는 것도 아니다. 지원 범위는 [Miri](https://github.com/rust-lang/miri), [Loom 제약](https://docs.rs/loom/latest/loom/#limitations-and-caveats), [Rust sanitizer target](https://doc.rust-lang.org/unstable-book/compiler-flags/sanitizer.html)을 기준으로 확인한다.

작성 시점에는 Windows fuzz 지원에 대해 [Fuzz Book](https://rust-fuzz.github.io/book/cargo-fuzz/setup.html)과 [cargo-fuzz README](https://github.com/rust-fuzz/cargo-fuzz/blob/main/README.md)의 설명이 일치하지 않는다. 따라서 특정 Windows/MSVC 조합을 이미 지원·검증했다고 기록하지 않고 고정 버전 smoke test로 판단한다. 운영 stable toolchain과 별도로 검증용 nightly를 고정할 수 있다.

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

**정확성·안전성·할당/복사 계약 위반은 1건도 기능 완료로 인정하지 않는다.** 시간 회귀는 잡음과 구분한다. 고정 성능 runner의 초기 noise 측정 후 benchmark별 허용 변화·통계 기준·tail latency 한도를 결과를 보기 전에 고정한다. 기준이 없는 상태를 성능 gate 통과로 표시하지 않는다.

새 최적화는 정확성·자원 계약을 지키면서 대표 workload에서 유의미한 이득을 보여야 기본 경로가 된다. 특정 크기/분포에서만 이득이면 그 조건의 dispatch로 제한한다. 좋은 평균으로 작은 입력·tail·동시성의 큰 회귀를 상쇄하지 않는다.

공용 CI의 noisy wall time으로 merge를 임의로 흔들지 않는다. 그곳에서는 정확성·할당·실행 가능성을 확인하고, 비교 가능한 runner에서 성능 gate를 별도로 수행한다. p99/p99.9는 충분한 요청 수·관측 구간·부하 생성 방식·신뢰도를 함께 표시한다. timeout·실패·거부 요청도 결과에 포함하고 완료된 요청만으로 지연을 보고하지 않는다. 느려지면 요청 발생도 멈추는 부하 도구로 실제 queue 지연을 숨기지 않는다. 통계적으로 구분할 수 없으면 통과/회귀를 억지로 선택하지 않고 판정 불가로 둔다.

절대 처리량·최대 지연 수치는 아직 **미확정**이다. 연구용 워크스테이션 및 실거래 대상의 baseline 후 용도별 수치를 고정한다. 임의의 GB/s·ns 목표를 이미 입증한 요구 충족으로 표시하지 않는다.

## 14. Rust 빌드·의존성·unsafe 관리

- 최초 workspace에서 Rust toolchain과 의존성을 고정하고 자동 업데이트로 benchmark 환경이 바뀌지 않게 한다.
- safe public API / 검증된 layout·소유권 / 작은 unsafe kernel 순서로 경계를 둔다. pointer arithmetic·length 계산은 overflow와 byte 범위를 먼저 검사한다.
- unchecked indexing, 수동 prefetch, SIMD, 원시 buffer 초기화는 safety 근거와 측정 근거가 있는 위치에만 허용한다. 운영 입력 검증을 debug assertion에만 맡기지 않는다.
- release의 산술 동작은 계약으로 정한다. Cargo profile의 overflow 설정만으로 모든 수치 의미를 해결했다고 보지 않는다.
- LTO, codegen units, PGO, inlining, allocator는 재현 가능한 비교 후 선택한다. PGO 학습 입력과 평가 입력을 구분한다.
- 프로파일링 가능한 빌드와 실제 배포 빌드를 구분한다. 성능을 위해 오류·관측 가능성을 모두 제거하지 않는다. panic/unwind/abort와 worker 복구 정책을 함께 정한다.
- Arrow·mmap·queue·allocator 같은 dependency도 코어 성능과 safety 경계에 포함한다. 버전 교체 시 관련 차등·성능·native 시험을 반복한다.
- 문서의 최신 Rust API 설명과 현재 설치 toolchain의 지원 여부를 혼동하지 않는다. 초기 배포 target마다 실제 compile/run으로 확인한다.

빌드 옵션과 profile 동작의 근거: [rustc codegen](https://doc.rust-lang.org/rustc/codegen-options/index.html), [Cargo profiles](https://doc.rust-lang.org/cargo/reference/profiles.html).

## 15. kernel마다 남겨야 하는 짧은 명세

새 kernel의 구현과 검토에는 아래 기록을 함께 둔다. 값은 측정 후 채우며 빈칸을 합격으로 처리하지 않는다.

| 항목 | 기록할 내용 |
|---|---|
| 의미 | 이름·입출력 타입·shape·null/NaN·overflow·오류·정렬·수치 모드 |
| 메모리 | 입력/출력 alias, alignment, offset, 읽고 쓰는 범위, backing 수명 |
| 비용 | 시간 복잡도, 논리 read/write bytes, scratch 식, 준비/steady-state 할당, spill 여부 |
| 실행 | scalar/reference, 자동 벡터화, ISA backend, dispatch 조건, chunk/cancel 경계 |
| 검증 | 독립 oracle, 경계/속성/fuzz, unsafe 검사, 지원 CPU 실행, 알려진 미검증 |
| 성능 | 입력 분포별 baseline·후보·채택 근거, 원시 결과·환경·회귀 한도 |

코드 작성자와 별도 검토자가 의미·unsafe·성능 증거를 확인한다. CI 녹색, 테스트 개수, 코드 줄 수만으로 완료를 판정하지 않는다.

## 16. 첫 구현 묶음과 단계별 합격선

### K0 — 코어와 측정 기반

Rust workspace, 지원 target/toolchain, 값·view 소유권, 고정 snapshot fixture, 독립 정답, 할당 계측과 release benchmark runner를 먼저 만든다. 새 runtime에서 일반 함수 두 개를 조합하는 예제를 포함한다. 제어 계층·kernel·저장/OS 계층의 의존 방향을 분리한다.

### K1 — 작은 kernel 집합의 엄격한 완료

첫 대상은 bitmap/validity, primitive 비교·선택, count, 의미를 고정한 수치 reduce다. 실제 HFT 열과 분봉 fixture를 owned/mapped 배열로 같은 함수에 전달한다.

**K1 합격:** 독립 정답, 지원 SIMD 경로·fallback, tail/offset/null/overflow 검사, pure loop 0 할당, 지원 직접 참조 경로의 입력 복사 0, 작업 예산 내 실행, chunk 변화에 대한 계약상 동일 결과, baseline과 최적 경로 비교를 모두 제시한다. 함수 조합 비용도 포함한다. 처음부터 전체 join·sort·APL evaluator를 한 번에 완성할 필요는 없다.

### K2 — 같은 실행을 worker로 이동

고정 snapshot을 지정한 비동기 요청·결과·취소·제한 queue를 구현한다. mapping 수명·동시 reader·worker 죽음·재실행과 연구/실시간 역할 간 간섭을 검증한다. 초기 단일 프로세스 코어부터 job budget과 chunk 취소 지점을 받아 이 단계에서 핵심 loop를 다시 설계하지 않게 한다.

### K3 — 대용량과 상태·영속성

group/rolling/as-of·외부 sort/join, snapshot 공개·정정·복구를 기능별로 추가한다. 각 연산은 이 문서의 동일 gate를 통과해야 한다. 작은 작업 예산 시험 이후 실제 RAM 초과·SSD/NAS·혼합 부하와 전체 instrument 범위로 확대한다. strict live/replay와 실거래 latency 합격 전 운영 전환을 완료로 표시하지 않는다.

APL 작성성 검증은 K0/K1과 병행한다. 그 결과가 아직 없다고 mmap·kernel 검증을 멈추지 않으며, Rust로 구현한다는 결정이 APL 선호를 폐기한다는 뜻도 아니다.

## 17. 현재 완료·미완료

- **완료:** Rust 구현 언어 확정, 이 성능·정확성 계약과 단계별 검증 기준 작성.
- **미완료:** workspace/런타임 코드, 함수별 수치 의미의 최종 표, reference/SIMD kernel, fixture/oracle, 실제 도구 지원 smoke test, 연구용 워크스테이션 topology·baseline, 성능 수치 gate, fuzz/model/native/crash/대용량 시험.
- **다음 작업:** K0와 첫 K1 kernel의 구현. 각 기능의 검증 증거를 코드와 함께 축적한다.

성능·정확성·복구는 서로 대체할 수 없다. 빠르지만 값이 틀리거나, 작은 입력에서만 빠르거나, 장애 뒤 같은 데이터를 설명할 수 없는 코어는 sicadb의 기초로 채택하지 않는다.
