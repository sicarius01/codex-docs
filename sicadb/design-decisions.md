# sicadb — 현재 결정과 작업 방식

갱신: 2026-09-17. 공유 메모리 요구사항과 검증 계획을 현재 기준으로 삼는다.

갱신: 2026-09-17. **Rust 계산 코어·Arrow/mmap reader와 K2 첫 TCP/worker 경로를 구현하고 Windows 11에서 검증했다.** 공개 문서다. 검증 범위와 성능은 K0/K1 결과 (프로젝트 내부 문서: k0-k1-results-2026-09-16.md)와 K2 결과 (프로젝트 내부 문서: k2-results-2026-09-16.md)를 따른다. 영속 저장 서버·APL·운영 전환은 미완료다. 짧은 진입점은 [README](README.md)다.

## 1. 읽는 순서와 문서 역할

1. [README](README.md): 현재 상태와 문서 진입점. 이 문서: 현재 결정과 작업 방식.
2. [다음 작업](next-work.md): shared memory live·recording·복구의 현재 우선순위. 첫 작업 정의 (프로젝트 내부 문서: k0-k1-work-item.md)와 K2 작업 계약 (프로젝트 내부 문서: k2-work-contract.md)은 과거 근거로 보존했다.
3. [성능·정확성 계약](performance-and-correctness-contract.md): 관련 기능의 필수 조건.
4. [전체 실행 계획](implementation-plan.md): 컴포넌트 경계와 후속 단계. 작업에 필요한 절만 읽는다.

기존 리서치 문서는 근거다. 최신 합의와 다른 과거 제안을 현재 요구로 다시 채택하지 않는다. 이 진입점이 상세 수치·안전 계약을 생략하거나 약화하는 근거가 되지는 않는다. 충돌을 발견하면 관련 문서를 함께 수정한다.

## 2. 합의한 방향

메모리·I/O의 현재 기본은 불변 FILE과 필요 범위의 RAM 준비이며 NAS→shared RAM과 선택 HDD cache를 구분한다. SSD 필수로 해석하지 않는다.

| 영역 | 현재 결정 |
|---|---|
| 목표 | 빠르고 가벼운 범용 값·배열·함수·테이블·메시지 런타임으로 연구와 실시간 역할을 조립한다. 전용 통계 RPC 목록으로 범위를 제한하지 않는다. |
| 위치·언어 | Full Trading 내부 독립 컴포넌트 `sicadb/`, Rust. 독립 Git, 고정 toolchain/Cargo workspace, core·Arrow reader·CLI가 있다. |
| 사용자 언어 | APL 우선 검증. 실제 Luna·CoT off 연결에서 작성성을 평가한다. q 호환·완전한 APL 호환은 초기 약속이 아니다. |
| 데이터 | HFT와 추가 원천 모두 최종 범위. 과거·현재·재생에서 데이터와 함수의 의미를 공유한다. |
| 저장 | `.arrow`(IPC FILE), `.arrow.stream`(IPC STREAM)을 사용한다. 크기·시간 기준으로 파일을 확정하며 하루 경계까지 기다리지 않는다. |
| 메모리·I/O | RAM 초과 데이터가 정상 입력. 필요한 NAS 범위를 shared RAM에 준비하고, 선택적 HDD/SSD의 불변 Arrow 파일+mmap을 재사용, 제한된 최근 상태·작업 메모리. NAS 자료는 필요한 범위를 준비해 재사용한다. |
| 저장 책임 | sicadb가 동시성·복구·snapshot·게시를 직접 관리한다. 코어 저장/metadata에 SQLite를 필수 의존성으로 넣지 않는다. |
| 실시간 가시성 | 최신 관측값은 디스크 영속화를 기다리지 않고 반드시 사용할 수 있어야 한다. live 공개와 durable 확인을 분리한다. |
| OS | 메인 PC·241 모두 Windows 11이 주 타깃. feeding PC·일일 배치 PC의 Windows 10도 가능한 범위에서 지원·실기 검증한다. OS별 재컴파일 허용. |
| 실행 | 시세 append writer와 불변 reader, 연구 worker와 실시간 역할을 분리하며 OMS는 다중 writer·lock을 유지한다. 비동기 job·취소·자원 예산을 코어 경계부터 갖춘다. |
| 느린 계산 | 모든 이벤트 처리, 최신 요청 합치기, 주기 실행을 연산별로 선택한다. DL의 최신 요청 합치기는 수집·호가 재구성·필요 피처 갱신을 생략하지 않는다. |
| 품질 | 저수준 함수는 첫 완료부터 SIMD 후보·할당·복사·메모리·동시성을 검증한다. 정답·안전성·성능 증거가 완료 조건이다. |

현재 data-service의 SQLite source/outbox는 다른 세션의 현행 구현이다. 그 자료와 재전송 근거를 임의로 지우지 않는다. 최종 source→sicadb 전환에서는 자체 저장 계약과 복구 시험을 통과한 뒤 책임을 넘긴다. 최신 source 범위는 producer 초안 (프로젝트 내부 문서: producer-contract-v0.md)에서 확인하며, fixture ACK를 sicadb durable ACK로 취급하지 않는다.

## 3. 공유 메모리 우선 결정

같은 호스트의 live 시세·feature·1초 테이블·배치 전달은 공유 메모리를 기본으로 한다. 공통 테이블·조회·구독 API는 direct read와 push를 함께 제공한다. push는 payload를 복제하지 않고 공통 row sequence의 범위, segment offset, generation을 알리며 소비자 executor가 callback을 실행한다. TCP는 원격 호스트와 선택적 제어·호환 경로이고 frame은 직렬화한다. 이 결정은 전체 DB가 하나의 물리 RAM이나 중앙 CPU라는 의미가 아니다.

column별 독립 회전은 금지한다. 하나의 row sequence를 공유하는 columnar batch/segment를 사용하고 `published_end`(exclusive), reader done, active pin, time retention, durable position을 별도 상태로 둔다. ring wrap과 resize는 sequence·generation·offset 길이를 검증한다. 고정 ring N을 바꾸어 modulo를 재해석하지 않으며, 예를 들어 [1000,1080), N=1024는 물리 [1000,1024)+[0,56)로 해석한다.

작성 규약은 시세·feature·1초 적재에서 main process가 single writer를 관리하는 것으로 한다. 프로세스 내 다중 thread는 partition별 실제 writer를 명시한다. 파생 worker는 결과를 main에 전달한 뒤 publish하는 비용을 시험한다. OMS는 사용자 확정 다중 writer와 lock 설계를 예외 없이 보존하며 owner·actor·command queue 단일화를 제안·명세·구현하지 않는다.

## 4. 이번 정리에서 채택한 설계 기본값

아래는 구현 전 명세를 구체화할 기본값이다. 성능 수치나 구현 완료를 뜻하지 않는다.

- **통신:** 같은 호스트 live 경로는 공유 메모리 descriptor와 검증된 offset을 사용한다. TCP는 원격·제어·호환 경로로 작은 frame을 직렬화한다. q wire 호환은 요구하지 않는다.
- **통신 제어:** request/job/stream 식별, framing·길이 제한·버전 협상, 취소·오류·완료, 재연결·재개, byte 단위 backpressure를 포함한다. 큰 결과 전송과 제어 경로를 분리해 실제 지연을 검증한다. TCP 수신은 durable ACK가 아니다. Arrow Flight는 대용량 전송의 비교 기준으로 사용한다.
- **저장·작성:** live와 recording은 bounded queue·worker·메모리를 분리한다. stream 기록과 FILE seal/manifest/recovery를 구현한다. 단일 writer 규약은 OMS에 적용하지 않는다. OMS의 다중 writer·lock 상태 갱신 의미는 그대로 유지한다.
- **파일 수명:** 진행 중 stream/재전송 기록 → 완성된 FILE segment → 복구 가능한 manifest/snapshot 게시 → reader 수명 종료 후 회수. producer가 이미 완성 FILE을 제공하면 불필요하게 stream으로 왕복하지 않는다. IPC STREAM 자체는 완전한 WAL 계약이 아니므로 sequence·commit 경계·잘린 tail·checksum·sync 규칙을 따로 정한다.
- **APL 범위:** 참조 방언 하나와 명시적 작은 지원 프로필부터 고정한다. 이름만 같은 연산에 임의 의미를 붙이지 않는다. 방언·인덱스 원점·수치/오류 프로필은 언어 과제 시작 전에 문서로 정하고 공개 호환 약속 전에는 변경할 수 있다.
- **긴 job:** `queued`, `running`, `cancel_requested`, `completed/failed/cancelled`를 구분한다. deadline과 취소 요청은 실제 실행 종료가 아니다. 실행 종료 확인 후 scratch·결과·snapshot pin을 회수한다. 비협조적 native 연산은 별도 worker로 격리한다.
- **DL latest 정책:** 모델·종목 또는 종목 묶음별 실행 중 작업과 최신 대기 상태를 관리한다. 입력 snapshot을 고정하고, 마지막 유효 완료 결과에 입력 기준 시각·모델 버전·게시 시각을 붙인다. 호출 간 상태를 유지하는 모델은 순서 보존/재생 계약이 먼저다. 일반 연구 job이나 주문 action에 자동으로 요청 생략을 적용하지 않는다.

## 5. 사용자 답변으로 확정한 운영 조건

| 항목 | 결정 | 구현·검증 조건 |
|---|---|---|
| 영속화 전 실시간 공개 | **반드시 지원.** 사용자 명시: 저장 완료를 기다리면 실시간 요구에 맞지 않음. | live 공개가 fsync/파일 seal/manifest 게시를 기다리지 않는 독립 경로. durable ACK는 실제 복구 가능한 경계에서만 발행. |
| 대상 OS | **Windows 11 우선, Windows 10도 가능한 범위에서 커버.** OS별 재컴파일 허용. | 두 Windows 버전의 실제 build·실행·mmap·IPC·복구 확인. Win11 통과나 크로스 컴파일만으로 Win10 지원 완료로 표시하지 않음. |

초기 target은 x86_64 Windows/MSVC를 기준으로 확인한다. 구체 Windows build·CPU ISA·사용 가능한 API를 각 머신에서 기록한다. Linux 운영 지원은 현재 초기 합격 조건에 넣지 않으며 OS 코드를 분리해 확장 여지를 둔다.

live 처리와 기록 큐·예산을 분리한다. 디스크 지연/실패가 live 공개의 정상 경로를 막지 않게 하며, 기록 지연·기록 중단·복구 가능한 범위를 명시적으로 표시한다. 무제한 큐나 거짓 durable ACK로 문제를 숨기지 않는다. 장애로 영속화되지 못한 live 관측까지 과거 조회/재생에서 복원된다고 주장하지 않는다.

**현재 첫 구현을 위해 추가로 답변받아야 할 질문은 없다.** 모델별 최대 예측 age와 실거래 지연·장애 대응 목표는 해당 전략을 연결할 때 실측 근거와 함께 정한다.

## 6. 질문 대신 구현·측정으로 닫을 항목

| 항목 | 정할 방법·시점 |
|---|---|
| segment 크기·seal 주기·종목 묶음·열군 | 실제 HFT/추가 데이터의 pruning·소파일 비용·읽기 증폭, ingest 지연으로 저장 구현 전에 비교 |
| SIMD·allocator·NUMA·worker 수 | 고정 환경에서 scalar/자동 벡터화/ISA 후보와 대역폭·tail latency 비교 |
| 시간·정수·scale·오류 의미 | 각 연산 착수 전 타입/실패 계약과 독립 정답 고정. source별 metadata를 검증하고 변환은 명시적 수행 |
| TCP byte layout·payload codec·Arrow 버전 | K2 첫 내부 제어 규약·parser/길이/deadline 검증 완료. 내부 API (프로젝트 내부 문서: k2-api.md). 큰 배열 전송·구독은 별도 확장·측정 |
| APL 방언·지원 primitive·LLM 과제 | 기존 의미와 정확값을 보존하는 프로필을 선택하고 30개 과제로 실제 연결 평가 |
| live 지연·허용 예측 age·GPU batch | live의 디스크 비대기는 확정. 구체 지연/예측 age는 실제 모델·전략과 장비 실측 후 결정. 정하지 않은 경로는 실거래 준비 완료로 표시하지 않음 |
| 복제·자동 failover·GPU backend | 단일 호스트 복구·worker 격리·모의 느린 연산을 검증한 뒤 독립 작업으로 확대 |

## 7. 리뷰 처리와 현재 증거

Claude 리뷰 (프로젝트 내부 문서: performance-and-correctness-contract-review-2026-09-16.md)는 원문을 보존한다. 유효한 지적 중 실패 출력·scaled 연산·실제 경계 fixture·복구 경로 발동·진행성은 코어 계약에 반영했다. 다음 해석은 그대로 채택하지 않는다.

- cache-builder의 arrow-rs 55 writer는 stream이다. 별도 변환기의 48컬럼 FILE과 구분한다. writer (프로젝트 내부 문서: writer.rs), [변환 실측 보고서](https://github.com/sicarius01/codex-docs/blob/main/quant-research/nas-arrow-conversion-benchmark-2026-09-16.md).
- Windows 공유 모드는 열린 파일에 대한 보호 수단이다. snapshot 전체 수명·reader 종료·회수 순서를 대체하지 않는다. 배타 open 실패를 곧바로 reader 생존 증거로 해석하지 않는다.
- 프로세스 강제 종료 시험은 전원 손실 보장과 다르다. Miri/ISA 지원은 고정 버전과 실제 실행 경로별로 확인한다.
- OI의 기존 ×1e4와 producer의 다른 scale을 이름만 보고 섞지 않는다. 단위·scale·원천 정의를 dataset별로 검증한다.

**구현·실행:** Rust workspace, 실제 HFT/펀딩 fixture의 owned/mapped 계산, 작은 함수 호출·조합, kernel/reader 검증과 로컬 benchmark에 더해 실제 TCP·worker pool·취소·강제 종료·부모 사망·재시작·페이지 공유를 확인했다. K0/K1 결과 (프로젝트 내부 문서: k0-k1-results-2026-09-16.md)와 K2 결과 (프로젝트 내부 문서: k2-results-2026-09-16.md)에 실제 범위·측정·미검증을 기록한다. **미완료:** APL 평가, 영속 저장·live, Windows 10/241 실기, RAM 초과·NAS·장기 혼합 부하. 외부 변환기·source의 성공을 sicadb의 검증으로 집계하지 않는다.

공개 저장소의 성능 계약은 기존 v0다. 이번 로컬 보완본은 게시하지 않았다.

## 8. Codex와 작업하는 방식

문서는 유지하되 현재 상태와 작업 증거를 계속 갱신한다. 매 작업은 **목표·필요한 입력/문서·제약·완료 증거**로 정의한다. 구현 방법은 이 경계 안에서 agent가 결정하고, 성능 가설은 측정으로 채택한다.

- README는 짧은 진입점으로 유지한다. 연구 기록 전체를 매번 필수 입력으로 넣지 않는다.
- 한 작업은 사용자가 확인할 수 있는 결과 하나를 만든다. 관련 설계→구현→검증→리뷰까지 이어서 수행한다.
- 새 세션은 완료 여부·관련 파일·검증 명령·원시 결과 위치·남은 실패가 적힌 다음 작업 문서에서 시작한다. 긴 작업 중 문맥 압축도 이 기록으로 이어간다.
- 문서의 주장과 실제 코드/실행 결과를 대조한다. 계획에 체크했다는 이유로 기능 완료로 판단하지 않는다.
- 독립적인 경계 검토·정답 검증은 별도 agent에 맡길 수 있다. 공유 파일 편집은 owner를 나누거나 worktree로 분리한다.
- 같은 목표의 작업 도중에는 불필요하게 세션을 나누지 않는다. 한 검증 가능한 결과가 끝나거나 실제로 다른 작업으로 분기할 때 새 세션을 사용한다.

이는 sicadb에 적용할 작업 방식이다. 공식 OpenAI 문서도 필요한 문맥과 완료 기준, 실용적인 짧은 AGENTS.md, 실행·검증·리뷰를 권장하며 긴 작업에는 갱신 가능한 실행 계획을 제시한다. [Best practices](https://learn.chatgpt.com/guides/best-practices), [Execution plans](https://developers.openai.com/cookbook/articles/codex_exec_plans).
