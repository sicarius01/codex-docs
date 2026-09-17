# 다음 작업 — 저장·snapshot·live의 최소 완성본

2026-09-17. 현재 구현의 성능 개선과 제어 지연 진단 (프로젝트 내부 문서: pacing-diagnosis-results-2026-09-16.md)을 마쳤습니다. 다음 기능 작업은 입력부터 복구까지 하나의 데이터 흐름을 완성하는 것입니다. 아래 저장 기능이 이미 구현됐다는 뜻은 아닙니다.

## 현재 확보한 기반

- 실제 TCP 비동기 접수·조회·취소·Wait, bounded worker pool과 이벤트 기반 완료 통지.
- 불변 FILE/mmap, 전체 batch 검증을 유지하는 제한된 준비 캐시, 큰 scan의 캐시 우회.
- 정확한 정수·순서 Float64 계산, 조건 결합·bitmap 최적화, Reference/Compiler 비교 경로.
- 실제 취소·강제 종료·부모 사망·재시작·자원 회수와 연결/queue/메모리 상한 시험.
- Windows 11 debug/release 각 106개 통과, fmt/clippy, 독립 코드 검토·수치 감사. 변경 없는 core의 이전 Miri 22개 근거 유지.
- 별도 observer/host와 sleep 대조군. 최초 예정 시각 기준 약 25% 회귀는 기존 바이너리 재실행에서 재현되지 않았습니다. 방향 반전과 OS 내부 원인은 아직 미확정입니다.
- 변경 전후 소스·binary·raw 보존. 초기 커밋 이후 구현 커밋·원격/공개 게시·운영 배포는 하지 않았습니다.

## 1. 바로 다음: shared memory 입력부터 재시작 복구까지

현재 순서는 bounded pool/layout→두 process direct read/push→recording STREAM·seal→DLL 재게시와 orders 통합이다. native batch 크기는 packet 80-row fixture와 분리해 sweep한다.

목표 흐름은 다음과 같습니다. 같은 호스트 데이터 경로는 shared memory를 기본으로 하고 TCP는 원격·제어·호환 경로로 둡니다.

**데이터 입력 → 공통 row sequence segment 완성 → 즉시 live direct/push 조회 → STREAM 기록 → 적정 크기/시간 단위 FILE 확정 → snapshot을 지정한 과거 조회 → 재시작 복구**

임시 DatasetStore를 영속 catalog로 취급하지 않습니다. 다음 저장 작업은 작은 단위로 분리합니다.

### 첫 구현 단위

1. 하나의 dataset·schema에서 입력 batch의 ID/순서, 공통 row sequence, segment offset/generation, live 가시성, 기록 완료, FILE 확정, snapshot 게시 상태를 정합니다. column별 독립 회전은 허용하지 않습니다.
2. live 최신값은 디스크 저장 완료 전에 반드시 공개합니다. 기록 예산과 실시간 계산 예산을 분리하고 durable ACK는 별도로 제공합니다.
3. STREAM 기록과 FILE seal을 구현합니다. 날짜 변경만을 유일한 seal 조건으로 고정하지 않습니다. 파일 크기·batch 수·시간 상한과 동시 query의 수명을 같이 정의합니다.
4. 확정된 파일·세대·범위를 manifest에 기록하고 reader가 지정한 snapshot의 파일 집합을 고정합니다. reader가 참조하는 FILE은 불변으로 유지하며 마지막 owner 이후 회수합니다.
5. pin/reclaim race, ring wrap, old-generation resize, reader pause/kill/resume/destruction과 기록 도중 종료, FILE 생성 도중 종료, manifest 게시 도중 종료, ACK 직전/직후 종료를 주입해 재시작 결과와 중복 입력 처리를 검증합니다.

### 이 단계에서 직접 볼 결과

- 한 실행에서 입력 전송·live 조회·기록 ACK·FILE 전환·과거 조회가 연결됩니다.
- 실행 중 프로세스를 강제 종료한 뒤 재시작해 어떤 데이터가 복구됐는지 독립 정답과 비교합니다.
- 아직 저장되지 않은 live 결과에는 그 상태를 표시합니다. live 가시성을 저장 완료로 오해하게 만들지 않습니다.
- 디스크 지연·기록 queue 포화·느린 reader에서도 자원 상한과 오류 의미를 유지합니다.

동시성·catalog는 자체 상태 모델로 설계합니다. SQLite를 필수 전제로 다시 넣지 않습니다. live 배열에는 불변 FILE의 검증 캐시를 그대로 적용하지 않습니다. OMS는 다중 writer와 lock 설계를 유지합니다. 다른 세션의 source 구현과 연동할 외부 계약 변경은 FT_ROOT 규칙에 따라 별도 범위로 다룹니다.

## 2. 실제 규모의 I/O와 운영 검증

- Windows 10 실기와 241의 NUMA·processor group·ISA·메모리 채널·backing device를 확인합니다. SSD는 필수 조건으로 고정하지 않습니다.
- 1,500 instruments 수개월, RAM보다 큰 입력, warm/cold와 순차/선택 scan을 분리합니다. 작은 18MB fixture를 대규모 분석 근거로 사용하지 않습니다.
- NAS는 일괄 준비·로컬 불변 FILE 재사용을 검증합니다. 소파일 반복 SMB stat/read나 전체 RAM 상주를 전제로 하지 않습니다.
- 여러 dataset/column 경쟁, 최대 worker/connection의 깨움 경합, 포화 처리량·장기 메모리 안정성을 별도 시험합니다.
- 2,000 instruments·32 GB feeding과 1초 burst에서 baseline engine allocation, ring/staging/private/shared resident, page fault, hot OS allocation, 호출별 copy, queue high-water를 분리합니다. TCP/SHM은 동일 의미의 packet 1/20/80과 여러 packet 합친 batch, 큰 native batch를 같은 조건으로 비교하고 수신자 1/2/8과 busy/sleep 대기는 분리해 p50/p95/p99/sample max를 측정합니다.
- 독립 정답으로 missing·duplicate·order를 0으로 확인하고 Loom 가능 모델, Miri 가능 부분, 실제 Windows multiprocess를 각각 실행합니다. 도구 통과만으로 전체 soundness를 선언하지 않습니다. 세부 failure matrix는 [공유 메모리 검증 계획](shared-memory-validation-plan.md)을 따릅니다.
- 엄격한 실거래 지연 목표는 별도 합격 조건입니다. API 호출 이후 처리 시간과 예정 시각 기준 전체 지연을 모두 보고하며 sleep overshoot를 분리합니다.
- Windows 내부 대기 원인은 커널 추적 권한을 갖춘 환경에서 ReadyThread/CSwitch로 이어서 확인합니다. 이번 개발 PC에서는 실제 수집이 접근 거부됐습니다. 이 미확정 항목을 해결된 것으로 표시하지 않되 저장 기능 구현과 구분합니다.

## 3. 함수·언어 확장

group/rolling/as-of·sort/join은 데이터·메모리·취소 의미를 먼저 정하고 독립 정답과 성능을 같이 검증합니다. APL 프로필과 실제 Luna/CoT off 작성성 평가는 별도 과제로 이어갑니다. 기존 함수를 조합하는 범용 evaluator가 구현된 것으로 표시하지 않습니다.

상세 내용은 sicadb 내부에 둡니다. 운영 데이터·다른 컴포넌트·공개 문서 저장소는 이번 작업의 부수 효과로 바꾸지 않습니다.
