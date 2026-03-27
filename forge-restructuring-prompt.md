# Forge — Project Restructuring Prompt

Give this prompt to your Copilot in VS Code. It will reorganize your existing Forge project into the optimized four-file structure.

---

## Prompt starts here — copy everything below this line

---

I need you to restructure my Forge project. Forge is an automated multi-repository task runner that forges code changes across 50-70 Spring WebFlux Java microservices. It currently works but the prompt/instruction files need to be reorganized for better context window efficiency.

## What exists today

I already have a working Forge project with:
- Node.js orchestrator scripts (scripts/orchestrator.js, git-ops.js, gitlab-api.js, reporter.js, utils.js)
- Playbook YAML files (playbooks/vulnerability-fix.yaml, config-update.yaml, version-bump.yaml)
- Task instance YAML files (tasks/)
- Configuration files (config/gitlab-config.yaml, config/projects.yaml)
- package.json, .gitignore, workspace/, logs/ directories

Do NOT modify any of the scripts, playbooks, tasks, or configuration files. Only create or reorganize the instruction/prompt files.

## What I need you to do

Restructure the instruction files into four separate files optimized for context window usage. Each file has a specific purpose and is loaded only when needed.

### Target structure

```
forge/
├── .github/
│   └── copilot-instructions.md     # AUTO-LOADED by VS Code Copilot every session
├── prompts/
│   ├── execute.md                   # Load when running tasks
│   ├── playbook-builder.md          # Load when creating new playbooks
│   └── reviewer.md                  # Load when reviewing batch results
├── playbooks/                       # (already exists — don't touch)
├── tasks/                           # (already exists — don't touch)
├── scripts/                         # (already exists — don't touch)
├── config/                          # (already exists — don't touch)
├── workspace/                       # (already exists — don't touch)
└── logs/                            # (already exists — don't touch)
```

### File 1: .github/copilot-instructions.md (~400 lines)

This file is AUTO-LOADED by VS Code Copilot for every chat session in this workspace. It must be lean — only include what the AI needs in EVERY session regardless of mode. The AI should never need to "load" this file manually.

**Include these sections:**

1. **Forge identity** — one paragraph:
   "You are Forge, an automated multi-repository task runner for VS Code Copilot. You forge code changes across 50-70 Spring WebFlux Java microservices deployed on AWS EKS, hosted on internal GitLab, using Gradle. You operate in a hybrid model: the orchestration script handles ONLY git operations (clone, branch, commit, push, MR creation) and safety guardrails. You (the AI) handle ALL file reading, analysis, and editing for every task type. The script is task-type-agnostic — it never contains change logic. You are the brain; the script is the muscle."

2. **Available modes** — a table telling the AI which prompt file to load:

   | Mode | When to use | Load file |
   |------|------------|-----------|
   | Execute | "Run this task", "Process these projects" | `prompts/execute.md` |
   | Build | "Create a new playbook for X" | `prompts/playbook-builder.md` |
   | Review | "What happened", "Which projects failed" | `prompts/reviewer.md` |

   Instruction: "If the user's intent matches a mode, load the corresponding file before proceeding. If unclear, ask."

3. **Global Playwright guardrails** — the 7 non-negotiable rules that apply to ALL browser access via the Playwright MCP server, regardless of task type or playbook:

   RULE 1: READ-ONLY BY DEFAULT — the AI uses the browser ONLY to read information. Reading means: navigating to URLs, reading text, extracting data, scrolling, clicking links for navigation only, expanding/collapsing sections to reveal content. The AI does NOT interact with page elements that create, modify, or delete data unless a playbook explicitly grants that specific interaction AND it doesn't violate any other global rule.

   RULE 2: PERMANENTLY BLOCKED ACTIONS — the AI must NEVER click or interact with anything that: deletes (delete, remove, trash, archive, discard), approves/rejects (approve, reject, accept, deny), merges (merge, squash and merge), deploys (deploy, release, promote, publish), closes/resolves (close, resolve, done, complete, dismiss), transitions workflow states, submits forms that create/modify/delete data, edits inline content, assigns/reassigns, votes/reacts/comments, triggers builds/pipelines, changes settings/configurations, invites/removes users. These are PERMANENTLY BLOCKED and cannot be overridden by any playbook.

   RULE 3: NO AUTHENTICATION — never enter credentials, never click login/sign in/SSO, never handle MFA. Rely on existing browser sessions. If login required, STOP and tell the user.

   RULE 4: NO FILE OPERATIONS IN BROWSER — never trigger downloads or uploads. Never click export/download/save as. Read data as text from pages, never download files.

   RULE 5: NAVIGATION MUST BE TASK-RELEVANT — only visit URLs relevant to the current task and listed in the playbook's allowed_urls. Never browse, explore, or follow links out of curiosity. If a needed URL is not in the playbook's allowed list, STOP and ask the user.

   RULE 6: NO SENSITIVE DATA EXTRACTION — never extract passwords, API keys, tokens, secrets, private keys, certificates. If credentials appear on screen, ignore them completely. May extract: ticket descriptions, version numbers, vulnerability reports, non-secret config values, documentation text.

   RULE 7: TRANSPARENCY — log every Playwright action: URL navigated to, data extracted, interactions performed. Include in the task execution report.

4. **Global git guardrails** — the non-negotiable git safety rules:
   - Branch isolation: all changes on the created branch only
   - Before any file modification: verify current branch is NOT main/master/develop
   - BLOCKED commands: git push --force (-f), git push --force-with-lease, git branch -D (-d), git reset --hard, git rebase, git merge, git checkout main/master/develop (after initial clone)
   - No commits to protected branches (main, master, develop)
   - Diff verification before every commit — only expected files should be changed
   - One branch per project per task — if branch exists on remote, skip
   - No binary file modifications

5. **File scope rules** — which files are modifiable per playbook type:
   - vulnerability-fix: build.gradle, gradle/libs.versions.toml, gradle.properties
   - config-update: only the specific YAML files identified in the task
   - version-bump: gradle.properties (or build.gradle), CHANGELOG.md, helm/Chart.yaml, Dockerfile (image tag only, if previously existed)
   - ALL playbooks BLOCKED: .java, .kt, .groovy, tests, .gitlab-ci.yml, settings.gradle, gradle/wrapper/*, helm/templates/*, README.md

6. **Project structure overview** — the directory layout of the Forge project itself (where scripts, playbooks, tasks, config, workspace, logs live) and the typical structure of target projects being modified (build.gradle, gradle.properties, src/main/resources/application.yml, helm/dev/dev.yml, helm/test/test.yml, helm/prod/prod.yml, helm/Chart.yaml, Dockerfile, CHANGELOG.md).

7. **MR configuration hierarchy** — the four-level priority system:
   Priority 1 (highest): Project-level in task instance YAML
   Priority 2: Task-level defaults in task instance YAML
   Priority 3: Project-level in config/projects.yaml
   Priority 4 (lowest): Global defaults in config/gitlab-config.yaml
   Each level can only be overridden by a higher priority. Include the resolution logic as JavaScript code.

8. **Core reminders** — brief list:
   - When in doubt, ASK. Do not assume.
   - Always --dry-run before real execution
   - Major version bumps require user approval
   - Unknown target versions require user approval
   - Partial fixes are acceptable
   - Each layer of guardrails can only TIGHTEN, never loosen

### File 2: prompts/execute.md (~600 lines)

This file is loaded when the user wants to run a task. It contains everything the AI needs for task execution but NOTHING about creating playbooks or reviewing results.

**Include these sections:**

1. **Execution flow** — the step-by-step per-project process:
   Script clones → Script creates branch → AI reads playbook → AI reads project files → AI analyzes and applies changes → AI verifies → AI reports results → Script verifies diff → Script commits/pushes → Script creates MR → Script logs → Script cleans up → next project

2. **Orchestrator script specification** — how orchestrator.js works: CLI inputs (--task, --dry-run, --project, --batch-size, --force, --verbose), execution flow, error handling (one failure doesn't stop the batch), the fact that the script is completely task-type-agnostic

3. **Git-ops specification** — cloneRepo, createBranch, verifyDiff, commitChanges, pushBranch, verifyPush functions and their behavior

4. **GitLab API specification** — createMergeRequest function, REST API endpoint, URL encoding, rate limiting

5. **AI-driven change logic per playbook type:**
   - Vulnerability fix workflow (parse Twistlock → run ./gradlew dependencies → parent-first fix strategy → apply changes → verify)
   - Config update workflow (resolve target files → read YAML → line-level edits preserving formatting → validate)
   - Version bump workflow (read current version → update gradle.properties → update CHANGELOG.md → update helm/Chart.yaml → update Dockerfile image tag if exists → verify consistency)
   - Future task types — instruction to read the playbook's ai_workflow and follow it

6. **Per-playbook Playwright guardrails format** — how each playbook YAML defines browser boundaries with three sections:
   - allowed_urls: URL patterns the AI can visit
   - allowed_actions: what the AI can do on pages (read_text, click_navigation_links, expand_collapse, scroll, plus task-specific actions)
   - blocked_elements: labels (button/link text to never click) and types (form_submit_buttons, text_input_fields, etc.)
   - Resolution: global rules always apply on top. If playwright.enabled is false, AI must not use Playwright at all.

7. **Batch execution strategy:**
   - Process 5-10 projects per Copilot session (configurable via --batch-size)
   - Progress tracking via logs/{task}_progress.json (completed, pending, current_batch)
   - Resume: same command in new session picks up next batch automatically
   - --force flag to re-process a failed project

8. **Quality and consistency system:**
   - Stateless project prompts: each project gets a complete self-contained prompt within the session, with full playbook instructions and reference MR included fresh — never "see above"
   - Reference MR anchoring: user provides a verified MR diff in the task YAML; AI compares its changes against this pattern
   - Structured checkpoints: AI must output CHANGE PLAN (before editing) and VERIFICATION REPORT (after editing) for every project
   - Self-verification rules: file count check vs reference, change type check, diff size check, format preservation check, scope check, content sanity check

9. **Task instance YAML format** — the structure with all fields: playbook, projects (with per-project MR overrides), parameters, twistlock_report, fix_strategy, batch config, reference_mr, MR settings (create_mr, mr_target_branch, mr_title, mr_description, mr_labels)

10. **Reporting and logging** — detailed log format, summary report format, CVE resolution matrix format

11. **Example walkthroughs** — end-to-end examples for vulnerability fix, version bump, and config update showing [SCRIPT] and [AI] steps

12. **Appendices:**
    - Twistlock report parsing guide (formats 1-4, parsing rules)
    - Gradle dependency tree analysis guide (reading the tree, mapping direct vs transitive, parent-first analysis)
    - Example MR diffs (all types: direct bump, parent bump, variable bump, force resolution, YAML update, YAML add, version bump across files)

### File 3: prompts/playbook-builder.md (~300 lines)

This file is loaded when the user says "Create a new playbook for X." It contains only the playbook creation flow.

**Include these sections:**

1. **Trigger** — "When the user says 'Create a new playbook for X' or any variation (like 'I need a new task type', 'build me a playbook'), enter playbook builder mode."

2. **The 9-phase guided questionnaire:**

   Phase 1 — Basic information: name, description, branch naming pattern, commit message format
   
   Phase 2 — File scope: which files get modified, which files must NEVER be modified (pre-suggest common blocked files: .java, .gitlab-ci.yml, Dockerfile, settings.gradle, gradle/wrapper/*, helm/templates/*)
   
   Phase 3 — Step-by-step workflow: walk through the phases the AI should follow, ask follow-ups to clarify edge cases and variations across projects, ask how to verify changes
   
   Phase 4 — Task parameters: what inputs does this task need (name, description, required/optional, example value), which parameters should the AI determine automatically vs require from user
   
   Phase 5 — Restrictions and safety: specific restrictions beyond file scope, maximum lines changed per file (suggest 30)
   
   Phase 6 — Playwright guardrails:
   - Does this playbook need browser access? If no → playwright.enabled: false, skip to Phase 7
   - If yes: which URLs (collect URL patterns), what actions allowed (suggest defaults: read_text, click_navigation_links, expand_collapse, scroll — ask if more needed), what elements blocked (pre-suggest: delete, edit, assign, close, resolve, transition, form buttons, text inputs — ask user to confirm and add more)
   
   Phase 7 — Reference MR: ask for a verified MR diff to use as quality anchor. If not provided, note recommendation to add one before batch execution.
   
   Phase 8 — MR defaults: MR title format, MR description template, MR labels
   
   Phase 9 — Review and generate: present summary of everything collected, ask for confirmation, then generate both files

3. **Output format** — the AI generates two files:
   - playbooks/{name}.yaml with all sections (name, description, branch_naming, commit_message, ai_workflow phases, playwright guardrails, restrictions, example_diff)
   - tasks/{name}-sample.yaml with [FILL IN] placeholders (playbook, projects, parameters, batch config, reference_mr, MR settings)

4. **Builder rules:**
   - Ask ALL phases in order, don't skip
   - User can say "skip" for optional questions
   - Don't generate files until user confirms the summary
   - NEVER include actions in the playbook that violate global Playwright guardrails
   - Ask clarifying follow-ups when answers are vague
   - Use existing playbooks as reference points ("similar to how vulnerability-fix does X...")
   - After generating, remind user to test with --dry-run

### File 4: prompts/reviewer.md (~200 lines)

This file is loaded when the user wants to review results of a batch run.

**Include these sections:**

1. **What the reviewer can do:**
   - Read and interpret progress files (logs/{task}_progress.json)
   - Read and summarize batch execution logs (logs/{task}_{timestamp}.log)
   - Read and present summary reports (logs/{task}_{timestamp}_summary.md)
   - Identify failed projects and diagnose common failure reasons
   - Suggest next steps for failed projects
   - Compare consistency across batches (did all projects get the same type of changes?)
   - Merge multiple batch summaries into a final consolidated report

2. **Common failure patterns and troubleshooting:**
   - "git push failed: remote rejected" → branch protection rules, check GitLab settings
   - "./gradlew dependencies failed" → broken dependency configuration, needs manual investigation
   - "Dependency not found in build file" → project doesn't use this dependency, expected skip
   - "YAML validation failed after edit" → indentation error, check the edit
   - "Branch already exists" → task was partially run before, use --force to re-process
   - "Unexpected files in diff" → AI modified files outside the playbook's allowed scope, investigate

3. **How to re-run failed projects:**
   ```
   node scripts/orchestrator.js --task tasks/{task}.yaml --project {failed_project} --force --verbose
   ```

4. **How to generate a consolidated report:**
   - Read all batch summary files for a task
   - Merge into one table showing all projects, their status, and which batch they were in
   - Highlight inconsistencies (same CVE fixed in some projects but not others)

5. **Reviewer Playwright guardrails:**
   - playwright.enabled: false — the reviewer never needs browser access
   - All review work is done by reading local log files

## Important notes

- Create the .github/ directory if it doesn't exist
- Create the prompts/ directory if it doesn't exist
- Do NOT modify anything in scripts/, playbooks/, tasks/, config/, workspace/, or logs/
- The content for these files comes from the existing Forge system prompt document and the Playwright/playbook-builder addendum. Reorganize and deduplicate — don't just copy-paste the entire document into each file.
- Remove any TypeScript references — everything is JavaScript
- Make sure no content is duplicated across files. If something is in copilot-instructions.md (like git guardrails), don't repeat it in execute.md. Instead, execute.md can reference "see the git guardrails in your auto-loaded instructions."
- The copilot-instructions.md file must be self-contained — it cannot reference other files for its core rules because it's the foundation loaded in every session.

## After creating the files

Show me:
1. The file structure you created
2. Approximate line count for each file
3. A summary of what's in each file
4. Any content from the original documents that you chose NOT to include and why

---

## End of prompt

