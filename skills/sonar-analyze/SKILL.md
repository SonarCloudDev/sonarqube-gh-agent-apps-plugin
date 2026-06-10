---
name: sonar-analyze
description: Analyze a file for quality and security issues using SonarQube
argument-hint: "[file-path]"
---

# SonarQube — Code Analysis

Analyze code for quality and security issues using the SonarQube MCP Server.

The server exposes **one of two** analysis tools depending on the organization's eligibility — they are mutually exclusive:

- `run_advanced_code_analysis` — **preferred.** Reuses the full CI-level analysis context from the most recent SonarQube Cloud scan of the branch, so results are higher precision. Available only to eligible organizations.
- `analyze_code_snippet` — **fallback.** Analyzes a single file in isolation from its content. Used when advanced analysis is not available to the organization.

**Always try `run_advanced_code_analysis` first; if it is not present in your available tools, use `analyze_code_snippet`.** Never assume which one exists — check your tool list.

## Disclaimer

Both tools analyze **one file at a time**, so results are not a substitute for a full project scan — mention this if the user might expect exhaustive coverage.

`run_advanced_code_analysis` additionally requires that a **previous SonarQube Cloud analysis exists** for the branch you target. If none has run yet, the tool has no context to build on — tell the user and suggest running CI analysis first (or fall back to snippet analysis if that tool were available, which it is not when advanced is).

## Usage

```
sonar-analyze                        # analyze the file currently in context
sonar-analyze src/auth/login.py      # analyze a specific file
```

## Prerequisites

This skill requires the SonarQube MCP Server to be configured with the repository mounted at `/app/mcp-workspace`, and **either** `mcp__sonarqube__run_advanced_code_analysis` **or** `mcp__sonarqube__analyze_code_snippet` to be available in your session.

If a tool fails due to authentication problems, ask the user to ensure they have given their consent for automatic token exchange through SonarQube Cloud > My Account > Access Tokens > Agent Apps (direct URL is https://sonarcloud.io/account/access-tokens?tab=github_agent_hq). Otherwise surface the tool error verbatim and stop.

## Instructions

### Step 1: Resolve what to analyze

Both tools analyze **one file at a time**. Resolve a single file path:

- If the user provided a file path, use it.
- If no path was provided, look at the current conversation context for a recently mentioned or edited file.
- If nothing is clear, ask: *"Which file would you like me to analyze?"*

Do not accept a directory as input. If the user provides one, ask them to specify a single file.

### Step 2: Build the `filePath` argument (prefix with the repository name)

The MCP server resolves `filePath` **relative to the workspace mount root** (`/app/mcp-workspace`). In the GitHub agent apps runtime the repository is **not** mounted at that root directly — it is checked out into a subdirectory named after the repository (`/app/mcp-workspace/<owner>/<repo>`, i.e. the value of `GITHUB_REPOSITORY`).

Therefore you **must prefix** the normal repo-relative path with the repository slug `<owner>/<repo>`:

| What the user gives you (repo-relative) | What you pass as `filePath`             |
| --------------------------------------- | --------------------------------------- |
| `src/auth/login.py`                     | `MyOrg/my-repository/src/auth/login.py` |
| `pom.xml`                               | `MyOrg/my-repository/pom.xml`           |

Use the actual `<owner>/<repo>` of the repository you are running in (the `GITHUB_REPOSITORY` value), not the example above. If you are unsure of the slug, derive it from the PR/branch context rather than guessing.

This prefix is **required** for `run_advanced_code_analysis` (it reads the file from the mount). It is also safe to apply for `analyze_code_snippet`.

### Step 3: Determine the file scope

If the path contains `test`, `spec`, or `__tests__`, the file is a **test** file; otherwise it is **main**. (The two tools name this parameter differently — see below.)

### Step 4: Call the analysis tool

#### Step 4a — Preferred: `mcp__sonarqube__run_advanced_code_analysis`

Use this when the tool is available. Parameters:

- `filePath` (**required**) — the repository-slug-prefixed path from Step 2.
- `branchName` (**required**) — the branch to pull the latest analysis context from. Use the branch currently being worked on (the PR's source branch when running on a PR).
- `projectKey` (optional) — **omit it** when `SONARQUBE_PROJECT_KEY` is configured in the server env (the common case); the tool ignores it then. Pass it only when targeting a different project.
- `fileScope` (optional) — `MAIN` or `TEST` from Step 3; defaults to `MAIN`.

```json
{
  "filePath": "SonarCloudDev/sonarqube-cli/src/auth/login.py",
  "branchName": "feature/my-branch",
  "fileScope": "MAIN"
}
```

#### Step 4b — Fallback: `mcp__sonarqube__analyze_code_snippet`

Use this only when `run_advanced_code_analysis` is **not** in your available tools. First read the file's full content, then detect the language from its extension:

| Extension              | Language key |
| ---------------------- | ------------ |
| `.py`                  | `py`         |
| `.js` `.jsx`           | `js`         |
| `.ts` `.tsx`           | `ts`         |
| `.java`                | `java`       |
| `.go`                  | `go`         |
| `.php`                 | `php`        |
| `.cs`                  | `cs`         |
| `.rb`                  | `rb`         |
| `.swift`               | `swift`      |
| `.kt`                  | `kotlin`     |
| `.c` `.cpp` `.cc` `.h` | `cpp`        |

Parameters:

- `filePath` (string) — the repository-slug-prefixed path from Step 2.
- `codeSnippet` (string) — the **full file content**.
- `language` (string) — the language key from the table above.
- `scope` (string) — `"TEST"` or `"MAIN"` from Step 3.
- `projectKey` (optional) — omit when `SONARQUBE_PROJECT_KEY` is configured; pass only when targeting another project.

```json
{
  "filePath": "SonarCloudDev/sonarqube-cli/src/auth/login.py",
  "codeSnippet": "<full file content>",
  "language": "py",
  "scope": "MAIN"
}
```

### Step 5: Format the results

**If issues are found**, present them as a table sorted by line number:

```markdown
## SonarQube Analysis — `src/auth/login.py`

Found **3 issue(s)**:

| Line | Severity   | Rule         | Message                                               |
| ---- | ---------- | ------------ | ----------------------------------------------------- |
| 12   | 🔴 Blocker | python:S2077 | Make sure that executing this SQL query is safe here. |
| 34   | 🟠 Major   | python:S1481 | Remove the unused local variable "token".             |
| 67   | 🟡 Minor   | python:S1135 | Complete the task associated to this "TODO" comment.  |
```

Report the file using its **repo-relative** path (without the `<owner>/<repo>` prefix) so it is clickable for the user.

Severity icons (the label depends on the server version):
- 🔴 Blocker
- 🟠 Critical / High
- 🟡 Major / Medium
- 🔵 Minor / Low
- ⚪ Info

**If no issues are found**:

```markdown
## SonarQube Analysis — `src/auth/login.py`

✅ No issues found.
```

### Step 6: Next steps

After the results, always add:

- If issues were found: *"Invoke the sonar-fix-issue skill with `<rule> <file>:<line>` to fix a specific issue, or ask me to fix them all."*
- If the user wants to analyze another file: remind them to invoke the sonar-analyze skill with the file path.
