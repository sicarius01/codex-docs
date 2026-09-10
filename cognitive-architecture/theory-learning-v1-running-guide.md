# V1 실행 및 검증 안내

2026-09-10. 요구사항 기준은 [고정 V1 스펙](https://github.com/sicarius01/codex-docs/blob/22fdb8ed09e4a4d8f8cf277c106066f427fa8e36/cognitive-architecture/theory-learning-v1-spec.md)입니다. 구현 기준 커밋은 `ce77a25`이며, 구현 설계는 구현 저장소의 `docs/design/theory-learning-v1-design.md`를 참고합니다.

## 실행

Windows PowerShell, Python 3.13+, uv, npm을 사용합니다. 의존성은 `uv.lock`과 `package-lock.json`으로 고정했습니다. PI는 `@earendil-works/pi-coding-agent@0.85.1`, 프로젝트 Node는 22.22.0입니다. 전역 Node를 변경하지 않습니다.

아래 명령은 **공개 문서 저장소가 아니라 구현 저장소(`context`) 루트**에서 실행합니다. `scripts/start.ps1`, `pyproject.toml`, `package.json`이 있는 폴더인지 확인합니다. PowerShell에서 해당 폴더로 이동한 뒤 최초 설치와 실행을 진행합니다:

```powershell
./scripts/start.ps1 -Install
```

이후 실행은 `./scripts/start.ps1`입니다. GUI는 <http://127.0.0.1:8765>입니다. 설치·빌드를 개별 실행하려면:

```powershell
uv sync --locked --python 3.13
npm ci --no-audit --no-fund
npm run build
uv run theory serve --standalone
```

모델 없이 기록 탐색과 문맥 미리보기만 실행할 때는 `uv run theory serve` 또는 `./scripts/start.ps1 -NoAgent`를 사용합니다. 실제 작업 데이터는 `.local/default/`에 저장됩니다. 프로젝트를 분리하려면 서로 다른 `--data-dir`와 포트를 사용합니다. 같은 data-dir의 동시 실행은 OS 파일 잠금으로 차단합니다. 서버 종료는 실행 터미널에서 Ctrl+C입니다.

## 모델·Langfuse 설정

`.env.example`을 `.env`로 복사하고 로컬에서 값을 입력합니다. 키는 채팅이나 Git에 올리지 않습니다. 시작 스크립트는 `.env`가 있으면 `uv run --env-file .env`로 읽습니다. 직접 실행할 경우에도 해당 옵션을 붙여야 합니다.

- `THEORY_MODEL_PROVIDER`, `THEORY_MODEL_ID`: PI에서 실제 사용할 제공자와 모델 ID.
- 해당 제공자의 PI 인증: PI 인증 저장소 또는 PI가 지원하는 제공자 환경변수. 이 프로젝트가 임의의 모델·계정을 선택하지 않습니다.
- `THEORY_MODELS_PATH`: 선택 사항. PI 형식의 custom `models.json` 파일 경로. 로컬 모델·프록시도 PI가 지원하는 설정을 사용합니다.
- `THEORY_ROLE_MODELS`: 선택 사항. 역할별 `provider`·`model` JSON 설정이며 기본은 공통 모델입니다.
- `THEORY_PROMPT_VERSIONS`: 선택 사항. 역할별 Langfuse prompt의 `name`·명시적 정수 `version`을 지정합니다. 원문과 버전을 harness snapshot에 고정하고 오프라인에서는 이전에 확인한 동일 버전만 재사용합니다.
- `LANGFUSE_BASE_URL`, `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`: 사용할 Langfuse 프로젝트.

```powershell
uv run --env-file .env theory doctor
uv run --env-file .env theory serve --standalone
```

`doctor`는 설정 존재 여부만 표시하며 비밀 값은 출력하지 않습니다. Langfuse 서버 설치는 기존 스펙의 Windows Docker Desktop/WSL2 실행 안내를 따릅니다. **2026-09-10 구현 검증 환경에서는 Docker CLI와 Langfuse 키가 확인되지 않았으므로 서버 설치·실제 API 왕복 검증 완료를 주장하지 않습니다.** Python SDK 4.15.2의 실제 메서드 시그니처를 확인해 adapter를 구현했습니다.

Langfuse 미설정·전송 지연·조회 불일치는 학습 완료로 처리하지 않습니다. Worker 기록은 로컬에 먼저 저장되고 outbox로 전송됩니다. 학습은 서버 조회로 원문 해시가 확인된 snapshot만 사용합니다. `flush()` 반환은 조회 성공을 뜻하지 않습니다. 조회 실패는 `waiting_for_rollout`으로 남으며 설정·서버 상태를 복구한 뒤 재개할 수 있습니다. 통신 재시도로 커널 실험을 다시 실행하지 않습니다.

## 한 번의 학습

모델과 Langfuse를 설정한 뒤 standalone 서버를 실행해 둡니다. GUI 개요에서 Worker 작업을 입력하거나 **별도 PowerShell 터미널을 구현 저장소 루트에서 열어** CLI를 사용합니다. 아래 명령은 한 줄씩 실행하며, `WORKER_RUN_ID`와 `LEARNING_RUN_ID`는 실행 목록에서 확인한 실제 ID로 바꿉니다. Worker 작업이 완료된 뒤 학습을 시작합니다.

```powershell
uv run theory worker '주어진 데이터를 Python으로 확인하고 결과를 설명해 주세요.'
uv run theory inspect runs
uv run theory learn "WORKER_RUN_ID"
uv run theory inspect runs "LEARNING_RUN_ID"
```

진행 중인 학습을 취소하거나 중단된 학습을 재개할 때만 다음 명령을 사용합니다:

```powershell
uv run theory cancel "LEARNING_RUN_ID"
uv run theory resume "LEARNING_RUN_ID"
```

Extractor → Reviewer → 실제 커널 조사 → Reviewer 결과 해석 및 필요 시 후속 조사 → Integrator → 검증·원자적 KB 반영 순서입니다. 후보가 없으면 이후 단계를 명시적으로 건너뜁니다. Reviewer 조사는 최초 두 개, 각각 후속 한 개 이내이고 역할별 도구 예산도 함께 적용됩니다. Integrator는 변경안을 제안하며 직접 저장하거나 실험하지 않습니다.

채택·보류·기각은 잠정적 판단입니다. 시스템은 증거 타입과 실제 참조 위치, 중복 root, 실행 여부 및 KB 버전을 검사합니다. **모델이 내린 의미론적 판단 자체가 옳다는 보장은 없습니다.** 보류 이론은 조건·한계와 함께 표시하고 기각 이론은 문맥에서 제외합니다.

GUI 좌측 8개 탭에서 실행 기록, 역할별 실제 모델 입력·응답, 도구 결과, 이론·관계·수정 이력, 실험, KB snapshot, 문맥 미리보기, 운영 상태를 확인합니다. 실제 전송 입력과 미리보기는 표시를 구분합니다. 이론의 근거 버튼으로 원래 증거 참조를 확인할 수 있습니다. 큰 원문은 JSON을 펼쳐 읽으며 실행 산출물의 HTML을 자동 실행하지 않습니다.

## 팀 PI 연결

팀의 PI에서 이 저장소의 `pi/extension.ts`를 명시적인 extension으로 로드합니다. 예를 들어 PI CLI의 `--extension <absolute-path>`를 사용합니다. 모델·Langfuse 환경변수는 PI를 시작하는 셸에 설정합니다. extension은 Python을 `uv run theory serve --integrated`로 시작하므로 **standalone 서비스와 동시에 같은 data-dir에서 실행하면 안 됩니다.**

팀 Worker의 실제 세션을 바꾸거나 학습 역할로 재사용하지 않습니다. 별도 RoleHost가 세 역할을 생성합니다. `/theory-learn`은 직전 Worker 기록을 학습 대상으로 전달합니다. Worker 시작 때 KB 문맥을 추가하고 입력·출력·도구 이벤트를 기록합니다. 학습 역할은 AGENTS.md·자동 검색 extension·skill을 로드하지 않는 독립 resource loader를 사용합니다.

학습 역할은 단독 캡처 hook을 사용하여 provider 입력을 `exact`로 기록합니다. 팀 Worker에는 다른 extension이 뒤에서 payload를 변경할 수 있으므로 `pre_final`로 표시합니다. 최종 송신점까지 캡처를 보장하려면 팀 하네스의 hook 순서·provider wrapper와 함께 통합 확인해야 합니다. 팀 `/evolve`, 거래 과제, LLM judge는 구현하지 않았습니다.

## 중단·복구와 데이터

SQLite는 WAL, `busy_timeout=5000`, `synchronous=FULL`, 단일 writer 스레드를 사용합니다. 시작 시 실행 중이던 작업은 중단 상태로 복구합니다. 호출 의도를 먼저 커밋하고 실제 호출 직전 `running`을 기록합니다. 실행 중 죽은 도구는 `outcome_unknown`으로 남아 자동 재실행을 차단합니다. 원래 프로세스나 외부 산출물로 결과를 확인하기 전에는 성공으로 바꾸지 않습니다. 새 실행을 명시적으로 시작하는 것과 기존 호출의 자동 재전송을 구분합니다.

재개는 완료된 단계의 snapshot·결과를 사용합니다. Python 변수나 모델 내부 상태를 되살렸다고 가정하지 않습니다. 취소 뒤 늦은 도구 응답도 저장하지만 취소된 모델로 결과 본문을 전달하지 않습니다. 이미 커밋한 KB는 취소로 되돌리지 않습니다.

Jupyter는 같은 Windows 사용자 권한의 프로세스이며 OS 보안 경계가 아닙니다. 코드가 읽을 수 있는 파일·네트워크 권한은 사용자 권한에 따릅니다. 학습 데이터와 평가 정답은 프롬프트에 무단 포함하지 않지만 악의적인 커널 코드에 대한 격리 보장은 하지 않습니다.

## 검증 범위

```powershell
uv run pytest -q
uv run ruff check src tests
npm test
npm run build
```

자동 테스트는 실제 SQLite, 실제 Windows Jupyter 커널, 실제 PI SDK와 로컬 HTTP 응답 fixture를 사용합니다. Langfuse 지연·페이지네이션·중복 검증은 명시적 test double입니다. 테스트용 응답을 모델의 추론 성능이나 Langfuse 실서버 검증으로 집계하지 않습니다. 테스트 데이터는 임시 디렉토리에 생성됩니다.

2026-09-10 구현 검증 환경에서는 브라우저 제어 표면이 연결되지 않아 GUI 시각·클릭 검증은 미완료입니다. TypeScript 검사, production 빌드, HTTP API 검증을 수행했습니다. 실제 모델·Langfuse 연결, 팀 PI 세션 통합, 216회 본평가까지 통과한 출시 완료 상태와 구분합니다. 평가 준비는 구현 저장소의 `docs/implementation/evaluation-v1.md`를 따릅니다.
