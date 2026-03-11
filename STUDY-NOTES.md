# Claude Forge 소스코드 분석 — 스터디 자료

> 분석일: 2026-02-23
> 대상: https://github.com/sangrokjung/claude-forge (main 브랜치)

---

## 1. 3줄 요약

1. Claude Code의 `~/.claude/` 디렉토리를 **symlink 허브**로 활용하여 에이전트·명령·훅·스킬을 일괄 관리하는 프레임워크
2. **PreToolUse/PostToolUse 훅 체계**로 보안(시크릿 필터, 명령 가드, DB 가드, Rate Limiter)과 학습(Continuous Learning)을 자동화
3. 에이전트 프롬프트에 `Role → Why → Success Criteria → Constraints → Protocol → Failure Modes → Checklist` 구조를 적용하여 **일관된 품질**을 확보

---

## 2. 아키텍처 개요

```
claude-forge (Git Repo)
├── agents/          # 11개 전문 에이전트 (.md)
├── commands/        # 36개 슬래시 명령 (.md)
├── skills/          # 15개 스킬 워크플로
├── hooks/           # 14개 자동화 훅 (.sh)
├── rules/           # 8개 규칙 파일 (.md)
├── knowledge/       # Anthropic 공식 문서 요약 (RAG용)
├── reference/       # 에이전트 팀·설정 레퍼런스
├── settings.json    # 핵심: 권한/훅/상태라인 통합 설정
└── install.sh       # symlink 기반 원클릭 설치
         │
         │  symlink
         ▼
    ~/.claude/
    ├── agents/ → repo/agents/
    ├── commands/ → repo/commands/
    ├── hooks/ → repo/hooks/
    ├── skills/ → repo/skills/
    ├── rules/ → repo/rules/
    └── settings.json → repo/settings.json
```

**핵심 설계 결정**: `git pull` 한 번이면 모든 설정이 즉시 업데이트됨 (symlink 덕분).

---

## 3. 주요 정책적 결정 (배울 점)

### 3-1. 권한 정책: 화이트리스트 + 블랙리스트 이중 구조

**`settings.json`의 `permissions` 섹션이 핵심.**

```
allow: 대부분의 도구를 허용 (Bash(*), Read(*), Write(*), mcp__*...)
deny:  구체적인 위험 패턴을 차단
```

**deny 목록에서 주목할 패턴들:**

| 차단 대상 | 패턴 | 이유 |
|-----------|------|------|
| 루트 삭제 | `Bash(rm -rf /)*` | 시스템 파괴 방지 |
| 홈 삭제 | `Bash(rm -rf ~)*` | 사용자 데이터 보호 |
| sudo | `Bash(sudo:*)` | 권한 상승 차단 |
| force push main | `Bash(git push --force*main)*` | 공유 브랜치 보호 |
| 패키지 배포 | `Bash(npm publish)*` | 실수 배포 방지 |
| 쉘 프로파일 덮어쓰기 | `Bash(*>~/.zshrc)*` | 환경 오염 방지 |
| 파이프-to-쉘 | `Bash(curl*\|*sh)*` | 원격 코드 실행 차단 |
| eval/exec | `Bash(eval *)*` | 명령 주입 차단 |
| osascript | `Bash(osascript*)*` | macOS 스크립트 차단 |

**배울 점**: WebFetch를 deny하고 MCP fetch 도구로 대체한 점. 내장 WebFetch보다 MCP 기반 도구가 더 통제 가능하다고 판단.

---

### 3-2. 훅 아키텍처: 6계층 이벤트 기반 자동화

```
SessionStart ──→ context-sync-suggest.sh (컨텍스트 동기화 안내)
UserPromptSubmit ──→ work-tracker-prompt.sh (작업 추적 시작)
PreToolUse:
  ├── Bash ──→ remote-command-guard.sh (위험 명령 차단)
  └── mcp__* ──→ rate-limiter.sh + mcp-usage-tracker.sh
PostToolUse:
  ├── 전체 ──→ work-tracker-tool.sh + output-secret-filter.sh + observe.sh
  ├── Edit ──→ code-quality-reminder.sh
  ├── Write ──→ code-quality-reminder.sh
  └── Edit|Write ──→ security-auto-trigger.sh (보안 파일 수정 감지)
Stop ──→ work-tracker-stop.sh + session-wrap-suggest.sh
TaskCompleted ──→ task-completed.sh
```

**각 훅의 설계 원칙:**
- exit 0 = 허용/계속, exit 2 = 차단
- stdin으로 JSON 수신 → Python으로 파싱·처리
- 차단 메시지는 stderr로 출력
- 세션당 중복 방지 (마커 파일 `/tmp/` 활용)

---

### 3-3. 에이전트 프롬프트 구조 (가장 배울 점이 많은 부분)

모든 에이전트가 동일한 XML 구조를 따름:

```xml
<Agent_Prompt>
  <Role>         <!-- 정체성 + 책임 범위 + NOT 책임 범위 -->
  <Why_This_Matters>  <!-- 왜 이 규칙이 필요한지 (동기 부여) -->
  <Success_Criteria>  <!-- 구체적 성공 조건 -->
  <Constraints>       <!-- 절대 하지 말아야 할 것 -->
  <Investigation_Protocol>  <!-- 단계별 실행 절차 -->
  <Tool_Usage>        <!-- 사용할 도구와 용도 -->
  <Execution_Policy>  <!-- 기본 effort 수준, 중단 조건 -->
  <Output_Format>     <!-- 출력 템플릿 -->
  <Failure_Modes_To_Avoid>  <!-- 흔한 실패 패턴 (안티패턴) -->
  <Final_Checklist>   <!-- 자기 검증 체크리스트 -->
</Agent_Prompt>
```

**왜 이 구조가 효과적인가:**

1. **`<Role>`에 "NOT 책임"을 명시** — 에이전트가 범위를 넘어서 행동하는 것을 방지
   - planner: "You never implement. You plan."
   - code-reviewer: "not responsible for implementing fixes"
   - security-reviewer: "not responsible for code style"

2. **`<Why_This_Matters>`로 동기 부여** — 규칙의 "왜"를 설명해야 AI가 엣지 케이스에서도 올바르게 판단
   - "cost of missing a vulnerability in review is orders of magnitude higher than the cost of a thorough check"

3. **`<Failure_Modes_To_Avoid>`가 핵심** — 흔한 안티패턴을 명시적으로 나열
   - planner: "30 micro-steps with implementation details" (과잉 계획)
   - code-reviewer: "Style-first review: Nitpicking formatting while missing a SQL injection" (우선순위 역전)
   - tdd-guide: "Adding retries or sleep instead of fixing root cause" (증상 치료)

---

### 3-4. 보안 훅 상세 분석

#### output-secret-filter.sh (시크릿 마스킹)
- **2단계 탐지**: 원본 텍스트 직접 매칭 → Base64/URL 디코딩 후 재매칭
- 25+개 시크릿 패턴 (OpenAI, AWS, Slack, GitHub, GitLab, NPM 토큰 등)
- 마스킹 형식: `sk-proj-ab***MASKED***wxyz` (앞 8자 + 뒤 4자 보존)
- 보안 로그에 "어떤 타입이 감지됐는지"만 기록 (값 자체는 기록 안 함)
- `OPENCLAW_SESSION_ID`가 없으면 검사 건너뜀 (로컬에선 비활성)

#### remote-command-guard.sh (명령 가드)
- 7개 차단 범주: 파괴적 삭제, 시크릿 유출, 경로 순회, 외부 통신, 권한 변경, 프로세스 종료, 명령 주입
- **localhost curl은 허용** (개발용 예외 처리가 세심함)
- 명령 정규화: 여러 공백 → 단일 공백으로 정규화 후 매칭

#### rate-limiter.sh (속도 제한)
- 분당 30회 / 시간당 500회 / 일일 5000회 (슬라이딩 윈도우)
- **제한값 하드코딩** — 환경변수로 오버라이드 불가 (보안 정책)
- `fcntl.flock`으로 파일 잠금 (동시 접근 경쟁 조건 방지)
- 차단된 요청은 카운트하지 않음 (DOS 방지)

#### db-guard.sh (DB 보호)
- DROP TABLE/DATABASE/SCHEMA 차단
- TRUNCATE 차단
- DELETE without WHERE 차단
- ALTER TABLE ... DROP COLUMN 차단
- 심플하지만 **핵심적인 파괴 방지**에 집중

---

### 3-5. 워크플로 자동화: /handoff-verify 분석

이 프로젝트의 가장 정교한 명령. v5 → v6 진화 과정이 인상적:

```
v5: /handoff → /clear → /verify (3단계, 수동)
v6: /handoff-verify (1단계, 서브에이전트 자동)
```

**핵심 아이디어**: Task 도구로 생성한 서브에이전트가 fresh context를 자동 제공
→ `/clear` (컨텍스트 초기화) 없이도 동일 효과
→ 부모 컨텍스트는 보존됨

**effort 기반 Adaptive Thinking:**

| effort | 코드 리뷰 범위 | thinking 수준 |
|--------|---------------|--------------|
| low | 변경 파일만 빠른 스캔 | 기본 |
| medium | 변경 파일 + 직접 의존성 | think hard |
| high | 의존성 그래프 추적 | think harder |
| max | 전체 프로젝트 영향 분석 | ultrathink |

**Fixable vs Non-Fixable 자동 분류:**
- Fixable: import 누락, lint, unused import, semicolons → 자동 수정 시도
- Non-Fixable: 로직 오류, 아키텍처, 순환 의존성 → 보고만

---

### 3-6. Continuous Learning v2 (자가 학습 시스템)

가장 야심찬 기능. 세션에서 패턴을 학습하여 "instinct"(본능)으로 저장:

```
세션 활동 → 훅 캡처 → observations.jsonl
  → Observer Agent (Haiku, 백그라운드)
  → 패턴 탐지 (사용자 교정, 에러 해결, 반복 워크플로)
  → instincts/personal/ (atomic 행동 단위, 신뢰도 0.3~0.9)
  → /evolve로 클러스터링 → commands/skills/agents 자동 생성
```

**신뢰도 체계:**
- 0.3 = 잠정적 (제안만)
- 0.5 = 보통 (관련 시 적용)
- 0.7 = 강함 (자동 승인)
- 0.9 = 거의 확실 (핵심 행동)

**v1→v2 핵심 변경**: Skills(확률적, 50-80% 발동)에서 Hooks(결정적, 100% 발동)로 관찰 메커니즘 변경.

---

### 3-7. 전략적 컴팩션 (Strategic Compact)

자동 컴팩션의 문제를 인식하고 **논리적 경계**에서 수동 컴팩션을 제안:

- 도구 호출 50회 도달 시 첫 제안
- 이후 25회마다 리마인더
- "탐색 끝 → 실행 전", "마일스톤 완료 후", "컨텍스트 전환 전"에서 제안

---

## 4. 코딩 규칙 (Golden Principles)

7가지 황금 원칙을 rules에 명문화:

| # | 원칙 | 핵심 규칙 |
|---|------|----------|
| 1 | 불변성 | spread operator로 새 객체. 원본 수정 금지 |
| 2 | 시크릿 환경변수화 | `process.env`로만 접근. 미설정 시 throw |
| 3 | TDD | RED → GREEN → IMPROVE. 커버리지 80%+ |
| 4 | 결론 먼저 | 첫 문장에 결론 → 근거는 나중 |
| 5 | 작은 파일/함수 | 파일 800줄, 함수 50줄, 중첩 4단계 한계 |
| 6 | 시스템 경계 검증 | zod로 입력 검증, 파라미터화된 쿼리 |
| 7 | 비유로 설명 | 일상 비유 1-2문장 → 기술 설명 순서 |

---

## 5. 우리 개발에 반영할 수 있는 아이디어

### 즉시 적용 가능 (Low Effort, High Value)

1. **에이전트 프롬프트 구조 도입**
   - `Role + Why + Success Criteria + Constraints + Failure Modes + Checklist` 구조
   - 특히 `Failure_Modes_To_Avoid` (안티패턴 명시)가 품질에 큰 영향

2. **settings.json deny 패턴 참고**
   - `rm -rf`, `sudo`, `force push main`, `eval`, `npm publish` 등 차단
   - 우리 환경에 맞게 커스터마이즈하여 적용

3. **Golden Principles 채택**
   - 파일 800줄/함수 50줄/중첩 4단계 규칙을 CLAUDE.md에 추가
   - 불변성 패턴 강제를 룰로 명문화

4. **결론-먼저 커뮤니케이션 규칙**
   - 이미 우리 CLAUDE.md에도 비슷한 규칙이 있지만, "비유로 설명" 규칙 추가 검토

### 중기 적용 (Medium Effort)

5. **보안 훅 시스템**
   - output-secret-filter의 Base64/URL 디코딩 우회 탐지 로직
   - security-auto-trigger의 보안 민감 파일 자동 감지 → 리뷰 제안

6. **handoff-verify 패턴**
   - 서브에이전트로 fresh context 검증하는 아이디어
   - effort 레벨별 adaptive thinking 적용

7. **knowledge 디렉토리 운영**
   - Anthropic 공식 문서 요약을 로컬에 보관하여 RAG처럼 활용
   - index.json으로 메타데이터 관리

### 장기 검토 (High Effort, Experimental)

8. **Continuous Learning v2**
   - 세션 관찰 → 패턴 추출 → instinct 생성 → 스킬 진화
   - 개념은 강력하나, 실효성 검증이 필요

9. **Strategic Compact**
   - 논리적 경계에서 컨텍스트 컴팩션 제안
   - 장시간 세션에서 유용할 수 있음

---

## 6. 비판적 관점 (주의점)

1. **복잡도 vs 실용성**: 36개 명령, 11개 에이전트, 14개 훅은 개인 프로젝트로서 상당히 복잡함. 실제 팀 도입 시 학습 곡선이 높을 수 있음.

2. **원격 세션 의존**: 많은 보안 훅이 `OPENCLAW_SESSION_ID` 환경변수를 체크하여 로컬에서는 비활성됨. 이는 원격 제어 시나리오에 특화된 설계.

3. **Python 의존**: 훅들이 bash에서 Python heredoc을 인라인으로 실행. Python3 필수 의존성이 암묵적.

4. **테스트 부재**: 훅/스크립트 자체에 대한 테스트 코드가 보이지 않음. 정규식 기반 보안 훅이 우회될 가능성.

5. **에이전트 프롬프트의 실효성**: 마크다운 기반 에이전트 정의는 Claude Code의 실제 에이전트 실행 메커니즘에 따라 효과가 달라질 수 있음.

---

## 7. 파일 경로 퀵 레퍼런스

| 용도 | 경로 |
|------|------|
| 핵심 설정 | `settings.json` |
| 에이전트 프롬프트 | `agents/*.md` |
| 훅 스크립트 | `hooks/*.sh` |
| 슬래시 명령 | `commands/*.md` |
| 코딩 규칙 | `rules/*.md` |
| 학습 시스템 | `skills/continuous-learning-v2/` |
| 보안 규칙 | `rules/security.md` |
| 설치 스크립트 | `install.sh` |
| 지식 베이스 | `knowledge/` |
