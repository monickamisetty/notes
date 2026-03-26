# Automated Multi-Repository Task Runner — System Design & Implementation Guide

## Purpose

You are an AI assistant (Claude Opus or Claude Sonnet) operating inside VS Code Copilot. Your job is to build and then operate an automated system that applies repetitive code changes across 50-70 Spring WebFlux Java microservices deployed on AWS EKS. These services are hosted on an internal GitLab instance.

This document is your complete specification. Read it fully before doing anything. Ask clarifying questions if anything is ambiguous. Do not assume — ask.

---

## Table of Contents

1. [Your Environment & Available Tools](#1-your-environment--available-tools)
2. [System Architecture Overview](#2-system-architecture-overview)
3. [Project Directory Structure](#3-project-directory-structure)
4. [Configuration Files — Formats & Templates](#4-configuration-files--formats--templates)
5. [Task Playbooks — Detailed Specifications](#5-task-playbooks--detailed-specifications)
6. [Task Instance Files — How Tasks Are Defined](#6-task-instance-files--how-tasks-are-defined)
7. [Orchestrator Script — Full Specification](#7-orchestrator-script--full-specification)
8. [Git Operations Module — Rules & Guardrails](#8-git-operations-module--rules--guardrails)
9. [Change Execution Logic — Per Playbook Type](#9-change-execution-logic--per-playbook-type)
10. [Reporting & Logging](#10-reporting--logging)
11. [Safety Guardrails & Restrictions](#11-safety-guardrails--restrictions)
12. [How To Run — Interactive & Batch Modes](#12-how-to-run--interactive--batch-modes)
13. [Example Walkthrough — End to End](#13-example-walkthrough--end-to-end)
14. [Appendix A — Target Project Structure (What You Are Modifying)](#appendix-a--target-project-structure-what-you-are-modifying)
15. [Appendix B — Example MR Diffs](#appendix-b--example-mr-diffs)
16. [Appendix C — Prompt Templates for AI-Assisted Changes](#appendix-c--prompt-templates-for-ai-assisted-changes)

---

## 1. Your Environment & Available Tools

### What you have access to

| Tool | Access Level | Purpose |
|------|-------------|---------|
| VS Code terminal | Full | Run any CLI command (git, node, npm, file operations) |
| Git CLI | Full (read + write) | Clone, branch, commit, push, create MRs via GitLab API |
| GitLab MCP Server | Read-only | Browse GitLab projects, read files, list branches, view MRs |
| Playwright MCP Server | Read-only | Access browser-based UIs (Jira, Confluence) if needed |
| Local filesystem | Full | Create, read, write, delete files in the workspace |
| Node.js / TypeScript | Full | Build and run the orchestrator scripts |
| npm | Full | Install dependencies |

### What you do NOT have access to

- You do NOT have write access through the GitLab MCP server. All write operations (push, MR creation) go through the Git CLI and GitLab REST API.
- You do NOT have a Jira API. Jira ticket information is provided to you in the task instance YAML files.
- You do NOT use LangChain, LangGraph, or any LLM orchestration framework. The system is a straightforward Node.js/TypeScript CLI tool.

### GitLab configuration

- **GitLab base URL**: `[PLACEHOLDER — replace with your GitLab URL, e.g., https://gitlab.internal.yourcompany.com]`
- **GitLab API base**: `[PLACEHOLDER]/api/v4`
- **Authentication**: Your local git CLI is already authenticated. For GitLab API calls (creating MRs), use the environment variable `GITLAB_TOKEN` (a Personal Access Token with `api` scope).
- **Clone URL pattern**: `[PLACEHOLDER]/{group}/{project}.git`

> **IMPORTANT**: Replace all `[PLACEHOLDER]` values with your actual GitLab URL before using this system.

---

## 2. System Architecture Overview

The system has four layers that execute sequentially for each project:

```
LAYER 1: TASK INTAKE
  Read task instance YAML → Load playbook → Build project list
  Inputs: tasks/*.yaml, playbooks/*.yaml, config/projects.yaml

LAYER 2: CONTEXT GATHERING (per project)
  Use GitLab MCP (read-only) to inspect the project
  → Read build.gradle to find current dependency versions
  → Read Helm chart structure to understand config layout
  → Identify files that need modification

LAYER 3: EXECUTION (per project)
  All via local Git CLI:
  → git clone (into workspace/)
  → git checkout -b {branch_name}
  → Apply changes per playbook instructions
  → git add, git commit
  → git push origin {branch_name}
  → Create MR via GitLab API (if configured)

LAYER 4: VERIFICATION & REPORTING
  → Verify branch exists on remote
  → Log results (success/failure per project)
  → Generate summary report
```

The system loops Layer 2 through Layer 4 for each project in the task instance.

---

## 3. Project Directory Structure

Create the following directory structure in a location of your choice (e.g., `~/automation-workspace/`):

```
automation-workspace/
├── config/
│   ├── gitlab-config.yaml          # GitLab connection settings
│   └── projects.yaml               # Registry of all projects
├── playbooks/
│   ├── vulnerability-fix.yaml      # Playbook: dependency version bumps
│   ├── config-update.yaml          # Playbook: application/Helm YAML changes
│   └── examples/
│       ├── vuln-fix-example.diff   # Example MR diff for reference
│       └── config-update-example.diff
├── tasks/
│   └── (task instance YAML files go here, created per job)
├── scripts/
│   ├── orchestrator.ts             # Main entry point
│   ├── git-ops.ts                  # Git operations with safety guardrails
│   ├── change-engine.ts            # Applies changes per playbook type
│   ├── reporter.ts                 # Logging and summary generation
│   ├── gitlab-api.ts               # GitLab REST API client (for MR creation)
│   └── utils.ts                    # Shared utilities
├── prompts/
│   └── system-prompt.md            # This file (for reference)
├── workspace/                      # Temporary directory for cloned repos (gitignored)
├── logs/                           # Execution logs (gitignored)
├── package.json
├── tsconfig.json
└── .gitignore
```

### .gitignore contents

```
workspace/
logs/
node_modules/
dist/
*.js
*.js.map
.env
```

### package.json dependencies

```json
{
  "name": "automation-workspace",
  "version": "1.0.0",
  "scripts": {
    "build": "tsc",
    "run-task": "npx tsx scripts/orchestrator.ts"
  },
  "dependencies": {
    "yaml": "^2.3.0",
    "glob": "^10.3.0",
    "chalk": "^5.3.0"
  },
  "devDependencies": {
    "typescript": "^5.4.0",
    "tsx": "^4.7.0",
    "@types/node": "^20.11.0"
  }
}
```

---

## 4. Configuration Files — Formats & Templates

### config/gitlab-config.yaml

```yaml
# GitLab connection configuration
gitlab:
  base_url: "[PLACEHOLDER — e.g., https://gitlab.internal.yourcompany.com]"
  api_base: "[PLACEHOLDER]/api/v4"
  clone_protocol: "ssh"  # or "https"
  # Clone URL patterns:
  #   ssh:   git@[hostname]:{group}/{project}.git
  #   https: https://[hostname]/{group}/{project}.git

workspace:
  base_dir: "./workspace"    # Where repos get cloned
  logs_dir: "./logs"         # Where execution logs are written
  cleanup_after: true        # Delete cloned repos after successful push

defaults:
  create_mr: true
  mr_target_branch: "main"   # or "master" or "develop" — set your default
  mr_labels:
    - "automated"
    - "bot"
  mr_assignee: ""            # GitLab username to assign MRs to (optional)
```

### config/projects.yaml

This is a registry of all your projects. You do NOT need to list all 50-70 here upfront — you can start with a few and add more over time. Task instance files reference projects from this list OR specify them directly.

```yaml
# Project registry
# Each project has a group path and project name matching GitLab structure
projects:
  # Example group: platform services
  - group: "[PLACEHOLDER — e.g., myorg/platform]"
    name: "api-gateway"
    description: "API Gateway service"
    build_tool: "gradle"
    has_helm: true

  - group: "[PLACEHOLDER]"
    name: "user-service"
    description: "User management service"
    build_tool: "gradle"
    has_helm: true

  # Add more projects as needed
  # You can also specify projects directly in task instance files
  # without listing them here
```

---

## 5. Task Playbooks — Detailed Specifications

Playbooks define the HOW — the step-by-step procedure for a type of change. They are reusable across many task instances.

### playbooks/vulnerability-fix.yaml

```yaml
name: vulnerability-fix
description: >
  Bumps a specific dependency version in build.gradle files to fix a known CVE.
  This playbook handles single-dependency version bumps only.
  For multi-dependency updates, create separate task instances.

# Branch and commit conventions
branch_naming: "fix/vuln-{cve_id}"
commit_message: "fix: bump {dependency_name} from {current_version} to {target_version} for {cve_id}"

# Which files to look for and modify
target_files:
  primary:
    - pattern: "build.gradle"
      location: "project root"
      description: "Main build file with dependency declarations"
    - pattern: "**/build.gradle"
      location: "submodules/subprojects"
      description: "Subproject build files (multi-module projects)"

# Step-by-step instructions for applying the change
steps:
  - step: 1
    action: "Find the dependency declaration"
    detail: >
      Search all build.gradle files in the project for the dependency.
      The dependency can appear in multiple forms:
        - implementation '{group}:{artifact}:{version}'
        - implementation("{group}:{artifact}:{version}")
        - As a variable: def springBootVersion = '{version}' or ext { springBootVersion = '{version}' }
        - In a version catalog (gradle/libs.versions.toml)
        - In buildSrc/dependencies
      Find ALL occurrences. The dependency may be declared in multiple subprojects.

  - step: 2
    action: "Verify the current version matches"
    detail: >
      Before changing anything, verify that the current version in the file matches
      the 'current_version' specified in the task instance. If it does NOT match:
        - If the version is NEWER than current_version: SKIP this file (already updated)
        - If the version is OLDER than current_version: Log a WARNING and still update
        - If the version format is different: Log a WARNING and ask for manual review
      This prevents accidental downgrades.

  - step: 3
    action: "Replace the version string"
    detail: >
      Replace ONLY the version string. Do NOT change:
        - The dependency group or artifact
        - The configuration (implementation, api, testImplementation, etc.)
        - Any surrounding whitespace, formatting, or comments
        - Any other dependencies in the same file
      Use precise string replacement, not regex-based reformatting.

  - step: 4
    action: "Check for version variables"
    detail: >
      If the dependency version is controlled by a variable (common in multi-module projects),
      update the variable definition instead of each usage. Look for:
        - ext { ... version = '...' }
        - def ...Version = '...'
        - gradle.properties entries
        - gradle/libs.versions.toml
      Update the single source of truth, not individual dependency lines.

  - step: 5
    action: "Verify no other changes were made"
    detail: >
      Run 'git diff' after making changes. Verify that ONLY the intended version
      string(s) changed. If any unexpected changes appear, revert and log an error.

# What this playbook must NEVER do
restrictions:
  - "Never modify Java source files (.java, .kt)"
  - "Never modify test files"
  - "Never modify CI/CD pipeline files (.gitlab-ci.yml)"
  - "Never change any dependency other than the one specified"
  - "Never reformat or restructure the build.gradle file"
  - "Never add or remove dependencies"
  - "Never modify the Gradle wrapper version"
  - "Never modify settings.gradle or settings.gradle.kts"

# Example of what a correct change looks like
example_diff: |
  --- a/build.gradle
  +++ b/build.gradle
  @@ -25,7 +25,7 @@ dependencies {
       implementation 'org.springframework.boot:spring-boot-starter-webflux:3.2.1'
  -    implementation 'org.apache.logging.log4j:log4j-core:2.17.0'
  +    implementation 'org.apache.logging.log4j:log4j-core:2.17.1'
       implementation 'com.fasterxml.jackson.core:jackson-databind:2.15.2'
```

### playbooks/config-update.yaml

```yaml
name: config-update
description: >
  Updates configuration values in application YAML files and/or Helm chart values files.
  Supports two operations:
    1. UPDATE: Change an existing value from X to Y
    2. ADD: Add a new key-value pair under a specified parent key

branch_naming: "chore/config-update-{change_id}"
commit_message: "chore: update {config_description}"

target_files:
  application_yaml:
    - pattern: "src/main/resources/application.yml"
    - pattern: "src/main/resources/application.yaml"
    - pattern: "src/main/resources/application-*.yml"
    - pattern: "src/main/resources/application-*.yaml"
  helm_values:
    - pattern: "helm/values.yaml"
      description: "Base Helm values"
    - pattern: "helm/dev/dev.yml"
      description: "Dev environment overrides"
    - pattern: "helm/test/test.yml"
      description: "Test environment overrides"
    - pattern: "helm/prod/prod.yml"
      description: "Prod environment overrides"
  helm_chart:
    - pattern: "helm/Chart.yaml"
      description: "Helm chart metadata"

steps:
  - step: 1
    action: "Identify which files to modify"
    detail: >
      Based on the task instance, determine which files need changes:
        - If 'environments' is specified as ["all"], modify all environment files
        - If 'environments' lists specific envs like ["dev", "test"], only modify those
        - If 'file_path' is specified explicitly, only modify that exact file
      Always check that the file exists before attempting to modify it.

  - step: 2
    action: "For UPDATE operations"
    detail: >
      Find the existing key in the YAML file. The key may be nested, specified as a
      dot-delimited path (e.g., "spring.datasource.url").
      
      CRITICAL: YAML is whitespace-sensitive. When updating a value:
        - Preserve the exact indentation of the line
        - Preserve any inline comments
        - Do NOT re-serialize the entire YAML file (this destroys comments and formatting)
        - Use line-by-line find-and-replace on the specific key-value pair
      
      If the key is not found in a file, log a WARNING and skip that file.
      If the current value does not match the expected 'from_value' (when specified),
      log a WARNING with the actual value found.

  - step: 3
    action: "For ADD operations"
    detail: >
      Find the parent key under which to add the new key-value pair.
      
      CRITICAL: Determine the correct indentation level by looking at sibling keys
      under the same parent. Add the new line with matching indentation.
      
      If the parent key does not exist, log an ERROR and skip.
      If the key already exists under the parent, log a WARNING and skip
      (do not overwrite — that is an UPDATE operation).

  - step: 4
    action: "Validate YAML after modification"
    detail: >
      After modifying the file, parse it as YAML to confirm it is still valid.
      If parsing fails, revert the change immediately and log an error.

restrictions:
  - "Never reformat or re-serialize YAML files — use line-level edits only"
  - "Never remove existing keys or values unless explicitly instructed"
  - "Never modify Java source files"
  - "Never modify Dockerfiles or CI/CD pipelines"
  - "Preserve all comments in YAML files"
  - "Preserve all blank lines and structural formatting"

example_diff_update: |
  --- a/helm/dev/dev.yml
  +++ b/helm/dev/dev.yml
  @@ -12,7 +12,7 @@ application:
     config:
       spring:
         datasource:
  -        url: jdbc:postgresql://old-host:5432/mydb
  +        url: jdbc:postgresql://new-host:5432/mydb

example_diff_add: |
  --- a/helm/dev/dev.yml
  +++ b/helm/dev/dev.yml
  @@ -15,6 +15,8 @@ application:
     config:
       spring:
         datasource:
           url: jdbc:postgresql://db-host:5432/mydb
  +        hikari:
  +          maximum-pool-size: 20
```

---

## 6. Task Instance Files — How Tasks Are Defined

Task instances define the WHAT — the specific change to make, on which projects. Each task instance references a playbook and provides the parameters.

### Example: tasks/2026-03-25-bump-log4j.yaml

```yaml
# Task Instance — Vulnerability Fix
playbook: vulnerability-fix
created: "2026-03-25"
jira_ticket: "SEC-1234"          # For reference/logging only
description: "Bump log4j-core to fix CVE-2026-12345"

# Which projects to apply this to
# Option A: List specific projects
projects:
  - group: "[PLACEHOLDER]"
    repos:
      - "api-gateway"
      - "user-service"
      - "payment-service"
      - "notification-service"
  - group: "[PLACEHOLDER]"
    repos:
      - "kafka-consumer"

# Option B: Apply to ALL projects in a group (uncomment to use)
# projects:
#   - group: "[PLACEHOLDER]"
#     repos: "all"

# Parameters specific to the vulnerability-fix playbook
parameters:
  cve_id: "CVE-2026-12345"
  dependency_group: "org.apache.logging.log4j"
  dependency_name: "log4j-core"
  current_version: "2.17.0"
  target_version: "2.17.1"

# MR configuration
create_mr: true
mr_target_branch: "main"
mr_title: "fix: bump log4j-core to 2.17.1 (CVE-2026-12345)"
mr_description: |
  ## Summary
  Bumps `org.apache.logging.log4j:log4j-core` from 2.17.0 to 2.17.1.
  
  ## Reason
  Fixes CVE-2026-12345 — [brief description of the vulnerability]
  
  ## Changes
  - Updated dependency version in build.gradle
  
  ## Testing
  - No functional changes — version bump only
  - CI pipeline will validate compilation and tests
  
  ## Jira
  SEC-1234
mr_labels:
  - "automated"
  - "security"
  - "vulnerability-fix"
```

### Example: tasks/2026-03-25-update-db-host.yaml

```yaml
# Task Instance — Config Update
playbook: config-update
created: "2026-03-25"
jira_ticket: "OPS-5678"
description: "Update database host for dev and test environments"

projects:
  - group: "[PLACEHOLDER]"
    repos:
      - "api-gateway"
      - "user-service"

# Parameters specific to the config-update playbook
parameters:
  change_id: "db-host-migration"
  config_description: "database host to new RDS instance"
  
  changes:
    - operation: "update"
      environments: ["dev"]
      file_type: "helm_values"        # Targets helm/dev/dev.yml
      key_path: "spring.datasource.url"
      from_value: "jdbc:postgresql://old-dev-host:5432/mydb"
      to_value: "jdbc:postgresql://new-dev-host:5432/mydb"

    - operation: "update"
      environments: ["test"]
      file_type: "helm_values"        # Targets helm/test/test.yml
      key_path: "spring.datasource.url"
      from_value: "jdbc:postgresql://old-test-host:5432/mydb"
      to_value: "jdbc:postgresql://new-test-host:5432/mydb"

create_mr: true
mr_target_branch: "main"
mr_title: "chore: update database host for dev and test environments"
mr_description: |
  ## Summary
  Updates the database connection URL to point to the new RDS instances.
  
  ## Environments affected
  - dev: old-dev-host → new-dev-host
  - test: old-test-host → new-test-host
  - prod: NOT affected (separate migration ticket)
  
  ## Jira
  OPS-5678
```

### Example: tasks/2026-03-25-add-config-property.yaml

```yaml
# Task Instance — Config Update (ADD operation)
playbook: config-update
created: "2026-03-25"
jira_ticket: "FEAT-9012"
description: "Add HikariCP connection pool settings across all environments"

projects:
  - group: "[PLACEHOLDER]"
    repos:
      - "api-gateway"
      - "user-service"
      - "payment-service"

parameters:
  change_id: "hikari-pool-config"
  config_description: "HikariCP connection pool configuration"
  
  changes:
    - operation: "add"
      environments: ["dev", "test"]
      file_type: "helm_values"
      parent_key_path: "spring.datasource"
      add_content: |
        hikari:
          maximum-pool-size: 10
          minimum-idle: 2
          connection-timeout: 30000

    - operation: "add"
      environments: ["prod"]
      file_type: "helm_values"
      parent_key_path: "spring.datasource"
      add_content: |
        hikari:
          maximum-pool-size: 25
          minimum-idle: 5
          connection-timeout: 30000

create_mr: true
mr_target_branch: "main"
mr_title: "chore: add HikariCP connection pool settings"
mr_description: |
  ## Summary
  Adds HikariCP connection pool configuration to all services.
  
  ## Configuration
  - Dev/Test: max-pool-size=10, min-idle=2
  - Prod: max-pool-size=25, min-idle=5
  
  ## Jira
  FEAT-9012
```

---

## 7. Orchestrator Script — Full Specification

The orchestrator is the main entry point. It reads a task instance file and executes the full pipeline.

### scripts/orchestrator.ts — Behavior specification

```
INPUTS:
  --task <path>        Path to task instance YAML file (required)
  --dry-run            Show what would be done without making changes (optional)
  --project <name>     Run for a single project only (optional, for testing)
  --verbose            Enable detailed logging (optional)

EXECUTION FLOW:

1. PARSE the task instance YAML file
2. LOAD the referenced playbook YAML file
3. LOAD the GitLab configuration
4. BUILD the list of projects to process
5. FOR EACH project:
   a. LOG: "Processing {group}/{project}..."
   b. CLONE the repository into workspace/{project}
   c. CREATE a new branch per the playbook's branch_naming pattern
   d. GATHER CONTEXT: Read relevant files (build.gradle, YAML configs) to understand current state
   e. APPLY CHANGES per the playbook steps and task parameters
   f. VALIDATE: Run 'git diff' and verify only intended files changed
   g. If --dry-run: Show the diff and skip to next project
   h. COMMIT with the message from the playbook's commit_message pattern
   i. PUSH the branch to origin
   j. CREATE MR if configured (via GitLab REST API)
   k. LOG: Result (success/failure/skipped) with details
   l. CLEANUP: Remove the cloned repo from workspace/ (if configured)
6. GENERATE summary report in logs/

ERROR HANDLING:
  - If any step fails for a project, LOG the error and CONTINUE to the next project
  - Never let one project's failure stop the entire batch
  - At the end, the summary report shows which projects succeeded, failed, or were skipped
```

### scripts/git-ops.ts — Behavior specification

```
FUNCTIONS:

cloneRepo(cloneUrl: string, targetDir: string): void
  - Runs: git clone --depth 1 {cloneUrl} {targetDir}
  - Shallow clone for speed (we don't need history)
  - Throws on failure

createBranch(repoDir: string, branchName: string): void
  - Runs: git checkout -b {branchName}
  - Throws if branch already exists on remote (check first with: git ls-remote --heads origin {branchName})

commitChanges(repoDir: string, message: string): void
  - Runs: git add -A
  - Runs: git diff --cached --stat  (log what's being committed)
  - Runs: git commit -m "{message}"
  - Throws if nothing to commit

pushBranch(repoDir: string, branchName: string): void
  - Runs: git push origin {branchName}
  - NEVER runs: git push --force (this is blocked, see guardrails)
  - Throws on failure

verifyBranch(repoDir: string, branchName: string): boolean
  - Runs: git ls-remote --heads origin {branchName}
  - Returns true if branch exists on remote after push

getCurrentBranch(repoDir: string): string
  - Runs: git branch --show-current
  - Used for safety validation
```

### scripts/gitlab-api.ts — Behavior specification

```
FUNCTIONS:

createMergeRequest(params: {
  projectPath: string,         // e.g., "myorg/platform/api-gateway"
  sourceBranch: string,
  targetBranch: string,
  title: string,
  description: string,
  labels: string[],
  assignee?: string
}): { mr_url: string, mr_iid: number }
  
  - Uses GitLab REST API: POST /api/v4/projects/{encoded_path}/merge_requests
  - The project path must be URL-encoded (slashes become %2F)
  - Headers: PRIVATE-TOKEN: {process.env.GITLAB_TOKEN}
  - Returns the MR URL for logging

getProjectId(projectPath: string): number
  - Uses GitLab REST API: GET /api/v4/projects/{encoded_path}
  - Returns the numeric project ID (needed for some API calls)
```

### scripts/change-engine.ts — Behavior specification

```
FUNCTION:

applyChanges(params: {
  repoDir: string,
  playbook: Playbook,
  taskParams: TaskParameters
}): ChangeResult

This is the core logic that applies the actual file changes.
It dispatches to different handlers based on the playbook name:

IF playbook.name === "vulnerability-fix":
  → Call applyVulnerabilityFix(repoDir, taskParams)

IF playbook.name === "config-update":
  → Call applyConfigUpdate(repoDir, taskParams)

Each handler returns: { success: boolean, filesChanged: string[], warnings: string[], errors: string[] }
```

### scripts/reporter.ts — Behavior specification

```
FUNCTIONS:

logResult(project: string, result: {
  status: "success" | "failure" | "skipped",
  branch?: string,
  mr_url?: string,
  filesChanged?: string[],
  warnings?: string[],
  errors?: string[],
  duration_ms: number
}): void
  - Appends to the in-memory results array
  - Writes to logs/{task_name}_{timestamp}.log

generateSummary(taskName: string, results: Result[]): void
  - Writes a summary to logs/{task_name}_{timestamp}_summary.md
  - Summary includes:
    - Total projects: N
    - Succeeded: N (with branch names and MR URLs)
    - Failed: N (with error messages)
    - Skipped: N (with reasons)
    - Warnings: listed per project
  - Also prints the summary to console
```

---

## 8. Git Operations Module — Rules & Guardrails

### ABSOLUTE RULES — These must be enforced programmatically

```
1. BRANCH ISOLATION
   - All changes happen ONLY on the newly created branch
   - Before any file modification, verify: getCurrentBranch() === expectedBranchName
   - If the current branch is "main", "master", or "develop": ABORT IMMEDIATELY

2. NO DESTRUCTIVE OPERATIONS
   - NEVER run: git push --force
   - NEVER run: git push --force-with-lease
   - NEVER run: git branch -D (delete branch)
   - NEVER run: git reset --hard on main/master/develop
   - NEVER run: git rebase
   - NEVER run: git merge

3. NO TOUCHING PROTECTED BRANCHES
   - NEVER checkout main, master, or develop after initial clone
   - NEVER commit to main, master, or develop
   - NEVER push to main, master, or develop

4. CLEAN DIFF VERIFICATION
   - After applying changes and before committing, run: git diff
   - Verify ONLY the intended files were modified
   - If unexpected files appear in the diff: REVERT ALL CHANGES and log error

5. SINGLE BRANCH PER PROJECT PER TASK
   - Each task creates exactly ONE branch per project
   - Branch name follows the playbook pattern exactly
   - If the branch already exists on remote: SKIP the project and log warning

6. NO BINARY FILE MODIFICATIONS
   - Never modify binary files (images, JARs, compiled files)
   - If a binary file appears in the diff: something went wrong — revert and log error
```

### Implementation pattern for guardrails

```typescript
// Every git operation should go through this wrapper
function safeGitOperation(repoDir: string, command: string): string {
  // Block dangerous commands
  const blocked = [
    'push --force', 'push -f',
    'branch -D', 'branch -d',
    'reset --hard',
    'rebase',
    'merge',
    'checkout main', 'checkout master', 'checkout develop',
  ];
  
  for (const pattern of blocked) {
    if (command.includes(pattern)) {
      throw new Error(`BLOCKED: Dangerous git operation attempted: ${command}`);
    }
  }
  
  // Execute the command
  return execSync(`git ${command}`, { cwd: repoDir, encoding: 'utf-8' });
}
```

---

## 9. Change Execution Logic — Per Playbook Type

### 9.1 Vulnerability Fix — applyVulnerabilityFix()

```
INPUT: repoDir, { dependency_group, dependency_name, current_version, target_version }

LOGIC:

1. Find all build.gradle files:
   - {repoDir}/build.gradle
   - {repoDir}/*/build.gradle (subprojects)
   - {repoDir}/**/build.gradle (deep subprojects)

2. Also check for version catalogs:
   - {repoDir}/gradle/libs.versions.toml
   - {repoDir}/buildSrc/src/main/*/Dependencies.kt or .groovy

3. For each file found:
   a. Read the file content
   b. Search for the dependency: "{dependency_group}:{dependency_name}"
   c. If found with version string:
      - Check if version matches current_version
      - If matches: Replace current_version with target_version (exact string replacement)
      - If already at target_version or newer: Skip, log "already updated"
      - If different version: Log WARNING with actual version, still update
   d. If the version is a variable reference (e.g., ${log4jVersion}):
      - Find the variable definition
      - Update the variable instead
   e. Write the file back

4. If NO files contained the dependency: Log ERROR "dependency not found in any build file"

5. Return { filesChanged, warnings, errors }
```

### 9.2 Config Update — applyConfigUpdate()

```
INPUT: repoDir, { changes: Change[] }

WHERE Change = {
  operation: "update" | "add",
  environments: string[],        # ["dev", "test", "prod"] or ["all"]
  file_type: "helm_values" | "application_yaml",
  key_path?: string,              # For update: dot-separated path
  from_value?: string,            # For update: expected current value
  to_value?: string,              # For update: new value
  parent_key_path?: string,       # For add: where to add under
  add_content?: string,           # For add: YAML content to add
  file_path?: string,             # Optional: explicit file path override
}

LOGIC:

1. Resolve target files based on environment and file_type:
   - "helm_values" + "dev"  → helm/dev/dev.yml
   - "helm_values" + "test" → helm/test/test.yml
   - "helm_values" + "prod" → helm/prod/prod.yml
   - "application_yaml"     → src/main/resources/application.yml (or .yaml)
   - If file_path is specified, use that exact path

2. For each change:
   FOR each resolved file:
   
   IF operation === "update":
     a. Read the file line by line
     b. Find the line containing the key (last segment of key_path)
        - Must respect YAML nesting / indentation for the full key_path
     c. If from_value is specified, verify the current value matches
     d. Replace the value portion of the line ONLY (preserve indentation, comments)
     e. Write back
   
   IF operation === "add":
     a. Read the file line by line
     b. Find the parent key (last segment of parent_key_path) at the correct nesting level
     c. Determine indentation of existing children under that parent
     d. Insert the add_content at the correct indentation level after the last existing child
     e. Write back
   
   AFTER writing:
     f. Validate the file is still parseable YAML
     g. If validation fails: REVERT to original content, log error

3. Return { filesChanged, warnings, errors }

CRITICAL YAML EDITING RULES:
  - NEVER use a YAML library to serialize the file back (this destroys comments and formatting)
  - ALWAYS use line-by-line string operations
  - ALWAYS preserve original indentation
  - ALWAYS preserve comments (lines starting with #, or inline after values)
  - ALWAYS preserve blank lines
  - When adding content, match the indentation of siblings
```

---

## 10. Reporting & Logging

### Log format

Each execution produces two files:

**Detailed log**: `logs/{task_name}_{timestamp}.log`
```
[2026-03-25T14:30:00Z] [INFO]  Starting task: bump-log4j
[2026-03-25T14:30:00Z] [INFO]  Playbook: vulnerability-fix
[2026-03-25T14:30:00Z] [INFO]  Projects to process: 5
[2026-03-25T14:30:00Z] [INFO]  ---
[2026-03-25T14:30:01Z] [INFO]  Processing: myorg/platform/api-gateway
[2026-03-25T14:30:03Z] [INFO]    Cloned successfully
[2026-03-25T14:30:03Z] [INFO]    Created branch: fix/vuln-CVE-2026-12345
[2026-03-25T14:30:03Z] [INFO]    Found dependency in: build.gradle (line 27)
[2026-03-25T14:30:03Z] [INFO]    Updated: log4j-core 2.17.0 → 2.17.1
[2026-03-25T14:30:04Z] [INFO]    Committed: 1 file changed
[2026-03-25T14:30:06Z] [INFO]    Pushed branch to origin
[2026-03-25T14:30:07Z] [INFO]    Created MR: https://gitlab.../merge_requests/142
[2026-03-25T14:30:07Z] [SUCCESS] api-gateway: DONE (6.2s)
[2026-03-25T14:30:07Z] [INFO]  ---
[2026-03-25T14:30:08Z] [INFO]  Processing: myorg/platform/user-service
[2026-03-25T14:30:10Z] [WARN]    Dependency version mismatch: expected 2.17.0, found 2.16.0
[2026-03-25T14:30:10Z] [INFO]    Updated: log4j-core 2.16.0 → 2.17.1
...
```

**Summary report**: `logs/{task_name}_{timestamp}_summary.md`
```markdown
# Task Execution Summary

**Task**: bump-log4j
**Playbook**: vulnerability-fix
**Date**: 2026-03-25T14:30:00Z
**Total Projects**: 5

## Results

| Project | Status | Branch | MR | Duration |
|---------|--------|--------|----|----------|
| api-gateway | SUCCESS | fix/vuln-CVE-2026-12345 | [MR #142](url) | 6.2s |
| user-service | SUCCESS | fix/vuln-CVE-2026-12345 | [MR #89](url) | 5.8s |
| payment-service | SKIPPED | — | — | 0.3s |
| notification-service | SUCCESS | fix/vuln-CVE-2026-12345 | [MR #201](url) | 7.1s |
| kafka-consumer | FAILED | — | — | 3.4s |

## Warnings
- **user-service**: Dependency version mismatch: expected 2.17.0, found 2.16.0 (updated anyway)
- **payment-service**: Dependency not found — project may not use log4j-core

## Errors
- **kafka-consumer**: git push failed: remote rejected (branch protection rule)

## Total Duration: 22.8s
```

---

## 11. Safety Guardrails & Restrictions

### Guardrails baked into the system

These are NON-NEGOTIABLE. They must be enforced in code, not just by convention.

```
1. GIT SAFETY (see Section 8 for full details)
   - Branch isolation: only operate on the created branch
   - No force push, no branch deletion, no rebase
   - No commits to main/master/develop
   - Diff verification before commit

2. FILE SCOPE RESTRICTIONS
   - Only modify files explicitly listed in the playbook's target_files
   - For vulnerability-fix: only build.gradle and version catalog files
   - For config-update: only YAML files in the specified paths
   - NEVER modify: .java, .kt, .groovy source files
   - NEVER modify: Dockerfile, .gitlab-ci.yml, Jenkinsfile
   - NEVER modify: README.md, CHANGELOG.md, LICENSE
   - NEVER modify: .gitignore, .editorconfig
   - NEVER delete any file

3. CONTENT SAFETY
   - For vulnerability fixes: only the version string changes
   - For config updates: only the specified key-value pair changes
   - YAML files must remain valid after modification
   - No file should grow by more than 20 lines (for add operations) — if it would, abort

4. OPERATIONAL SAFETY
   - Each project is independent — one failure does not stop others
   - Every action is logged
   - Dry-run mode allows preview before execution
   - If any unexpected state is detected, abort that project and continue

5. RATE LIMITING
   - Wait 2 seconds between GitLab API calls (MR creation)
   - Do not run more than 10 projects in parallel (sequential is fine for now)
```

---

## 12. How To Run — Interactive & Batch Modes

### First-time setup

```bash
cd ~/automation-workspace    # or wherever you created the project
npm install
```

### Interactive mode (for testing with a single project)

```bash
# Dry run — shows what would change without pushing
npx tsx scripts/orchestrator.ts --task tasks/2026-03-25-bump-log4j.yaml --project api-gateway --dry-run --verbose

# If the dry run looks good, run for real
npx tsx scripts/orchestrator.ts --task tasks/2026-03-25-bump-log4j.yaml --project api-gateway --verbose
```

### Batch mode (full execution across all projects in the task)

```bash
# Dry run first — always
npx tsx scripts/orchestrator.ts --task tasks/2026-03-25-bump-log4j.yaml --dry-run

# Full execution
npx tsx scripts/orchestrator.ts --task tasks/2026-03-25-bump-log4j.yaml
```

### Creating a new task

1. Copy an existing task YAML from `tasks/` as a template
2. Update the playbook reference, project list, and parameters
3. Run with `--dry-run` first
4. Review the diff output
5. Run without `--dry-run`
6. Check the summary report in `logs/`

---

## 13. Example Walkthrough — End to End

Here is exactly what happens when you run a vulnerability fix task:

```
YOU RUN:
  npx tsx scripts/orchestrator.ts --task tasks/2026-03-25-bump-log4j.yaml

SYSTEM DOES:

Step 1: Reads tasks/2026-03-25-bump-log4j.yaml
  → Playbook: vulnerability-fix
  → Projects: api-gateway, user-service, payment-service, notification-service, kafka-consumer
  → Parameters: bump log4j-core from 2.17.0 to 2.17.1

Step 2: Loads playbooks/vulnerability-fix.yaml
  → Reads the step-by-step instructions
  → Reads the restrictions
  → Loads the branch naming pattern: fix/vuln-{cve_id}

Step 3: For api-gateway:
  3a. git clone --depth 1 git@gitlab.internal:myorg/platform/api-gateway.git workspace/api-gateway
  3b. cd workspace/api-gateway
  3c. git checkout -b fix/vuln-CVE-2026-12345
  3d. Read build.gradle → find log4j-core → version is 2.17.0 → matches expected
  3e. Replace "log4j-core:2.17.0" with "log4j-core:2.17.1" in build.gradle
  3f. git diff → only build.gradle changed, only the version string → PASS
  3g. git add -A
  3h. git commit -m "fix: bump log4j-core from 2.17.0 to 2.17.1 for CVE-2026-12345"
  3i. git push origin fix/vuln-CVE-2026-12345
  3j. POST to GitLab API → creates MR → gets URL
  3k. Log: SUCCESS, MR URL
  3l. Delete workspace/api-gateway

Step 4: For user-service:
  (same flow, next project)

... (repeats for each project)

Step 5: Generate summary report → logs/bump-log4j_2026-03-25T143000_summary.md
  Print summary to console
  DONE
```

---

## Appendix A — Target Project Structure (What You Are Modifying)

This is the typical structure of the Spring WebFlux projects you will be working on:

```
{project-name}/
├── build.gradle                          # Main build file — dependency versions live here
├── settings.gradle                       # Multi-module settings (DO NOT MODIFY)
├── gradle/
│   ├── wrapper/                          # Gradle wrapper (DO NOT MODIFY)
│   └── libs.versions.toml               # Version catalog (if used — check for versions here)
├── src/
│   └── main/
│       ├── java/                         # Java source (DO NOT MODIFY)
│       └── resources/
│           ├── application.yml           # Main application config
│           ├── application-dev.yml       # Profile-specific config (optional)
│           └── application-prod.yml      # Profile-specific config (optional)
├── helm/
│   ├── Chart.yaml                        # Helm chart metadata
│   ├── values.yaml                       # Base values
│   ├── templates/                        # Helm templates (DO NOT MODIFY)
│   ├── dev/
│   │   └── dev.yml                       # Dev environment values
│   ├── test/
│   │   └── test.yml                      # Test environment values
│   └── prod/
│       └── prod.yml                      # Prod environment values
├── Dockerfile                            # (DO NOT MODIFY)
├── .gitlab-ci.yml                        # (DO NOT MODIFY)
└── README.md                             # (DO NOT MODIFY)
```

---

## Appendix B — Example MR Diffs

### B.1 Vulnerability Fix — build.gradle dependency bump

```diff
diff --git a/build.gradle b/build.gradle
index abc1234..def5678 100644
--- a/build.gradle
+++ b/build.gradle
@@ -23,7 +23,7 @@ dependencies {
     implementation 'org.springframework.boot:spring-boot-starter-webflux:3.2.4'
     implementation 'org.springframework.boot:spring-boot-starter-actuator:3.2.4'
     implementation 'org.springframework.boot:spring-boot-starter-security:3.2.4'
-    implementation 'org.apache.logging.log4j:log4j-core:2.17.0'
+    implementation 'org.apache.logging.log4j:log4j-core:2.17.1'
     implementation 'com.fasterxml.jackson.core:jackson-databind:2.15.2'
     implementation 'io.projectreactor:reactor-core:3.6.2'
 
```

### B.2 Vulnerability Fix — version variable in ext block

```diff
diff --git a/build.gradle b/build.gradle
index abc1234..def5678 100644
--- a/build.gradle
+++ b/build.gradle
@@ -8,7 +8,7 @@ ext {
     springBootVersion = '3.2.4'
     jacksonVersion = '2.15.2'
-    log4jVersion = '2.17.0'
+    log4jVersion = '2.17.1'
     reactorVersion = '3.6.2'
 }
 
```

### B.3 Config Update — Helm values file UPDATE operation

```diff
diff --git a/helm/dev/dev.yml b/helm/dev/dev.yml
index abc1234..def5678 100644
--- a/helm/dev/dev.yml
+++ b/helm/dev/dev.yml
@@ -10,7 +10,7 @@ application:
   config:
     spring:
       datasource:
-        url: jdbc:postgresql://old-dev-host:5432/mydb
+        url: jdbc:postgresql://new-dev-host:5432/mydb
         username: app_user
         driver-class-name: org.postgresql.Driver
 
```

### B.4 Config Update — Helm values file ADD operation

```diff
diff --git a/helm/dev/dev.yml b/helm/dev/dev.yml
index abc1234..def5678 100644
--- a/helm/dev/dev.yml
+++ b/helm/dev/dev.yml
@@ -12,6 +12,9 @@ application:
       datasource:
         url: jdbc:postgresql://dev-host:5432/mydb
         username: app_user
+        hikari:
+          maximum-pool-size: 10
+          minimum-idle: 2
         driver-class-name: org.postgresql.Driver
 
```

---

## Appendix C — Prompt Templates for AI-Assisted Changes

If you use Copilot's AI capabilities to analyze build files or determine the right change strategy, here are prompt patterns you can embed in the orchestrator:

### C.1 Finding a dependency in build.gradle

```
You are analyzing a Gradle build file. Find the dependency "{dependency_group}:{dependency_name}" 
and tell me:
1. The exact line number where it appears
2. The current version
3. Whether the version is hardcoded or uses a variable
4. If it uses a variable, where is that variable defined?

Here is the build.gradle content:
---
{file_content}
---

Respond ONLY in this JSON format:
{
  "found": true/false,
  "line_number": N,
  "current_version": "x.y.z",
  "version_type": "hardcoded" | "variable",
  "variable_name": "varName or null",
  "variable_defined_in": "file path or null",
  "variable_line_number": N or null
}
```

### C.2 Finding a key in a YAML file

```
You are analyzing a YAML configuration file. Find the key at path "{key_path}" 
(dot-separated, e.g., "spring.datasource.url") and tell me:
1. The exact line number
2. The current value
3. The indentation level (number of spaces)

Here is the YAML content:
---
{file_content}
---

Respond ONLY in this JSON format:
{
  "found": true/false,
  "line_number": N,
  "current_value": "...",
  "indentation_spaces": N
}
```

---

## Quick Start Checklist

When you (the AI assistant in Copilot) receive this document:

1. [ ] Read this entire document
2. [ ] Ask the user to fill in all `[PLACEHOLDER]` values (GitLab URL, group paths)
3. [ ] Create the directory structure as specified in Section 3
4. [ ] Create the configuration files (Section 4)
5. [ ] Create the playbook files (Section 5)
6. [ ] Implement the TypeScript scripts (Section 7) following the exact behavior specifications
7. [ ] Implement the guardrails (Section 8) — these are non-negotiable
8. [ ] Implement the change engine (Section 9)
9. [ ] Implement the reporter (Section 10)
10. [ ] Test with a single project using --dry-run
11. [ ] Show the user the dry-run output before proceeding
12. [ ] Wait for user approval before running any actual git push

**REMEMBER**: When in doubt, ASK. Do not assume. Do not guess GitLab URLs, project names, or configurations. The user will provide all specifics.
