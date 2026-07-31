# Ralphy 실행 프로세스

> 전제: 저장소 루트에 `SETUP_PROMPTS.md`와 `PREFLIGHTS.md`가 있다.
>
> 두 파일은 완성된 프롬프트이므로 내용을 그대로 전달하고 추가 지시를 붙이지 않는다.
> 본 실행에는 태스크 수나 loop 수 제한을 두지 않으며, 실행 환경은 최소 3시간 동안 프로세스를 유지할 수 있어야 한다.

## 1. 제품 기획

```bash
git init
git commit --allow-empty -m "chore: initialize repository"
claude "$(cat SETUP_PROMPTS.md)"
```

대화가 끝나면 `PRODUCT.md`, `AGENTS.md`, `tasks.yaml`을 검토한다. `tasks.yaml`의 태스크 개수에는 상한을 두지 않는다.

## 2. Seed scaffold와 Ralphy 초기화

제품 기능을 구현하지 않은 최소 실행 골격만 만든다.

```bash
ralphy \
  --no-commit \
  --max-retries 1 \
  --no-browser \
  "Read PRODUCT.md, tasks.yaml, and AGENTS.md.
Create only the minimal runnable seed project for the fixed stack.
Add npm run check, one placeholder page, one unit smoke test, and one browser smoke test.
Do not implement tasks.yaml, modify planning files, or commit."

ralphy --init
ralphy --add-rule "Before coding, read PRODUCT.md and AGENTS.md"
ralphy --add-rule "Work on exactly one task and avoid unrelated refactoring"
ralphy --add-rule "Run npm run check before reporting completion"
ralphy --add-rule "Do not add, remove, or update dependencies"
ralphy --add-rule "Do not weaken or delete tests"
```

`.ralphy/config.yaml`의 명령과 수정 금지 영역을 확인한다. 최소 금지 대상은 `.env*`, `PRODUCT.md`, `AGENTS.md`, `SETUP_PROMPTS.md`, `PREFLIGHTS.md`다.

## 3. 실행 환경 검증

`PREFLIGHTS.md`만 그대로 실행한다.

```bash
claude -p "$(cat PREFLIGHTS.md)"
```

최종 상태가 `PASS`가 아니면 feature task를 실행하지 않는다.

```bash
bash -n scripts/ralph-preflight.sh
./scripts/ralph-preflight.sh verify tasks.yaml
```

## 4. Seed 확정과 dry run

`.ralphy/preflight/`의 runtime artifact는 Git에서 제외한 뒤 seed를 확정한다.

```bash
git diff --check
git status --short
git add .
git commit -m "chore: create ralph-ready product seed"
git tag product-seed
git switch -c "ralph/session-$(date +%Y%m%d-%H%M)"
git push -u origin HEAD

ralphy --yaml tasks.yaml --dry-run
```

## 5. 3시간 본 실행

터미널, 워크숍 세션 또는 CI job의 허용 실행 시간을 **최소 180분**으로 설정한다. Ralphy 명령에는 task/loop 개수 제한이나 별도 wrapper를 넣지 않는다. 모든 태스크가 먼저 완료되면 정상적으로 조기 종료한다.

실행 직전에 preflight를 다시 확인한다.

```bash
./scripts/ralph-preflight.sh verify tasks.yaml
```

최종 실행 명령은 다음과 같다.

```bash
ralphy \
  --yaml tasks.yaml \
  --max-retries 2 \
  --no-browser
```

이 실행은 기본 sequential mode로 남은 태스크를 모두 처리한다. `--max-iterations`를 사용하지 않으므로 처리할 태스크 수를 제한하지 않으며, `--no-commit`을 사용하지 않으므로 Ralphy의 task별 자동 커밋을 유지한다.

## 6. 종료 확인과 GitHub 반영

```bash
npm run check
git diff --check
git status --short
git log --oneline --decorate product-seed..HEAD
git push
```

3시간 실행 창이 끝났는데 미완료 태스크가 남았다면 현재 변경을 먼저 검토한다. 완료된 커밋을 push한 뒤 동일한 최종 명령을 다시 실행하면 `completed: false`인 다음 태스크부터 이어서 처리한다.

