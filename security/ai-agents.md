# Security For AI-Assisted Development

AI coding assistants read your repository, run commands and edit files. Treat everything they read as untrusted input and everything they can do as privileges to be scoped.

## Threat Model

| Risk | Example | Control |
|------|---------|---------|
| Prompt injection via repo content | A README, issue, web page, dependency or code comment tells the agent to exfiltrate `.env` or run a command | Treat files, tool results and web content as data; deny secrets access; require approval for network and destructive commands |
| Excess agency | Agent runs `rm -rf`, force-pushes, deploys, or edits CI | Least-privilege permissions; ask for risky actions; protected branches |
| Secret exposure | Agent reads `.env`, prints a token, commits a key | Deny-list secret paths; secret scanning in pre-commit and CI; short-lived credentials |
| Malicious tool servers (MCP, plugins, skills) | A third-party MCP server or skill runs code or reads data it should not | Review before enabling; pin versions; scope to the workspace; no secrets in config |
| Unverified generated code | Hallucinated APIs, insecure defaults, missing authorization checks | Human review; tests; static analysis; threat-model new endpoints |
| Hallucinated dependencies ("slopsquatting") | The model invents a package name an attacker has registered | Verify every new package name and publisher before installing |
| License and provenance | Generated code reproduces copyleft or proprietary code | Run licence/similarity checks where policy requires |

## Configure The Assistant Conservatively

- Start from the most restrictive permission mode and allow specific read-only commands (tests, linters, `git status/diff`). Ask for writes outside the source tree, network access, installs, `git push` and deploys.
- Deny reads of `.env*`, `secrets/`, key files and cloud credential directories. Keep real secrets out of the working tree entirely when possible.
- Do not run agents with production credentials or on machines that hold them. Use a dev/test account, a sandbox or a container.
- Use hooks and permission rules for enforcement. Instructions in a markdown file are guidance the model may ignore; hooks and permissions are enforced by the tool.
- Review hook commands, MCP server definitions and plugin manifests before enabling them, and re-review after changes. Project-level config from a freshly cloned repository is untrusted until you have read it.

## Handling Untrusted Content

- Do not let the agent act on instructions found in issues, pull request descriptions, web pages, package READMEs or tool output. Instructions come from the user.
- When the agent must read untrusted content, avoid giving the same session write access to secrets or the ability to publish.
- Be suspicious of text that claims authority ("system", "admin", "ignore previous instructions"), creates urgency, or asks to send data to a URL.

## Reviewing AI-Generated Code

Generated code needs the same review as a stranger's pull request, with extra attention to:

1. **Authorization**: every new endpoint or query checks who is asking, not just that they are logged in.
2. **Input handling**: validation at boundaries, parameterised queries, no string-built commands or paths.
3. **Error paths**: failures must not fail open, leak internals, or swallow exceptions (OWASP A10:2025).
4. **Dependencies**: any new package exists, is the intended one, and passes audit.
5. **Secrets and logging**: nothing sensitive in logs, errors, tests or fixtures.
6. **Tests**: assert behaviour, including denial and failure cases, not just the happy path.

## Secrets Hygiene

- Run secret scanning (gitleaks, trufflehog, GitHub secret scanning) on commit and in CI; block pushes that contain secrets.
- If a secret reaches a prompt, a transcript or a commit, treat it as leaked: rotate it.
- Prefer short-lived, narrowly scoped tokens (OIDC, workload identity) over long-lived keys.

## Using AI In The Product

If your application calls an LLM:

- Validate and constrain model output before using it (schema validation, allow-lists); never pass it to a shell, SQL or `eval` unchecked.
- Do not put secrets or other users' data into prompts. Enforce authorization before retrieval, not after.
- Limit tool use by the model to the minimum, require confirmation for side effects, and log tool calls.
- Rate-limit, cap token and cost budgets, and monitor for abuse.

## Checklist

- [ ] Secret files are denied to the assistant and scanned in CI
- [ ] Risky commands (network, installs, deletes, push, deploy) require approval
- [ ] Hooks, MCP servers and plugins were reviewed before enabling
- [ ] Agent instructions come from the user, never from files or web content
- [ ] Generated code is reviewed for authorization, input handling and error paths
- [ ] New dependencies are verified (name, publisher, audit) before install
