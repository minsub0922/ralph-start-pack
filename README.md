아래 순서는 기존의 **기획 → seed scaffold → Ralphy 초기화 → preflight → 기준점 commit → 1-task pilot → 본 실행** 흐름을 그대로 따른 것입니다. 

현재 Ralphy 공식 CLI는 `npm install -g ralphy-cli`로 설치하며, `--init`, `--yaml`, `--dry-run`, `--max-iterations`, `--max-retries`, `--no-commit`, `--no-browser`를 지원합니다. npm 버전은 Node.js 18 이상이 필요합니다. ([GitHub][1])

---

# 0. 빈 저장소 시작

```bash
cd ~/Dev/Projects
mkdir my-ralph-project
cd my-ralph-project

# 이 두 파일이 이미 있어야 함
ls -l SETUP_PROMPT.md PREFLIGHT.md
```

```bash
git init
git branch -M main

git commit --allow-empty -m "chore: initialize repository"

git add SETUP_PROMPT.md PREFLIGHT.md
git commit -m "docs: add setup and preflight prompts"
```

```bash
git log --oneline --decorate -3
git status --short
```

예상:

```text
xxxxxxx docs: add setup and preflight prompts
xxxxxxx chore: initialize repository
```

Commit 사용자 정보 오류가 날 때만 실행:

```bash
git config user.name "YOUR_NAME"
git config user.email "YOUR_EMAIL"
```

---

# 1. Claude Code로 제품 기획

```bash
claude
```

Claude Code에 아래 프롬프트를 입력:

```text
Read SETUP_PROMPT.md and follow it exactly.

Additional constraints:

- Do not read or execute PREFLIGHT.md yet.
- Do not modify SETUP_PROMPT.md or PREFLIGHT.md.
- Start by asking only question 1.
- Continue asking one question at a time.
- Do not create files until I say "확정".

When creating tasks.yaml, use exactly this top-level structure:

tasks:
  - title: "T01 - task title"
    completed: false
    description: |
      Task: T01 - task title

      Goal:
      ...

      Acceptance criteria:
      - ...

      Verification:
      - npm run check

Requirements for tasks.yaml:

- Every task must contain title, completed, and description.
- The first line of description must repeat the task title.
- All completed values must be false.
- Do not include project initialization or dependency installation tasks.

Begin with question 1 only.
```

질문에 하나씩 답변합니다.

마지막 질문까지 합의되면 입력:

```text
확정
```

파일 생성이 끝나면:

```text
/exit
```

Ralphy의 현재 YAML parser는 최상위 `tasks:` 배열을 기대하고, `description`을 실제 agent 작업 본문으로 사용합니다. 

---

# 2. 기획 결과 검토 및 commit

```bash
ls -l \
  SETUP_PROMPT.md \
  PREFLIGHT.md \
  PRODUCT.md \
  tasks.yaml \
  AGENTS.md
```

```bash
sed -n '1,240p' PRODUCT.md
sed -n '1,320p' tasks.yaml
sed -n '1,240p' AGENTS.md
```

```bash
# 최상위 tasks 확인
grep -n '^tasks:' tasks.yaml

# 모든 task 상태 확인
grep -n 'completed:' tasks.yaml

# true가 하나라도 나오면 중단
if grep -Eq 'completed:[[:space:]]*true' tasks.yaml; then
  echo "STOP: completed:true가 존재함"
else
  echo "OK: 모든 task가 false"
fi
```

```bash
git diff --check
git status --short
```

예상:

```text
?? PRODUCT.md
?? tasks.yaml
?? AGENTS.md
```

Commit:

```bash
git add PRODUCT.md tasks.yaml AGENTS.md
git commit -m "docs: define product and Ralph tasks"
```

```bash
git status --short
git log --oneline --decorate -5
```

---

# 3. Claude Code로 최소 seed scaffold 생성

Ralphy로 제품 task를 실행하기 전에 최소 실행 가능한 프로젝트를 만듭니다.

```bash
claude
```

Claude Code에 입력:

```text
Read PRODUCT.md, tasks.yaml, and AGENTS.md.

Create only the minimal runnable seed project required before Ralph/Ralphy execution.

Use the fixed stack:

- TypeScript
- Next.js App Router
- npm
- Local JSON storage unless the planning documents explicitly require SQLite
- Vitest
- Playwright
- ESLint

Create these npm scripts:

- dev
- build
- start
- typecheck
- lint
- test:run
- test:e2e
- check

`npm run check` must run all non-interactive checks in this order:

1. typecheck
2. lint
3. unit tests
4. build
5. Playwright smoke test

Requirements:

- Create package.json and package-lock.json.
- Declare every required dependency in package.json.
- Do not use global project dependencies.
- Do not implement any product feature from tasks.yaml.
- Do not modify PRODUCT.md, tasks.yaml, AGENTS.md, SETUP_PROMPT.md, or PREFLIGHT.md.
- Follow the directory structure documented in AGENTS.md.
- Add only one placeholder page proving Next.js runs.
- Add one minimal unit smoke test.
- Add one minimal Playwright browser smoke test.
- Configure Playwright webServer so test:e2e is non-interactive.
- Do not add authentication, payment, deployment, or external APIs.
- Do not run Ralphy.
- Do not commit or push.

You may install the declared dependencies to generate package-lock.json.

Run npm run check if the current environment allows it.

If installation or verification fails, do not hide the error. Report the exact failing command and leave the repository in a diagnosable state.
```

완료 후:

```text
/exit
```

확인:

```bash
ls -l package.json package-lock.json
cat package.json

git status --short
git diff --check
```

가능하면 실행:

```bash
npm run check
```

성공하면:

```text
typecheck PASS
lint PASS
unit test PASS
build PASS
e2e PASS
```

실패하더라도 아직 commit하지 않고 다음 preflight 단계로 이동합니다.

---

# 4. Ralphy CLI 설치

```bash
node --version
npm --version
claude --version
```

Node 18 이상 확인:

```bash
node -e '
const major = Number(process.versions.node.split(".")[0]);
console.log("Node:", process.version);
if (major < 18) {
  console.error("ERROR: Ralphy requires Node.js 18+");
  process.exit(1);
}
'
```

Ralphy 설치:

```bash
npm install -g ralphy-cli
```

```bash
command -v ralphy
ralphy --help | sed -n '1,180p'
```

예상:

```text
/usr/.../ralphy
```

권한 오류가 나면 다음처럼 실행하지 않습니다.

```bash
# 실행하지 않음
sudo npm install -g ralphy-cli
```

사용 중인 `nvm`, `fnm`, `asdf` 등의 사용자 Node 환경을 활성화한 뒤 다시 설치합니다.

---

# 5. Scaffold가 생긴 뒤 Ralphy 초기화

```bash
ralphy --init
```

확인:

```bash
find .ralphy -maxdepth 2 -type f -print
ralphy --config
```

예상:

```text
.ralphy/config.yaml
.ralphy/progress.txt
```

공통 rule 추가:

```bash
ralphy --add-rule \
  "Before coding, read PRODUCT.md and AGENTS.md"

ralphy --add-rule \
  "Work on exactly one task and avoid unrelated refactoring"

ralphy --add-rule \
  "Run npm run check before reporting task completion"

ralphy --add-rule \
  "Do not add, remove, or update dependencies"

ralphy --add-rule \
  "Do not weaken, skip, delete, or rewrite tests merely to make them pass"

ralphy --add-rule \
  "Do not modify SETUP_PROMPT.md, PREFLIGHT.md, PRODUCT.md, AGENTS.md, ralph.environment.json, scripts/ralph-preflight.sh, or files under .ralphy/preflight"
```

---

# 6. `.ralphy/config.yaml` 정리

```bash
claude
```

Claude Code에 입력:

```text
Read PRODUCT.md, AGENTS.md, package.json, and .ralphy/config.yaml.

Update only `.ralphy/config.yaml`.

Preserve correctly detected project information.

Set or verify:

project:
  language: TypeScript
  framework: Next.js
  description: Read PRODUCT.md for the product definition.

commands:
  test: npm run test:run
  lint: npm run lint
  build: npm run build

Ensure these rules exist without duplicates:

- Before coding, read PRODUCT.md and AGENTS.md
- Work on exactly one task and avoid unrelated refactoring
- Run npm run check before reporting task completion
- Do not add, remove, or update dependencies
- Do not weaken, skip, delete, or rewrite tests merely to make them pass
- Do not modify preflight infrastructure

Set boundaries.never_touch to include:

- SETUP_PROMPT.md
- PREFLIGHT.md
- PRODUCT.md
- AGENTS.md
- package.json
- package-lock.json
- ralph.environment.json
- scripts/ralph-preflight.sh
- .ralphy/config.yaml
- .ralphy/preflight/**
- .env
- .env.*
- node_modules/**

Do not modify tasks.yaml.
Do not modify any other file.
Do not run Ralphy.
Do not commit or push.
```

완료 후:

```text
/exit
```

확인:

```bash
cat .ralphy/config.yaml
ralphy --config
git diff --check
```

---

# 7. Claude Code로 `PREFLIGHT.md` 수행

```bash
claude
```

Claude Code에 입력:

```text
Read PREFLIGHT.md and execute it completely in this repository.

Use:

- PRODUCT.md
- tasks.yaml
- AGENTS.md
- package.json
- package-lock.json
- .ralphy/config.yaml
- all build, test, lint, Playwright, environment, and CI configuration

Requirements:

- Create the preflight infrastructure described in PREFLIGHT.md.
- Create ralph.environment.json.
- Create scripts/ralph-preflight.sh.
- Create .ralphy/preflight/report.md.
- Run plan, setup, and verify.
- Do not implement any product task.
- Do not modify any task completed value.
- Do not modify PRODUCT.md or AGENTS.md.
- Do not run an actual Ralphy feature task.
- Do not commit or push.
- Do not automatically execute sudo commands.
- Do not create or expose secrets.

For Ralphy dry-run validation, always use a bounded command:

ralphy --yaml tasks.yaml --dry-run --max-iterations 1 --no-browser --no-commit

Never run an unbounded dry-run.

If a manifest, lockfile, permission, environment variable, authentication, or system dependency issue remains, finish with exactly one of:

- BLOCKED
- USER ACTION REQUIRED

Only report PASS after the full quality command succeeds.
```

완료 후:

```text
/exit
```

현재 순차 실행 코드에서 dry-run은 task를 완료 처리하지 않으므로, 종료 범위를 제한하기 위해 `--max-iterations 1`을 함께 사용하는 것이 안전합니다. 

---

# 8. 사용자가 preflight를 직접 재실행

```bash
chmod +x scripts/ralph-preflight.sh
bash -n scripts/ralph-preflight.sh
```

```bash
./scripts/ralph-preflight.sh plan tasks.yaml
```

```bash
sed -n '1,320p' .ralphy/preflight/report.md
```

Blocker가 없다면:

```bash
./scripts/ralph-preflight.sh setup tasks.yaml
./scripts/ralph-preflight.sh verify tasks.yaml
```

확인:

```bash
test -f .ralphy/preflight/PASSED \
  && echo "PREFLIGHT PASS" \
  || echo "PREFLIGHT NOT PASSED"
```

예상:

```text
PREFLIGHT PASS
```

---

# 9. `USER ACTION REQUIRED`가 나온 경우

Privileged script가 생성된 경우:

```bash
sed -n '1,260p' .ralphy/preflight/privileged-actions.sh
```

내용을 검토한 후에만:

```bash
chmod +x .ralphy/preflight/privileged-actions.sh
./.ralphy/preflight/privileged-actions.sh
```

필수 환경변수가 있는 경우:

```bash
cat ralph.environment.json
```

`.env.example`이 있다면:

```bash
cp .env.example .env.local
${EDITOR:-vi} .env.local
```

실제 secret을 입력한 후:

```bash
./scripts/ralph-preflight.sh verify tasks.yaml
```

---

# 10. `BLOCKED`가 나온 경우

Report 확인:

```bash
cat .ralphy/preflight/blockers.txt
sed -n '1,320p' .ralphy/preflight/report.md
```

Manifest 또는 lockfile 문제만 수정하도록 Claude Code 실행:

```bash
claude
```

입력:

```text
Read:

- PREFLIGHT.md
- .ralphy/preflight/report.md
- .ralphy/preflight/blockers.txt
- package.json
- package-lock.json
- all package scripts and configuration files

Fix only the seed manifest and lockfile blockers identified by preflight.

Requirements:

- Add only dependencies already required by existing scripts or configuration.
- Keep package.json and package-lock.json consistent.
- Use npm.
- Do not install project dependencies globally.
- Do not implement product features.
- Do not modify PRODUCT.md, tasks.yaml, AGENTS.md, SETUP_PROMPT.md, or PREFLIGHT.md.
- Do not modify task completion states.
- Do not run Ralphy.
- Run npm install only as needed to update package-lock.json.
- Run the preflight plan, setup, and verify commands again.
- Do not commit or push.

Finish with PASS or BLOCKED.
```

완료 후:

```text
/exit
```

다시 검증:

```bash
./scripts/ralph-preflight.sh plan tasks.yaml
./scripts/ralph-preflight.sh setup tasks.yaml
./scripts/ralph-preflight.sh verify tasks.yaml
```

---

# 11. Ralph-ready 기준점 검증

다음 명령이 모두 성공해야 합니다.

```bash
npm run check
```

```bash
git diff --check
```

```bash
test -x node_modules/.bin/tsc
node_modules/.bin/tsc --version
```

```bash
test -f .ralphy/preflight/PASSED
```

```bash
grep -n 'completed:' tasks.yaml
```

모든 task가 아직 `false`인지 확인:

```bash
TRUE_COUNT="$(
  grep -Ec \
    '^[[:space:]]*completed:[[:space:]]*true[[:space:]]*$' \
    tasks.yaml || true
)"

echo "completed true: $TRUE_COUNT"
test "$TRUE_COUNT" -eq 0
```

예상:

```text
completed true: 0
```

---

# 12. Ralph-ready 기준점 commit 및 tag

```bash
git status --short
git diff --stat
git diff --check
```

```bash
git add -A
```

Commit 전에 반드시 확인:

```bash
git diff --cached --name-only
git diff --cached --check
```

`.env`, `.env.local`, secret, `node_modules`가 stage되어 있으면 제거:

```bash
git restore --staged .env .env.local 2>/dev/null || true
git restore --staged node_modules 2>/dev/null || true
```

Commit:

```bash
git commit -m "chore: create verified Ralph-ready seed"
git tag ralph-ready
```

확인:

```bash
git status --short
git log --oneline --decorate -8
```

예상:

```text
xxxxxxx (HEAD -> main, tag: ralph-ready)
        chore: create verified Ralph-ready seed
```

---

# 13. Ralphy dry-run

현재 Ralphy는 기본적으로 auto-commit이 활성화되어 있으므로, 아래 과정에서는 사람이 검증 후 commit하도록 항상 `--no-commit`을 사용합니다. 

```bash
./scripts/ralph-preflight.sh run tasks.yaml -- \
  --yaml tasks.yaml \
  --dry-run \
  --max-iterations 1 \
  --max-retries 1 \
  --no-browser \
  --no-commit
```

예상:

```text
Starting Ralphy with Claude Code
Tasks remaining: N
Task 1: ...
(dry run) Skipped
Reached max iterations (1)

Completed: 0
Failed:    0
```

Dry-run 후 변경이 없어야 합니다.

```bash
git status --short
```

---

# 14. 첫 task pilot 실행

실행 전 상태:

```bash
git status --short
git rev-parse --short HEAD
grep -n 'completed:' tasks.yaml
```

Task 파일 백업:

```bash
mkdir -p .ralphy/preflight
cp tasks.yaml .ralphy/preflight/tasks.before-pilot.yaml
```

Pilot:

```bash
./scripts/ralph-preflight.sh run tasks.yaml -- \
  --yaml tasks.yaml \
  --max-iterations 1 \
  --max-retries 1 \
  --no-browser \
  --no-commit
```

예상:

```text
Starting Ralphy with Claude Code
Tasks remaining: N
Mode: Sequential

Task 1: ...
Completed: 1
Failed:    0
```

---

# 15. Pilot 직후 검증

```bash
./scripts/ralph-preflight.sh verify tasks.yaml
```

```bash
git diff --check
git status --short
git diff --stat
git diff -- tasks.yaml
```

정확히 한 task만 완료됐는지 확인:

```bash
TRUE_COUNT="$(
  grep -Ec \
    '^[[:space:]]*completed:[[:space:]]*true[[:space:]]*$' \
    tasks.yaml || true
)"

FALSE_COUNT="$(
  grep -Ec \
    '^[[:space:]]*completed:[[:space:]]*false[[:space:]]*$' \
    tasks.yaml || true
)"

echo "completed: $TRUE_COUNT"
echo "remaining: $FALSE_COUNT"

test "$TRUE_COUNT" -eq 1
```

예상:

```text
completed: 1
remaining: N-1
```

추가 확인:

```bash
npm run check
git diff --check
```

Pilot commit:

```bash
git add -A
git diff --cached --check
git diff --cached --stat

git commit -m "feat: complete Ralph pilot task"
git tag ralph-pilot-pass
```

```bash
git status --short
git log --oneline --decorate -8
```

---

# 16. Pilot이 실패한 경우

검증이 실패하면 commit하지 않습니다.

```bash
git diff --binary > ../ralph-pilot-failed.patch
git stash push -u -m "failed Ralph pilot"
```

깨끗한 기준점에서 재시작:

```bash
git switch -c ralph-pilot-retry ralph-ready
```

```bash
./scripts/ralph-preflight.sh verify tasks.yaml
```

Task 설명이 부족한 것이 원인이라면 `tasks.yaml`을 수정한 뒤:

```bash
git add tasks.yaml
git commit -m "docs: clarify failed Ralph task"
```

```bash
./scripts/ralph-preflight.sh verify tasks.yaml
```

Pilot 재실행:

```bash
./scripts/ralph-preflight.sh run tasks.yaml -- \
  --yaml tasks.yaml \
  --max-iterations 1 \
  --max-retries 1 \
  --no-browser \
  --no-commit
```

---

# 17. 본 실행: 두 task씩 수행

남은 task 확인:

```bash
grep -n 'completed:[[:space:]]*false' tasks.yaml
```

안전 기준점 기록:

```bash
SAFE_HEAD="$(git rev-parse HEAD)"
echo "$SAFE_HEAD"
```

두 task 실행:

```bash
./scripts/ralph-preflight.sh run tasks.yaml -- \
  --yaml tasks.yaml \
  --max-iterations 2 \
  --max-retries 2 \
  --no-browser \
  --no-commit
```

---

# 18. 각 batch 후 검증 및 commit

```bash
./scripts/ralph-preflight.sh verify tasks.yaml
```

```bash
npm run check
git diff --check
git status --short
git diff --stat
```

진행 상태:

```bash
TRUE_COUNT="$(
  grep -Ec \
    '^[[:space:]]*completed:[[:space:]]*true[[:space:]]*$' \
    tasks.yaml || true
)"

FALSE_COUNT="$(
  grep -Ec \
    '^[[:space:]]*completed:[[:space:]]*false[[:space:]]*$' \
    tasks.yaml || true
)"

echo "completed: $TRUE_COUNT"
echo "remaining: $FALSE_COUNT"
```

검증 성공 후 commit:

```bash
git add -A
git diff --cached --check
git diff --cached --stat

git commit -m "feat: complete next Ralph task batch"
```

다음 batch:

```bash
./scripts/ralph-preflight.sh run tasks.yaml -- \
  --yaml tasks.yaml \
  --max-iterations 2 \
  --max-retries 2 \
  --no-browser \
  --no-commit
```

다음 과정을 반복합니다.

```text
run 2 tasks
→ verify
→ npm run check
→ diff 확인
→ commit
→ 다음 2 tasks
```

---

# 19. Batch 실패 시 복구

실패 내용을 보존:

```bash
git diff --binary > ../ralph-batch-failed.patch
git stash push -u -m "failed Ralph batch"
```

마지막 성공 commit에서 retry branch 생성:

```bash
git switch -c "ralph-batch-retry-$(date +%Y%m%d-%H%M%S)" "$SAFE_HEAD"
```

```bash
./scripts/ralph-preflight.sh verify tasks.yaml
```

실패한 task 설명을 보완한 뒤 같은 batch 명령을 다시 실행합니다.

---

# 20. 전체 task 완료 확인

```bash
if grep -n \
  '^[[:space:]]*completed:[[:space:]]*false[[:space:]]*$' \
  tasks.yaml; then
  echo "STOP: 아직 미완료 task가 있음"
else
  echo "ALL TASKS COMPLETE"
fi
```

예상:

```text
ALL TASKS COMPLETE
```

---

# 21. 최종 검증

```bash
./scripts/ralph-preflight.sh verify tasks.yaml
```

```bash
npm run check
git diff --check
git status --short
```

```bash
git log --oneline --decorate -15
```

마지막 변경사항이 남았다면:

```bash
git add -A
git diff --cached --check
git commit -m "chore: finalize Ralph implementation"
```

최종 상태:

```bash
git status --short
```

예상:

```text
# 출력 없음
```

애플리케이션 수동 실행:

```bash
npm run dev
```

---

# 전체 명령 흐름만 다시 보기

```text
git init
→ prompt 파일 commit
→ claude + SETUP_PROMPT.md
→ PRODUCT.md / tasks.yaml / AGENTS.md commit
→ claude로 seed scaffold 생성
→ ralphy 설치
→ ralphy --init
→ .ralphy/config.yaml 설정
→ claude + PREFLIGHT.md
→ plan
→ setup
→ verify
→ npm run check
→ ralph-ready commit/tag
→ bounded dry-run
→ 1-task pilot
→ verify
→ pilot commit
→ 2-task batch 반복
→ 최종 verify
```

[1]: https://github.com/michaelshimeles/ralphy/blob/main/cli/README.md?utm_source=chatgpt.com "ralphy/cli/README.md at main · michaelshimeles/ralphy · GitHub"

# ralph-start-pack
