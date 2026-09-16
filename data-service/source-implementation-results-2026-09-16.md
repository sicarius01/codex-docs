# Source 구현·검증 기록 — 2026-09-16

사용자 요청으로 게시한 구현 결과 공개본. 코드·DB·실행 로그·fixture·별도 계약 문서는 이번 게시 범위에 포함하지 않는다. 인수인계 A1~A4 source 구현·독립 검증과 A5 수집 예산·staging 한계 보고를 완료했다. 최종 Rust 테스트 71/71, fmt·clippy·build 및 별도 CLI·PyArrow·HTTP·Node 검증을 통과했다. sicadb 코어·양쪽 프로토콜 확정·전 종목 실행은 별도 검증이다.

## 시작 기준

- 기존 M1: `cargo test --all-targets` 30/30, `cargo fmt --all -- --check`, `cargo clippy --all-targets -- -D warnings`, `cargo build` 통과.
- 구현 작업은 원격이 없는 독립 로컬 Git 저장소에서 수행했다. 기존 파일을 보존했으며 소스 코드 커밋·푸시·배포는 하지 않았다. 이후 사용자 요청으로 이 결과 문서만 별도 게시했다.
- 기존 검토 프로세스와 수집 DB는 변경하지 않았다. 이번 검증은 임시 DB·loopback fixture 및 별도 소량 공개 응답으로 진행했다.
- 변경 전 코드·설정·테스트 사본은 별도 로컬 검증 산출물로 보존했다. 사본 자체는 공개하지 않는다.

## 코드에서 확인한 문제와 구현 방향

기존 `collect_symbol`은 마지막 저장 봉에서 순차 전진하며, `candles`는 중간 슬롯 하나가 없어도 페이지 전체를 거부한다. 31일을 넘긴 재시작은 오류로 끝난다. `Store::ingest`는 원본과 revision을 한 transaction으로 보존하지만 복구 checkpoint는 없다. 원천 cooldown은 프로세스 메모리에만 있다.

HTTP 조회·catalog·health와 collector가 하나의 `Arc<Mutex<Store>>`를 공유한다. health도 catalog 전체 집계를 호출한다. 최신/과거 조회 의미는 유지하면서 bounded reader 연결과 writer를 분리하고, 수집 상태는 과거 완전성과 별도로 표시한다.

## 진행 상태

| 단계 | 상태 | 완료 증거 |
|---|---|---|
| A1 최신·복구 분리 | 구현·독립 검증 완료 | 전체 49개 테스트·fmt·clippy·build 통과, 32일 중단·영구 결측·429/418 재시작·구 schema 이관 CLI 검증 |
| A2 Arrow·outbox | 구현·독립 검증 완료 | PyArrow 23.0.1로 별도 구성한 6개 revision 및 제공 fixture 대조, rollback·재전송·quota 검증 |
| A3 확정 펀딩 | 구현·독립 검증 완료 | 실제 소량 응답 재생·Decimal·migration·페이지·재시작·Arrow 대조 |
| A4 현재 OI·5분 통계 | 구현·독립 검증 완료 | 별도 계열·시각·정밀도·복구·오류·migration·Arrow·HTTP 대조 |
| A5 1,500 instruments 예산·staging | 한계 보고 완료 | 8회 원천 조회·현재 catalog·예산 계산·합성 10,000행 측정. 전 종목 실행은 미검증/비활성 |

## 검증 경계

고장 주입 결과와 실제 원천 관측을 구분한다. 이번 소량 검증은 24시간 운영 관찰·전원 손실·전 종목 수집 SLA를 증명하지 않는다. 각 단계의 요청 수·복구 시간·schema/설정/API 영향·실제 실행 상태를 아래에 갱신한다.

## A2 독립 검증

작성자와 다른 검증 담당이 기존 schema의 합성 DB를 별도로 만들고 producer CLI → Arrow IPC → PyArrow 23.0.1로 대조했다. 같은 seq를 한 행씩 나누는 checkpoint, 2^53을 넘는 정수, unknown null, A→B→A 정정, 늦은 백필, raw SHA-256을 확인했다. 제공한 정답 fixture는 4개 IPC 파일·6개 revision·총 68,864 bytes이며 독립 기대값과 일치했다.

outbox checkpoint 갱신 실패 시 batch와 cursor rollback, quota 초과 시 checkpoint 불변, 미완성 materialized 파일 재생성, 별도 프로세스 재전송과 ACK 유실 후 중복 제거, consumer 저장 실패 시 receipt rollback, manifest 경계 변조 거부, source checkpoint 레코드 변경 감지를 확인했다. 시험 ACK로 outbox를 정리하거나 sicadb 게시 완료로 표시하지 않는다.

독립 보고와 Arrow fixture 대조 결과는 별도 로컬 검증 산출물로 보존했다. 코드·fixture 자체는 공개하지 않았다.

## A1·A2 통합 검사

`cargo fmt --all -- --check`, `cargo clippy --all-targets -- -D warnings`, `cargo test --all-targets`, `cargo build`를 통합 담당이 실행해 통과했다. 테스트는 수집/원천/API 21, A1 저장 복구 5, producer 8, 기존 저장 계약 15로 총 49개다.

최신 구간을 먼저 요청하고 과거 복구는 cycle 전체 요청·경과시간 예산으로 제한한다. 명시적 범위는 작업 원장을 사용해 반복 호출 시 이미 처리한 앞부분을 다시 작업으로 쌓지 않는다. 결측/무효 행 주변의 유효한 값과 원본을 보존하며, 같은 영구 결측의 작업 ID·시도 횟수·다음 시각을 재시작 후 유지한다.

저장과 복구 진행은 같은 transaction으로 갱신한다. HTTP 최신/기간·catalog/health의 연결과 동시성 한도를 분리했고, catalog는 증분 요약, 기간 coverage는 제한된 고정 view cache를 사용한다. SQLite는 rusqlite 0.40.2의 bundled 3.53.2이며 실제 런타임 버전 검증을 통과했다.

구현자의 느린 원천 fixture에서 200ms 복구 예산은 약 204ms·요청 1회로 끝났으며 최신 데이터는 이미 저장돼 있었다. 독립 CLI fixture의 32일 중단은 첫 cycle 약 438ms에 최신 조회가 회복됐다. 이는 loopback·축소 fixture 측정이며 실제 거래소 운영 지연 분포가 아니다.

## A3 확정 펀딩

`derivatives.funding.settled`는 source fundingTime의 실제 millisecond와 nullable rateType을 함께 키로 삼는다. 비율은 signed ×1e8 정수, 가격은 nullable ×1e8 정수다. 같은 시각의 Regular/Special/null, A→B→A 정정, 늦은 과거 사건을 구분하고 provenance를 보존한다. 예상 펀딩·과거 정산 일정·원천 공개 시각을 추정하지 않으며 coverage/freshness는 unknown이다.

최신 정산을 먼저 요청하고 과거 구간을 영속 작업으로 처리한다. 기본 worker는 펀딩 비활성이며 활성화 시 bootstrap 24시간, limit 100, 복구 2페이지/cycle, funding 요청 간격 667ms다. 명시적 범위는 millisecond 그대로 최대 31일이며 완료 전 반복 명령은 cursor를 이어간다. inclusive 원천 pagination은 마지막 timestamp를 겹쳐 읽고 같은 시각에서 더 진행할 수 없으면 오류·작업을 남긴다. 429/418 cooldown과 endpoint pacing은 재시작 뒤에도 유지한다.

서로 다른 검증 담당의 CLI 검증을 통과했다. 저장한 실제 BTC/ETH 응답을 loopback에서 재생하여 independent Decimal 값 및 raw hash를 대조했고, baseline M1 schema 이관과 immutable revision 유지, 중복 재수집, 페이지 경계, 7시간 전 최신 정산과 32일 backlog 분리, 무효 9번째 소수·원본 보존, 재시작 뒤 429 재호출 억제를 확인했다. 별도 검증은 URI 대소문자/끝 slash를 바꾼 봉 수집 경로에서도 cooldown 우회를 막는 것을 확인했다.

source→query→Arrow→consumer 재전송이 연결됐으며 펀딩 제공 fixture 3파일·6revision·43,350bytes가 독립 PyArrow 23.0.1 정답과 일치했다. 독립 CLI·Arrow 검증 결과는 별도 로컬 산출물로 보존했다. 전체 검사에서 테스트용 불필요한 vec lint와 dataset 수를 2로 고정한 기존 catalog assertion을 발견했다. vec는 배열로 바꾸고 catalog는 정확한 5개 dataset 목록을 검증하도록 갱신했으며 최종 전체 검사는 통과했다.

## A4 현재 OI·5분 통계

현재 `derivatives.open_interest.snapshot`은 실제 응답을 받은 시각별 관측을 남긴다. 원천의 `time`은 `source_time_ms`로 분리하며 동일 원천 값·시각을 다시 받아도 다른 수신 millisecond의 관측을 보존한다. 조회 freshness는 선택 범위의 최신 관측에 붙은 원천 시각으로 평가한다. 최근 수신으로 오래된 원천 값을 fresh로 바꾸지 않으며 미래 원천 시각은 unknown이다. 수집 전 과거 snapshot의 백필은 HTTP 호출 전에 거부한다.

`derivatives.open_interest.statistics.5m`는 원천 timestamp를 기간 종료 시각으로 보존한다. 수량·가치·nullable CMC는 정확한 ×1e8 정수이며 기존 Feeding ×1e4/UDP를 변경하지 않는다. 원천 endpoint의 물리 단위 설명이 부족한 부분은 source-native 필드와 해석 근거를 명시했다. Arrow timestamp resolution 1ms와 통계 period 300000ms를 구분한다.

최신 통계와 과거 복구를 나누고 경계 millisecond를 보존한다. 양끝의 inclusive/exclusive 차이는 요청을 `start=cursor-1`, `end=exclusive_end`로 넓힌 뒤 `[start,end)`로 필터링하고 경계를 겹쳐 읽어 처리한다. 새 수집은 보수적인 최근 28일 안이며 원천의 최근 1개월을 정확한 30일 보장으로 바꾸지 않는다. 오래된 미완료 prefix는 `outside_collection_window`로 남겨 `suspended_jobs`와 null 재시도 예정으로 표시한다. 최신과 남은 최근 범위는 계속 진행하며 보류 범위가 있으면 `has_more=true`다.

기본 두 OI worker는 비활성이다. 통계 기본값은 24시간 bootstrap, limit 100(최대 500), 복구 2페이지/cycle, 자체 요청 간격 334ms이며 일반 요청 간격과 공통 cooldown도 적용한다. snapshot 실패는 누락 관측으로 기록하고 가짜 값을 만들지 않는다. health의 실패 수는 원자적으로 누적한 counter를 읽어 실패 이력을 매번 스캔하지 않는다.

서로 다른 검증 담당이 loopback/별도 DB에서 실제 보존 응답 05~08을 재생하고 Decimal·원본 SHA-256을 대조했다. 동일 source time의 다른 관측, source-time stale, millisecond range, 정밀도/overflow/음수 거부, 원본·실패 상태 보존, inclusive/exclusive 페이지, null CMC, A→B→A, 고정 view, 32일 중단과 보류 작업, 429/418 이후 다른 dataset 경로의 재호출 억제를 확인했다. 기존 M1과 A3 DB migration 모두 불변 revision/key/view를 보존했다. source→query→Arrow→consumer 재전송도 통과했다.

OI 제공 fixture는 5파일·8revision·75,970bytes다. 통합 담당이 exporter/helper와 독립된 PyArrow 코드로 수기 정답·typed schema·scale·원천 필드 metadata·null·서로 다른 시각·checksum·manifest 경계를 대조했다. 독립 CLI·PyArrow 검산 결과는 별도 로컬 산출물로 보존했다. 추가 독립 검증의 상세 경로는 독립 보고(별도 비공개 문서)에 있다.

## 변경 범위와 sicadb 전달물

| 영역 | 최종 변경 |
|---|---|
| 수집 | 기존 collector/source에 최신 우선·복구 예산·원본 오류 격리·영속 cooldown/pacing. funding.rs와 open_interest.rs에 계열별 원천 계약. |
| 저장 | 기존 records 제약을 5계열로 transaction 이관. 봉/펀딩/통계 작업·frontier·요청 원장, 증분 series summary, source 대기, OI 관측 상태/실패 이력. 기존 revision/view API 유지. |
| 조회·화면 | 제한된 별도 reader, 가벼운 health, catalog/cache. 기존 query 구조에 새 dataset 추가. 원천·관측·기간 종료를 구분하는 표·상태. |
| 출력 | src/producer.rs 및 producer CLI, typed Arrow 60.0.0, 별도 outbox WAL/FULL, composite checkpoint·trial consumer. SQLite bundled 3.53.2. |
| 전달 자료 | producer 계약(별도 비공개 문서), fixtures/producer-v0의 M1·funding·OI input/수기 expected/IPC manifest, examples/fixture_consumer.py, demo/demo-funding/demo-oi 명령. |

producer 계약은 source adapter v0 초안이다. sicadb가 파일을 받아 검증·영속화·catalog에 게시하는 transaction 및 durable ACK는 양쪽 통합에서 확인해야 한다. 시험 ACK로 outbox를 지우지 않는다. 알려지지 않은 source publication/consumer availability 시각은 null이다.

## 최종 통합 검증·실행 상태

- `cargo fmt --all -- --check`, `cargo clippy --all-targets -- -D warnings`, `cargo test --all-targets`, `cargo build` 모두 통과.
- Rust **71/71**: main 33, A1 저장 복구 5, A3 저장 4, A4 저장 4, producer 10, 기존 저장 계약 15.
- `node tests/ui_dataset_contract.mjs` 통과. 실제 inline 화면 스크립트를 실행해 큰 정확 정수·음수·null·millisecond·dataset 전환·원천 freshness·nested health·관측 범위 요청을 확인했다.
- 별도 copied executable/SQLite backup의 loopback HTTP에서 HTML·health·5계열 catalog, 펀딩 음수 값, 두 OI 계열의 실제 원천 값·서로 다른 시각·고정 view range를 확인했다. HTTP 검증 결과는 별도 로컬 산출물로 보존했다.
- 실제 렌더링 검증은 미실시다. CUA가 `No browser is available`을 반환했고 apps/browsers inventory가 비어 있었다. HTTP·offline DOM 확인을 브라우저 시각 검증으로 표시하지 않는다.
- 공개 원천 요청은 초기 표본 **8회**뿐이며 이후는 loopback/보존 응답으로 검증했다. 임시 HTTP 서버는 검증 후 종료했다. 구현 종료 당시 기존 M1 검토 프로세스는 변경 없이 실행 중이었다. 새 상시 수집기·서비스는 실행하지 않았다.
- 구현 작업에서 기존 파일을 보존했고 소스 코드 커밋·푸시·sicadb/feeding/bv_sync 수정·운영 배포는 하지 않았다. 이 결과 문서의 공개는 후속 사용자 지시에 따른 별도 작업이다.
- 최종 검사 목록과 source/test/config/UI SHA-256은 별도 로컬 검증 산출물로 보존했다. 구현 작업에서 갱신한 주요 문서 8개의 로컬 링크가 존재함을 확인했다. 그 문서들은 이번 게시 범위에 포함하지 않는다.

## 남은 운영·통합 검증

24시간 연속 관찰과 실제 조회 가능 지연 분포, 전원 손실 내구성, 디스크 전체 용량·원본/revision/outbox 보관 정책, 다른 프로세스까지 합친 거래소 IP 예산, sicadb durable ACK·게시, 실제 연구 소비자 대조는 남아 있다. A5 보고(별도 비공개 문서)의 합성 10,000행 결과는 전 종목 실행 성능이 아니다. 현재 기본 outbox logical quota를 1,500종목 분봉으로 환산하면 약 6.1시간이며 자동 정리는 없다. 전 종목 실행 gate를 통과하거나 가동했다고 보고하지 않는다.
