# sicadb 공유 메모리 검증 계획

작성·갱신: 2026-09-17. 문서 단계이며 검증은 미실행이다. 모든 결과는 fixture hash/seed, code/toolchain, OS/CPU 설정, 실행 명령, raw latency/trace, oracle 비교와 fault 발동 증거를 함께 보존한다.

## 1. 기준선과 정확성

independent oracle는 같은 serializer·ring·allocator를 재사용하지 않는다. 값·null·dictionary·event/available time까지 exact 대조한다. Loom/Miri 도구 통과도 전체 soundness 보장이 아니다.

구현 전/초기에 workload와 baseline을 고정하고 구현 gate 전에 성능 합격선을 확정한다. packet fixture의 80-row는 최대치일 뿐이다. packet 1/20/80, 여러 packet을 합친 batch, 큰 native batch를 별도 sweep하며 부족한 데이터는 80행을 채우려고 기다리지 않는다. independent oracle 기준 missing=0, duplicate=0, order error=0은 즉시 gate다.

## 2. 동시성·수명 case

| ID | 입력·fault | 관측 | 합격 조건 | 상태 |
|---|---|---|---|---|
| SM-01 | packet 1/20/80, 합친/큰 batch | seq·column·validity·APBT | zero 조건, 대기 없음 | 미실행 |
| SM-02 | pause 중 pin/reclaim | stable metadata·linearization | 회수 공간 접근 0 | 미실행 |
| SM-03 | ring [1000,1080), N=1024 | spans·offset | [1000,1024)+[0,56) 순서 | 미실행 |
| SM-04 | resize·old reader | generation·old refs | 모든 old ref 종료 전 회수 0 | 미실행 |
| SM-05 | slow reader 1/2/8·pool 고갈 | pin cap·high-water | 신규 pin 거부/격리·overwrite 0 | 미실행 |
| SM-06 | callback exception·DLL 지연 | done/ACK·pin life | layout 오염·조기 reclaim 0 | 미실행 |
| SM-07 | snapshot cut+tail reconnect | cut/tail seq | 누락·중복·혼합 세대 0 | 미실행 |
| SM-08 | push notify merge/loss | notification·consumer seq | 알림 횟수와 row 수를 동일시하지 않고 sequence에서 복구 | 미실행 |
| SM-09 | busy/sleep lost wake | wake trace·progress | lost wake 없이 진행 | 미실행 |

Loom 가능한 작은 모델로 SM-02~04, Miri 가능한 ownership/alignment 부분을 분리한다. 실제 Windows multiprocess에서 pause/kill/resume/destruction, shutdown/restart generation, reconnect, slow reader, long analysis와 disk read를 실행한다. 같은 머신 2,000 instruments는 2,000 port/process를 뜻하지 않으며 worker별 instrument 분할·종목 내 순서·CPU 연산과 I/O event loop를 분리한다. task/worker 수는 실측으로 정한다.

## 3. 저장·복구 case

ST-04는 durable ACK를 보낸 prefix가 100% 복구되는지 확인한다. ST-05는 fsync 지연 중 live 값이 실제 관측되는지 확인한다. SM case에는 generation 재시작, 범위·offset overflow, invalid descriptor를 포함한다.

| ID | fault | 관측 | 합격 조건 | 상태 |
|---|---|---|---|---|
| ST-01 | STREAM torn tail | length/checksum/prefix | 유효 prefix만 복구, gap 표시 | 미실행 |
| ST-02 | fsync failure | durable boundary/ACK | durable ACK 없음, 재전송 범위 명시 | 미실행 |
| ST-03 | seal/manifest 전후 crash | files/manifest | 완전 FILE만 snapshot 게시 | 미실행 |
| ST-04 | ACK 전후 crash/retry | batch ID/commit | 중복 반영 0 | 미실행 |
| ST-05 | recording queue cap/full disk | live visibility/p99 | live 유지, record 불가·gap 명시 | 미실행 |

## 4. 메모리·성능

32 GB 시험은 engine·table·recording·OS 전체 사용량과 작은 scratch 예산을 분리한다. 장기 bounded memory/high-water와 최소 기간·부하 프로파일은 후보 검증 전에 고정한다. 예정 입력→수신 queue→소비→계산→publish를 구간별 trace하고 누락·느린 표본을 제외하지 않으며 copy bytes와 allocation calls를 기록한다.

2,000 instruments·32 GB feeding과 1초 burst를 사용한다. engine baseline allocation, ring/shared resident, staging/private resident, page fault를 분리한다. 내부 pool block 배정은 허용·필수이며 논리 pool allocation 0을 요구하지 않는다. steady-state hot path의 OS reserve/map/commit과 unplanned heap growth 호출 0을 계측하고 warmup commit/touch·page fault·paging은 별도 기록한다. pool 고갈 시 전체 copy 없이 명시 상태를 낸다.

direct read·push·TCP는 동일 payload 의미와 batch ID/sequence/column/validity를 비교한다. 80-row packet fixture와 native batch sweep을 혼동하지 않는다. 수신자 1/2/8, busy/sleep, warm/cold, same-app callback은 분리 교차 반복한다. 예정 입력 시각부터 측정해 coordinated omission을 피하고 p50/p95/p99/sample max, throughput, CPU, bytes를 기록한다. shared RSS 합산과 실제 물리 상주량을 구분하며 NAS 무단 접근은 하지 않는다.

## 5. 단계·산출물·현재 상태

| 단계 | 범위 | 원시 산출물 | gate | 상태 |
|---|---|---|---|---|
| V0 | shared view/layout | descriptor·alignment log | OS atomic/layout | 미구현 |
| V1 | direct/push | batch·callback·wake trace | 의미·pin·수명 | 미구현 |
| V2 | record/recovery | stream·manifest·crash log | gap·ACK·복구 | 미구현 |
| V3 | DLL/orders | C ABI·derived trace | ABI·재시도·OMS | 미구현 |
| V4 | Win11·32 GB | raw memory/perf/latency | baseline 후 고정한 선 | 미구현 |

Win11과 Win10은 각각 실제 build·실행·mmap·IPC·복구 근거가 있을 때만 해당 지원을 표시한다. 성능 SLO·burst 예산은 baseline 뒤 구현 gate 전에 고정한다. 이 문서는 새 운영 승인 절차를 만들지 않는다.
