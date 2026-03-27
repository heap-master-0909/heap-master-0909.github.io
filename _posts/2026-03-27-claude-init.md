---
title: Claude Code 시작하기 - AI 코딩 도우미 활용법
date: 2026-03-27 00:00:00 +0900
categories: [AI, Claude]
tags: [AI, Claude, Anthropic, Claude-Code, Tool]
pin: true
---

## 설치 및 시작

```bash
npm install -g @anthropic-ai/claude-code
```

설치 후 프로젝트 디렉토리에서 `claude` 명령어로 실행한다.

---

## 한글 출력 설정

Claude Code의 응답 언어를 한글로 고정하려면 `.claude/settings.json` 또는 `CLAUDE.md`에 지정한다.

```
# CLAUDE.md 예시
모든 응답은 한글로 작성해주세요.
```

---

## 프로젝트 초기화

```bash
/init
```

프로젝트 루트에 `CLAUDE.md` 파일을 생성한다. Cursor의 `.cursorrules`와 유사한 역할로, Claude Code가 대화 시작 시 우선적으로 읽어들인다.

`CLAUDE.md`에 프로젝트 컨벤션, 기술 스택, 주의사항 등을 작성하면 매 대화마다 컨텍스트를 반복 설명할 필요가 없다.

---

## Subagent (하위 에이전트)

`.claude/agents/` 디렉토리에 커스텀 에이전트를 정의할 수 있다.

- 기능별로 에이전트를 분리하여 역할을 명확히 한다
- 병렬 처리가 가능하여 복잡한 작업을 동시에 수행할 수 있다
- 예: 테스트 실행 에이전트, 코드 리뷰 에이전트, 탐색 에이전트 등

```markdown
<!-- .claude/agents/test-runner.md -->
테스트를 실행하고 실패한 항목을 분석하여 수정안을 제시합니다.
```

---

## Hooks

특정 이벤트 발생 시 자동으로 실행되는 셸 명령을 설정할 수 있다. `.claude/settings.json`에서 구성한다.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          { "type": "command", "command": "npx prettier --write $CLAUDE_FILE_PATH" }
        ]
      }
    ]
  }
}
```

- `PreToolUse` — 도구 실행 전
- `PostToolUse` — 도구 실행 후
- `Notification` — 알림 발생 시

---

## 주요 슬래시 명령어

| 명령어 | 설명 |
|--------|------|
| `/init` | 프로젝트 초기화 (`CLAUDE.md` 생성) |
| `/clear` | 현재 대화의 컨텍스트를 초기화 |
| `/resume` | 이전 대화 컨텍스트를 복원 |
| `/export` | 컨텍스트를 Cursor 등 다른 도구로 내보내기 |
| `/model` | 사용 모델 변경 (예: `/model opus`) |
| `/help` | 도움말 표시 |
| `/commit` | 변경사항 커밋 |
| `/review-pr` | PR 리뷰 |
| `/compact` | 대화 컨텍스트 압축 |

---

## 커스텀 슬래시 명령어

`.claude/commands/` 디렉토리에 마크다운 파일을 만들어 나만의 명령어를 정의할 수 있다.

```markdown
<!-- .claude/commands/optimize.md -->
선택된 코드의 성능을 분석하고 최적화 방안을 제시해주세요.
```

사용 시: `/optimize`

---

## 빠른 셸 명령 실행

대화 중 `!`를 접두사로 붙이면 바로 셸 명령을 실행할 수 있다.

```
! git status
! npm test
```

---

## 유용한 팁

- **컨텍스트 관리**: 대화가 길어지면 `/compact`로 압축하거나 `/clear`로 초기화
- **모델 전환**: `/model opus`로 고성능 모델, `/model haiku`로 빠른 응답 전환
- **메모리**: Claude Code는 `.claude/` 디렉토리에 프로젝트별 메모리를 저장하여 대화 간 컨텍스트를 유지
- **권한 설정**: `settings.json`에서 도구별 자동 허용/거부 규칙을 설정 가능
- **멀티 파일 작업**: 여러 파일을 동시에 읽고 수정할 수 있어 리팩토링에 유용
