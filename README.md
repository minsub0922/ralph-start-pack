# Ralph Starter Package Guide

Claude Code와 Ralphy를 사용해 제품 기획부터 반복 구현까지 실행하는 최소 가이드입니다.

> 예시는 Claude Code 기준입니다. 다른 Code Agent를 사용할 경우 같은 Agent를 기획과 Ralphy 실행에 사용하세요.

## 1. 저장소 초기화

프로젝트 저장소 루트에서 실행합니다.

```bash
git init
git config user.name "YOUR_NAME"
git config user.email "YOUR_EMAIL"
```

## 2. Code Agent로 제품 기획

Claude Code를 실행합니다.

```bash
claude
```

다음 문서의 전체 내용을 복사해 Claude Code에 입력합니다.

```text
https://github.com/minsub0922/ralph-start-pack/blob/main/SETUP_PROMPTS.md
```

기획 결과를 검토한 뒤 확정합니다.

```text
확정
```

Claude Code를 종료합니다.

```text
/exit
```

다음 파일이 생성되었는지 확인합니다.

```text
PRODUCT.md
AGENTS.md
tasks.yaml
```

생성된 기획 파일을 커밋합니다.

```bash
git add PRODUCT.md AGENTS.md tasks.yaml
git commit -m "docs: define product and Ralph tasks"
```

## 3. Ralphy 초기화
intall: https://github.com/michaelshimeles/ralphy

```bash
ralphy --init
```

## 4. Ralphy 규칙 등록

아래 명령 전체를 한 번에 복사해 실행합니다.

```bash
ralphy --add-rule \
  "Before coding, read PRODUCT.md and AGENTS.md and identify the target TUI behavior, keybindings, states, and acceptance criteria"

ralphy --add-rule \
  "Work on exactly one task and avoid unrelated refactoring or changes to unrelated TUI screens"

ralphy --add-rule \
  "Preserve the existing TUI architecture, state management, rendering conventions, and keybinding patterns unless the task explicitly requires changing them"

ralphy --add-rule \
  "Do not add, remove, or update dependencies"

ralphy --add-rule \
  "Handle terminal input, resize events, small terminal sizes, long text, empty states, and Unicode text without crashing or corrupting the layout"

ralphy --add-rule \
  "Restore terminal state on normal exit, cancellation, errors, and SIGINT; do not leave raw mode, cursor visibility, or alternate-screen state corrupted"

ralphy --add-rule \
  "Do not write debug logs or unstructured output over the active TUI; use the repository's existing logging mechanism"

ralphy --add-rule \
  "Keep rendering, keyboard handling, and state transitions deterministic and testable without timing-dependent sleeps or assertions"

ralphy --add-rule \
  "Before reporting task completion, run the repository-defined tests and validation commands relevant to the task, including TUI rendering, input handling, state transitions, and terminal cleanup where applicable"

ralphy --add-rule \
  "Do not weaken, skip, delete, or rewrite tests merely to make them pass"

ralphy --add-rule \
  "Do not modify SETUP_PROMPTS.md, PREFLIGHTS.md, PRODUCT.md, AGENTS.md, ralph.environment.json, scripts/ralph-preflight.sh, or files under .ralphy/preflight"

ralphy --add-rule \
  "Complete exactly one task per iteration and create exactly one descriptive Git commit containing the task implementation, tests, tasks.yaml update, and .ralphy/progress.txt update before reporting completion"
```

## 5. Ralphy 설정 생성

Claude Code를 다시 실행합니다.

```bash
claude
```

다음 프롬프트를 입력합니다.

```text
@PRODUCT.md @AGENTS.md @tasks.yaml을 기반으로 @.ralphy/config.yaml을 채워.
```

설정 내용을 확인한 뒤 Claude Code를 종료합니다.

```text
/exit
```

설정 파일이 생성되었는지 확인합니다.

```bash
cat .ralphy/config.yaml
```

## 6. Ralphy 실행

```bash
ralphy --yaml tasks.yaml
```

중단 후 다시 시작할 때도 같은 명령을 실행합니다.

```bash
ralphy --yaml tasks.yaml
```
