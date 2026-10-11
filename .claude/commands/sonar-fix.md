---
description: Run SonarQube on the current PR (vs main), generate a report, and fix the issues in changed code
argument-hint: [pr-number] [--report-only]
allowed-tools: Bash(git:*), Bash(gh pr:*), Bash(mvn:*), Bash(./mvnw:*), Bash(./gradlew:*), Bash(gradle:*), Bash(curl:*), Bash(jq:*), Read, Edit, Write, Grep, Glob, AskUserQuestion
---

# Fix SonarQube issues in a Spring Boot PR

Arguments: `$ARGUMENTS`
- First argument (optional): PR number. If omitted, detect it with `gh pr view --json number -q .number`; if there is no PR, use the current branch against `main`.
- `--report-only`: generate the report but do not change any code.

## Prerequisites (check first, stop and tell the user if missing)

- Environment variables `SONAR_HOST_URL` and `SONAR_TOKEN` are set. Never print the token or put it in a file, commit, or the report.
- `SONAR_PROJECT_KEY` is set, or can be read from `sonar.projectKey` in `pom.xml`, `build.gradle(.kts)` or `sonar-project.properties`.
- The build tool is detected from the repo root: `pom.xml` (use `./mvnw` if present, else `mvn`) or `build.gradle(.kts)` (use `./gradlew` if present).

## Steps

1. **Scope the change.** List files changed against main with `git diff --name-only origin/main...HEAD` (run `git fetch origin main` first). Keep `.java`, `.kt`, `.xml`, `.yml`, `.properties` files under `src/`. These are the only files you may edit.

2. **Run the analysis.**
   - Maven: `mvn -B verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.host.url=$SONAR_HOST_URL -Dsonar.token=$SONAR_TOKEN -Dsonar.projectKey=$KEY`
   - Gradle: `./gradlew test jacocoTestReport sonar -Dsonar.host.url=$SONAR_HOST_URL -Dsonar.token=$SONAR_TOKEN -Dsonar.projectKey=$KEY`
   - Add pull request parameters when a PR number is known: `-Dsonar.pullrequest.key=<n> -Dsonar.pullrequest.branch=<head-branch> -Dsonar.pullrequest.base=main`. Pull request analysis requires SonarQube Developer Edition or higher. On Community Edition, omit them, scan the branch, and filter results to the changed files from step 1.
   - If the build or tests fail, report that and stop. Do not fix Sonar issues on a broken build.

3. **Fetch results** from the SonarQube API (authenticate with `curl -s -u "$SONAR_TOKEN:"`):
   - Quality gate: `$SONAR_HOST_URL/api/qualitygates/project_status?projectKey=$KEY&pullRequest=<n>`
   - Issues: `$SONAR_HOST_URL/api/issues/search?componentKeys=$KEY&pullRequest=<n>&resolved=false&ps=500` (use `&branch=<name>` instead of `pullRequest` on Community Edition)
   - Security hotspots: `$SONAR_HOST_URL/api/hotspots/search?projectKey=$KEY&pullRequest=<n>`
   - Wait for the analysis task to finish before querying (poll `api/ce/task` or `.scannerwork/report-task.txt`).

4. **Write the report** to `reports/sonar-report.md` (create the directory; this is generated output, do not commit it unless asked). Include:
   - PR number, branch, commit SHA, timestamp, quality gate status
   - Counts by type (Bug, Vulnerability, Code Smell, Security Hotspot) and severity
   - Coverage and duplication on new code, if the gate reports them
   - Issues grouped by file, each with rule key, severity, line, message
   - A "Fixed / Skipped / Needs human decision / Not selected" section, filled in after step 5

5. **Ask what to fix, then fix by criticality** (skip this whole step with `--report-only`):
   - **Present the report first.** Show a short table in the chat: severity (Blocker, Critical, Major, Minor, Info) x type (Bug, Vulnerability, Code Smell, Security Hotspot) with counts, plus the five most severe issues (file:line, rule, message). Mention the full report path.
   - **Ask before changing any code** with `AskUserQuestion`. Do not start fixing until the user answers.
     - Question 1 (single select), "Which severity levels should I fix?": `Blocker + Critical only`, `Blocker + Critical + Major`, `All severities`, `Let me pick specific issues`. Only offer tiers that actually have issues.
     - If the user picks specific issues, or the count is large, ask a follow-up (multi-select, up to 4 options per question, grouped by severity or by file) to choose the exact issues or files. List more with several questions or accept an "Other" answer such as rule keys or issue IDs.
     - Question 2 (single select), "How should I handle security hotspots and vulnerabilities?": `Fix where the fix is clear, list the rest`, `Report only, do not touch`.
   - If the user selects nothing or cancels, stop and keep the report only.
   - Fix **only the issues the user selected**, in files changed by the PR, in this order: Blocker, Critical, Major, Minor, Info. Finish and verify each severity tier (compile, then the affected tests) before starting the next, so a problem surfaces at the most critical level first.
   - Record each selected issue as Fixed, Skipped (with reason), or Needs human decision in the report.
   - Unselected issues stay open and are listed in the report as "Not selected".
   - Fix the root cause. Never silence an issue with `@SuppressWarnings`, `//NOSONAR`, or by editing Sonar rules or exclusions.
   - Typical Spring Boot fixes: constructor injection instead of field `@Autowired`; no `@Transactional` on private methods; close resources with try-with-resources; avoid returning entities from controllers (use DTOs); parameterised queries instead of string-built SQL; no hardcoded credentials (move to config or environment); replace `System.out`/`printStackTrace` with SLF4J logging; avoid catching generic `Exception`/`Throwable`.
   - Security hotspots and vulnerabilities: fix when the fix is clear. Otherwise list them under "Needs human decision" with an explanation. Do not mark hotspots as reviewed.
   - Missing coverage on new code: add focused unit tests for the changed logic when the gate fails on coverage.
   - Do not change public API behavior, database schemas, or dependency versions without asking.

6. **Verify.** Run the unit tests (`mvn -B test` or `./gradlew test`) after fixing. If tests fail because of your change, fix or revert it. Then re-run step 2 and 3 once and update the report with the new counts and gate status.

7. **Summarize** to the user: gate status before and after, issues fixed, issues skipped or needing a decision (with reasons), and the report path. Do not commit or push. Offer to do so.

## Rules

- Do not modify files outside the PR's changed files unless a fix cannot work without it; if so, say so in the summary.
- If Sonar is unreachable or authentication fails, report the HTTP status and stop. Do not guess results.
- Keep fixes minimal and consistent with the surrounding code style.
