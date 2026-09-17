# sicadb

Rust로 만드는 범용 값·배열·함수·테이블·메시지 런타임. HFT와 추가 데이터의 연구·실시간 처리를 같은 데이터·함수 의미로 연결한다.

## 현재 상태

- **2026-09-17 — 공유 메모리 통합 요구사항과 검증 계획 문서화·독립 검토 완료.** Windows 11 x64/MSVC를 주 대상으로 하고 Windows 10 feeding/batch 가능 범위를 별도 실기 검증한다. 같은 호스트 live 시세·피처·1초 테이블·배치 전달은 공유 메모리를 기본으로 하며, TCP는 원격 호스트와 선택적 제어·호환 경로다. 세부 계약은 [공유 메모리 런타임 계약](shared-memory-runtime-contract.md), 검증 순서는 [공유 메모리 검증 계획](shared-memory-validation-plan.md)에 기록한다. 구현·실행 검증은 아직 시작하지 않았다.
- **2026-09-17 — 프로세스 간 대기형 락 비교 측정 완료.** 같은 Win32 Mutex의 인계 지연 합산 p50은 스레드 8.0µs·프로세스 8.7µs였다. 별도 진단의 18개 case와 warmup이 통과했고 원시 표본 601,206개를 독립 감사했다. 측정 보고서 (프로젝트 내부 문서: ipc-lock-probe-results-2026-09-17.md)에 무경쟁 비용·대기 CPU·한계와 NAS→RAM/HDD·DLL 선택을 정리했다. 개발 PC의 제한된 관측이며 OMS 다중 작성자 설계와 운영 코드는 변경하지 않았다.
- **2026-09-16 — 제어 지연 진단·재검증 완료.** 최신 진단 결과 (프로젝트 내부 문서: pacing-diagnosis-results-2026-09-16.md)를 먼저 읽는다. 별도 observer/host와 sleep 대조군을 구현하고 원래 바이너리도 재실행했다. 생산 core·Arrow·worker·IPC·service 47개 파일은 변경하지 않았다.
- **2026-09-16 — 현재 구현 전체의 성능 개선·전후 비교 완료.** 직전 단계의 성능 개선 내역 (프로젝트 내부 문서: performance-sweep-results-2026-09-16.md)에 6개 crate의 계산·입력 준비·IPC·worker 알림·Windows TCP 대기 개선을 기록했다. 변경 전후 소스·binary·raw는 `results/perf-sweep-20260916/`에 보존했다.
- **실제 동작:** TCP 비동기 접수·조회·취소·Wait, 지속 실행 worker pool, 불변 Arrow FILE/mmap 재사용, 제한된 metadata 캐시, 예산·종료 확인·재시작. live ingest와 영속 저장 서버는 아직 없다.
- **동일 API release 비교:** 5회씩 교차 실행했다. HFT 입력 준비 포함 계산 p50 131.2→47.6µs, client 완료 p50 400.1→258.7µs, tiny 완료 321.4→233.0µs, 종료 약 2014→2.8ms. 로컬 warm fixture 결과이며 NAS·전체 48열·대규모 분석 처리량으로 해석하지 않는다.
- **지연 판단:** 최초 예정 시각 기준 p99 +24.9%/+24.5% 관측은 보존했다. 같은 바이너리 재실행에서는 -11.9%/-18.1%, 고정 observer·worker의 별도 서버 비교는 -7.6%/-15.0%로 악화가 재현되지 않았다. sleep 안의 수백 µs 지연을 확인했지만 방향 반전과 OS 내부 원인은 미확정이다. 지속적 서버 회귀라는 판단과 운영 지연 목표 합격은 보류한다.
- **전체 검증:** 최신 Windows 11 debug/release 각 106개 통과·실패 0, fmt/clippy 통과. core는 변경이 없어 이전 Miri 22개 근거를 유지한다. 106개 중 1개는 부모 종료 시험용 subprocess host이며 수동 benchmark 1개는 정상 test 실행에서 제외한다. 별도 프로세스·지연/오류 표본 보존 시험, 독립 코드 검토·수치 감사를 완료했다.
- **정확성:** 정수 범위·Float64 순서·null·취소 의미와 Arrow 전체 검증을 유지했다. Arrow reader 6개 파일은 기준선과 같다. 직전 성능 개선에서 변경한 core의 native/Miri·할당·성능 근거를 확보했고 이번 진단에서는 변경하지 않았다. 과거 K0/K1 (프로젝트 내부 문서: k0-k1-results-2026-09-16.md)·K2 (프로젝트 내부 문서: k2-results-2026-09-16.md) 결과는 당시 기록으로 보존한다.
- **다음 우선순위:** [저장·snapshot·live의 최소 완성본](next-work.md). 입력→즉시 live→STREAM 기록→FILE 확정→과거 조회→재시작 복구를 연결한다. 엄격한 지연 목표와 Windows 커널 추적은 운영 검증에 남아 있으며 APL/범용 evaluator는 후속 과제다.
- **Git:** 독립 `main`의 초기 커밋은 `e8cdee4`다. 구현은 작업 트리에 있으며 원격은 연결하지 않았다. 이번 구현을 커밋·공개 게시·운영 배포하지 않았다.
- **운영 조건:** Windows 11이 주 타깃이며 Windows 10 실기 검증은 남아 있다. 입력이 RAM보다 큰 조건을 최종 지원한다. live 최신값은 영속화 전에 반드시 제공해야 하지만 현재 read-only prototype에 ingest/live/durable ACK가 구현된 것은 아니다.

## 직접 실행

이 디렉터리에서:

```powershell
cargo build --release --locked -p sicadb-worker -p sicadb-cli
cargo run --release --locked -p sicadb-cli -- k2-demo
cargo run --release --locked -p sicadb-cli -- demo
cargo run --release --locked -p sicadb-cli -- bench
cargo run --release --locked -p sicadb-cli -- perf-sweep
cargo run --release --locked -p sicadb-cli -- perf-wait
cargo run --release --locked -p sicadb-cli -- pacing-probe
cargo run --release --locked -p sicadb-cli -- pacing-sleep
```

결과는 JSON으로 출력한다. `--output results/my-demo.json`처럼 아직 없는 파일 경로를 지정할 수도 있다. 상위 디렉터리는 먼저 만들어야 한다. `k2-demo`는 실제 자식 프로세스와 TCP를 사용하고 준비·계산·제어 지연을 구분한다. 최초 관리 파일 복사 비용도 기록한다. 측정은 로컬 warm fixture 기준이며 NAS/cold I/O 시험이 아니다.

## 문서

| 문서 | 읽을 때 |
|---|---|
| 프로세스 간 대기형 락 실측·RAM/HDD·DLL 검토 (프로젝트 내부 문서: ipc-lock-probe-results-2026-09-17.md) | 동일 Win32 Mutex 비교, 측정 한계, SSD가 필수가 아닌 적재 정책과 독립 모듈 배포 검토 |
| 제어 지연 진단·재검증 (프로젝트 내부 문서: pacing-diagnosis-results-2026-09-16.md) | 최신 측정 판단·새 진단 도구·재현 안 된 회귀·OS 추적 한계 확인 |
| 전체 성능 개선·전후 비교 (프로젝트 내부 문서: performance-sweep-results-2026-09-16.md) | 최초 성능 개선의 구현·수치·관측·검증·재현 근거 확인 |
| 성능 비교 방법 (프로젝트 내부 문서: performance-sweep-measurement.md) / 수치 독립 감사 (프로젝트 내부 문서: performance-numeric-independent-review.md) | 표본·관측 구간·잡음·할당 수치 해석 확인 |
| K2 구현·검증 결과 (프로젝트 내부 문서: k2-results-2026-09-16.md) | 성능 개선 이전의 프로세스·TCP 구현 기록 확인 |
| 첫 구현·검증 결과 (프로젝트 내부 문서: k0-k1-results-2026-09-16.md) | 실제 코드, 정답, 성능, 실행 명령과 한계 확인 |
| [다음 작업](next-work.md) | 공유 메모리 기반 live·저장·복구의 다음 작업과 gate 확인 |
| [공유 메모리 런타임 계약](shared-memory-runtime-contract.md) | segment/ring/pool, publish·pin·회수, API·DLL·저장 경계의 필수 계약 확인 |
| [공유 메모리 검증 계획](shared-memory-validation-plan.md) | 독립 정답, 다중 프로세스, failure, 메모리·성능 gate와 미검증 범위 확인 |
| K2 내부 API (프로젝트 내부 문서: k2-api.md) | TCP 메시지·worker·service 경계 확인 |
| worker 독립 검토 (프로젝트 내부 문서: k2-worker-independent-review-2026-09-16.md) / service·IPC 독립 검토 (프로젝트 내부 문서: k2-service-independent-review-2026-09-16.md) | native 수명·상태 전이·framing 검토와 남은 한계 확인 |
| [현재 결정과 작업 방식](design-decisions.md) | 합의·미결 사항·작업 방식 확인 |
| [성능·정확성 계약](performance-and-correctness-contract.md) | 해당 기능의 필수 조건 확인 |
| [전체 실행 계획](implementation-plan.md) | 후속 단계와 통합 순서 확인 |
| core 함수 계약 (프로젝트 내부 문서: CONTRACT.md) | 숫자·소유권·backend·오류/취소·예산 의미 확인 |
| Arrow reader 안전 경계 (프로젝트 내부 문서: README.md) | 실제 지원 포맷, 불변 파일·mmap 전제 확인 |
| 실물 fixture 근거 (프로젝트 내부 문서: fixture-provenance.md) | 출처·writer·hash·정답 연결 확인 |
| 독립 구현 리뷰 (프로젝트 내부 문서: k0-k1-independent-review-2026-09-16.md) | 발견·수정·재검증 확인 |
| Claude 명세 리뷰 원문 (프로젝트 내부 문서: performance-and-correctness-contract-review-2026-09-16.md) | 원래 리뷰 근거 확인 |
| Full Trading 연동 경계 (프로젝트 내부 문서: sicadb.md) | 다른 컴포넌트와의 책임·전달 경계 확인 |

## 저장소와 작업 규칙

`sicadb/`는 FT_ROOT와 별도 Git 저장소이며 부모 `.gitignore`에서 제외한다. 서브모듈이 아니다. 내부 구현·설계·리뷰·검증 결과는 여기에 두고 FT_ROOT의 `docs/`에는 다른 컴포넌트가 알아야 하는 내용만 둔다.

작업 전 AGENTS.md (프로젝트 내부 문서: AGENTS.md)를 읽고 README와 현재 작업 문서를 갱신한다. 문서는 UTF-8로 읽고 쓰며 한국어에 한자를 섞지 않는다. Cargo.lock/toolchain과 작은 실제 fixture는 재현 입력이고 `target/`, `results/`, 운영 데이터는 Git 제외 대상이다. 공개 게시·원격 게시·운영 전환은 별도 지시 범위다.
