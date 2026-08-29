# Prime Agent 분석 정리 (한국어)

이 문서는 Prime Agent 저장소를 살펴보고 정리한 분석 노트입니다.
개념 이해 → 설치/사용법 → 수익화 및 PHP 연동 검토 순서로 구성했습니다.

## 저장소 주소

- 원본(Upstream): https://github.com/PrimeIntellect-ai/prime-agent
- 현재 저장소(Fork): https://github.com/bmshin94/prime-agent
- 논문: https://arxiv.org/abs/2608.23552 — *Prime Agent: A Self-Improving RLM Harness*
- RLM 소개 글: https://www.primeintellect.ai/blog/rlm
- 기반이 된 프로젝트 `pi`: https://github.com/earendil-works/pi
- 관련 프로젝트: https://github.com/PrimeIntellect-ai/verifiers · https://github.com/PrimeIntellect-ai/prime-rl

---

## 1. 이게 뭔가요?

**터미널에서 동작하는 AI 코딩 · 리서치 에이전트**입니다. 프로젝트 폴더에서
`prime-agent`를 실행하면 AI가 그 폴더 안에서 파일을 읽고 수정하고 명령어를
실행합니다. Claude Code, Codex CLI와 같은 부류입니다.

| 항목 | 내용 |
|---|---|
| 만든 곳 | Prime Intellect |
| 라이선스 | MIT (상업적 이용·수정·재배포 가능) |
| 규모 | TypeScript 소스 약 944개 파일 / 약 35만 줄 |
| 버전 | 0.8.1 |
| 요구사항 | macOS 또는 Linux, Node.js 22.8.0+ |

### 핵심 차별점 4가지

**① 모델이 쓰는 도구가 파이썬 REPL 하나뿐 (RLM)**

보통 에이전트는 `파일읽기`, `파일쓰기`, `검색`, `배쉬`처럼 도구를 여러 개
제공합니다. Prime Agent는 `ipython` 하나만 주고 나머지를 전부 코드로 처리합니다.

```python
from pathlib import Path
config_files = list(Path(".").rglob("*.toml"))
large_files = [p for p in config_files if p.stat().st_size > 10_000]

result = await bash("npm run check")
print(result.output)
```

가장 큰 장점은 **파이썬 커널 상태가 컨텍스트 압축(compaction) 후에도
살아남는다**는 점입니다. 대화 기록은 요약되어도 변수·함수·파싱 결과는
커널에 그대로 남아 이어서 사용할 수 있습니다.

**② 서브에이전트가 함수 호출처럼 동작**

```python
api_review = await rlm("공개 API를 리뷰해줘", name="api-reviewer")
test_review = await rlm("테스트 커버리지를 확인해줘", name="test-reviewer")
```

`rlm()`은 **자식의 답을 반환하지 않습니다.** 작업 접수 시점에 핸들만 즉시
돌려주고, 자식은 나중에 `agent_message`로 부모에게 결과를 보고하거나 파일로
남깁니다. 덕분에 부모는 기다리지 않고 계속 다른 일을 할 수 있습니다.

**③ 스스로 개선 (Continual Harness)**

`/refine`을 실행하면 지금까지의 대화 흐름을 스스로 복기해서, 반복된 실패나
재사용할 만한 패턴을 보조 프롬프트·메모리·스킬·서브에이전트 명세로 저장합니다.
변경 이력 스냅샷이 남아 롤백도 가능하며, 변경 불가능한 기본 시스템 프롬프트는
절대 덮어쓰지 않습니다.

**④ 터미널을 닫아도 계속 실행**

세션은 데몬 워커 프로세스가 소유합니다. TUI를 닫으면 클라이언트만 분리될 뿐
작업은 계속되고, 나중에 `prime-agent attach`로 다시 붙을 수 있습니다.

---

## 2. 폴더 구조

| 경로 | 역할 |
|---|---|
| `packages/coding-agent/` | 본체. CLI, 슬래시 명령어, 세션 관리, 문서 |
| `packages/agent/` | 에이전트 코어 엔진 (대화 루프, 상태 관리) |
| `packages/ai/` | LLM 제공자 통합 레이어 (도구 호출 지원 모델만 포함) |
| `packages/tui/` | 터미널 UI 프레임워크 (차분 렌더링, 깜빡임 없음) |
| `prime-agent-runtime/` | 파이썬 런타임 `rlm` 패키지 (REPL, bash, 스킬, MCP) |
| `packages/coding-agent/skills/` | 내장 스킬 13종 |
| `packages/coding-agent/docs/` | 공식 문서 (mermaid 다이어그램 포함) |
| `install.sh` | 설치 스크립트 (SHA-256 체크섬 검증 포함) |
| `AGENTS.md` | 개발 규칙 (AI에게 주는 프로젝트 컨벤션) |

### 내장 스킬 13종

`agent-message` · `agent-observe` · `attach-image` · `compact` · `edit` ·
`goal` · `linear` · `notion` · `prime-intellect` · `refine` ·
`rlm-heartbeat` · `skill-creator` · `websearch`

---

## 3. 설치

### 기본 설치

```bash
curl -fsSL https://app.primeintellect.ai/prime-agent/install.sh | sh
```

설치 스크립트는 버전 확인 → 다운로드 → SHA-256 검증 → `prime-agent` 명령
등록 → 파이썬 런타임 준비까지 처리합니다.

베타 채널:

```bash
curl -fsSL https://app.primeintellect.ai/prime-agent/install.sh | sh -s -- beta
```

채널은 `stable`(기본), `beta`, 또는 특정 버전 번호를 지정할 수 있습니다.

### 소스에서 실행

```bash
git clone https://github.com/PrimeIntellect-ai/prime-agent
cd prime-agent
npm ci
./prime-agent.sh
```

### 설치 확인

```bash
prime-agent --version
prime-agent doctor          # 상태 진단
prime-agent doctor --fix    # 자동 복구
```

---

## 4. 로그인

### 구독 계정

에이전트 안에서 `/login` 실행 후 선택합니다.

- Claude Pro/Max
- ChatGPT Plus/Pro (Codex)
- GitHub Copilot

> **주의:** Claude Pro/Max로 연결할 경우, 서드파티 하니스 사용분은 플랜 한도가
> 아니라 **추가 사용량(extra usage)에서 토큰당 과금**됩니다.
> https://claude.ai/settings/usage 에서 확인하세요.

### API 키

```bash
export ANTHROPIC_API_KEY=sk-ant-...
prime-agent
```

| 제공자 | 환경변수 |
|---|---|
| Anthropic | `ANTHROPIC_API_KEY` |
| OpenAI | `OPENAI_API_KEY` |
| Google Gemini | `GEMINI_API_KEY` |
| Prime Inference | `PRIME_API_KEY` |
| DeepSeek | `DEEPSEEK_API_KEY` |
| Mistral / Groq / Cerebras | `MISTRAL_API_KEY` / `GROQ_API_KEY` / `CEREBRAS_API_KEY` |
| Azure OpenAI Responses | `AZURE_OPENAI_API_KEY` |
| Cloudflare AI Gateway | `CLOUDFLARE_API_KEY` (+ 계정/게이트웨이 ID) |

자격증명은 `~/.prime/agent/auth.json`에 저장되고 만료 시 자동 갱신됩니다.

---

## 5. 사용법

### 에디터 조작

| 기능 | 방법 |
|---|---|
| 파일 참조 | `@` 입력 후 파일 검색 |
| 경로 자동완성 | `Tab` |
| 여러 줄 입력 | `Shift+Enter` (윈도우 터미널은 `Ctrl+Enter`) |
| 이미지 첨부 | `Ctrl+V` 붙여넣기 또는 드래그 |
| 쉘 명령 (결과를 모델에 전달) | `!command` |
| 쉘 명령 (모델에 전달 안 함) | `!!command` |
| 외부 에디터 | `Ctrl+G` (`$VISUAL` / `$EDITOR`) |
| 단축키 목록 | `/hotkeys` |

### 메시지 큐 (작업 중 개입)

- `Enter` — 현재 턴이 끝난 뒤 전달 (steering, 방향 수정)
- `Alt+Enter` — 모든 작업이 끝난 뒤 전달 (follow-up)
- `Ctrl+C` — 현재 작업 중단 / 힌트 표시 중 다시 누르면 종료
- `Esc` — 입력창만 비움 (작업은 유지)
- `Alt+Up` / `Alt+Down` — 대기 중인 메시지 탐색

### 주요 슬래시 명령어

```text
/login /logout        자격증명 관리
/model /effort        모델·사고 수준 변경
/settings             설정 화면
/usage /context       토큰·비용·컨텍스트 확인
/new /resume /name    세션 관리
/compact [지시]        컨텍스트 압축
/tree /fork /clone    세션 분기·이동
/export [파일]         HTML로 내보내기
/share                비공개 GitHub gist로 공유
/btw /side <질문>      본 세션에 남기지 않고 곁가지 질문
/reload               스킬·확장·설정 다시 읽기
```

Prime Agent 고유 명령어:

```text
/refine [지시]                         하니스 자기개선 실행·롤백
/goal <목표>                           지속 목표 설정
/heartbeat every 10m <지시>            주기적 반복 프롬프트
/heartbeats                            하트비트 목록 관리
/autonomous on | status | off          자율 모드
```

### CLI 명령어

```bash
# 세션
prime-agent -c                   # 최근 세션 계속
prime-agent -r [경로|id]          # 세션 선택·재개
prime-agent --fork <경로|id>      # 세션 분기
prime-agent --no-session         # 저장하지 않는 일회성 세션

# 비대화형
prime-agent -p "이 코드베이스 요약해줘"
cat README.md | prime-agent -p "요약해줘"
prime-agent -p @screenshot.png "이 이미지 설명해줘"
prime-agent @code.ts @test.ts "리뷰해줘"

# 백그라운드 에이전트
prime-agent agents               # 에이전트 뷰
prime-agent list [--all]
prime-agent attach <에이전트>
prime-agent send <에이전트> "메시지"
prime-agent rename <에이전트> <새이름>
prime-agent stop <에이전트>
prime-agent status
prime-agent shutdown [--force]

# 예약 실행
prime-agent schedule add worker "in 30m" -- "벤치마크 결과 확인"
prime-agent schedule add worker "0 9 * * 1-5" -- "열린 작업 리뷰"
prime-agent schedule list --all
prime-agent schedule cancel <job-id>

# 모델 · 패키지
prime-agent model list [검색어]
prime-agent package install <소스> [--local]
prime-agent package list
prime-agent update [--force]
```

### 주요 옵션

| 옵션 | 설명 |
|---|---|
| `--provider <이름>` | 제공자 지정 |
| `--model <패턴>` | 모델 지정 (`provider/id` 형식 지원) |
| `--thinking <수준>` | `off`~`max` |
| `--cwd <경로>` | 작업 디렉터리 지정 |
| `--offline` | 시작 시 네트워크 작업 비활성화 |
| `--no-context-files` | `AGENTS.md` / `CLAUDE.md` 로딩 끄기 |
| `--skill <경로>` | 스킬 추가 로드 (반복 가능) |
| `-e, --extension <소스>` | 확장 로드 (반복 가능) |
| `--mode json` | 이벤트를 JSON 라인으로 출력 |
| `--mode rpc` | stdin/stdout JSON 프로토콜 |

### 프로젝트 설정 파일

```
~/.prime/agent/
├── auth.json          자격증명 (공유 금지)
├── AGENTS.md          전역 지침
├── SYSTEM.md          시스템 프롬프트 교체
├── APPEND_SYSTEM.md   시스템 프롬프트 추가
├── skills/            전역 스킬
└── sessions/          세션 기록 (JSONL)
```

프로젝트 단위로는 `.prime/agent/` 및 프로젝트 루트의 `AGENTS.md` /
`CLAUDE.md`를 사용합니다. 컨텍스트 파일은 전역 → 상위 디렉터리 →
현재 디렉터리 순으로 로드됩니다.

### 자율 모드

```bash
prime-agent -p \
  --autonomous \
  --autonomous-gate "npm run check" \
  --autonomous-gate-retries 2 \
  --autonomous-max-continuations 3 \
  --autonomous-max-turns 12 \
  --autonomous-max-tokens 80000 \
  --autonomous-timeout-ms 1800000 \
  --thinking high \
  "실패하는 검사를 고치고 검증된 결과를 보고해줘"
```

게이트(gate)는 **통과해야만 작업을 끝낼 수 있는 검증 명령**입니다. 실패하면
출력이 다음 이어하기에 전달되어 에이전트가 수정을 시도합니다.

기본 한도: 이어하기 3회 / 턴 12회 / 토큰 80,000 / 30분 / 게이트 재시도 3회.

> 한도 도달로 종료된 것과 작업 성공은 다릅니다. 문서에도 "한도 도달이 성공을
> 의미하지 않는다"고 명시되어 있으므로 결과는 반드시 직접 확인해야 합니다.

---

## 6. 안전 주의사항

공식 문서의 경고를 요약하면 다음과 같습니다.

1. **보안 샌드박스가 아닙니다.** 워커·커널 프로세스 분리는 생명주기 격리와
   장애 복구를 위한 것이며, 사용자와 동일한 OS 권한으로 실행됩니다.
2. 모델이 생성한 파이썬 코드와 프로젝트 명령이 그대로 실행됩니다.
   **git으로 커밋된 상태**나 별도 클론·워크트리에서 사용하세요.
3. 신뢰할 수 있는 저장소·지침·스킬·확장만 사용하세요. 스킬은 임의 동작을
   지시할 수 있고 실행 코드를 포함할 수 있습니다.
4. 신뢰할 수 없는 코드나 지침은 외부 샌드박스에서 실행하세요.
5. `/usage`로 토큰과 비용을 주기적으로 확인하세요.

---

## 7. 수익화 아이디어 검토

라이선스가 MIT이므로 상업적 이용·수정·재배포·유료 판매가 모두 가능합니다.
조건은 저작권 고지문 포함뿐이며 소스 공개 의무는 없습니다.

### 아이디어 우선순위

**1. PHP 레거시 현대화 에이전트 (가장 유력)**

국내에 PHP 5.x, 구버전 CodeIgniter 등으로 운영되는 시스템이 다수 남아 있으나,
해외 스타트업 중심의 기존 코딩 에이전트 제품들은 이 시장을 겨냥하지 않습니다.
Prime Agent의 특성과 궁합도 좋습니다.

- 마이그레이션은 장기 작업 → 데몬 백그라운드 실행이 유효
- 파일 수천 개 분석 → 파이썬 커널에 상태를 유지해 컨텍스트 초과를 회피
- 파일·모듈 단위 분할 → 서브에이전트 병렬 처리
- "테스트 통과까지" → 자율 모드 게이트 활용

과금: 무료 진단 리포트 후 프로젝트 단위 견적, 또는 유지보수 월 구독.

**2. 웹 기반 에이전트 관제 대시보드**

Prime Agent의 강점은 장시간 실행 에이전트인데 관리 인터페이스가 터미널뿐입니다.
브라우저 대시보드(실행 상태, 토큰·비용 집계, 웹에서 메시지 전송, 스케줄 관리,
팀별 정산, 알림 연동)는 PHP/Laravel로 만들기 적합한 영역입니다.

과금: 팀 시트당 월 구독.

**3. 온프레미스 · 폐쇄망 구축**

금융·공공·의료·대기업은 코드를 외부 클라우드로 전송할 수 없습니다.
MIT 라이선스 + 로컬 실행이라 사내망 배치가 가능하며, 여기에 SSO, 감사 로그,
사내 LLM 연동을 얹어 구축비 + 연간 유지보수로 과금합니다.

**4. 비개발 직군용 업무 자동화**

파이썬 REPL 기반이라 데이터 처리에 강합니다. 쇼핑몰 리포트 자동화, 회계 자료
정리, 마케팅 데이터 수집·분석 등에 적용할 수 있습니다.

**5. 스킬 마켓플레이스**

수요·공급이 동시에 필요해 초기 단독 아이템으로는 부적합하며, 위 사업이
자리잡은 뒤 부가 기능으로 붙이는 편이 낫습니다.

### 리스크

| 리스크 | 대응 |
|---|---|
| LLM 토큰이 곧 원가 | 정액제 대신 종량제 또는 사용량 상한 |
| 원 개발사가 동일 기능 출시 | 국내 시장·PHP 레거시 등 로컬 특화로 차별화 |
| 고객 서버에서 코드 실행에 따른 사고 책임 | 계약 조건, 백업, 격리 환경 필수 |
| 경쟁 심화 | 정면 경쟁 대신 틈새 공략 |
| 원본 업데이트 추적 부담 | 포크 개조가 아니라 외부에서 감싸는 구조 채택 |

---

## 8. PHP 연동 가능성

### 결론: 재구현하지 말고 감쌀 것

Prime Agent는 이미 외부 프로그램이 제어할 수 있는 헤드리스 모드를 제공합니다.

```bash
prime-agent --mode rpc     # stdin/stdout JSON 프로토콜
prime-agent --mode json    # 이벤트를 JSON 라인으로 출력
prime-agent -p "질문"       # 단발 실행
prime-agent send <에이전트> "메시지"
```

따라서 PHP 쪽에서 LLM 로직을 새로 구현할 필요가 없습니다.

### PHP 예제

```php
<?php
// 방법 1: 단발 실행
$result = shell_exec('prime-agent -p ' . escapeshellarg('이 저장소를 요약해줘'));
echo $result;

// 방법 2: RPC 모드로 제어
$proc = proc_open(
    ['prime-agent', '--mode', 'rpc', '--model', 'anthropic/claude-sonnet-4-20250514'],
    [0 => ['pipe', 'r'], 1 => ['pipe', 'w'], 2 => ['pipe', 'w']],
    $pipes,
    '/var/www/target-project'
);

fwrite($pipes[0], json_encode([
    'id'      => 'req-1',
    'type'    => 'prompt',
    'message' => 'PHP 5 문법을 PHP 8로 변환해줘',
]) . "\n");

while (($line = fgets($pipes[1])) !== false) {
    $event = json_decode($line, true);
    if (($event['type'] ?? null) === 'tool_execution_start') {
        echo "작업 중: {$event['toolName']}\n";
    }
}
```

RPC 프로토콜 주의사항 (공식 문서 기준):

- 레코드 구분자는 **LF(`\n`)만** 사용합니다. `\r\n` 입력은 `\r`를 제거해 처리하고,
  유니코드 구분자(`U+2028`, `U+2029`)까지 줄바꿈으로 처리하는 범용 라인 리더는
  프로토콜에 맞지 않습니다.
- 에이전트가 스트리밍 중일 때 프롬프트를 보내려면 `streamingBehavior`를
  `steer` 또는 `followUp`으로 지정해야 하며, 생략하면 오류가 반환됩니다.
- 모든 명령에 선택적 `id` 필드를 넣어 요청·응답을 매칭할 수 있습니다.

### 권장 아키텍처

```
사용자 (브라우저)
      |
[ Laravel (PHP) — 직접 구현할 영역 ]
  · 인증 / 팀 / 권한
  · 대시보드 UI
  · 작업 이력 · 토큰 비용 정산
  · 알림 연동 (슬랙 등)
      |  proc_open / Queue Job
[ Prime Agent — 수정 없이 그대로 사용 ]
  · --mode rpc
      |
   LLM 제공자
```

PHP 스택 제안: Laravel + Horizon(큐), Laravel Reverb 또는 SSE(실시간 로그),
`symfony/process`(프로세스 제어). 상주 프로세스가 필요하면 Swoole 또는 ReactPHP.

이 구조의 장점은 원본이 업데이트되어도 `prime-agent update` 한 번이면 되고,
자체 코드에는 영향이 없다는 점입니다.

### PHP로 처음부터 재구현하지 않는 이유

LLM 호출 자체는 HTTP + JSON이라 PHP로도 가능하지만, 약 35만 줄을 다시
작성해야 하며 PHP가 상대적으로 약한 영역이 핵심에 몰려 있습니다.

- 장시간 살아 있는 프로세스 (PHP는 요청-응답 모델이 기본)
- 터미널 UI
- 파이썬 커널 생명주기 관리
- 스트리밍 처리

---

## 9. 진행 제안

1. 1주차 — 실제로 `prime-agent`를 사용해 보며 동작 감각 익히기
2. 2주차 — PHP `proc_open` 연동 POC (웹에서 프롬프트 입력 → 결과 표시)
3. 3~4주차 — 실제 레거시 PHP 프로젝트 하나를 대상으로 마이그레이션 시연
4. 5주차 이후 — 시연 결과(before/after)를 근거로 영업 시작

아이디어보다 동작하는 데모의 설득력이 훨씬 큽니다.
