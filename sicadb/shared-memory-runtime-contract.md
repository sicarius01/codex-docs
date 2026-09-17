# sicadb 공유 메모리 런타임 계약

작성·갱신: 2026-09-17. 필수 계약과 구현 전 계획을 구분한다. 현재 구현은 SIMD, 불변 FILE/mmap, TCP job과 worker pool까지이며 shared live ingest/subscription, DLL, recording은 미완료다.

## 1. 범위

Windows 11 x64/MSVC를 주 대상으로 하고 Windows 10 feeding·batch는 실제 근거가 있는 범위만 지원으로 표시한다. 같은 호스트 live 시세·feature·1초 table·batch 전달은 shared memory가 기본이고, TCP는 원격 호스트와 선택적 control/compatibility 경로다. 공통 table/query/subscribe API는 direct read와 push를 함께 제공한다. 전체 DB가 하나의 물리 RAM이나 중앙 CPU라는 뜻은 아니다. OMS의 다중 writer와 lock 상태 갱신은 별도 사용자 확정 계약으로 유지한다.

## 2. Layout과 sequence

column별 독립 회전은 금지한다. 공통 row sequence의 columnar batch/segment를 사용하고 모든 column·validity·dictionary·variable 영역을 완성한 뒤 Release publish한다. descriptor는 schema/version, table, sequence, `published_end`(exclusive), segment, offset, length, generation, checksum, event/available time을 포함한다. reader done, active pin, time retention, durable position은 별도 상태다.

고정 용량 ring의 physical slot은 N개이며, 논리 sequence는 그 범위에 갇히지 않는다. N을 바꾸어 modulo를 재해석하지 않는다. [1000,1080), N=1024는 [1000,1024)+[0,56)이다. bounded pool은 고정 page/segment 배정·반환과 metadata/descriptor queue 예산을 포함한다. OS physical page와 sicadb block은 구분한다.

logical sequence는 단조 증가하는 `[begin,endExclusive)`이고 physical slot/offset과 구분한다. pool 초기 reserve/commit/touch를 계측하되 물리 pinned를 보장하지 않는다. logical table block 추가 증설을 기본으로 하여 전체 복사를 피한다.

## 3. 작성·읽기·push

시세·feature·1초 적재만 main process single writer와 partition별 규칙을 사용한다. 파생 worker 결과를 main에 전달한 뒤 publish하는 비용은 별도 측정한다. 첫 live slice는 논리적으로 자라는 append table과 publish 후 불변 batch다. 임의 inplace update/delete, 다중 table ACID, 거래소 전체 total order를 완료했다고 표시하지 않는다.

reader는 안정된 descriptor/control metadata를 관측한 뒤 pin 권리와 retire/reclaim 권리가 경쟁하는 프로토콜을 거치고, pin 성공 후 generation과 범위를 재확인한 뒤 접근한다. 이미 회수된 공간 접근은 금지한다. CAS ref-state·epoch·hazard와 선형화 지점은 모델 검증 뒤 선택한다. `syncReadGuard`는 Drop에서 pin을 해제하고 async DLL은 실제 사용 완료까지 pin한다. push는 payload 복제 없이 sequence 범위와 segment offset/generation을 알리며 소비자 executor가 callback을 실행한다. 함수 포인터 callback은 layout에 저장하지 않는다.

초기화→내용 쓰기→Release publish→Acquire observe 순서를 지킨다. 일반 Rust pointer·Vec·std mutex를 layout에 저장하지 않는다. APBT는 native batch/packet 식별자, 완전한 내용, 내부 순서, 수신·available time, stream/session/gap과 필요한 merge 규칙을 보존한다. 80-row는 기존 packet fixture 최대이며 모든 native batch에 상한을 강제하지 않는다. UDP raw bytes는 필수가 아니며 wire 동일성이 요구될 때만 가역 encoding을 사용한다.

완성 batch와 null bitmap은 공유 byte를 reader가 읽는 동안 수정하지 않는다. Windows interprocess atomic, alignment, ABI 보장은 실제 대상 OS에서 확인한다. worker별 instrument 분할·종목 내 순서·CPU 계산과 I/O event loop를 분리하며 process/port/task 수는 실측으로 정한다.

## 4. 수명·회수·resize

time retention 만료와 모든 reader reference·active pin 종료를 모두 만족한 block만 재사용한다. reader별 pin 한도, 신규 pin 거부, 느린 reader 격리, pool 고갈 상태를 명시한다. 정상 bounded 예산에서는 live writer가 old reader를 기다리지 않지만 pool 고갈 시 종료 확인 전 덮어쓰지 않고 오류·backpressure를 낸다. timeout/heartbeat만으로 회수하지 않는다.

최근 1초 retention과 burst, reader lag, recording backlog 예산은 서로 구분한다. live 값은 fsync·FILE seal·manifest·durable ACK 이전에도 항상 제공한다.

snapshot→tail은 cut/tail cursor를 포함한 불변 descriptor 식별자·offset을 한 번 교체해 게시한다. 새 generation reader 전환과 old reader 종료 뒤 old 공간을 반환하며 필드를 다른 세대로 섞지 않는다.

## 5. 저장·DLL

live와 recording은 bounded queue·worker·memory를 분리한다. recording이 live global cursor를 무한 pin하지 않도록 전용 staging과 bounded copy를 허용한다. 유한 RAM·무손실·무한 disk stall을 동시에 보장하지 않는다. STREAM torn tail, fsync failure, seal/manifest/ACK 전후 crash, recording 용량 소진에서 gap·불가·재전송 범위와 durable ACK를 정확히 표시한다. recording은 별도 writer thread를 기본으로 하고 추후 process writer를 지원한다. FILE seal은 크기·시간 기준이다. NAS 영속 저장과 NAS→shared RAM 분석, 선택 HDD cache를 지원 대상으로 검토하며 SSD 필수는 아니다. 수정 중 원격 파일 동시 mapping은 금지한다.

버전별 C ABI DLL은 core 재빌드 없이 독립 빌드하고 struct size, opaque handle, buffer free, thread safety, multi-call, core/plugin/schema 호환과 exception unwind 금지를 검사한다. 통제된 관련 process 재시작을 기본으로 하며 hot reload는 필수가 아니다. raw→matching feature→derived table→subscription/1초 결과와 `forwardtestorders`를 실주문 없는 fixture부터 검증하며 재시도 중복·OMS 동시성을 보존한다.

## 6. 최소 흐름

ingest→row sequence segment→live direct/push→recording STREAM→FILE seal/manifest→snapshot query→restart recovery가 최소 수직 흐름이다. 문서화는 구현·검증·운영 완료를 뜻하지 않는다.

세부 검증은 [공유 메모리 검증 계획](shared-memory-validation-plan.md)을 따른다.
