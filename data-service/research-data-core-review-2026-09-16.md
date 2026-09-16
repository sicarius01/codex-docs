# 추가 데이터와 HFT를 위한 연구·실거래 통합 데이터 플랫폼 검토

작성·통합 플랫폼 관점 갱신: 2026-09-16 KST. 상태: **조사·설계 제안, 구현 미착수·후보 간 실측 미실시**.

대상은 약 **1,500 instruments × 수개월의 반복 통계**, 추가 데이터·HFT의 과거·현재 조회, 그리고 **하나의 논리적 데이터 원천으로 연구와 실거래를 지원하는 플랫폼**이다. 연구 머신은 사용자가 제시한 **241, RAM 약 760GB**를 기준으로 한다. `loop`의 공유 메모리 구현, 현재 HFT 코드와 기존 실행 보고서, KX·Arrow·분석 엔진의 공식 문서를 검토했다.

이 문서는 [앞선 연구 I/O 검토](https://github.com/sicarius01/codex-docs/blob/main/data-service/research-io-review-2026-09-16.md)를 확장한다. 사용자 최종 지시에 따라 **kdb의 방식과 아이디어, 자체 구현 가능성**을 다룬다. 제품 도입 조건이나 가격은 비교하지 않는다. 241·NAS의 현재 성능을 새로 측정한 보고서는 아니다.

읽는 순서: **1절 갱신 결론 → 21~26절 통합 플랫폼·질의·비동기·실거래 관점 → 14~20절 내부 엔진 비교**. 기존 본문의 번호는 유지했다. 용량은 3절, kdb 저장 아이디어는 6~7절, 현재 HFT의 문제는 2절과 9절이다.

## 1. 추천 결론

**연구와 실거래가 같은 데이터·질의·함수 정의를 사용하는 통합 플랫폼을 설계하는 방향에 동의한다. 공통 질의 계층, 다중 프로세스 실행, 비동기 요청·액션, 실시간 구독은 이 목표의 핵심이다.** 기존 조사는 연구 I/O와 엔진 비교에 치우쳐 이 관점을 충분히 반영하지 못했다. 21~26절에서 보완한다.

먼저 **플랫폼의 데이터 의미·질의/함수·메시지·프로세스·복구 규약**을 정하고, 각 역할의 저장·계산 구현을 선택한다. DuckDB·Polars·분석 서버를 재사용할 수 있어도 이 상위 설계가 없어지지는 않는다. 독자적인 q 호환 언어·범용 저장 엔진까지 만드는 일은 별도 선택이다. mmap과 shared memory는 이 플랫폼 안에서 반복 입력과 프로세스 간 사본을 줄이는 수단이다.

연구 I/O의 권장 조합은 다음과 같다. 상시 실거래 경로의 배치는 23절에서 별도로 정의한다.

1. **NAS:** 원본·이력 보관과 백업. 실험마다 전량을 다시 읽는 장소로 사용하지 않는다.
2. **241 연구용 로컬 SSD/NVMe:** 필요한 데이터의 검증된 사본, 압축 열 기반 이력, 계산 중 임시 파일. 용량을 확인한 전용 연구 볼륨이 전제다.
3. **분할 실행:** RAM보다 큰 이력도 열·시간·instrument 묶음으로 읽고 계산할 수 있어야 한다.
4. **반복 입력:** 엔진 내부 재사용 또는 상주 프로세스의 배열로 중복 준비를 줄인다. 반복 decode·사본이 병목이면 자주 쓰는 열·기간의 읽기 전용 mmap도 비교한다.
5. **계산 위치:** 241에 작업을 보내고 작은 통계 결과를 받는다. 큰 원자료가 필요하면 제한된 batch로 받는다.
6. **같은 규약:** 최신·과거, 추가 데이터·HFT를 같은 식별·시각·품질·snapshot 규칙으로 요청한다. 내부 저장 형식은 데이터 성격에 맞춰 선택한다.

**760GB는 큰 장점이지만, HFT 전 종목·수개월·전 컬럼을 모두 상주시키는 설계의 근거는 될 수 없다.** 1,500개 × 90일 × 1초 격자에서는 숫자 열 8개만으로 약 746.5GB다. 반면 같은 범위의 1분 데이터는 숫자 열 100개가 약 155.5GB다. 두 데이터를 같은 메모리 정책으로 다루면 안 된다.

연구 실행부의 첫 비교 후보는 **① DuckDB의 로컬 Parquet 직접 조회와 native DB 적재, ② Polars의 선택 읽기·streaming, ③ 현재 Rust 계산을 한 번 로드해 반복 실행하는 worker**다. 이 비교만으로 플랫폼 전체를 선택하지 않는다. 기존 엔진이 역할별 요구를 충족하면 독자 저장·범용 계산 엔진의 개발은 줄일 수 있지만, **공통 질의 의미와 비동기 실행·구독·실거래 재현 규약은 계속 필요하다.** 특정 엔진의 일반 벤치마크 순위를 우리 데이터의 성능으로 대신하지 않는다.

## 2. 현재 코드와 과거 기록에서 확인한 것

### 2.1 HFT에는 서로 다른 두 읽기 경로가 있다

```text
호가·틱 패킷
  ├─ 패킷 아카이브 → SPKR 색인 → 사건 순서 재생/APBT
  └─ record/replay → 1초 raw → 피처 캐시 → 통계·학습·1초 백테스트

추가 데이터 원천
  └─ data-service 수집 → 정규화 revision → 최신/과거 조회
```

1초 피처 통계에는 열 기반 읽기가 잘 맞는다. 패킷 재생에는 정확한 사건 순서와 호가장·주문 상태가 필요하다. **같은 데이터 준비 규약을 사용하되, 패킷 재생까지 1초 테이블로 대체하지 않는다.**

| 확인 항목 | 현재 동작 | 설계에 주는 의미 |
|---|---|---|
| 1초 raw | `ts_ns: i64` + `List<f64>` 46슬롯. 캐시 빌더가 앞 42슬롯을 행 우선 배열로 복사 | raw를 바로 열 단위 통계 배열처럼 쓸 수 없음 |
| 피처 파일 | `feature.arrow`지만 실제 writer는 **Arrow IPC stream**, 시각과 피처별 f64 열 | 확장자로 형식이나 mmap 가능성을 판단하면 안 됨 |
| 공통 피처 reader | 전 RecordBatch를 모은 뒤 전열을 소유 `Vec`로 복사 | 1열 요청에도 전열 읽기·복사가 생김 |
| 여러 날짜 로더 | 날짜별 Frame을 모은 뒤 새 배열에 이어붙임. 정렬도 새 배열 생성 | 파일을 mmap해도 소비자가 소유 배열을 요구하면 복사가 남음 |
| 공통 cross 피처 | 빌드 중 `Arc`로 공유하지만 출력에는 venue별로 반복 포함 | 동일 정의·입력인 열은 저장/계산 공유 후보. 절감률은 별도 측정 |
| DTW 전용 저장소 | 필요한 1열을 심볼별 파일로 추출, 날짜 오프셋으로 선택 읽기 | 프로젝트 안에 이미 읽는 바이트를 줄이는 선례가 있음 |
| DTW의 한계 | f32 저장 후 읽을 때 f64 배열 생성. 날짜 추가 시 전체 파일 재작성 | 정밀도·세대·증분 갱신을 갖춘 범용 저장소는 아님 |
| 다중 실험 | `onesec_bt`는 한 번 로드한 입력에 여러 신호를 실행하는 경로가 있음 | 별도 shared-memory 서버보다 먼저 비교할 간단한 기준 |
| 패킷 재생 | SPKR의 instrument별 chunk·footer index로 선택 읽기. APBT는 bytes를 읽고 사건 참조를 정렬 | 불변 입력·공통 색인은 공유 가능. 전략별 엔진 상태는 개별 소유 |

코드 근거는 13절에 기록했다. 여기서 확인한 복사 경로가 사용자가 겪은 지연의 몇 %였는지는 아직 계측하지 않았다.

### 2.2 실제 규모에 대한 기존 기록

2026-09-06 재생 보고서에는 당시 집계로 **1초 raw 약 506GB, 피처 캐시 약 712GB, 캐시 성공 39일**이 기록되어 있다. 이는 여러 venue와 제한된 심볼 범위의 산출물이며, 현재 저장량이나 1,500 instruments 전체의 실측은 아니다. 그래도 전체 캐시를 RAM에 넣는 것만으로 해결하기 어려운 규모라는 근거다.

2026-09-05 APBT 보고서는 241을 **Xeon 8173M 2개, 56물리/112논리 코어, RAM 767GB**로 기록한다. 당시 약 463GB 패킷을 로컬 디스크에서 읽어 90심볼 × 2변형 × 7일, 총 1,260런을 41분에 실행했다. 보고서는 SMB 병목이 해소됐다고 서술한다. **CPU·실행 조건이 통제된 NAS/SSD 비교가 아니므로, 41분을 SSD만의 개선 효과로 해석하지 않는다.**

같은 재생 작업 기록에는 로컬 디스크가 산출물로 가득 차 작업이 중단된 사건도 있다. 따라서 기존 시스템 드라이브에 수백 GB를 일괄 복사하는 제안은 부적절하다. 연구 볼륨·여유 공간·cache 상한·중단/회수 정책을 먼저 정해야 한다.

### 2.3 `loop`에서 가져올 수 있는 것

`loop`는 같은 backing을 Host와 VM이 공유하고, 확정 이력과 갱신되는 현재값을 구분한다. 검증 기록은 BTC 5분 봉 26,496행, 8MiB backing, 여러 VM의 읽기 전용 공유 수준이다. 수백 GB 통계 성능을 입증한 시스템은 아니다.

특히 현재 통계 소비 경로는 C에서 레코드 128B를 복사하고 Python 정수 목록으로 바꾼 뒤 전체 행 목록을 만든다. 공유 영역을 만들었다고 계산 끝까지 복사가 사라지는 것은 아니다. 차용할 부분은 **데이터 소유권·불변성·게시 순서·읽기 전용 공유**이며, 241의 같은 OS 프로세스에 ARM64 VM/ivshmem 장치를 도입할 필요는 없다.

## 3. 메모리 용량: 추가 데이터와 HFT를 분리해서 계산

가정: 1,500은 거래소·상품 종류를 구별한 **instrument 총수**다. 전 기간에 매 시각 한 행이 있는 조밀한 격자를 가정한다. 각 열은 8바이트 숫자, 단위는 decimal GB/TB다. 타임스탬프를 별도 저장하면 그만큼 열을 추가해야 한다. validity·색인·정정본·계산 결과·임시 배열·객체 overhead는 아래에 포함하지 않았다.

| 주기 / 기간 | 행 수 | 숫자 10열 | 숫자 20열 | 숫자 50열 | 숫자 100열 |
|---|---:|---:|---:|---:|---:|
| 1분 / 90일 | 1.944억 | 15.6GB | 31.1GB | 77.8GB | 155.5GB |
| 1분 / 180일 | 3.888억 | 31.1GB | 62.2GB | 155.5GB | 311.0GB |
| 1초 / 30일 | 38.88억 | 311.0GB | 622.1GB | 1.56TB | 3.11TB |
| 1초 / 90일 | 116.64억 | 933.1GB | 1.87TB | 4.67TB | 9.33TB |
| 1초 / 180일 | 233.28억 | 1.87TB | 3.73TB | 9.33TB | 18.66TB |

계산식은 `instrument 수 × 일수 × 하루 행 수 × 선택 열 수 × 8`이다. 압축률 예측이나 현재 Arrow 파일의 실측 크기가 아니다. 원시 틱은 종목별 사건 빈도가 달라 이 격자 공식으로 계산할 수 없다.

**실용적인 묶음의 예:** 1초 데이터 1,500개 × 7일 × 20열은 값 배열 약 145.2GB다. 하루 × 50열이면 약 51.8GB다. 이 정도 묶음을 읽고, 같은 입력에 여러 통계를 계산한 뒤 다음 묶음으로 넘어갈 수 있다. 여러 달을 분석한다는 요구가 여러 달의 전행을 동시에 RAM에 보유한다는 뜻은 아니다.

초기 RAM 정책 후보는 반복 입력 200~300GB와 제한된 계산 메모리다. 최종 상한은 전체 물리 메모리 사용량과 worker별 임시 배열을 측정해 정한다. OS·다른 서비스·새 segment 작성·압축 해제·정렬/조인에 여유를 남긴다. 동일 mmap 페이지를 프로세스 RSS마다 중복 합산하거나, OS 파일 cache와 공유 매핑을 서로 다른 데이터 사본으로 계산하지 않는다. 반대로 압축 파일 cache와 별도 해제 배열은 함께 존재할 수 있다.

## 4. shared memory·mmap의 정확한 역할

**둘은 양자택일이 아니다.** 로컬 파일을 여러 프로세스가 읽기 전용으로 매핑하는 것도 공유 페이지를 사용하는 방법이다. Windows에서도 일반 파일 또는 paging-file backing을 사용하는 file mapping이 가능하다. 다른 머신에 있는 소비자가 같은 물리 RAM을 공유하는 것은 아니다. [Windows File Mapping](https://learn.microsoft.com/en-us/windows/win32/memory/file-mapping), [Named Shared Memory](https://learn.microsoft.com/en-us/windows/win32/memory/creating-named-shared-memory).

| 방법 | 장점 | 남는 비용 / 제한 | 우리 용도 |
|---|---|---|---|
| 로컬 파일의 읽기 전용 mmap | 같은 파일 페이지 재사용, 필요한 부분 접근, 재시작 후 SSD에서 복구 | 최초 page fault·SSD 읽기, RAM 압박 시 재읽기, 소비자가 복사하면 이점 감소 | **불변 피처·이력의 반복 조회에 우선 후보** |
| paging-file/익명 공유 메모리 | writer가 만든 배열을 여러 프로세스가 공유, 파일 준비 과정 축소 가능 | 수명·게시·복구 관리 필요. 모든 페이지의 영구 RAM 상주 보장 아님 | 최신 제한 버퍼, 매우 자주 바뀌는 계산 결과 등에 후보 |
| 한 프로세스의 배열 + 여러 스레드 | 프로세스 간 전송·공유 객체 관리 없이 입력 재사용 | 프로세스 장애 범위 큼, RAM 상한·가변 상태 분리 필요 | **Rust 반복 실험의 첫 비교군** |

mmap 호출은 파일 내용을 즉시 전부 RAM으로 옮기는 작업이 아니다. 실제 접근 시 페이지가 필요해지고, 반복 접근은 남아 있는 페이지를 재사용한다. warm-up/prefetch로 미리 읽을 수 있으나 Windows의 `PrefetchVirtualMemory`도 메모리 여건에 따른 최적화 요청이다. 전체 데이터의 영구 상주 약속이 아니다. [Microsoft PrefetchVirtualMemory](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-prefetchvirtualmemory).

Parquet의 압축·인코딩된 bytes를 mmap한다고 바로 숫자 배열이 되지 않는다. 계산할 때 해제·변환이 필요하다. 반복 계산에서 그 비용이 지배적이면 선택한 열만 한 번 계산용 배열로 준비한다. Arrow도 **비압축 buffer와 이를 직접 참조하는 reader**라는 조건이 중요하다. 압축 IPC, 타입 변환, rechunk, 정렬, Python 목록 변환에는 새로운 메모리가 생길 수 있다. [Arrow IPC](https://arrow.apache.org/docs/python/ipc.html), [Arrow buffer·압축 명세](https://arrow.apache.org/docs/format/Columnar.html).

241의 이중 CPU 기록을 고려하면 NUMA도 측정 대상이다. 모든 페이지를 한 CPU 쪽에서 먼저 만들고 다른 CPU가 계속 읽는 배치가 유리하다고 가정하면 안 된다. 입력 파티션과 worker 배치를 맞추고 스레드 수를 단계적으로 늘린다. 공유 메모리는 용량 중복을 줄이지만, 여러 worker가 같은 전체 배열을 읽는 **메모리 대역폭**까지 늘려주지는 않는다.

## 5. 다른 방식들과 비교

이 절은 최초 조사 요약이다. **저장·실행 구조, 메모리 한계, 반복 재사용, 선택 조건을 확장한 비교는 14~20절**에 있다. 이 표만으로 후보의 우열을 결정하지 않는다.

| 후보 | 해결하는 문제 | 한계 | 판단 |
|---|---|---|---|
| 로컬 Parquet + DuckDB | 필요한 열/구간을 골라 SQL 집계·조인. RAM보다 큰 일부 연산은 SSD로 임시 데이터를 내리며 실행 | 매 실행 decode 비용, 복잡한 조인/정렬의 임시 공간. 모든 연산이 무제한 out-of-core인 것은 아님 | **전 기간 통계의 기본 비교 기준** |
| 로컬 Parquet + Polars lazy/streaming | 열·필터 선택, batch 실행, DataFrame 중심 연구 | 모든 연산이 streaming인 것은 아니며 일부는 메모리 실행으로 전환 | Python/Rust 연구 파이프라인 대안 |
| 비압축 열 segment + mmap | 반복 실험의 로드·해제·사본 생성 감소 | 디스크 사용량, 세대 관리, 실제 배열 뷰 지원 필요 | **반복 입력 재사용 경로 후보** |
| 상주 Rust worker + 스레드 | 한 번 로드하고 여러 설정/통계 실행, 기존 계산 코드 활용 | 변경 가능한 입력 공유 금지, 작업 취소·메모리 제어 필요 | **가장 먼저 비교할 간단한 재사용 방식** |
| 사전 집계·증분 통계 | 반복해서 같은 원행을 읽는 일을 줄임 | 새 통계 질문에 충분하지 않을 수 있음. 정의·정정·오차 정책 필요 | 저장 형식과 무관하게 우선 적용할 원칙 |
| QuestDB/ClickHouse형 분석 서버 | 데이터가 있는 노드에서 조회·집계하고 결과만 전송, 다중 소비자 서비스 | 이력 이관·스키마·운영·수집 자원 분리 필요. 원행을 전부 반환하면 네트워크 비용 그대로 | 다수 사용자·상시 조회가 중요해지면 비교 |
| Zarr형 다차원 chunk 배열 | time × instrument × feature의 부분 tensor 접근 | chunk 모양에 민감, 작은 파일 증가 가능, 비정형 사건 시각에는 추가 설계 | 학습 tensor/반복 window 추출이 핵심일 때 후보 |
| Ray형 공유 object store·분산 실행 | 작업 스케줄링·노드별 객체 재사용 | 같은 노드의 일부 배열만 직접 공유. 다른 노드에는 전송, 객체화·spill 비용 존재 | 단일 241에서 부족한 것이 확인된 뒤 검토 |
| NAS 근처 계산·네트워크 증설 | 원행 전송량 또는 최초 준비 시간 감소 | NAS CPU/디스크 경합, 계산 위치 관리. 반복 전열 복사 문제는 별도로 남음 | 실제 첫 준비 병목을 측정한 뒤 판단 |

DuckDB는 Parquet에서 열 선택과 필터를 reader에 내려 불필요한 읽기를 줄인다. 다만 모든 종목·기간·열을 요구하면 그만큼 읽어야 한다. 큰 정렬/그룹/조인에 spill을 지원하지만 일부 집계 상태와 복합 연산은 메모리 한계를 만날 수 있다. [Parquet 선택 읽기](https://duckdb.org/docs/stable/data/parquet/overview), [메모리 초과 작업과 제한](https://duckdb.org/docs/current/guides/performance/how_to_tune_workloads).

Polars는 lazy plan과 streaming을 제공하며 일부 연산의 메모리 실행 전환을 공식 문서에 설명한다. 따라서 최종 결과만 작다고 중간 메모리가 작다고 가정하지 않는다. [Polars Streaming](https://docs.pola.rs/user-guide/concepts/streaming/).

QuestDB도 시간 분할 열 저장·mmap·시계열 조인을 사용하는 비교 대상이다. 기존 인프라에 이름이 있다는 이유로 현재 NAS의 QuestDB를 HFT 통계 주 엔진으로 정하는 것은 별개의 결정이다. ClickHouse 같은 서버를 선택해도 연구 데이터와 계산을 빠른 로컬 저장소에 배치하는 원칙은 같다. [QuestDB Architecture](https://questdb.com/docs/architecture/questdb-architecture/), [ClickHouse 자체 운영 설치](https://clickhouse.com/docs/get-started/setup/self-managed/quick-install).

Zarr의 chunk와 shard는 선택 읽기 단위와 파일 수를 조절하는 아이디어로 유용하다. Ray의 배열 공유는 같은 노드 기준이며 원격 객체는 다운로드된다. 둘 다 NAS 전송을 자동으로 없애는 기술은 아니다. [Zarr 성능](https://zarr.readthedocs.io/en/stable/user-guide/performance/), [Ray Objects](https://docs.ray.io/en/latest/ray-core/objects.html).

### 읽기 자체를 줄이는 통계 설계

- 건수·합·최솟값·최댓값·평균은 block별 부분 결과로 합칠 수 있다. 분산/공분산은 수치적으로 안정적인 결합 공식을 사용한다.
- 같은 입력에 대한 임계값 여러 개·여러 horizon·여러 후보를 가능한 범위에서 한 번의 읽기에 묶는다. 연산량은 남지만 파일 재오픈·decode·복사는 줄어든다.
- rolling 계산에는 앞 block의 필요한 이력 또는 상태를 넘긴다. EMA는 단순 고정 warm-up만으로 비트 동일성이 보장되지 않으므로 정의된 시작 상태/checkpoint가 필요하다.
- 정확한 분위수·전역 순위·대규모 조인·경로 의존 시뮬레이션은 작은 요약만으로 대체되지 않을 수 있다. 외부 정렬/선택, 두 번 읽기, spill 또는 문제별 알고리즘이 필요하다.
- 근사 분위수나 표본은 별도 모드로 표시한다. 원래 요구한 정확한 통계를 몰래 근사하지 않는다.

## 6. kdb에서 차용할 핵심

### 6.1 왜 요구와 잘 맞는가

kdb의 참고 가치는 **열 기반 데이터, 시계열 연산, 현재와 이력의 분리, 통합 조회 진입점**을 함께 설계했다는 데 있다. 공식 tick architecture는 수집 로그·배포를 맡는 tickerplant, 현재 데이터를 다루는 RDB, 디스크 이력 HDB, 요청을 연결하는 gateway를 설명한다. 무거운 분석이 수집 경로를 잡지 않게 나누는 구조다. [KX 시스템 아키텍처](https://code.kx.com/q/architecture/).

| kdb 아이디어 | 우리 설계로 옮길 내용 |
|---|---|
| 열을 개별적으로 접근하는 splayed storage | 필요한 피처 buffer만 읽기. 파일 수는 우리 instrument·열 수에 맞게 묶음 단위 결정 |
| 날짜 partition, 여러 저장 장치의 segment | 시간 pruning, 부분 준비/복구, 작업 분배. 하루가 너무 크면 더 작은 불변 묶음 사용 |
| 저장한 열을 메모리에 매핑해 계산 | 파일과 계산 배열 사이의 불필요한 변환을 줄이는 반복 조회 경로 |
| symbol dictionary, 정렬/그룹 정보 | 문자열 반복 감소, instrument 범위 색인, 정렬된 시간 탐색 |
| vector 연산 | 행마다 객체를 만들지 않고 연속된 열을 묶어서 계산 |
| as-of/window join | HFT 사건 시각에 추가 데이터의 그때 사용 가능한 값을 결합 |
| TP/RDB/HDB/gateway 분리 | 내구성 있는 수집·제한된 최신 영역·불변 이력·하나의 조회 계약 |
| 데이터가 있는 프로세스에서 함수 실행 | 원행 전량을 연구 클라이언트로 보내기보다 계산 작업과 결과 전달 |

열 파일·partition·segment의 구조는 KX의 공개 저장 문서에서 확인할 수 있다. 이를 참고하되 **파일 하나당 열 하나를 모든 symbol/date에 그대로 곱하는 배치**를 정답으로 삼지는 않는다. 현재의 소파일 문제를 다시 만들 수 있으므로 manifest와 열 buffer 색인을 가진 묶음도 비교한다. [KX Database](https://code.kx.com/q/database/), [Partitioned tables](https://code.kx.com/q/kb/partition/), [열 기반 조회 최적화](https://code.kx.com/q/wp/columnar-database/).

### 6.2 kdb도 전체 이력을 RAM에 올려야 하는 모델은 아니다

디스크의 열·partition을 필요한 만큼 접근하는 구조 자체가 핵심이다. HFT 수개월 전량이 RAM에 안 들어간다는 이유로 포기할 필요가 없다. 반대로 저장소 이름을 바꿔도 NAS에서 모든 열을 반복 읽으면 그 바이트 비용은 남는다.

우리에게는 **디스크에서 처리 가능한 전체 이력 + RAM에서 재사용하는 선택 영역 + 제한된 최신 영역**의 결합이 맞다. 실시간 영역에서 이력 영역으로 넘어가는 기준은 시각과 게시 sequence로 명시하고, 24시간 시장에서는 UTC 구간을 지속적으로 닫을 수 있어야 한다. 날짜가 바뀔 때 대량 작업으로 수집을 멈추는 구조는 피한다.

### 6.3 시간 조인은 성능과 거래 시점 의미를 함께 설계

KX `aj`는 같은 키에서 기준 시각까지의 최근 행을 결합하며, 중복 시각의 선택 및 반환 시각에도 정의가 있다. DuckDB와 QuestDB에도 ASOF 기능이 있다. **이 연산 전체를 처음부터 구현해야만 시작할 수 있는 것은 아니다.** [KX aj](https://code.kx.com/q/ref/aj/), [DuckDB ASOF](https://duckdb.org/docs/current/guides/sql_features/asof_join), [QuestDB ASOF](https://questdb.com/docs/query/sql/asof-join/).

그러나 `event_time <= 거래 판단 시각`만으로 미래정보 사용을 막을 수는 없다. 예를 들어 12:00~12:01 봉은 12:00에 확정된 값이 아니다. 12:01 이후 실제 관측·조회 가능 시각을 조건으로 삼아야 한다. 정정된 과거값도 그 정정이 알려진 시각 이전에는 사용할 수 없다.

필수 정의는 **instrument 일치, 시간 단위, 사용 가능 시각, 허용 지연(tolerance), 중복 시각의 sequence, 정정 revision, 결측 처리**다. 원천 공개 시각을 알 수 없으면 보유한 관측 시각의 한계를 표시한다. 현재 `data-service`의 `view_seq`는 저장 snapshot이며 완전한 거래 시점 PIT를 제공하지 않는다.

## 7. 필요한 코어를 직접 만들 수 있는가

이 절은 저장·연구 실행부의 최소 범위다. 통합 플랫폼의 질의·프로세스·비동기·지속 계산 범위는 22~25절을 함께 적용한다.

**가능하다. 우리 작업에 필요한 저장·조회·시계열 연산의 범위를 정하면 현실적인 프로젝트다.** 이 절은 구현 가능 범위를 정리하며 자체 엔진 채택을 결정한 것은 아니다. 14~20절의 비교로 기존 엔진의 한계를 확인한 부분에 적용한다. 가장 어려운 부분은 파일에 숫자를 쓰거나 mmap하는 코드보다, 정정·결측·시간 의미·다중 읽기·메모리 한계·장애 후 복구를 일관되게 만드는 일이다.

### 7.1 자체 명세의 최소 범위

아래는 제안 명세다. 아직 API나 디스크 형식으로 확정/구현한 것은 아니다.

| 코어 | v1에서 정의할 동작 | 검증할 핵심 |
|---|---|---|
| 데이터/상품 식별 | dataset, venue, market, instrument의 안정 ID와 표시명. 스키마·단위·scale 별도 관리 | 재상장·이름 변경·다른 venue를 같은 심볼로 혼합하지 않음 |
| 타입 배열 | i64 시각/정확한 scaled 값, 필요한 정수·f64 피처, 명시적 null/품질 | 기존 f64 원본은 그대로 보존. f32 축소나 정확한 가격의 float 변환을 자동 수행하지 않음 |
| 불변 segment | 열 buffer, 행 수, dtype, 정렬 범위, instrument 범위/offset, checksum, 생성 버전 | offset/length 검증, 부분 파일 거부, 64비트 전체 행/바이트 위치 |
| manifest와 snapshot | 어떤 segment·revision·정의 조합인지 고정. 데이터군별 coverage 포함 | 과거/최신 경계의 누락·중복 방지, 실행 중 세대 혼합 방지 |
| 선택 읽기 | instrument·기간·columns를 저장소 단계에 적용, chunked column view 반환 | 사용하지 않는 열을 소유 배열로 읽지 않음. 같은 값/순서 유지 |
| 실행 | 필터, 구간 집계, group, backward as-of, 필요한 rolling/lag | 결측·중복 시각·경계·분할 실행이 전체 실행 정의와 일치 |
| RAM·SSD 예산 | 작업 전 예상 입력/임시/결과 크기, 동시성·prefetch·cache 상한 | 초과 요청을 분할/대기/거부. 디스크 가득 참·취소 후 회수 |
| 최신/이력 연결 | bounded 최신 batch, 내구성 로그 또는 기존 영속 저장 참조, seal 후 이력 게시 | crash 후 재생·중복 제거·게시 watermark 일관성 |
| 소비자 전달 | 같은 논리 요청으로 작은 table, batch iterator, 로컬 읽기 핸들, 계산 job | 큰 요청을 자동 전량 JSON/Python 객체로 만들지 않음 |

instrument 범위 색인과 정렬 정보만으로 필요한 첫 기능을 시작할 수 있다. 범용 B-tree/bitmap/모든 q attribute를 동시에 만들 필요는 없다. 116억 행을 한 배열·한 batch·32비트 row index에 밀어넣지 않고 segment 내부와 전역 offset의 범위를 구분한다.

### 7.2 직접 만들 부분과 기존 구현을 활용할 부분

**직접 소유할 부분:** 시장 데이터 계약, source/replay/feature 버전, snapshot 게시·수명, cache 준비와 회수, 시점 의미, HFT 입력 어댑터, 작업 배치·예산, 실제 병목인 전용 계산 kernel.

**검증된 구현을 활용할 부분:** Arrow/Parquet 직렬화와 압축, 표준 타입·버퍼 관리, 범용 SQL 필터/집계/조인, OS file mapping. 처음에는 DuckDB 또는 Polars를 기준 실행기로 두고 자체 kernel과 동일 결과를 비교할 수 있다. DuckDB 내부 파일을 택할 경우 일반 embedded 방식의 다중 프로세스 쓰기 제약까지 설계해야 하므로, 불변 파일 조회와 작업 소유권을 구분한다. [DuckDB 동시성](https://duckdb.org/docs/current/connect/concurrency).

**별도 제품 규모가 되는 범위:** q 언어·실행기 호환, 모든 qSQL/내장 함수, kdb 바이너리 파일/IPC 호환, 범용 최적화기, 임의 UDF 격리, 분산 복제·고가용성 전체. 사용자가 요구한 방식 차용에는 이 전체 호환성이 필요하지 않다. 공개 구조와 동작을 바탕으로 우리 요구의 독립 명세를 작성한다.

v1을 위 최소 범위로 닫으면 단계별 구현과 검증이 가능하다. 수개월 운영·정정·장애·스키마 진화까지 검증된 코어는 PoC보다 훨씬 큰 작업이다. 현재 자료만으로 완료 날짜나 kdb와 같은 성능을 약속할 수 없다. **첫 실제 HFT 조회와 반복 실험을 통과시키고, 측정으로 필요한 기능만 추가하는 방식**을 권한다.

### 7.3 제안 조회 계약

아래는 연구 데이터 접근의 부분 계약이다. 전체 사용자 인터페이스에는 22절의 함수 실행·지속 질의와 24절의 비동기/액션 계약이 추가로 필요하다.

하나의 SDK에서 다음 의미를 공통으로 다루는 것이 목표다. 아래 이름은 설명용이다.

```text
요청: dataset + instruments + [start, end) + columns
      + snapshot 또는 latest + quality 조건 + temporal mode

prepare → 준비 작업 ID, 입력/예상 바이트, 진행률, 준비된 snapshot
scan    → 제한된 크기의 typed batch 또는 로컬 column view
compute → 집계/조인/연구 작업 ID, 진행률, 결과
latest  → 같은 스키마의 최신값과 신선도/품질
```

`prepare`가 반환한 immutable snapshot을 분석에 고정한다. `latest`는 요청 시점에 어떤 snapshot/watermark로 해석했는지 알려준다. 여러 데이터군의 snapshot은 단일 원천 시각을 가정하지 않고 **각 입력 버전·관측 경계의 조합**으로 표현한다.

원격 PC에는 241의 로컬 포인터를 전달할 수 없다. 로컬 worker는 핸들/manifest로 열고, 원격 소비자는 batch 전송 또는 작업 결과를 받는다. 사용자에게는 같은 데이터 선택 규약을 제공한다. UDP 구독 여부는 live 전달의 별도 결정이며, 역사 데이터 준비·재전송을 UDP 하나로 통일할 이유는 없다.

### 7.4 자체 실행기의 첫 연산은 어떻게 만들 것인가

예를 들어 “전 종목 90일 중 spread 조건에 맞는 행의 수익률 평균”이라면 다음 순서로 실행한다.

```text
snapshot 고정
→ manifest에서 날짜·instrument 묶음 선택
→ 조건 열과 결과 열만 열기
→ 제한된 batch에서 조건 mask/행 위치 계산
→ 종목별 count·평균 상태 갱신
→ 부분 상태 병합
→ 작은 결과 테이블 반환
```

전체 90일을 이어붙인 거대한 Frame이나 필터된 모든 행의 복사본을 먼저 만들 필요가 없다. row 위치를 담는 선택 벡터 또는 bit mask도 크기를 예산에 포함하며 batch 단위로 재사용한다. 결과 행 자체를 요청했다면 필요한 결과 buffer는 별도로 생긴다.

첫 backward as-of kernel은 같은 instrument 안에서 정렬된 두 입력을 앞으로 진행하며 오른쪽의 마지막 유효 행을 유지하는 방식으로 만들 수 있다. 필요한 정렬·revision 해석이 끝난 입력에서는 대략 두 입력의 행 수 합에 비례해 처리할 수 있다. 정렬 비용은 별도이며, 작은 표본 질의는 색인 탐색이 더 나을 수 있다. 구간 시작 전의 마지막 유효 행을 가져오는 경계 처리, tolerance, 중복 시각 선택을 명세·테스트로 고정한다.

실거래 시점 재현은 이 단일 시간 조인만으로 끝나지 않는다. 먼저 해당 판단 시각에 알려진 revision을 해석한 뒤 유효 사건을 선택해야 한다. 늦게 도착한 과거 봉이 “가장 최근 관측된 행”이라는 이유만으로 최신 시장 상태를 대체하지 않게 해야 한다.

초기 실행 계획은 제한된 연산의 조합과 명시적인 메모리 상한으로 충분하다. 자동 비용 최적화기·JIT·범용 분산 실행은 첫 정확한 scan/aggregate/as-of의 선행 조건이 아니다. 이 범위로 시작하면 저장·계산 인터페이스를 검증하면서 기능을 늘릴 수 있다.

## 8. 연구 경로의 배치와 데이터 수명

아래는 연구 경로의 부분 배치다. **실거래·구독·비동기 gateway를 포함한 전체 배치는 23절**에 있다. 실행 엔진과 저장 형식은 플랫폼 계약 아래에서 14~20절의 비교로 선택한다. 반복 입력은 엔진 cache·상주 배열로도 재사용할 수 있으며 mmap은 조건부 확장이다.

```mermaid
flowchart TB
    A["추가 데이터 수집 / HFT 원본·재구성"] --> B["불변 이력 + 원천·처리 버전"]
    B --> C["NAS 아카이브"]
    B --> D["공통 catalog·snapshot·준비 API"]
    C -->|"필요한 묶음만 최초 준비"| E["241 연구 볼륨: 열 기반 파일 또는 native DB"]
    D --> E
    E --> F["전 기간: 선택 읽기 + 분할 계산"]
    E --> G["반복 입력: 엔진 cache / 상주 배열 / 필요 시 mmap"]
    G --> H["241 상주 worker / 여러 연구 작업"]
    F --> I["통계 결과 / 필요한 batch"]
    H --> I
    J["연구 클라이언트"] -->|"데이터 선택·작업 요청"| D
    I --> J
```

**컴포넌트 배치:** Full Trading 안의 독립 `data-service`에 공통 catalog·snapshot·준비/조회 계약을 두고, 241의 연구 worker를 별도 실행 단위로 둔다. 라이브 수집기와 연구 worker의 CPU·RAM·I/O 예산을 분리한다. 241은 필요할 때 켜는 머신이라는 운영 기록이 있으므로 상시 수집의 유일한 저장처로 의존하지 않는다.

HFT 패킷 복원·호가장·feature 정의는 기존 HFT 컴포넌트가 책임진다. 공통 코어는 그 산출물과 의미/버전을 어댑터로 받아 재사용한다. 이 확장으로 `data-service`가 모든 HFT 전략 코드와 mutable 엔진 상태를 소유하게 만들 필요는 없다.

### 파일 형식과 배치 후보

- **분석 이력:** Parquet의 압축·열 선택 경로를 기본 비교 대상으로 둔다. 쿼리 형태에 따라 native 분석 DB와도 비교한다.
- **반복 계산 영역:** 비압축 Arrow IPC 또는 명세가 있는 고정폭 열 buffer. 직접 배열 뷰를 제공하는 reader인지 실측한다.
- **호환성:** 현재 Rust writer가 IPC stream을 선택한 이유는 당시 Julia reader와의 호환성 문제였다. IPC file을 새 연구 형식으로 쓰려면 Rust·Python·Julia의 실제 사용 버전으로 왕복/직접 읽기를 검증한다. 기존 파일을 이름만 유지한 채 바꾸지 않는다.
- **파티션:** 날짜/시간 묶음 + instrument 묶음 + 열/열군. 시계열 연구는 instrument별 연속 구간, 전 종목 횡단면은 시간별 묶음이 유리하다. 두 접근을 시험하고 자주 쓰는 것만 보조 배치로 만든다.
- **탐색:** manifest의 segment 목록과 통계로 선택한다. 요청마다 NAS 전체 디렉터리와 모든 파일을 `stat`하지 않는다.
- **압축 선택:** 초기 준비·decode·반복 횟수·SSD 공간을 함께 비교한다. 한두 번 읽을 자료까지 전부 비압축 사본으로 만들지 않는다.

### 게시·정정·회수

1. 완성되지 않은 파일은 준비 중 상태로 둔다. 내용·schema·행 수·hash를 확인하고 durable manifest에 게시한다.
2. 독자는 시작 시 snapshot을 고정한다. 읽는 파일의 내용을 덮어쓰거나 길이를 줄이지 않는다.
3. 늦은 데이터·정정은 새 segment/revision으로 게시한다. 정리 작업 뒤에도 고정된 과거 snapshot을 재현할 수 있어야 한다.
4. 독자가 사용 중인 매핑은 lease/refcount 등으로 추적한다. Windows의 열린 매핑 수명도 고려해 이전 파일을 나중에 회수한다.
5. manifest 자체는 원자적 교체와 재시작 복구 규칙을 갖춘다. rename만 성공하면 전원 손실 내구성까지 보장된다고 간주하지 않는다.
6. checksum은 준비·게시·검증 시 사용한다. 실험을 시작할 때마다 수백 GB 전체를 재해시하지 않는다.

## 9. HFT에 적용할 때의 필수 조건

### 피처 통계 경로

현재 `Frame { Vec... }` 소유 모델과 read/concat를 그대로 둔 채 mmap만 추가하지 않는다. 새 연구 경로는 **chunked column view**를 받아 계산하고, join/정렬/수정이 필요한 결과에만 작업 배열을 만든다. mutable 학습 전처리는 읽기 전용 입력과 분리한다.

공통 cross 피처는 정의·입력 버전·lag·시간 격자가 정말 같을 때 한 번만 저장한다. 여러 종목/venue에 복제된 열을 manifest에서 조합할 수 있다. 같은 이름이어도 계산 규칙이 다르면 같은 열로 취급하지 않는다.

기존 DTW f32 저장소는 아이디어의 선례지만 범용 정확값 저장소로 그대로 승격하지 않는다. 새 조회는 원본 f64와 정확한 정수의 dtype를 보존한다. 용량 절감을 위한 f32 변환은 허용 오차가 정해진 별도 파생 dataset으로 다룬다.

### 패킷 재생 경로

SPKR의 instrument별 chunk 선택과 사건 순서를 유지한다. 같은 심볼·날짜의 실험들은 불변 bytes와 검증한 순서 색인을 재사용할 수 있다. 원본 chunk를 mmap하는 것과 호가장·가상 주문·전략 상태를 공유하는 것은 다르다. **전략별 mutable 상태는 독립**이어야 한다.

한 번의 사건 읽기로 여러 전략 상태를 진행시키는 방식도 비교할 만하다. 다만 전략별 연산량·실패 격리·메모리 사용량이 커질 수 있으므로 mmap worker 방식과 실제로 비교한다. 1초 피처에서의 속도 개선을 패킷 재생 속도로 일반화하지 않는다.

### 의미를 바꾸면 성능 개선으로 인정하지 않는다

- 재생본과 라이브본의 원천·수신 시각·호가 복원 버전 차이를 보존한다.
- 결측 날짜, 상장 기간, 늦은 파일, duplicate sequence, warm-up, ffill, cross lag를 명세에 포함한다.
- HFT raw의 일부 flow 값은 이미 trailing/carry 집계다. 다시 단순 합산하면 이중 집계가 된다.
- rolling/forward label의 날짜 경계를 이어야 한다. 미래 결과를 설명변수의 사용 가능 시각으로 착각하지 않는다.
- snapshot, 원본 세대, replay engine 버전, feature 정의 hash, 열·dtype·기간을 cache key에 포함한다.

## 10. 구현 전 비교 실험

현재 수행한 것은 코드/문서 검토와 용량 산술이다. 아래 실험은 **아직 실행하지 않았다**. 241을 켜거나 설정을 바꾸거나 NAS의 대량 데이터를 옮기지 않았다.

### 10.1 데이터와 질의 고정

추가 데이터, HFT 1초 피처, SPKR 패킷을 별도 세트로 둔다. 실제 값 분포·결측·늦은 데이터·정정·여러 instrument를 포함한 작은 표본에서 시작한다. 같은 상수 행을 복제한 압축률을 실제 데이터 크기로 사용하지 않는다.

필수 질의는 다음과 같다.

1. 모든 대상 × 여러 날짜에서 **2~5열만** 읽어 종목별 집계.
2. 20~50열의 조건부 통계·상관·여러 horizon 계산.
3. 추가 데이터와 HFT를 사용 가능 시각으로 backward as-of join.
4. 같은 입력에 대한 여러 파라미터/전략 반복 실행.
5. RAM보다 큰 기간의 정확한 집계와 중간 상태 병합.
6. 동일 SPKR 입력을 여러 전략이 재사용하는 사건 재생.

### 10.2 비교군

| 비교군 | 확인할 것 |
|---|---|
| 현재 reader, NAS 표본 | 현재 대기 시간과 읽는 바이트의 기준 |
| 현재 reader, 같은 표본의 로컬 SSD | 네트워크/소파일 비용과 전열 복사 비용 분리 |
| 한 번 로드 + 같은 프로세스 다중 실험 | shared memory 서비스 없이 얻는 재사용 효과 |
| 로컬 Parquet + DuckDB/Polars | 선택 읽기·분할 계산의 기준 성능 |
| 비압축 열 segment + mmap + 직접 뷰 | decode·복사 제거의 효과와 page fault 비용 |
| 필요할 때만 별도 공유 배열 | 파일 mapping보다 이득이 있는 실제 live/계산 구간 |

기존 운영 위치를 바꾸지 않고 한정된 표본과 별도 연구 영역으로 비교한다. 크기는 작은 정확성 표본 → 약 10~20GB → 약 100GB → 목표 작업 크기로 늘린다. 각 단계는 앞 단계의 정확성·용량 예산이 통과해야 진행한다.

### 10.3 기록할 지표와 통과 기준

- **첫 준비 / 재시작 후 첫 읽기 / RAM에 남은 반복 읽기 / 증분 갱신**을 분리한다. cache 상태와 빌드 모드를 기록한다.
- NAS 전송 bytes, 로컬 디스크 읽기 bytes, 실제 선택 열 bytes, decode 시간, 계산 시간, peak 물리 메모리·private 메모리, page fault, 임시 파일 크기를 측정한다.
- worker 1·4·8개부터 시작해 CPU/NUMA 배치와 병목을 본다. 과거 APBT의 56병렬이 모든 통계에 적합하다고 가정하지 않는다. worker 내부 스레드 수까지 합산한다.
- 같은 snapshot에서 행·키·dtype·정확한 정수는 일치해야 한다. 부동소수 집계는 연산 순서 차이를 고려한 사전 허용 오차와 재현 모드를 정한다.
- 동일 snapshot의 로컬 cache가 유효하면 반복 실행의 **원자료 NAS 재전송은 0**이어야 한다. 모든 페이지가 RAM에 남아야 한다는 조건과는 다르다.
- 열을 줄인 요청에서 실제 읽기량이 줄어야 한다. 전체 Arrow를 읽고 반환만 2열로 줄이는 것은 통과가 아니다.
- 메모리를 넘는 요청은 계획된 분할/spill/거부가 되어야 한다. OOM·무제한 pagefile·무제한 SSD 증가로 끝나면 실패다.
- 정정 게시 중 기존 독자의 결과가 바뀌지 않아야 한다. 중단·재시작·손상 파일·디스크 부족·취소와 매핑 회수를 검증한다.
- 수집/최신 조회의 지연을 함께 측정한다. 연구 성능을 위해 최신 수집을 직렬로 막으면 통과가 아니다.

초 단위 목표는 대표 질의와 연구 볼륨·현재 링크 속도를 확인한 뒤 정한다. 이번 조사만으로 1,500개 수개월 분석 시간을 보장하지 않는다.

## 11. 실제 작업 순서 제안

통합 플랫폼의 전체 진행 순서는 26절로 보완한다. 아래는 그 안의 연구 I/O 작업이며, 질의·비동기·구독 명세를 연구 성능 확인 뒤로 미루지 않는다.

| 단계 | 결과물 | 다음 단계로 넘어가는 조건 |
|---|---|---|
| **A. 명세·기준 데이터** | 7절 및 22~25절의 공통 의미/실행 계약, HFT/추가 데이터 매핑, 대표 질의, 241 볼륨·RAM 예산 | 연구와 실거래의 관계, 같은 질문의 정답·데이터 버전·측정 방법이 고정됨 |
| **B. 한정된 비교 구현** | 현재 reader/로컬 재사용/Parquet/mmap을 같은 표본으로 비교 | 어떤 비용을 줄였는지 실제 bytes·시간·메모리로 설명 가능 |
| **C. 공통 읽기 계층** | 선택한 기존 엔진 또는 필요한 자체 reader 위에 manifest·snapshot·열/기간 선택·bounded scan, 연구 소비자 1개 | 기존 결과 재현, 불필요한 전열 복사 제거, RAM 초과 처리 |
| **D. 241 반복 실행** | 상주 worker, 묶음 실험, cache 수명·예산·복구, job/result API | 반복 NAS 재전송 제거, 취소·증분 갱신·메모리 한계 검증 |
| **E. 현재/이력 통합** | 최신 영역과 seal된 이력의 조회 경계, 정정·시점 계약 | 경계 중복/누락·시점 혼입 없이 추가 데이터와 HFT 결합 |
| **F. 확장** | 필요한 전용 시간 조인/rolling kernel, 더 큰 규모/다중 사용자 | 기존 실행기로 부족한 지점을 측정한 뒤 범위를 결정 |

기존 `data-service` M1.1의 수집/연구 자원 분리는 유지한다. M1.2는 **추가 데이터만의 SSD 내보내기**에서 **HFT에도 적용 가능한 공통 연구 데이터 경로 검증**으로 확장하는 것이 맞다. HFT의 어댑터와 새 소비자 한 개부터 연결하고, 기존 수집·학습·전략을 일괄 전환하지 않는다.

먼저 플랫폼의 최소 명세를 고정하고, A와 B를 통해 C의 연구 읽기를 기존 엔진 어댑터로 충분히 구현할 수 있는지 판단한다. **플랫폼 계약은 우리가 소유하고, 저장·계산을 직접 만드는 범위는 역할별 검증으로 정한다.** 연구 I/O 기준은 20절, 실거래·비동기·구독을 포함한 필수 기준은 26절에 있다.

## 12. 지금 정할 것과 남겨둘 것

**지금 반영할 방향:**

- kdb의 열·시간 분할·벡터 계산·현재/이력 분리·통합 gateway 아이디어를 차용한다.
- 대량 연구 준비·계산은 241의 한정된 연구 볼륨과 RAM 예산 안에서 수행한다. 상시 실거래 경로는 별도 자원으로 운영한다.
- 전 기간 scan 경로와 반복 입력 공유 경로를 모두 지원한다.
- HFT 피처, HFT 패킷, 추가 데이터는 공통 snapshot 규약 아래 각각 맞는 reader로 연결한다.
- 공통 질의·함수 정의를 과거 실행·실시간 구독·재생에 연결한다. 비동기 요청과 상태 변경 액션의 완료·재시도 의미를 정한다.
- 연구 I/O의 첫 성과는 **필요한 입력을 한 번 준비해 여러 분석이 같은 정확한 값을 재사용하는 것**으로 둔다. 통합 플랫폼의 첫 검증은 26.3처럼 과거 실행·비동기·구독·복구를 함께 포함한다. 이를 수행할 독자 저장·범용 계산 엔진의 필요성은 아직 확정하지 않았다.

**측정 뒤 결정할 것:**

- Parquet/Arrow IPC/고정 열 buffer의 실제 조합, segment 크기와 열군.
- RAM 상주 범위, NUMA/worker 수, 로컬 SSD 공간과 임시 공간 상한.
- DuckDB/Polars 연산을 재사용할 범위와 자체 kernel이 필요한 범위.
- 네트워크 증설, 별도 분석 서버, 여러 계산 노드의 필요성.

## 13. 검토 근거와 한계

아래는 로컬 검토에서 확인한 **레포 상대 경로**다. 공개 저장소에 소스·운영 로그를 함께 게시한 것은 아니다. 실행 시간·용량은 해당 과거 보고서의 기록이며 이번에 재측정하지 않았다.

| 근거 | 위치 |
|---|---|
| HFT raw 스키마·슬롯·복사 | `hft-oms_rust/crates/runner/src/record.rs:45`, `crates/cache-builder/src/rawload.rs:18`, `:73`, `:157` |
| IPC stream 선택 이유·전열 reader | `hft-oms_rust/crates/cache-builder/src/writer.rs:1`, `:49` |
| 다일 concat·정렬 사본 | `hft-oms_rust/crates/fit/src/cacheload.rs:16`, `crates/fit/src/frame.rs:66`, `:81` |
| cross 공유와 출력 | `hft-oms_rust/crates/cache-builder/src/build.rs:123`, `:492` |
| 기존 단일 열 저장소 | `hft-oms_rust/crates/analyzer/src/dtw/colstore.rs:1`, `:57`, `:79` |
| 한 번 로드·여러 신호 | `hft-oms_rust/crates/fit/src/bin/onesec_bt.rs:187`, `:269` |
| 패킷 선택 읽기·APBT 정렬 | `hft-oms_rust/crates/replay/src/lib.rs:397`, `crates/analyzer/src/source/apbt.rs:426` |
| 기존 241 하드웨어·로컬 패킷 실행 | `hft_research/docs/reports/APBT_ALL_SYMBOLS_E0_vs_E1f0_2026-09-05.md:5`, `:207` |
| 재생 데이터 용량·결손·원본 차이·디스크 부족 기록 | `hft_research/docs/reports/REPLAY_V2_REGEN_2026-09-06.md:6`, `:22`, `:33`, `:39` |
| HFT raw flow의 집계 의미 | `hft_research/crates/rlab/src/raw.rs:7` |
| loop의 공유 backing·읽기와 Python 사본 | `loop/examples/binance-ivshmem/native/shared.c:21`, `:64`, `shared.py:47`, `council_tool.py:32` |
| loop의 실제 검증 범위 | `loop/examples/binance-ivshmem/RESULTS.md:6`, `:68`, `loop/docs/PROJECT_GUIDE.md` §7~8 |

외부 기술 설명은 각 절에 연결한 공식 문서를 기준으로 했다. kdb 실행파일의 내부 구현을 검사하거나 성능을 비교 실행하지 않았다. 소스·프로토콜·운영 데이터·기존 실행 프로세스는 변경하지 않았으며, 이 문서와 관련 로컬 안내만 갱신한다. 공개본에는 실제 로컬 절대경로·접속 정보·전략 성과표·원시 데이터를 포함하지 않는다.

<a id="alternatives-deep-dive"></a>

## 14. 대안 심화: 무엇을 같은 기준으로 비교할 것인가

**kdb가 다른 모든 방식보다 낫다고 결론 낼 근거는 없다.** 최초 조사는 kdb·mmap·공유 메모리에 비해 다른 후보의 검토가 얕았다. 이번 보완은 주요 대안을 실제 데이터 이동 경로와 실행 방식까지 비교한다. 모든 제품을 망라하거나 우리 장비에서 성능 순위를 측정한 조사는 아니다.

### 14.1 서로 다른 층을 구분한다

| 층 | 후보 / 결정 | 줄이는 비용 | 이것만으로 해결되지 않는 것 |
|---|---|---|---|
| 데이터 배치·보관 | Parquet, Arrow IPC, 엔진 native 저장, Zarr | 선택 열·구간 읽기, 압축, 파일 수 | 계산 알고리즘·결과 크기 |
| 질의 실행 | DuckDB, Polars, DataFusion, ClickHouse, QuestDB | 불필요한 연산 제거, 병렬 집계, 일부 연산 spill | 물리 링크 속도·모든 연산의 메모리 초과 |
| 입력 재사용 | OS 파일 cache, 상주 배열, mmap, object store | 반복 디스크 읽기·decode·사본 | 최초 준비·RAM보다 큰 전체 입력 |
| 작업 배치 | Rust worker, Dask, Ray | 입력 가까이 작업 실행, 병렬 자원 관리 | 읽을 bytes·shuffle·전략별 상태의 자동 소멸 |
| 계산량 자체 축소 | 부분 집계, 증분 계산, 중간 결과 재사용 | 같은 원행의 재방문 | 새로운 질문·정정 시 재계산 |

따라서 “Parquet 대 mmap 대 kdb” 하나를 고르는 문제로 보면 안 된다. 예를 들어 **Parquet + DuckDB + 상주 작업 프로세스**도 가능하고, **Arrow IPC + Rust worker + 부분 집계**도 가능하다. 공유 메모리를 도입하지 않고 입력을 한 프로세스에서 재사용하는 구성도 독립적인 유력 대안이다.

### 14.2 작업을 일곱 종류로 나눈다

| 작업 | 구체적인 질문 | 중요한 비용 / 보존해야 할 의미 |
|---|---|---|
| W1 전 기간 통계 | 전 instrument 수개월, 3~10개 피처의 분포·조건별 수익 통계 | 선택 열 scan, 그룹 상태, 정확/근사 구분 |
| W2 시간 결합·rolling | 1초 피처에 당시 알려진 OI·funding·시장 상태 결합 | 정렬, 종목별 상태, 경계 이력, 정보 사용 가능 시각 |
| W3 반복 실험 | 같은 입력으로 수십~수백 설정 비교 | 반복 로드·decode·중간 피처 재계산 |
| W4 학습 입력 | time × instrument × feature window 반복 추출 | 접근 모양, chunk, batch 생성·전송 |
| W5 공유 조회 | 여러 소비자의 최신값·이력·집계 요청 | 동시성, 갱신 지연, 자원 격리, snapshot |
| W6 패킷 재생 | 틱·호가 사건 순서로 전략/체결 상태 재생 | 사건 순서·동시각 tie-break, 전략별 가변 상태 |
| W7 실거래 지속 계산 | 데이터 도착마다 피처·조건 결과를 갱신하고 전략에 전달 | 구독 연속성, 늦은 데이터, 상태 복구, 처리 지연, live/replay 일치 |

한 엔진이 W1에 좋다는 사실은 W6·W7도 대체한다는 뜻이 아니다. 사용자 API는 통일하되 요청의 의미와 반환 형태를 구분해야 한다. `scan`은 제한된 행 batch, `aggregate`는 작은 통계, `prepare`는 재사용 가능한 입력 식별자, `job`은 계산 결과, `subscribe`는 지속적인 갱신을 제공하는 식이다. 이는 제안이며 외부 프로토콜 변경은 아니다.

### 14.3 이미 진행된 Arrow 변환과의 관계

별도 작업의 [Rust Arrow 변환 실측 보고서](https://github.com/sicarius01/codex-docs/blob/aadc7bf/quant-research/nas-arrow-conversion-benchmark-2026-09-16.md)는 기존 벡터를 **48컬럼 무압축 IPC FILE**로 변환하는 경로를 확인했다. 따라서 새 비교가 반드시 원본부터 다시 변환하는 작업으로 시작할 필요는 없다. 이미 검증된 변환본을 입력 후보로 포함한다. 2절의 기존 HFT writer/reader 문제와 별도 변환 산출물의 존재는 구분한다.

그 보고서의 8개 파일·203.03MB 표본에서 최종 Rust 변환의 중앙값은 NAS 2스레드 8.95초, 워크스테이션 8스레드 8.61초였다. 전송·검증을 포함한 변환 작업이며, 서로 같은 코어 수나 완전히 통제된 부하 조건은 아니다. **통계 질의·mmap·DB 엔진 비교 결과로 사용할 수 없다.** 다만 워크스테이션의 원격 읽기/반환과 NAS의 검증 계산이 서로 다른 병목이라는 근거는 된다. 현재 이관을 바꾸거나 그 산출물을 재생성하자는 제안은 아니다.

이후의 엔진 특성은 공식 문서에서 확인한 기능이고, **우리 작업에 대한 적합성·우선순위는 그 기능과 현재 코드로부터 도출한 판단**이다. 후보 간 성능 수치는 아직 없다.

## 15. 단일 머신에서 충분히 해결하는 대안

### 15.1 DuckDB + 로컬 Parquet: 전 기간 통계의 첫 기준

**실행 경로:** 필요한 파일/row group/열 선택 → 압축 해제한 batch → 필터·집계·조인 → 작은 결과 또는 분할 출력. Parquet의 통계와 partition으로 제외할 수 있는 부분을 건너뛴다. 전 instrument·전 기간을 요청하면 기간/종목 필터의 이득은 작다. 이때 주효한 것은 **50열 중 5열만 읽기, 원행 대신 집계 결과 반환하기**다. [Parquet 읽기와 pushdown](https://duckdb.org/docs/current/data/parquet/overview).

우리에게 좋은 이유는 별도 저장 엔진을 만들지 않고 W1의 기준 구현을 만들 수 있다는 점이다. 데이터가 RAM보다 크다는 이유만으로 탈락하지 않는다. 다만 큰 그룹·정렬·조인의 중간 상태, 일부 집계 함수와 복합 blocking 연산은 메모리 한계가 있다. `memory_limit`도 프로세스 전체 RSS를 엄격하게 봉쇄하는 값은 아니다. 스레드·동시 job·임시 SSD 공간까지 예산에 포함해야 한다. [메모리 관리](https://duckdb.org/2024/07/09/memory-management), [OOM 주의점](https://duckdb.org/docs/current/guides/performance/oom).

**반복 실행의 한계:** 파일 bytes가 OS cache에 남는 것과 계산 가능한 모든 배열이 영구 보관되는 것은 다르다. 같은 Parquet의 같은 열을 반복 해제하는 비용이 크면 15.2 또는 15.4와 비교한다. 반대로 선택 열이 적고 계산이 무겁다면 별도 mmap 포맷 준비 비용이 회수되지 않을 수 있다.

**도입 시 주의할 DuckDB Rust API:** `query_arrow()`는 iterator 형태지만 전체 결과를 버퍼링한다. 큰 결과에는 `stream_arrow()` 또는 파일 출력 경로를 검토해야 한다. 엔진 내부가 분할 실행해도 API 어댑터가 전행을 모으면 다시 RAM 문제가 생긴다. [Rust 결과 처리](https://duckdb.org/docs/current/clients/rust/result_handling).

**선택 조건:** SQL로 표현되는 전 기간 통계·필터·그룹·시간 결합이 주력이고, 매번 거대한 중간 테이블을 소비자에게 넘길 필요가 없을 때. **탈락 조건:** 실제 대표 질의의 중간 상태/반복 decode가 목표를 초과하고, 질의·분할 조정으로도 해결되지 않을 때다. ASOF도 지원하지만 시각·동률·유효기간 규약은 우리가 고정해야 한다. [ASOF JOIN](https://duckdb.org/docs/current/guides/sql_features/asof_join).

### 15.2 DuckDB native DB: 자체 열 저장소보다 먼저 비교할 후보

이 후보는 Parquet를 매번 직접 읽는 것과 별개다. **한 번 로컬 DB에 적재하고 반복 조회**하면 엔진 자체 저장·통계·buffer 관리와 최적화 경로를 사용할 수 있다. 공식 가이드도 native 형식과 Parquet의 장단점을 구분한다. “DuckDB는 Parquet를 다시 해제하니 자체 코어가 필요하다”는 논리는 이 후보를 빠뜨린 것이다. [파일 형식별 성능 고려](https://duckdb.org/docs/current/guides/performance/file_formats).

비교에는 최초 적재 시간·추가 디스크·증분 게시 비용을 포함해야 한다. immutable 연구 release별 DB, 또는 한 writer 프로세스가 관리하는 DB가 설계 후보다. 여러 독립 프로세스가 같은 DB 파일에 자유롭게 동시에 쓰는 구조를 전제하지 않는다. 공식 동시성 모델과 우리 writer/reader 배치를 맞춰야 한다. [동시성](https://duckdb.org/docs/current/connect/concurrency).

**적합성 판단:** 같은 수개월 입력에 SQL 연구를 계속 반복하고, 데이터 변경보다 조회가 훨씬 많다면 우선순위가 높다. Parquet 원본 + native 연구 DB의 중복 저장이 자체 실행 cache보다 비싸다고 미리 판단할 수 없다. 두 방식 모두 별도 파생 저장 공간을 사용하기 때문이다. native DB가 목표를 만족하면 직접 파일 포맷·인덱스·범용 쿼리 실행기를 만들 필요가 크게 줄어든다.

### 15.3 Polars lazy/streaming: DataFrame 연구를 유지하는 대안

**실행 경로:** `scan_parquet`로 지연 실행 계획 구성 → 열·필터를 읽기 단계로 이동 → 가능한 연산을 batch로 실행 → 집계 결과 수집 또는 sink로 출력. 처음부터 `read_parquet`로 전부 읽은 후 열을 버리는 코드와 다르다. 큰 결과를 한 DataFrame으로 `collect`하는 대신 sink를 쓰는 것도 별도 판단이다. [읽기·쓰기 실행 모델](https://docs.pola.rs/user-guide/lazy/sources_sinks/), [scan_parquet](https://docs.pola.rs/api/python/stable/reference/api/polars.scan_parquet.html).

**메모리 판단:** streaming 선택만으로 전체 계획의 bounded memory를 보장하지 않는다. 일부 연산은 메모리 실행으로 전환될 수 있다. 실제 physical plan, 그룹 수, 정렬·join 상태, 최종 출력 크기를 확인해야 한다. lazy 최적화의 공통 부분식/부분계획 재사용도 독립적인 여러 연구 실행에 걸쳐 영구 배열 cache를 제공한다는 뜻은 아니다. [Streaming](https://docs.pola.rs/user-guide/concepts/streaming/), [최적화 범위](https://docs.pola.rs/user-guide/lazy/optimizations/).

**우리 규모에서 놓치면 안 되는 설정:** 기본 index 크기는 32비트다. 단일 거대 DataFrame이나 관련 상태가 2³²행을 넘는 경로에는 Python `rt64`, Rust `bigidx` 등 큰 index 구성이 필요할 수 있다. 1초·90일의 116.64억 행은 이 경계를 넘는다. 다만 **streaming 입력의 누적 행 수가 2³²를 넘는 모든 계획이 무조건 실패한다는 뜻은 아니다**. 실제 materialization 단위와 사용 연산을 확인한다. [설치·큰 데이터 설정](https://docs.pola.rs/user-guide/installation/).

ASOF는 instrument별 정렬, backward/forward/nearest, tolerance 등을 지정할 수 있다. 우리는 당시 이용 가능한 값만 쓰도록 backward 의미와 knowledge cutoff를 고정해야 한다. 또한 내부 멀티스레드 엔진을 많은 프로세스에서 각각 실행하면 과도한 스레드와 메모리 경쟁이 생긴다. job 수와 `POLARS_MAX_THREADS`를 함께 제한한다. [join_asof](https://docs.pola.rs/api/python/stable/reference/lazyframe/api/polars.LazyFrame.join_asof.html), [멀티프로세스](https://docs.pola.rs/user-guide/misc/multiprocessing/), [thread pool](https://docs.pola.rs/api/python/stable/reference/api/polars.thread_pool_size.html).

**선택 조건:** 현재 연구를 DataFrame 표현식으로 자연스럽게 만들 수 있고, 피처 가공과 통계를 한 계획으로 연결할 때. Python 행별 callback이나 중간 `collect`가 많이 필요하면 그 비용까지 포함해 Rust worker와 비교한다. Windows Rust CI도 존재하므로 Linux 이전을 필수 전제로 삼을 이유는 없다. 사용할 버전·feature의 빌드 검증은 필요하다. [Polars Rust CI](https://github.com/pola-rs/polars/blob/main/.github/workflows/test-rust.yml).

### 15.4 상주 Rust worker: DB 없이도 입력을 반복 사용하는 대안

이것은 현재 `onesec_bt`의 load-once 접근을 확장하는 설계다. **입력 block을 한 번 읽기 → 읽기 전용 배열/뷰를 보유 → 여러 통계·파라미터 실행 → 다음 block** 순서로 바꾼다. 같은 프로세스의 스레드는 `Arc`와 slice로 불변 입력을 공유할 수 있다. 별도의 프로세스 간 공유 메모리 프로토콜은 필요하지 않다.

예를 들어 “설정 100개 각각 90일을 읽기”를 “하루 또는 7일 block을 읽고 설정 100개를 처리하기”로 바꾸면 원자료 재방문을 줄일 수 있다. 다만 전략별 포지션·주문·rolling 상태는 각각 유지한다. 날짜 간 이어지는 상태는 다음 block으로 전달해야 하므로 날짜를 무조건 독립 병렬화해서는 안 된다. 여러 instrument의 포트폴리오가 상호 작용하면 그 결합도 보존한다.

**장점:** 기존 HFT 계산 코드를 살리면서 반복 로드·전열 concat·설정별 복사부터 줄일 수 있다. **부족한 것:** 범용 질의 최적화, 자동 spill, 임의 join, 여러 언어/프로세스 간 장애 격리는 따로 제공되지 않는다. 입력·결과·scratch에 상한을 두고 취소/재시작을 설계해야 한다.

**선택 조건:** W3와 W6처럼 같은 입력에 기존 네이티브 알고리즘을 반복할 때. 한 프로세스의 메모리 공유로 충분하면 named shared memory 서버를 추가하지 않는다. 여러 독립 소비자가 동일한 계산용 열 배열을 실제로 반복 요구할 때 파일 mmap 또는 공유 영역을 다음 단계로 비교한다. 이 판단은 현재 코드에서 도출한 설계 제안이며 새 worker를 구현했다는 뜻은 아니다.

### 15.5 DataFusion: 엔진 전체를 직접 만드는 것과의 중간 선택

DataFusion은 Arrow 기반 Rust 질의 실행 구성요소다. 자체 source를 `TableProvider`로 연결하고 기존 필터·집계·정렬 실행기를 사용하는 방향을 검토할 수 있다. **완성된 시계열 저장 서비스는 아니므로** catalog, snapshot, 정정 이력, 파일 게시, 작업 API는 우리 계층에 남는다. 이 경우에도 SQL parser·optimizer·일반 집계기를 전부 새로 만들 이유는 없다. [기능과 확장점](https://datafusion.apache.org/user-guide/features.html).

일부 Parquet 최적화와 predicate 실행은 설정·버전에 따라 다르다. 메모리도 기본 pool을 그대로 쓰지 말고 `FairSpillPool` 등 예산과 spill을 명시해야 한다. 여러 session이 각자 큰 예산을 받지 않게 공유 `RuntimeEnv`를 검토한다. 엔진에 등록하지 않은 사용자 buffer까지 자동 회계되는 것은 아니다. 결과 역시 `collect`는 전체 batch를 모으므로 `execute_stream`과 구분한다. [설정](https://datafusion.apache.org/user-guide/configs.html), [MemoryPool](https://docs.rs/datafusion/latest/datafusion/execution/memory_pool/trait.MemoryPool.html), [DataFrame 실행](https://datafusion.apache.org/library-user-guide/using-the-dataframe-api.html).

**시간 조인은 특히 버전 확인이 필요하다.** 조사 시점의 rolling SQL 문서에는 ASOF가 있지만, 확인한 55.0.0 태그의 해당 SQL 문서에는 없다. rolling 문서는 오른쪽 입력의 RAM 적재, ASOF spill/repartition 제한, 같은 시각 후보의 비결정성도 설명한다. 따라서 “우리 채택 버전에 준비된 범용 대용량 ASOF가 있다”고 전제하지 않는다. 배포 버전과 동률 규약을 고정한 검증 전에는 이 기능을 선택 근거로 쓰지 않는다. [rolling ASOF 문서](https://datafusion.apache.org/user-guide/sql/select.html#asof-join), [55.0.0 태그 문서](https://raw.githubusercontent.com/apache/datafusion/55.0.0/docs/source/user-guide/sql/select.md).

**선택 조건:** Rust 서비스 안에 질의 엔진을 내장하고, HFT 전용 source/operator를 자연스럽게 결합해야 할 때. 단순 통계만 필요하면 DuckDB/Polars보다 조립·운영할 부분이 많을 수 있다. Windows 구성도 공식 개발 문서에서 다루므로 OS 때문에 처음부터 배제할 후보는 아니다. [Windows 개발 환경](https://datafusion.apache.org/contributor-guide/development_environment.html#windows-setup).

## 16. 분석 서버를 두는 대안: ClickHouse와 QuestDB

두 후보의 가치는 **데이터 가까이에서 질의·집계를 수행하고 작은 결과를 여러 소비자에게 주는 것**이다. 연구자가 적어도 반복 질의가 많으면 검토할 가치가 있다. 반대로 API 서버를 설치한 뒤 수십억 원행을 그대로 반환하면 네트워크와 클라이언트 메모리 문제는 남는다.

### 16.1 ClickHouse: 반복 전종목 집계의 강한 비교 후보

MergeTree 계열은 정렬된 열 기반 part를 만들고 background merge를 수행한다. 희소 인덱스로 불필요한 구간을 건너뛰고 필요한 열을 읽는다. 우리에게 중요한 선택은 `(instrument, time)`과 `(time, instrument)` 중 어느 질의에 맞춘 배치를 주로 쓸 것인지다. 다른 정렬의 projection은 읽기 이득과 추가 저장·쓰기 비용을 교환한다. 전종목·전기간의 모든 선택값이 필요하면 인덱스가 그 scan 자체를 없애주지는 않는다. [MergeTree](https://clickhouse.com/docs/engines/table-engines/mergetree-family/mergetree).

`GROUP BY`의 중간 상태를 디스크로 내리는 external aggregation과, 정렬 순서에 맞춰 집계 상태를 일찍 확정하는 최적화를 사용할 수 있다. **이것을 모든 window·시간 조인에 대한 자동 spill 보장으로 확대하면 안 된다.** 그룹 수·임시 공간·쿼리 수를 함께 제한한다. [GROUP BY와 외부 메모리](https://clickhouse.com/docs/sql-reference/statements/select/group-by).

**우리에게 잘 맞는 경우:** 같은 수개월 테이블을 대상으로 많은 SQL 통계를 반복하고, 여러 소비자가 결과를 공유하며, 집계된 작은 결과만 받는 연구. 1,500개 테이블을 먼저 만드는 대신 instrument 열을 가진 공통 schema와 시간 partition을 출발점으로 비교하는 편이 낫다. 피처와 1분 보조 데이터는 별도 유지하여 불필요한 60배 복제를 피한다.

**운영 비용:** 최초 적재·중복 저장·merge·정정 처리·재시작 후 cold read가 추가된다. workload별 자원 제어도 모든 background 작업을 완전히 격리하지는 않는다. 공식 문서는 merge/mutation의 CPU scheduling 제한과 일부 메모리 scheduling의 experimental 상태를 명시한다. 수집과 연구의 지연 요구가 충돌하면 별도 연구 인스턴스/복제본을 고려한다. [Workload scheduling](https://clickhouse.com/docs/operations/workload-scheduling).

### 16.2 QuestDB: 시간순 현재/이력과 ASOF 중심 후보

QuestDB는 designated timestamp를 중심으로 시간을 분할하고, native 열 파일을 mmap해 OS page cache를 활용한다. WAL 및 늦게 도착한 데이터의 처리를 제공한다. 현재 공식 문서는 native와 Parquet partition을 함께 다루므로 **“QuestDB는 압축 이력을 쓸 수 없다”는 식으로 제외해서는 안 된다.** [Storage Engine](https://questdb.com/docs/architecture/storage-engine/).

우리에게 매력적인 부분은 시간 범위 조회와 **각 1초 행에 그 시각 이하의 최근 보조값을 붙이는 ASOF**, 그리고 너무 오래된 값의 사용을 막는 tolerance다. 단, 경제적 event time과 실제 이용 가능 시각이 다르면 후자를 기준으로 정렬·결합해야 한다. 정렬되지 않은 결과에 단순히 timestamp 힌트를 붙여 정렬을 대신해서는 안 된다. [ASOF JOIN](https://questdb.com/docs/query/sql/asof-join/).

메모리 관리도 구체적으로 봐야 한다. 현재 문서에는 쿼리별 native allocation 제한이 있다. 그러나 mmap 페이지·JVM heap·thread stack 등을 포함한 전체 물리 메모리 상한은 아니다. 일반 집계·정렬·window의 상태가 커졌을 때 모두 자동 disk spill로 이어진다는 보장은 확인하지 못했다. 한도 초과 실패와 우리 쪽 분할 실행 정책도 필요하다. [Cairo memory limits](https://questdb.com/docs/configuration/cairo-engine/#memory-limits).

**선택 조건:** 계속 들어오는 시계열의 최근 상태, 시간 범위, ASOF 결합을 공유 API로 제공하는 일이 중요할 때. 전종목 수개월의 무거운 임의 집계에서 ClickHouse나 embedded engine보다 빠르다는 판정은 아직 없다. Windows 실행 파일/service도 공식 제공하므로 Windows라는 이유만으로 배제하지 않는다. [Quick start](https://questdb.com/docs/getting-started/quick-start/).

### 16.3 어느 서버든 정정 이력과 연구 재현성은 별도로 설계한다

| 동작 | 실제 의미 | 우리에게 필요한 보완 |
|---|---|---|
| ClickHouse ReplacingMergeTree | 같은 정렬 키의 버전을 background merge에서 최신으로 정리. merge 전 최신 조회에는 `FINAL` 등 중복 해소 필요 | 제거된 과거 버전은 `as_of` 필터로 복구할 수 없음. revision 원장/불변 release 분리 |
| QuestDB dedup | 지정 키가 같은 기존 행을 새 값으로 교체 | 최신 table과 당시 알려진 값의 원장을 분리 |
| ClickHouse incremental MV | 삽입 block 중심으로 집계 갱신 | 정정 삽입의 단순 합산은 이중 계산 가능. JOIN 오른쪽 변경만으로 기존 결과가 자동 재계산되지 않음 |
| QuestDB MV | 영향을 받은 시간 구간을 비동기 재계산 | base 변경 종류에 따라 invalid 가능. JOIN 상대 변경과 refresh 지연도 관리 |

근거: [ReplacingMergeTree](https://clickhouse.com/docs/engines/table-engines/mergetree-family/replacingmergetree), [QuestDB 저장·dedup](https://questdb.com/docs/architecture/storage-engine/), [ClickHouse incremental MV](https://clickhouse.com/docs/materialized-view/incremental-materialized-view), [QuestDB materialized views](https://questdb.com/docs/concepts/materialized-views/).

**일관된 한 번의 조회와 point-in-time 연구는 다른 요구다.** 당시의 원천 revision·사용 가능 시각·피처 정의·재구성 버전·결측 범위를 고정해야 한다. 현재 table만 유지하면서 “과거 timestamp가 있으니 과거 시점 재현도 가능하다”고 판단하면 안 된다.

서버를 채택한다면 데이터·임시 공간을 어디에 둘지도 엔진과 함께 평가한다. 241 로컬 연구 SSD의 서버와 NAS 파일을 직접 읽는 서버는 다른 구성이다. 필요할 때만 실행하는 연구 전용 서버도 가능하지만, 정지 중 신규 데이터와 재시작 후 cache 준비를 별도로 처리해야 한다.

## 17. 배열 저장·분산 실행·GPU 등 다른 접근

### 17.1 Zarr / HDF5: 표보다 tensor를 주로 읽는 경우

학습이 `time × instrument × feature`의 일정 모양 window를 반복 추출한다면, 압축 chunk 배열은 SQL 테이블과 다른 유력 대안이다. **chunk는 일부를 읽는 단위**이고, Zarr의 shard는 여러 chunk를 한 저장 객체에 묶어 작은 파일 수를 줄이는 수단이다. chunk 모양은 접근 패턴에 맞춰 정한다. [Zarr 성능](https://zarr.readthedocs.io/en/stable/user-guide/performance/), [sharding 명세](https://zarr-specs.readthedocs.io/en/latest/v3/codecs/sharding-indexed/index.html).

예를 들어 `(1시간, 32 instruments, 8 features)`와 `(1일, 1 instrument, 8 features)`는 같은 총량이라도 서로 다른 읽기를 유리하게 만든다. 전종목의 짧은 단면과 한 종목의 긴 과거를 모두 같은 효율로 읽을 수 있다고 기대하면 안 된다. 사용할 chunk 중 실제 필요한 값의 비율, 해제 bytes, 파일 요청 수를 측정한다. 이 예시는 권장 기본값이 아니다.

가격·수량 정수를 float tensor로 강제 통일하지 말고 dtype별 배열과 schema를 유지한다. 미상장·결측·관측 없음은 mask와 coverage로 표현한다. 불규칙한 틱까지 조밀한 시간 격자로 강제 변환하면 공간 증가뿐 아니라 사건 의미를 잃을 수 있다. **W4의 파생 학습 포맷 후보이며 모든 원장·패킷 저장을 대체할 후보는 아니다.**

HDF5도 chunk 배열 후보지만 Python `h5py`는 내부 전역 lock 때문에 여러 스레드의 h5py API 호출이 직렬화될 수 있다. 이는 h5py의 특성이지 모든 HDF5 구현이 같다는 주장으로 넓히지 않는다. SWMR도 single-writer/multiple-reader 모델이며 다중 writer DB를 뜻하지 않는다. [h5py threading](https://docs.h5py.org/en/stable/threads.html), [SWMR](https://docs.h5py.org/en/stable/swmr.html). 불변 연구 묶음이면 더 단순하게 쓸 수 있지만 파일 version·게시·동시 읽기는 여전히 설계 대상이다.

### 17.2 Dask: 분할 계산을 조립하되 shuffle을 계산에 넣는다

Dask는 DataFrame/array를 partition으로 나누고 작업 그래프로 실행한다. **단일 241에서도** RAM보다 큰 batch와 여러 작업을 조율하는 후보이며, 여러 머신이 필수는 아니다. 다만 작은 task를 매우 많이 만드는 scheduler 비용, index 재배치와 대규모 join의 shuffle을 포함해야 한다. 필요한 열·행 선택 전에 `persist`하면 후속 optimizer의 읽기 축소를 막을 수 있다. [DataFrame best practices](https://docs.dask.org/en/stable/dataframe-best-practices.html).

worker의 관리 대상 데이터는 spill할 수 있지만 task 내부의 native 배열·사용자 임시 메모리까지 같은 방식으로 모두 제어되지는 않는다. 프로세스 메모리 기준 pause/terminate도 큰 순간 할당을 없애지는 못한다. 한 worker 예산만 보고 전체 worker 합계를 놓치지 않는다. [Worker memory](https://distributed.dask.org/en/stable/worker-memory.html).

배열 연산에서는 저장 chunk와 계산 chunk를 맞추고, rolling 경계의 이웃 데이터를 `map_overlap`으로 넘길 수 있다. 그러나 halo 전달은 추가 읽기·복사이며, EMA 초기 상태나 주문장 재생의 무한히 이어지는 상태를 자동 해결하지 않는다. [Array chunks](https://docs.dask.org/en/stable/array-chunks.html), [Array overlap](https://docs.dask.org/en/stable/array-overlap.html).

**선택 조건:** 분할 가능한 Python/NumPy 알고리즘이 이미 있고, 여러 partition의 실행·실패·재시도를 관리해야 할 때. 단순 전 기간 SQL 집계 하나를 위해 도입하면 embedded engine보다 구성요소가 늘 수 있다. 큰 결과는 파일이나 작은 집계로 내보내고, 마지막에 전체 DataFrame을 호출자 RAM으로 모으지 않는다.

### 17.3 Ray: 반복 실험·상태 있는 worker·노드별 객체 재사용

Ray는 작업과 actor를 스케줄링하고 노드별 object store에서 입력을 재사용한다. 읽기 전용 NumPy buffer 등 지원되는 경우 같은 노드에서 사본을 줄일 수 있다. **프로세스의 모든 객체가 zero-copy가 되는 것도, 여러 노드가 하나의 물리 RAM을 공유하는 것도 아니다.** 원격 객체는 전송되고, writable 변환·다른 포맷 변환은 복사를 유발할 수 있다. [Objects](https://docs.ray.io/en/latest/ray-core/objects.html), [map_batches의 zero-copy 조건](https://docs.ray.io/en/latest/data/api/doc/ray.data.Dataset.map_batches.html).

object store가 차면 파일로 spill할 수 있다. 이것은 actor의 임의 private heap까지 자동으로 디스크화한다는 뜻은 아니다. object reference를 오래 잡으면 데이터 수명도 길어진다. store와 worker heap, spill/restore bytes, 같은 입력을 다른 노드로 보내는 횟수를 따로 측정해야 한다. [Memory management](https://docs.ray.io/en/latest/ray-core/scheduling/memory-management.html), [Object spilling](https://docs.ray.io/en/latest/ray-core/objects/object-spilling.html).

**선택 조건:** W3의 실험 scheduling, 모델 학습, 장시간 상태를 가진 actor, 향후 여러 계산 노드가 중요할 때. 날짜/instrument 묶음을 보유한 worker 가까이 다음 실험을 보내는 방식이 핵심이다. 각 실험에 수백 GB를 새 객체로 전달하는 사용법은 피해야 한다. 단일 Rust worker로 충분하면 Ray가 필수는 아니다. Windows의 다중 노드 구성을 검토한다면 공식 문서가 experimental/untested로 분류하는 범위도 확인해야 한다. [설치·플랫폼 상태](https://docs.ray.io/en/latest/ray-overview/installation.html).

### 17.4 GPU: 계산이 병목일 때의 다음 단계

GPU는 큰 수치 연산·학습·지원되는 집계의 계산 시간을 줄일 후보다. NAS의 파일 열기·SMB 지연·원자료 전송을 직접 해결하지는 않는다. 760GB host RAM과 GPU VRAM은 별도 자원이고, host/device 이동과 결과 반환까지 포함해야 한다. 241의 사용 가능한 GPU/VRAM은 이번 조사에서 확인하지 않았다.

Windows의 RAPIDS 검토에는 WSL2 등 실행 환경 조건이 붙는다. 현재 공식 설치 문서의 WSL2 제한에는 단일 GPU와 GPUDirect Storage 미지원이 포함된다. 따라서 기존 Windows mmap 파일을 그대로 GPU가 비용 없이 읽는 구조로 상정하지 않는다. [NVIDIA 데이터 과학 설치 안내](https://docs.nvidia.com/datascience/install/).

**선택 조건:** 로컬 준비·열 선택·반복 재사용 후에도 계산 kernel이 시간을 지배하고, GPU에 맞는 충분히 큰 batch를 만들 수 있을 때. 지금의 NAS 읽기 문제에 대한 첫 처방으로 GPU를 택할 근거는 없다.

### 17.5 NAS 근처 계산·네트워크·운영체제 경계

NAS 가까이에서 열 선택·부분 집계를 끝내면 네트워크 bytes를 줄일 수 있다. 반면 전행을 변환해 다시 보내는 작업은 NAS CPU·해시·디스크와 경합한다. 14.3의 기존 실측처럼 **“데이터 옆에서 실행하므로 무조건 더 빠르다”는 결론은 성립하지 않는다.** daily 소량 준비와 여러 달 연구를 별도 작업으로 평가한다.

링크가 1Gbps라고 가정하면 1초·90일·20열의 비압축 값 1.866TB를 한 번 보내는 데도 이상적 하한이 약 4.15시간이다. 7일·20열의 145.2GB는 약 19.4분이다. 프로토콜·디스크·검증 비용이 없는 단순 하한이며 실제 압축 전송량과는 다르다. 이 하한에 가깝다면 파일 포맷만 고쳐서는 최초 준비를 크게 줄일 수 없다. 반복 전송 제거, 필요한 열만 준비, 압축 전송, 링크 개선을 각각 평가한다.

Linux/WSL 엔진과 Windows 원본 경로 사이의 접근도 숨기지 않는다. Microsoft는 사용 도구의 OS 쪽 파일시스템에 데이터를 배치하는 것을 권한다. WSL VM의 RAM 예산도 host 전체 RAM과 동일하지 않으며 따로 설정된다. **OS/VM 경계를 넘으면 mmap/shared memory가 자동 공유된다고 가정하지 않는다.** [WSL 파일시스템](https://learn.microsoft.com/en-us/windows/wsl/filesystems), [WSL 자원 설정](https://learn.microsoft.com/en-us/windows/wsl/wsl-config). 이번 조사에서는 머신 설정을 변경하지 않았다.

## 18. 엔진을 바꾸기 전에 계산 방식으로 줄일 수 있는 일

### 18.1 고정 질문은 부분 결과를 저장하고 합친다

매일 같은 정의로 산출하는 건수·합·분산·조건별 수익 통계라면, 날짜/instrument별 부분 상태를 만들고 합치는 방식이 후보 엔진 공통으로 유효하다. 평균의 평균 대신 합과 유효 표본 수를 보존한다. 분산·공분산은 안정적인 결합 상태와 결측 처리 규약을 유지한다. 피처 정의나 원천 revision이 바뀌면 영향을 받은 부분 결과도 새 버전으로 만든다.

| 질문 | 재사용 가능한 것 | 다시 원행/큰 상태가 필요한 경우 |
|---|---|---|
| 동일 조건의 평균·분산·상관 | 조건별 유효 표본 수와 결합 가능한 통계 상태 | 조건 변경, 결측 규칙 변경, 서로 다른 유효 표본 집합 |
| 동일 bin의 histogram | bin별 개수/가중 합 | bin 경계 변경, 원래 정의의 정확한 분위수 |
| 여러 threshold 실험 | 공통 피처·수익 horizon·필터, 경우에 따라 정렬 결과 | 거래 경로가 threshold에 따라 달라지는 시뮬레이션 |
| rolling 피처 | 앞 구간의 필요한 이력, 정의된 상태/checkpoint | 정정이 이후 상태까지 전파되거나 window 정의가 바뀜 |
| 패킷 재생 | 불변 packet bytes·공통 사건 색인 | 전략별 주문·포지션·체결 상태 |

실시간으로 같은 feature와 통계를 계속 유지해야 한다면 Flink 같은 stateful stream processor도 별도 대안이다. event time·watermark·늦은 데이터 정책과 상태를 관리할 수 있다. 다만 watermark만으로 정보 이용 가능 시각이나 정정 문제까지 해결되지는 않는다. **고정된 반복 계산을 미리 수행하는 용도**가 적합하며, 임의의 과거 연구를 모두 대체하지 않는다. [Flink streaming analytics](https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/learn-flink/streaming_analytics/).

### 18.2 별도 실행 cache의 손익분기점을 계산한다

다음 식은 같은 snapshot·같은 결과·직렬 반복이라는 단순 모델이다. 계산 자체의 공통 시간은 양쪽에서 제외하고, 다른 부분만 비교한다.

```text
P = 선택 열을 별도 실행 포맷으로 준비하는 일회성 비용
C = 기존 경로의 매회 읽기 + decode + 필요한 변환 비용
H = 준비한 입력을 매회 다시 접근하는 비용
R = 같은 입력의 반복 횟수

별도 준비의 이득 조건: P + R × H < R × C
즉, C > H일 때 R > P / (C - H)
```

**가상의 예:** P=120초, C=20초, H=8초라면 10회에서 같고 11회부터 시간 이득이다. 실측값이 아니며 저장 공간·갱신·재시작·eviction 비용은 추가해야 한다. 파일 page cache 덕분에 C가 이미 작으면 mmap 변환 이득은 줄어든다. 반대로 기존 경로가 매번 전체 배열을 만들면 먼저 그 사본만 제거해도 효과가 클 수 있다.

모든 열을 실행 cache로 만들 필요도 없다. 자주 반복하는 피처·기간만 준비하고, 드문 질문은 압축 이력에서 읽는 혼합 정책이 가능하다. cache key에는 데이터 release·열·dtype·정규화/피처 버전·정렬·결측 규약을 포함해야 한다. 동일 파일명이라는 이유로 정정 전 cache를 재사용해서는 안 된다.

### 18.3 메모리에 한 번 올린 뒤에도 전체를 여러 번 읽으면 비싸다

1초·90일·20열은 값 영역만 약 1.87TB다. 이를 worker 8개가 각각 전부 통과하면 논리적 배열 접근량은 약 14.9TB가 된다. 물리 DRAM 트래픽은 CPU cache·계산 방식에 따라 달라지지만, **공유 페이지가 곧 공유 계산이나 무한 메모리 대역폭을 뜻하지는 않는다.** 동일 중간 피처를 함께 계산하고 block을 재사용하는 방식이 더 중요해질 수 있다.

비압축 포맷이 항상 빠르지도 않다. cold SSD/네트워크 읽기가 지배적이면 압축을 해제하는 CPU 비용보다 읽는 bytes 감소가 클 수 있다. warm 반복에서 decode가 지배적이면 비압축 배열이 유리할 수 있다. 따라서 **첫 준비·cold scan·warm 반복을 한 숫자로 합쳐 저장 형식 순위를 매기지 않는다.**

## 19. 어떤 비교를 해야 실제로 선택할 수 있는가

### 19.1 1차 비교는 네 구성으로 제한한다

| 구성 | 입력·실행 | 확인할 질문 |
|---|---|---|
| A | 로컬 Parquet + DuckDB 직접 조회 | 열 선택·집계·spill만으로 충분한가? |
| B | 같은 데이터의 DuckDB native 연구 DB | 적재 비용을 포함해 반복 SQL이 더 유리한가? |
| C | 로컬 Parquet + Polars lazy/streaming | 피처·통계 pipeline이 작은 중간 상태로 실행되는가? |
| D | 검증된 열 기반 IPC 등 + 기존 Rust load-once worker | 반복 실험의 입력/중간 계산 재사용으로 충분한가? |

원래 HFT reader는 개선 전 기준으로 남긴다. 기존 변환 IPC FILE도 D의 입력 후보로 사용하고, 필요하면 같은 연산의 IPC 직접 읽기를 추가한다. **새로운 자체 디스크 포맷을 만드는 것을 비교의 선행 조건으로 두지 않는다.**

A~D 중 충분한 것이 있으면 그 경로를 공통 규약에 연결한다. 다중 사용자·지속 수집 요구가 크면 ClickHouse/QuestDB를 추가하고, native source/operator 내장이 필요하면 DataFusion을 추가한다. tensor 추출이 주력이면 Zarr, 실행 scheduling이 병목이면 Dask/Ray를 해당 작업에만 추가한다. 모든 제품을 한꺼번에 설치하는 계획은 아니다.

### 19.2 데이터·질의·정답을 먼저 고정한다

**데이터:** 작은 정확성 표본 → 실제 전체 instrument의 짧은 기간 → 실제 수개월 순서다. 일부 instrument만으로 전체 그룹 수·결측·불균형을 대신하지 않는다. 전체 기간에 존재하지 않은 종목을 억지로 채우지 않고 coverage를 기록한다. 패킷 replay는 별도 기준을 둔다.

**공통 질의:** W1의 선택 5열/20열 집계, W2의 시간 결합과 rolling, W3의 같은 입력 10회 반복을 최소 세트로 한다. 파라미터 수는 실제 대표 연구에 맞춘다. exact quantile이나 전역 정렬처럼 작은 중간 상태로 끝나지 않는 작업도 포함한다. unsupported 연산은 누락하거나 다른 근사값으로 바꾸지 않고 그대로 표시한다.

**정확성:** 정수·시각·키·null은 같은 의미로 비교한다. NaN·무한대·동일 timestamp·중복·정정·상장/폐지·빈 partition을 포함한다. ASOF는 `available_at <= decision_time`, 동일 시각 tie-break, 최대 유효 시간, 종목 경계를 명시한다. rolling은 block 경계를 포함한 정답, 재생은 사건 순서와 전략별 결과를 확인한다. 부동소수 reduction 순서가 달라질 때 허용 오차와 재현 모드를 먼저 정한다.

### 19.3 측정표는 아래를 분리해야 한다

| 단계 | 필수 측정 | 잘못된 비교의 예 |
|---|---|---|
| NAS → 로컬 준비 | 선택 bytes·실제 전송 bytes·파일 수·전송/검증 시간 | 전체 파일 복사와 필요한 열만 반환하는 질의를 같은 일로 비교 |
| 포맷 변환·DB 적재 | 1회 시간·출력 공간·peak 메모리·검증 | warm native DB에서 최초 적재 비용 생략 |
| cold 실행 | 디스크 bytes·page fault·decode·계산·결과 시간 | 프로세스만 재시작하고 OS cache까지 비워졌다고 가정 |
| warm 반복 | 1회/10회 총시간·재decode·사본·입력 재사용 | 순수 kernel 시간과 결과 직렬화 포함 시간을 비교 |
| 큰 상태 | private/공유 메모리·scratch·spill bytes·임시 공간 | memory_limit 값만 기록하고 실제 사용량 생략 |
| 동시성 | 1/4/8 job, 내부 thread 수, 총 처리량·개별 지연 | 8 job 각각 CPU 전 코어를 쓰게 한 뒤 엔진이 느리다고 판단 |
| 증분·정정 | 추가 준비 bytes·갱신 시간·기존 독자 일관성 | 전체 재생성과 일부 partition 갱신 비용 혼합 |

운영 머신의 cache를 비우거나 메모리를 압박하는 실험은 자동 전제로 두지 않는다. 별도 검증 환경에서 cache 상태를 통제하거나, 통제하지 못한 조건을 명시한다. NVMe와 SATA, native Windows와 WSL, 원본 압축률·정렬·row group 크기도 기록한다. 측정용 도구가 만든 큰 사본도 비용에 포함한다.

**반환 결과를 맞춰야 한다.** DuckDB가 instrument별 수치 1,500개를 반환하고 Rust가 전행 배열을 반환하는 비교는 엔진 성능 비교가 아니다. 양쪽이 같은 계산을 끝내고 같은 결과를 반환하도록 맞춘다. 반대로 실제 소비자가 전행을 필요로 하면 서버가 작은 집계만 계산한 수치로 대체하지 않는다.

### 19.4 통과 기준

- 정해진 결과·데이터 시점·품질 의미가 일치한다.
- 유효한 로컬 입력을 반복 사용할 때 원자료를 NAS에서 다시 전송하지 않는다.
- 선택 열이 줄면 실제 읽기/해제/사본도 줄어든다.
- RAM·임시 SSD·동시 job 예산 안에서 끝나거나, 계획된 분할/제한을 설명한다.
- 정정 시 필요한 부분만 새로 준비할 수 있고, 진행 중인 연구는 고정된 release를 읽는다.
- 대표 작업의 허용 대기 시간과 반복 처리량을 만족한다. 그 시간 목표는 실제 대표 연구와 장비 볼륨을 확인해 별도로 정한다.

이번 보완에서는 위 비교를 실행하지 않았다. 공식 기능을 우리 데이터의 실측 결과로 표시하지 않으며, 새 benchmark 수치를 만들어 넣지 않는다.

## 20. 수정한 추천: 공통 규약은 만들고, 엔진은 검증해서 선택한다

이 절의 선택 기준은 **플랫폼 내부의 저장·연구 실행 구성요소**에 적용한다. 플랫폼 전체의 필수 질의·비동기·실거래 요구와 선택 기준은 21~26절로 확장했다.

### 20.1 작업별 현재 우선순위

아래는 **비교할 순서**이며 실측 성능 순위가 아니다.

| 실제 주력 작업 | 먼저 볼 방식 | 다음 후보 | 자체 구현이 남을 수 있는 부분 |
|---|---|---|---|
| 전종목·수개월의 임의 통계 | DuckDB Parquet / native, Polars | 반복 공유 SQL이 많으면 ClickHouse | 데이터 의미·snapshot·작업 예산·출력 어댑터 |
| 같은 HFT 피처의 반복 실험 | 상주 Rust worker, block 단위 다중 실험 | 읽기 전용 mmap, Ray actor | 피처 reader·입력 뷰·전략별 상태 분리 |
| 현재값·시계열 시간 결합 | QuestDB와 기존 엔진 ASOF | Rust 내장이 필요하면 DataFusion 검토 | 시점 규약·정정 이력·경계 병합 |
| 학습 tensor/window | Zarr 등 chunk 배열과 현재 열 형식 비교 | Dask, 계산 병목 확인 후 GPU | dtype/mask·batch adapter·재사용 정책 |
| 패킷 사건 재생 | 기존 SPKR/Rust reader 개선·load-once | 불변 bytes/색인의 mmap 공유 | 사건 순서·호가장/주문 상태·전용 replay kernel |

### 20.2 자체 저장·범용 계산 엔진을 개발하지 않아도 되는 조건

기존 엔진을 공통 API 뒤에 연결했을 때 19.4를 만족하고, 필요한 시간 조인·rolling·재현성을 표현할 수 있다면 해당 역할의 **독자 저장 포맷·범용 optimizer·범용 계산 엔진 개발을 범위에서 뺄 수 있다.** 그러나 이 판정으로 공통 질의 계층·다중 프로세스 실행·비동기 액션·실시간 구독의 설계까지 생략할 수는 없다. 플랫폼 채택에는 26절의 live 지연·연속성·복구·계산 일치 기준을 추가로 통과해야 한다.

추가 개발은 다음처럼 **남은 병목이 특정됐을 때 작은 단위로** 검토한다.

1. 기존 source가 필요한 열만 읽지 못함 → 해당 source adapter/reader.
2. 같은 입력 decode·사본이 반복 시간의 상당 부분임 → 선택 열 실행 cache 또는 mmap 뷰.
3. 기존 연산으로 정확한 시간/재생 의미를 표현하기 어렵거나 느림 → 해당 ASOF/rolling/replay kernel.
4. 여러 소비자의 입력 중복이 실제 RAM 한계를 만듦 → 공유 backing과 수명/예산 관리.

단, 입력 재사용이나 합법적인 분할로 해결되지 않는 문제를 모두 “mmap을 만들면 해결된다”고 취급하지 않는다. 큰 중간 상태·전역 정렬·동시 작업·메모리 대역폭은 각각 해결해야 한다.

### 20.3 지금 선택할 방향

**통합 플랫폼의 의미·질의·실행 규약을 먼저 설계하고, 그 안에서 로컬 데이터 준비와 기존 엔진/현재 Rust를 공정 비교한다.** kdb의 열·시간 배치에 더해, 프로세스 간 함수 실행·비동기 gateway·실시간 계산과 이력 분석의 연결도 차용 대상이다. 독자적인 전체 언어·저장 엔진을 만드는지와 통합 플랫폼을 만드는지는 별개의 결정이다.

사용자가 제안한 shared memory·mmap도 강한 후보로 남는다. 특히 W3/W6의 반복 입력 재사용에 의미가 있다. 다만 W1의 수개월 통계에서는 **압축 열 선택 scan + 엔진 내부 집계**, W4에서는 **chunk 배열**, W5에서는 **분석 서버**가 더 단순하거나 적합할 가능성도 동등하게 열어둔다.

조사 기준일은 2026-09-16이다. 본문의 공식 웹 문서 중 `current`, `latest`, `main`은 갱신되는 문서다. 구현 단계에서는 실제 배포 버전·feature·운영체제를 고정하고 특히 streaming/spill/ASOF 지원을 다시 확인해야 한다. 이번 결과물은 설계 비교와 문서 갱신이며, 소스 수정·설치·241/NAS 설정 변경·새 성능 실험은 하지 않았다.

<a id="unified-trading-platform"></a>

## 21. 연구와 실거래가 함께 쓰는 ‘우리의 kdb’ 관점

### 21.1 이전 조사에 이 관점이 충분히 적용됐는가

**충분히 적용되지 않았다.** 현재·이력 통합, TP/RDB/HDB, gateway를 언급했지만 실제 검토와 선택 기준은 대용량 연구 I/O에 집중돼 있었다. 사용자가 원하는 목표는 **하나의 데이터 원천에서 질의하고, 계산을 실행하고, 실시간 결과를 구독하며, 같은 정의를 연구와 실거래에 사용하는 시스템**이다. 저장 형식과 scan 성능만으로는 이 목표를 평가할 수 없다.

| 이전 검토 | 포함했던 것 | 부족했던 것 / 이번 보완 |
|---|---|---|
| 6절 kdb 구조 | 열 저장, 현재/이력, gateway | 질의·함수 실행·구독을 프로세스에 배치하는 전체 모델 |
| 7절 조회 계약 | prepare, scan, compute, latest | 지속 질의, 비동기 결과, 상태 변경 액션의 의미 |
| 8절 배치도 | NAS → 241 → 연구 결과 | 실거래 소비자, 실시간 피처와 최신 상태, 구독 경로 |
| 14~19절 비교 | 과거 데이터 집계·반복 실험 | 시장 데이터 도착에 따른 지속 계산과 거래 경로의 지연 |
| 20절 중단 기준 | 기존 엔진으로 연구가 충분하면 자체 엔진 축소 | 플랫폼의 공통 질의·실행 규약까지 생략할 수 없음을 명시 |

KX 공식 구조도 TP의 로그·배포, RDB의 현재 데이터, RTE의 이벤트 기반 계산, HDB의 이력, gateway의 질의 연결을 구분한다. kdb 프로세스 하나로 실행하는 구성도 가능하지만, 사용자가 말한 **다중 프로세스로 구성한 kdb 시스템**은 공식 아키텍처와 맞는 관점이다. [KX Architecture](https://code.kx.com/q/architecture/).

### 21.2 동의하는 부분과 반박하는 부분

사용자의 핵심 제안에는 동의한다. 표의 후반부는 구현 방식을 선택할 때의 판단이다. 사용자가 새 언어 문법·단일 물리 서버·매 틱 중앙 질의를 모두 요구했다고 해석한 것은 아니다.

| 주장 / 해석 | 판단 | 이유 |
|---|---|---|
| 연구와 실거래가 하나의 논리적 데이터 원천을 사용해야 한다 | **동의** | 수집·피처·시점 정의가 각각 달라지는 문제를 줄이고, 결과의 계보를 추적할 수 있음 |
| 자체적인 질의·함수 실행 체계가 필요하다 | **동의** | 공통 데이터 위에서 무엇을 계산하고 어디서 실행하며 어떤 결과를 돌려주는지 플랫폼이 정의해야 함 |
| 다중 프로세스와 비동기 질의·액션을 설계해야 한다 | **동의** | 실시간 상태, 연구 계산, 이력 조회, 명령 처리를 독립적으로 실행·복구할 수 있어야 함 |
| 이를 위해 q와 같은 새로운 언어 문법부터 만들어야 한다 | **필수라는 주장에는 반대** | 질의 의미와 실행 계획은 필요하지만 표현은 SDK·구조화된 식·SQL 부분집합으로 시작할 수 있음 |
| ‘하나’이므로 연구·실거래가 한 서버·한 작업 큐를 사용해야 한다 | **반대** | 연구 scan·정렬·메모리 압박·재시작이 실거래 지연과 장애로 전파됨 |
| 모든 틱마다 중앙 gateway에 동기 질의를 해야 한다 | **반대** | 실시간 구독으로 갱신되는 전략 인접 상태를 같은 플랫폼의 데이터로 사용할 수 있음 |
| 기존 분석 엔진을 쓰면 이런 플랫폼을 자체적으로 설계할 필요가 없다 | **반대** | 저장·계산 라이브러리가 통합 의미·구독·비동기 액션·live/replay 일치를 대신 정의해주지 않음 |

따라서 기존 엔진 재사용이라는 방향은 유지하되, **전체 플랫폼의 설계가 먼저이고 엔진 비교는 그 내부 구성요소의 검증**으로 위치를 바꾼다. kdb 방식의 매력은 빠른 이력 테이블에 더해, 데이터를 가진 프로세스에서 계산하고 프로세스들이 메시지로 연결되는 일관된 사용 모델에도 있다.

### 21.3 ‘하나의 데이터 소스’의 구체적인 뜻

다음을 플랫폼 전체에서 공유하는 것으로 정의한다.

- instrument ID, schema, 단위·dtype, 결측·정정 규약과 데이터 이름 공간.
- 원천 사건과 revision, 수집/관측 시각, 소비 가능한 진행 지점.
- 버전이 붙은 질의·함수·피처 정의와 해당 정의가 적용된 입력.
- 과거·현재·실시간 구독을 연결하는 snapshot/cursor와 복구 규칙.
- 동일한 SDK/요청 모델, 결과 타입, 오류·품질·신선도 표기.

물리적으로는 로그, 최신 상태, 열 기반 이력, 연구 SSD 복제본, 전략 옆의 최신값 cache가 있을 수 있다. 각 사본이 **어느 데이터 버전과 진행 지점까지 반영했는지** 확인할 수 있어야 한다. 복제본마다 원천 의미나 피처 정의를 따로 바꾸는 것은 허용하지 않는다.

공통 인터페이스도 단일 장애 지점 하나를 뜻하지 않는다. gateway는 여러 인스턴스일 수 있고, SDK는 같은 논리 서비스의 적절한 인스턴스에 연결할 수 있다. 여러 partition을 가진 데이터의 ‘현재’가 자동으로 전역 원자적 snapshot이 되는 것도 아니다. 필요한 일관성 범위를 요청과 결과에 명시한다.

## 22. 공통 질의·함수 체계는 무엇을 만들어야 하는가

### 22.1 질의 언어, 질의 의미, 실행기를 구분한다

| 부분 | 플랫폼이 정할 내용 | 기존 구현 재사용 가능성 |
|---|---|---|
| 표현 | SDK 표현식, 구조화된 요청, SQL 부분집합 또는 자체 문법 | parser·SDK를 재사용할 수 있음. 새 문법은 선택 |
| 의미 | 타입, null, 시간, 정정, 정렬, window, 함수 버전, 결과 확정 여부 | **플랫폼이 소유해야 함**. 엔진별 기본값에 맡기면 의미가 달라질 수 있음 |
| 논리 계획 | 데이터 선택 → 필터 → 계산/조인 → 집계 → 반환/지속 갱신 | 공통 표현 트리로 구성 가능 |
| 실행 계획 | partition·현재/이력 분할, 실행 위치, 병렬도, 중간 결과 병합 | 기존 optimizer/operator를 역할에 맞게 사용 가능 |
| 실행 수명 | 제출·진행·취소·완료·실패·재연결, 자원 한도 | job/runtime 도구를 활용하되 외부 계약은 통일 |

여기서 표현 트리란 계산식을 문자열이 아닌 **타입과 연산으로 표현한 구조**다. 예를 들어 ‘어떤 데이터의 spread를 instrument별로 60초 평균’이라는 의미를 저장하고, 과거 batch 실행 또는 실시간 상태 갱신으로 해석할 수 있다. 처음부터 q의 모든 타입·동사·함수 실행기를 복제할 필요는 없다.

반대로 엔진별 SQL을 그대로 노출하고 사용자가 시간·null·정정 차이를 알아서 처리하게 하면 공통 질의 체계를 달성했다고 보기 어렵다. 플랫폼 의미에 맞게 변환하거나, 해당 backend가 지원하지 않는 요청이라고 명시해야 한다. 의미를 바꾸는 묵시적 fallback은 하지 않는다.

### 22.2 조회뿐 아니라 데이터를 가진 프로세스에서 계산한다

필요한 연산 범위는 다음과 같다.

1. 열/행 선택, 타입이 있는 벡터 식, 그룹·구간 집계.
2. instrument별 시간 정렬·ASOF·rolling·lag와 명시된 경계 처리.
3. 입력·출력 schema와 버전이 있는 분석 함수 호출.
4. 현재 상태 조회와 데이터 도착에 따라 결과를 갱신하는 지속 질의.
5. 파생 결과의 저장·게시와 재사용.

KX IPC도 질의 문자열 외에 함수와 인자를 전달해 원격 프로세스에서 실행하는 방식을 제공한다. 이 **계산을 데이터 쪽으로 보내는 모델**을 차용할 가치가 있다. 다만 우리 v1이 임의 q 코드나 모든 사용자 프로그램의 원격 실행을 지원해야 한다는 결론은 아니다. [KX IPC의 메시지·함수 호출](https://code.kx.com/q/basics/ipc/).

제안하는 시작점은 버전이 붙은 등록 함수와 지원 연산의 조합이다. 연구용 사용자 함수 실행을 추가할 때는 일반 연구 worker에서 처리하고, 실거래 상태를 소유한 프로세스에는 처리 시간·메모리를 검증한 함수를 배치한다. 연구 중의 함수 수정이 실행 중인 전략의 정의를 자동으로 바꾸지 않게 버전을 고정한다.

함수 계약에는 외부 상태를 변경하는지도 표시한다. 읽기 전용 질의의 재실행 정책을 파생 데이터 게시나 설정 변경에 그대로 적용하지 않는다. 외부 상태를 바꾸는 호출은 24절의 액션 계약을 따르고, 지속 계산의 내부 상태는 해당 구독/프로세스가 소유한다.

### 22.3 같은 정의를 여러 실행 방식으로 사용한다

아래 이름은 **미구현 설명용 계약**이다. 현재 M1 API에 존재한다는 뜻이 아니다.

| 사용 형태 | 의미 | 결과 |
|---|---|---|
| `query(plan, view, range)` | 유한한 입력에 질의를 실행하고 결과를 기다림 | 작은 table 또는 제한된 batch stream |
| `submit(plan, view, range)` | 같은 질의를 제출하고 호출자는 다른 작업을 계속함 | request/job ID, 상태, 결과/오류 |
| `subscribe(plan, cursor)` | 지원되는 지속 질의를 데이터 도착에 따라 갱신 | subscription ID, 순서가 있는 결과 갱신·진행 지점 |
| `call(function, version, args)` | 등록된 분석 함수를 실행. sync/async 선택은 별도 | 함수 계약에 따른 타입 있는 결과 |
| `command(action, idempotency_key)` | 구독 등록·파생 데이터 게시 등 상태 변경을 요청 | 접수·적용·완료 상태와 관련 버전 |

동기 호출도 내부적으로는 같은 요청을 제출하고 완료를 기다리는 SDK 편의 기능으로 구현할 수 있다. 비동기 호출과 서로 다른 질의 의미를 만들 필요는 없다.

예를 들어 `spread_mean_60s@v3`라는 피처 정의를 등록하면 다음 세 실행에서 같은 정의를 참조한다.

- 연구: 고정된 과거 입력에서 기간별 값을 계산한다.
- 실거래: 도착한 사건으로 상태를 갱신하고 새 결과를 구독자에게 보낸다.
- 재생: 기록된 입력 순서와 초기 상태를 사용해 당시 결과를 검증한다.

**같은 이름만 붙인 세 구현은 충분하지 않다.** 가능한 부분은 공통 kernel을 사용하고, 실행기가 다르면 동일 의미를 검증하는 기준 데이터를 통과시킨다. 입력 순서·clock·warm-up·정정 규약이 달라지면 같은 함수 코드도 다른 값을 만들 수 있다. 상세 기준은 25절에 둔다.

### 22.4 모든 과거 질의를 실시간으로 실행할 수 있는 것은 아니다

각 연산/함수에 `batch 가능`, `stream 가능`, 필요한 상태 크기, 결과가 잠정/확정되는 조건을 명시한다. 과거의 전체 구간 순위나 미래 5분 수익률처럼 입력이 더 있어야 확정되는 값은 즉시 실거래 피처로 제공할 수 없다. live에서 지원할 수 없는 질의는 거부하거나, 어떤 지연·갱신 의미로 지원하는지 별도로 정의한다.

실시간 결과는 새 행 추가만 있는지도 정해야 한다. 늦은 사건으로 과거 window가 바뀐다면 갱신 결과의 키·revision과 이전 결과를 대체하는 의미가 필요하다. 잠정값을 매번 새 확정 행으로 취급하면 실거래와 연구가 서로 다른 데이터를 쓰게 된다.

## 23. 다중 프로세스로 구성하는 전체 플랫폼

### 23.1 역할 배치

```mermaid
flowchart TB
    FEED["거래소 데이터 / 추가 데이터 원천"] --> ING["수집·정규화 / 순서·복구 로그 / 배포"]
    ING --> RT["실시간 상태·피처 프로세스"]
    ING --> WR["이력 작성·seal·revision 게시"]
    WR --> HIST["불변 이력 / catalog·snapshot"]
    HIST --> RW["연구 worker / 이력 질의·재생"]
    RT -->|"계산 결과·버전 기록"| WR
    SDK["공통 SDK / 질의·함수 정의"] --> GW["비동기 gateway / 라우팅·요청 추적"]
    GW -->|"제한된 최신 조회·구독 등록"| RT
    GW -->|"과거 질의·계산 job"| RW
    RT -->|"구독 갱신"| LIVE["실거래 전략의 로컬 상태"]
    RW -->|"결과 / 재생 입력"| RES["연구·백테스트 소비자"]
    LIVE --> OMS["기존 OMS / 주문·체결 실행"]
    OMS -->|"주문·체결 사실의 별도 이벤트"| ING
```

전체가 하나의 논리 플랫폼이며 각 박스가 반드시 서버 한 대나 단일 프로세스 하나라는 뜻은 아니다. 주문·체결 이벤트를 포함할 경우 시장 데이터와 다른 schema/소유권으로 보관한다. 이 도식은 **주문 실행을 data-service에 합치자는 구현 지시가 아니다.** 현재 OMS의 책임은 유지한다.

KX의 TP/RDB/RTE/HDB/gateway 분리는 이런 역할 설계의 근거다. gateway는 여러 데이터 프로세스를 연결하고, 무거운 계산은 데이터 가까이에서 수행할 수 있다. 아래 세부 운영 정책은 우리의 요구에서 도출한 제안이다. [KX Architecture](https://code.kx.com/q/architecture/), [Gateway 설계](https://code.kx.com/q/wp/gateway-design/).

### 23.2 프로세스별 상태 소유권과 자원

| 역할 | 주로 소유하는 것 | 분리해야 하는 작업 |
|---|---|---|
| 수집·로그·배포 | 원천 연결, 정규화, partition별 순서·진행 지점 | 대형 이력 질의·실험용 함수 실행 |
| 실시간 상태/피처 | 자신이 담당한 instrument/partition의 가변 상태 | 수개월 scan, 무제한 결과 materialization |
| 최신 조회 서비스 | 공개된 최신 상태와 제한된 읽기 뷰 | writer의 긴 정지나 전역 잠금이 필요한 연구 |
| 이력 writer | seal, 정정 revision, manifest 게시 | 독자에게 보이는 파일의 제자리 수정 |
| 연구 worker | 고정 snapshot, 질의 scratch, 전략별 실험 상태 | 상시 실거래의 유일한 데이터 공급 역할 |
| gateway / 요청 관리자 | 요청 ID, 대상 worker, 진행·결과·오류 추적 | 원자료 전량을 모은 뒤 하는 무제한 중앙 계산 |

가변 상태는 partition별 소유 프로세스를 정해 갱신 순서를 관리한다. 읽기 전용 불변 열 buffer는 mmap/공유 메모리로 여러 worker가 재사용할 수 있다. 갱신 중인 최신 상태는 단순히 같은 포인터를 주는 것으로 충분하지 않다. 게시 sequence·세대 또는 읽기 snapshot을 정의해 반쯤 갱신된 상태를 읽지 않게 해야 한다.

다중 프로세스와 멀티스레드는 함께 사용할 수 있다. 프로세스는 역할·장애·상태 소유권을 나누고, 그 안에서 병렬 연산을 수행한다. 1,500 instruments마다 프로세스를 하나씩 만들겠다는 뜻은 아니다. partition 크기·hot instrument·프로세스 수·내부 thread 수는 실제 부하로 결정한다.

### 23.3 실거래와 연구는 공통 데이터를 사용하면서 자원을 분리한다

241은 대량 연구·이력 조회·재생의 중심으로 적합하다. 필요할 때 꺼질 수 있는 운영 방식이라면, **241 중단이 실시간 수집·피처·거래 입력 중단을 뜻하지 않도록** 상시 경로를 다른 실행 자원에 둔다. 동일 플랫폼은 여러 머신에 걸칠 수 있다.

같은 머신에 일부 역할을 배치해도 프로세스 분리만으로 충분한 격리가 되지는 않는다. CPU·RAM·디스크뿐 아니라 page cache와 메모리 대역폭이 경쟁한다. 연구 작업의 동시성·메모리·임시 쓰기·네트워크 전송을 제한하고, 실거래 지연 기준을 만족하지 못하면 물리 자원도 분리한다.

실거래 전략은 초기 snapshot/필요한 이력을 준비한 뒤 구독으로 로컬 상태를 갱신할 수 있다. 매 틱마다 중앙 gateway나 NAS 이력 조회가 완료되기를 기다릴 필요는 없다. gateway가 구독을 설정하고 실제 데이터는 담당 서비스가 소비자에게 직접 전달하는 구성도 가능하다. 데이터 의미와 재개 규약은 같은 SDK에 남는다.

수집 로그의 ‘접수’, 디스크에 기록한 상태, 내구성이 보장된 상태, 구독자에 게시한 상태도 구별해야 한다. 모든 이벤트의 동기 디스크 반영을 실거래 경로에 넣을지, batch 내구성을 택할지는 지연·복구 손실 범위를 함께 정할 사항이다. 비동기로 게시했다는 이유만으로 무손실 복구를 보장하지 않는다.

## 24. 비동기 질의·액션·구독을 정식 실행 모델로 둔다

### 24.1 비동기 전송과 병렬 실행은 다르다

비동기 호출은 호출자가 결과를 기다리며 멈춰 있지 않는다는 의미다. 긴 계산을 받는 worker가 다른 일을 동시에 처리할 수 있는지는 별도 문제다. **비동기 I/O만 붙이고 무거운 질의를 실시간 상태 프로세스에 몰아주면 실거래 지연은 남는다.** 다중 worker, 역할별 큐, 실행 예산, 상태 소유권을 함께 설계한다.

KX gateway 문서는 요청 식별자로 여러 서비스의 비동기 응답을 추적·결합하는 방식을 설명한다. deferred response는 클라이언트가 결과를 기다리는 호출 형태를 유지하면서 gateway가 worker 응답을 기다리는 동안 다른 요청을 처리하도록 할 수 있다. 이런 동작을 플랫폼의 기본 실행 모델로 참고할 수 있다. [Gateway의 비동기 요청](https://code.kx.com/q/wp/gateway-design/#synchronous-vs-asynchronous), [Deferred response](https://code.kx.com/q/kb/deferred-response/).

또한 **비동기는 UDP를 뜻하지 않는다.** KX IPC도 TCP/IP 기반 메시지에서 sync·async·response를 구분한다. 우리는 요청/제어, 대량 결과, live 배포 각각에 맞는 TCP·로컬 IPC·공유 메모리 등의 운반 방식을 선택할 수 있다. UDP/multicast를 택하면 순서·누락 감지·재전송·snapshot 복구 규약이 추가로 필요하다. [KX IPC](https://code.kx.com/q/basics/ipc/).

### 24.2 유한 질의의 수명

최소 상태는 `accepted → running → succeeded / failed / cancelled`로 둔다. 취소 요청은 별도 상태/표시로 관리한다. 취소가 접수됐다는 사실을 이미 실행이 중단됐다는 뜻으로 사용하지 않는다. 시작 전 거부와 실행 중 실패도 구분한다.

| 계약 | 필요한 내용 |
|---|---|
| 요청 식별 | 재연결 후에도 식별 가능한 request/job ID, 요청 내용·질의/함수 버전 |
| 입력 고정 | 접수 시 또는 실제 실행 시 어느 시점에 snapshot을 고정하는지 명시. 재시도는 같은 snapshot 사용 |
| 배정·분할 | 대상 역할·partition, worker, 하위 요청 ID, 분할/병합 계획 |
| 완료 | 결과 schema, snapshot/cursor, 품질·coverage, 결과 handle 또는 batch |
| 오류 | 일부 partition 실패, 지원하지 않는 연산, 자원 부족, deadline 초과를 구분 |
| 취소 | worker에 전파, 중간 상태·임시 파일·공유 handle 회수. 이미 완료된 결과와의 경합 규약 |
| 재연결 | 상태 조회·결과 재수신 가능 기간, 만료 후 동작, client가 끊겨도 작업을 계속할지 여부 |

응답 도착 순서는 제출 순서와 다를 수 있다. request ID로 연결해야 하며, 여러 worker의 실패나 중복 응답도 처리해야 한다. 부분 결과를 완전한 결과처럼 반환하지 않는다. 부분 성공을 허용한 요청이면 누락 partition과 완료 범위를 결과에 표시한다.

순수 조회는 입력 snapshot을 고정하면 재실행이 가능하다. 큰 조회의 취소는 연산 경계에서 협력적으로 처리하거나 전용 worker를 종료할 수 있지만, 실거래 가변 상태와 연구 질의를 같은 프로세스에 넣고 전체 프로세스를 죽이는 구조는 피한다. deadline 초과도 실행 중단이나 부수 효과의 롤백을 자동 의미하지 않는다.

### 24.3 비동기 액션은 ‘전송 성공’과 ‘적용 완료’를 분리한다

여기서 액션은 피처 등록, 구독 생성/해제, 파생 데이터 계산·게시, snapshot 준비 등 플랫폼 상태를 바꾸는 동작이다. 이를 단순한 결과 없는 질의로 취급하면 재시도와 복구 의미가 불명확해진다.

최소한 다음을 구별한다.

- **접수:** 서비스가 명령을 받아 처리 책임을 수락했다. 수락 기록의 내구성 수준도 명시한다.
- **적용:** 소유 프로세스의 상태나 durable metadata에 변경이 반영됐다.
- **완료/관측 가능:** 결과 버전이 게시됐고 지정된 조회·구독 경로에서 확인할 수 있다. 다른 복제본의 반영까지 의미하는지는 별도다.

재시도할 수 있는 액션에는 idempotency key와 요청 내용의 대응을 보관한다. 같은 키·같은 요청은 기존 결과/상태를 반환하고, 같은 키·다른 요청은 거부하는 규칙이 후보다. 기록의 보존 기간, 재시작 후 복구, 이미 게시한 결과와의 대조까지 있어야 한다. **키를 붙이는 것만으로 모든 외부 부수 효과가 정확히 한 번 실행되는 것은 아니다.**

명령을 처리한 직후 응답 연결이 끊겼다면 클라이언트에는 결과가 불확실할 수 있다. 같은 ID로 상태를 확인하고, 소유자가 적용 상태를 복구·판정하도록 한다. 외부 효과와 기록을 함께 확정할 수 없다면 `unknown/reconciliation_required` 같은 판정 불가 상태도 드러내야 한다. 무조건 새 ID로 재전송해서는 안 된다.

주문·취소까지 비동기 액션의 범위에 넣는 경우에는 기존 OMS가 주문 상태의 소유자로 남는다. 데이터 플랫폼의 job 재시도 정책을 주문 전송에 그대로 적용하지 않는다. 이 문서는 데이터·실행 모델을 정리하며 거래 권한이나 주문 전송 구현을 변경하지 않는다.

### 24.4 구독은 계속되는 계산과 전달의 계약이다

구독에는 결과 데이터 외에 subscription ID, 정의 버전, 결과 키/revision, partition별 cursor, 신선도·처리 지연을 포함해야 한다. heartbeat는 연결이 살아 있다는 정보이며, 데이터가 최신 상태라는 증거와 구분한다.

느린 소비자가 생겼을 때도 데이터 종류별 정책이 다르다.

| 구독 의미 | 허용 가능한 처리 후보 | 금지할 해석 |
|---|---|---|
| 틱·호가 사건을 모두 소비 | 제한된 큐, 지연 통지, 재개 가능한 로그, 필요 시 연결 종료 후 복구 | 몰래 중간 사건을 버리고 연속했다고 표시 |
| 최신값 상태만 필요 | 계약에 명시한 coalescing으로 최신값 유지 | 최신값으로부터 중간 사건·거래량을 재구성할 수 있다고 주장 |
| 피처/window 결과 | 추가·대체·확정 의미와 revision 전달 | 잠정값 갱신을 새로운 독립 확정 행처럼 누적 |

TCP도 애플리케이션 재시작 후의 중복 처리나 로그 보존 기간까지 해결하지 않는다. 소비자가 어디까지 적용했는지와 재개 가능한 범위를 정의해야 한다. **최신 상태 구독**은 명시적 reset과 새 snapshot으로 재개할 수 있다. **전사건 소비·경로 의존 계산**은 보존 로그 또는 검증된 checkpoint와 후속 사건으로 복구해야 한다. 현재 snapshot만으로 누락된 거래량·사건 순서·전략 상태를 복원했다고 표시하지 않는다. 복구가 불가능하면 gap을 표시하고 계약된 중단/재초기화 정책을 적용한다. 무제한 큐가 실시간 프로세스의 RAM을 고갈시키지 않게 해야 한다.

## 25. 같은 플랫폼의 연구·실거래 결과를 연결하는 조건

### 25.1 snapshot에서 실시간 갱신으로 끊김 없이 넘어간다

“현재 상태를 받고 그다음 구독”을 별개 요청으로 단순 연결하면 그 사이에 발생한 사건을 놓칠 수 있다. 반대로 먼저 구독한 뒤 snapshot을 합치면 이미 반영된 사건을 다시 적용할 수 있다. 다음과 같은 계약이 필요하다.

1. 대상 partition과 schema/계산 정의 버전을 정한다.
2. snapshot이 반영한 **partition별 cursor**를 결과와 함께 반환한다. cursor에는 재시작으로 sequence가 바뀌는 경우를 구별할 세대/epoch도 필요하다.
3. snapshot을 만드는 동안 필요한 후속 사건을 로그/버퍼에서 보존하고, 해당 cursor 이후부터 전달한다.
4. 소비자는 snapshot과 그 이후 사건을 순서대로 적용하고 중복·누락을 확인한다.
5. 재연결 시 마지막 적용 cursor에서 재개한다. 복구 범위를 벗어나면 최신 상태 구독은 명시적으로 reset한 뒤 snapshot을 다시 받을 수 있다. 전사건 소비·경로 의존 계산은 로그/checkpoint 복구 또는 gap 처리로 전환하며, 새 snapshot이 누락 구간까지 복원했다고 간주하지 않는다.

여러 거래소·partition의 cursor 목록은 각 진행 지점의 집합이다. 이것만으로 전 시장의 단일 원자적 시점을 보장하지 않는다. cross-instrument 계산에는 허용 지연, 기준 시각, 필요한 입력의 준비 여부와 결측 처리를 별도로 정한다.

### 25.2 현재 데이터가 이력으로 이동해도 같은 질의가 성립해야 한다

현재 영역과 이력 영역을 조회할 때는 **조회에 고정된 manifest 세대와 partition별 경계**로 담당 범위를 나눈다. 이력으로 seal되는 도중이라도 같은 행이 두 번 나오거나 빠지면 안 된다. 새 이력이 게시됐다는 통지와 모든 reader가 그것을 적용했다는 상태도 구분한다.

gateway가 `현재 + 과거`를 각각 조회해 합칠 때 두 조회의 경계를 따로 잡으면 경합이 생긴다. 같은 조회 계획에서 경계·revision 해석을 고정한다. 일부 최신 프로세스가 늦으면 그 지연을 반환하거나 요구된 신선도를 만족하지 못했다고 응답해야 한다. 시간상 최신 행이 있다는 것만으로 전체 입력이 최신이라고 판단하지 않는다.

### 25.3 ‘같은 데이터’에는 당시 관측과 사후 정정의 구별이 포함된다

예를 들어 실거래 당시에는 OI 값 A가 알려졌고, 나중에 과거 시각의 값이 B로 정정됐다면 두 값은 모두 보존할 필요가 있다.

| 연구 목적 | 사용할 의미 |
|---|---|
| 지금 알고 있는 가장 정확한 과거 상태 분석 | 사후 정정을 반영한 이력과 해당 revision |
| 당시 이용 가능했던 정보로 전략 평가 | 판단 시각에 이용 가능했던 원천·revision |
| 실제 실행된 전략 결과 재현 | 그 전략이 실제 적용한 입력 cursor·순서·함수 버전·초기 상태와 설정 |

세 목적은 같지 않다. 원천 event time만 있는 과거 자료로는 나머지 시점 정보를 복원했다고 주장할 수 없다. 실제 소비 순서를 기록하지 않았다면 ‘당시 수집기가 받은 데이터’와 ‘전략이 판단 전에 적용한 데이터’도 완전히 같다고 단정할 수 없다.

재구성한 HFT 피처와 실제 live에서 만든 피처도 생성 경로·피처 버전·입력 coverage를 구별한다. 공통 플랫폼은 이 차이를 숨기는 대신, 어떤 실험이 어떤 의미의 입력을 사용했는지 확인할 수 있게 해야 한다.

### 25.4 피처 하나로 batch·live·replay의 의미를 검증한다

`spread_mean_60s@v3`를 예로 들면 피처 명세에 다음이 들어가야 한다.

- spread의 원천과 단위, bid/ask 갱신을 같은 상태로 보는 규칙.
- window가 event time 기준인지 수신/처리 시각 기준인지, 끝점 포함 여부.
- 무거래·결측·중복·같은 시각 사건의 순서, instrument 경계.
- 초기 snapshot, 필요한 warm-up, checkpoint에서 이어지는 상태.
- 늦은 사건·정정의 처리, 잠정 결과를 확정하는 조건.
- 계산 version과 정밀도·반올림·부동소수 오차 계약.

같은 기록된 입력을 분할 크기와 실행 속도를 달리해 재생해도 이 규약에 맞는 결과가 나와야 한다. wall clock을 읽는 계산은 재생에서도 제어 가능한 clock을 사용해야 한다. 정수·키·순서는 정확히 맞추고 부동소수 결과는 정해진 재현/오차 기준으로 비교한다. 의미 차이를 허용 오차로 감추지는 않는다.

실시간으로 확정되지 않은 미래 수익률 label은 연구 결과로 저장할 수 있지만 그 시각의 실거래 피처로 노출하지 않는다. “같은 함수 체계”는 모든 분석 함수를 즉시 실거래에 투입한다는 뜻이 아니다.

## 26. 플랫폼 관점에서 바뀌는 선택 기준과 작업 순서

### 26.1 반드시 소유할 설계와 선택적으로 만들 구현

| 반드시 소유할 설계·검증 | 기존 엔진을 이용하거나 자체 구현 여부를 나중에 정할 것 |
|---|---|
| 공통 데이터·시간·정정·품질 의미 | 파일 포맷, 압축 codec, 범용 인덱스 |
| 질의·함수 정의, 지원 연산과 결과 계약 | 새 질의 문법·parser, 범용 optimizer/JIT |
| 프로세스 역할, 상태 소유권, 라우팅 | worker 실행 라이브러리와 메시지 운반 구현 |
| 비동기 요청·명령·구독의 수명과 복구 | 결과 저장 방식·작업 관리 도구 |
| snapshot→구독, 현재→이력의 연속성 | mmap·공유 메모리·엔진 cache의 조합 |
| live/batch/replay 의미 일치와 자원 격리 | 일반 집계·조인 엔진, 병목인 전용 kernel |

‘소유한다’는 모든 코드를 직접 쓴다는 뜻이 아니다. 외부 구성요소를 쓰더라도 이 규약의 책임과 검증 기준을 우리 플랫폼에 둔다는 뜻이다. 구현 시 첫 `data-service` M1의 범위를 자동 확장한 것으로 해석하지 않는다. 이 문서는 목표 구조와 후속 설계 제안이다.

### 26.2 연구 I/O 외에 통과해야 할 검증

| 검증 | 통과해야 할 상태 |
|---|---|
| 공통 질의 의미 | 같은 지원 연산이 backend와 batch/live 실행에서 동일한 계약을 따름. 미지원은 명시적으로 거부 |
| snapshot→구독 | snapshot 작성·구독 등록·재연결 중 적용 사건의 누락·중복을 처리하고 경계를 증명. 상태 reset과 전사건의 복구 불가 gap을 구분 |
| 현재→이력 전환 | seal·정정 게시 중 고정 조회의 누락·중복·세대 혼합 없음 |
| 실거래 지연 | 연구 scan·spill·cache 준비를 병행해도 정의한 입력→피처→소비자 지연과 신선도 목표 충족 |
| 혼잡·느린 소비자 | 큐/RAM이 한도 안에 머물며 지연·재개·명시적 중단이 계약대로 동작 |
| 비동기 질의 | 응답 순서 변경·worker 실패·취소 경합·재연결에도 올바른 ID와 결과/오류 연결 |
| 비동기 액션 | 적용 직후 연결/프로세스 장애가 나도 중복 적용을 막거나 불확실한 상태를 명시하고 대조 가능 |
| live/replay 일치 | 동일 입력 순서·상태·clock·정의에서 같은 계산 의미와 정밀도 기준 충족 |
| 역할별 복구 | 허용한 복구 시간·손실 범위를 검증. 연구 머신 중단이 상시 경로를 중단시키지 않음 |

지연은 평균만 보지 않고 p99 등 긴 지연과 시장 burst 중 큐 길이를 함께 본다. 단순 데이터 도착, durable 기록, 피처 계산 완료, 전략 적용 시각을 각각 측정한다. 허용 지연·복구 시간·로그 보존·손실 범위는 실제 전략 및 원천별로 명세해야 한다. 이번 조사에서 임의의 마이크로초/밀리초 목표나 달성 수치를 제시하지 않는다.

### 26.3 다음 작업은 플랫폼을 관통하는 작은 사례부터

| 순서 | 결과물 | 다음 단계 조건 |
|---|---|---|
| 1. 플랫폼 명세 | 공통 데이터/질의/함수 모델, 시간 의미, 프로세스 역할, async/구독 상태도 | 연구와 실거래가 어떻게 같은 정의를 사용하는지 설명 가능 |
| 2. 기준 입력·정답 | 작은 HFT 표본 + 보조 데이터, 대표 인과적 피처 하나, 지연·정정·결측 사례 | 과거·live·replay의 정답과 달라져야 하는 경우가 명시됨 |
| 3. 최소 경로 검증 | 기록된 입력 → 현재 상태 → 과거 질의·async job·구독 → 소비자, 재시작/재개 | 26.2의 의미·연속성·복구를 제한된 범위에서 입증 |
| 4. 역할별 실행기 비교 | 19절의 연구 엔진 비교와 별도의 실시간 실행/전달 비교 | 엔진별 담당 역할·한계·자원 예산이 결정됨 |
| 5. 규모·격리 검증 | 전 instrument·수개월, burst, 동시 연구, 241 중단 | 연구 성능과 상시 경로의 지연·복구를 함께 충족 |
| 6. 소비자 전환 | 연구 소비자와 실시간 shadow 소비자가 같은 계약 사용 | 값·시점·성능을 확인한 후 운영 전환 범위 결정 |

처음부터 모든 언어·모든 질의·분산 고가용성 전체를 만드는 계획은 아니다. 하지만 최소 경로부터 질의·비동기·구독·실시간 상태를 포함해야, 연구용 cache를 완성한 뒤 다시 통합 플랫폼으로 재설계하는 일을 줄일 수 있다. 저장·scan microbenchmark는 이 단계들과 독립적으로 수행할 수 있어도 플랫폼 검증을 대체하지 않는다.

### 26.4 최종 판단

**사용자가 말한 ‘우리의 kdb’라는 통합 데이터·계산 플랫폼 방향에 동의한다.** 이전 문서는 그 목표를 연구 저장·읽기 성능 쪽으로 좁혀 다뤘고, 이를 보완했다. 필요한 것은 공통 질의·함수 체계와 비동기 다중 프로세스 runtime, 그리고 연구와 실거래가 공유하는 데이터 의미다.

반박하는 것은 새로운 q 호환 언어·독자 범용 엔진이 반드시 선행해야 한다는 해석, 그리고 하나의 데이터 소스가 하나의 물리 서버·실행 큐·매 틱 중앙 질의를 뜻한다는 해석이다. **논리적으로 통합하고, 실시간/연구의 실행 자원과 가변 상태 소유권을 분리하는 구조**를 추천한다.

따라서 DuckDB·Polars·mmap 중 어느 것이 빠른지는 여전히 중요한 질문이지만, 그것으로 전체 설계를 끝내지 않는다. 먼저 플랫폼의 사용·실행 모델을 정하고 각 역할을 구현할 수단을 선택한다. 이번 갱신은 문서 설계이며 실제 질의 언어·IPC·구독·거래 경로의 코드를 구현하거나 기존 프로토콜을 변경하지 않았다.
