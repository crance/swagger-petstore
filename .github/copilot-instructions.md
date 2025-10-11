This project will be used to demonstrate Fortify's capability in DevSecOps, specifically SAST (Static Application Security Testing) using Fortify SAST and Aviator (Generative AI).
As part of DevSecOps, we will integrate Fortify SAST and Aviator into CI/CD pipelines using GitHub Actions, GitLab pipeline, and other tools.
FCLI (Fortify Command Line Interface) has launched MCP (Model Context Protocol) to enable AI agents to interact with Fortify products, such as Fortify on Demand and Fortify Software Security Center.

# Instructions for GitHub Copilot

### Guidance: variable values, workflows, and secrets
1. Do not duplicate authoritative secret or environment values in this instructions file. Instead, show canonical usage examples that reference CI environment variables and GitHub Actions secrets (e.g., `${{ secrets.SSC_URL }}`, `${{ env.SSC_APP_NAME }}`).
2. Link to the workflow files where variables and secrets are defined. For GitHub Actions this repository currently defines these in `.github/workflows/fortify.yml` and `.github/workflows/aviator.yml`.
3. For multi-platform/CI support (GitLab, Jenkins), document a single canonical mapping: show the variable name (e.g., `SSC_URL`, `SSC_TOKEN`, `SC_SAST_SARGS`) and state that each platform should map its secret store to the same variable names. This avoids divergence across CI providers.

Example mapping paragraph you can reuse across CI docs:

"Platform secret mapping: the repository uses the logical variables `SSC_URL`, `SSC_TOKEN`, `SCSAST_URL`, `CLIENT_AUTH_TOKEN`, `SSC_APP_NAME`, and `SSC_APP_VER_NAME`. In GitHub Actions these are provided as `secrets.SSC_URL` and `secrets.SSC_TOKEN`; in GitLab CI they should be defined as project or group variables with the same names. Do not place credential values in this file."

### Safe pattern for showing variable usage (do not duplicate values)
When documenting usage, follow this safe pattern so instructions remain accurate without embedding authoritative values:

- Always use environment interpolation or placeholders in examples. For example:
	- Correct: `--publish-to "${{ env.SSC_APP_NAME }}:${{ env.SSC_APP_VER_NAME }}"`
	- Avoid: `--publish-to "petstore:v2"` in this instructions file.
- Point readers (and agents) to the authoritative location of values (e.g., the GitHub Actions workflow file) when literal values are required. Add a sentence like: "Authoritative values live in `.github/workflows/fortify.yml` — update them there, not in this file."
- If an automation needs a literal mapping, obtain it from the CI environment at runtime (e.g., `env.SSC_APP_NAME`) instead of copying it here.

Note: you've chosen not to create per-CI README files or a single-source manifest. Respecting that choice, this file will continue to show placeholders and point to the workflow files as the source of truth.

### MCP session conventions and defaults
When calling FCLI MCP tools that accept an SSC session, prefer the named session `default` unless a project-specific session has been created and documented. Example convention for the agent:

- Use `--ssc-session default` for one-off or CI invocations.
- If your CI creates and names a session, the agent should use that session name instead; the session name is defined by the CI environment and passed to MCP calls.

This avoids implicit session creation and keeps behavior predictable across environments.

### Agent MCP query acceptance criteria (contract)
When an automated agent (or Copilot) executes MCP commands against SSC on behalf of this repo, follow these acceptance rules:

1. Use the environment variables documented above (`FOD_RELEASE`, `SSC_APP_NAME`, `SSC_APP_VER_NAME`) rather than embedding literal app/version strings.
2. Use `--ssc-session default` unless instructed otherwise by the CI job.
3. Always pass `--store <name>` to capture a machine-parsable response when supported. The agent should read the stored variable and return the JSON payload instead of only printing human text.
4. Surface errors: on non-zero exit codes return the raw `stderr` and the `exitCode` back to the caller.
5. When grouping or filtering via `--by` or other SSC parameters, validate that the argument is supported; if the server rejects it, surface the server error and suggest valid options.

These rules make programmatic automation reliable and auditable.

### Runtime agent guidance (resolve values from workflows)
When an automated agent runs outside of GitHub Actions (for example, in a developer workstation or an AI agent executing MCP calls), the `${{ ... }}` placeholders in workflow files are not expanded. To avoid failed MCP calls, agents should resolve the concrete application/version values from the repository's workflow files at runtime instead of hard-coding them here.

Recommended safe pattern (do not duplicate authoritative values in this file):

- Attempt to read the workflow files where the values are defined, for example `.github/workflows/fortify.yml` or `.github/workflows/aviator.yml`, and extract the variables `SSC_APP_NAME`, `SSC_APP_VER_NAME`, or `FOD_RELEASE`.

`FOD_RELEASE` - this the application information in `application:release` format for Fortify on Demand (FoD)
`SSC_APP_NAME` - this the application name for Fortify Software Security Center (SSC)
`SSC_APP_VER_NAME` - this the application version name for Fortify Software Security Center (SSC)
Strictly use the above variables for MCP queries.

- Use a small, local parse/grep step rather than relying on GitHub interpolation. Example approaches (agent implementation may vary):
	- Use a YAML parser (recommended): parse the workflow YAML and read the `env` mapping for `SSC_APP_NAME` and `SSC_APP_VER_NAME`.
	- If a YAML parser is not available, use a conservative grep that extracts simple assignments under `env:` blocks. Only accept literal scalars (no `${{ ... }}` references).

- Only accept values that are literal scalars (no `${{ ... }}` templates). If the extracted value contains `${{` or other template syntax, treat it as unresolved and continue to fallback steps.

Fallback order for agents:

1. If CI environment variables are available in the current environment (for example `process.env.SSC_APP_NAME`), use them.
2. If not, parse `.github/workflows/fortify.yml` and `.github/workflows/aviator.yml` and extract the `env` entries. Use the resolved string when required by MCP commands.
3. If parsing fails or the workflows use templated values (for example `${{ vars.SOME_VAR }}`), prompt the user to supply the concrete values, or fail with a clear error showing the workflow fragment and a suggested resolution command.

Minimal example commands (agent may run these locally):

 - With a YAML parser (python example):
	 python -c "import sys,yaml; d=yaml.safe_load(open(' .github/workflows/fortify.yml')); print(d.get('env',{}).get('SSC_APP_NAME'))"

 - Quick grep (conservative, will only match simple literals):
	 grep -E "^\s*SSC_APP_NAME:\s*\"?[^\$\{].+\"?$" .github/workflows/*.yml || true

Behavioral contract reminder:

 - Do not copy literal application names or versions into this instructions file. Instead, the instructions now point agents to the authoritative workflow files and give safe, non-invasive ways to resolve values at runtime.
 - When an agent resolves values at runtime, log the source (which workflow file and line) in its output for auditability.

### Examples: common fcli / sc-sast / aviator / ssc workflows
Below are a few compact, canonical examples showing how the repository variables are intended to be used. Do not hard-code secret values in this file; CI injects them as secrets or environment variables.

- Login to SSC (example):
	java -jar fcli.jar ssc session login --url "${{ secrets.SSC_URL }}" --token "${{ secrets.SSC_TOKEN }}" --ssc-session default

- Start a ScanCentral SAST scan and publish to SSC (example):
	java -jar fcli.jar sc-sast scan start -f fortifypackage.zip --publish-to "${{ env.SSC_APP_NAME }}:${{ env.SSC_APP_VER_NAME }}" --store sc_sast_scan --ssc-session default

- Run an Aviator audit against an SSC appversion (example):
	java -jar fcli.jar ssc session login --url "${{ secrets.SSC_URL }}" --token "${{ secrets.SSC_TOKEN }}" --ssc-session default
	java -jar fcli.jar aviator ssc audit --av "${{ env.SSC_APP_NAME }}:${{ env.SSC_APP_VER_NAME }}" --app "${{ env.AVIATOR_APP_NAME }}" --ssc-session default
	java -jar fcli.jar ssc session logout --ssc-session default || true

Keep these examples as canonical usage patterns; the actual values (secrets and env) must come from the CI environment.

### Tip: always consider pagination when listing resources
When using MCP to list resources (for example `ssc issue list-groups`, `ssc issue list`, or `ssc appversion list`), responses are paged. Agents should either request a large `--page-size` when supported or iterate using `pagination-offset` (or follow the `nextPageOffset` value) to retrieve all pages. This avoids missing entries that appear on later pages (for example, an OWASP external list may appear on page 2 or later).

Example: fetch all groups in one request (if the server supports it):
	java -jar fcli.jar ssc issue list-groups --appversion "${{ env.SSC_APP_NAME }}:${{ env.SSC_APP_VER_NAME }}" --page-size 200 --ssc-session default --store groups_all

Example: iterate pages using offsets:
	java -jar fcli.jar ssc issue list-groups --appversion "${{ env.SSC_APP_NAME }}:${{ env.SSC_APP_VER_NAME }}" --pagination-offset 0 --ssc-session default --store groups_p0
	java -jar fcli.jar ssc issue list-groups --appversion "${{ env.SSC_APP_NAME }}:${{ env.SSC_APP_VER_NAME }}" --pagination-offset 20 --ssc-session default --store groups_p20

### Remediation Advice
Prioritize Aviator's remediation advice (found in issue comments) over standard SAST results when both are available. Aviator leverages generative AI to provide context-aware recommendations, which can be more effective in addressing complex security issues.

## CI/CD Integration
### GitHub Action workflows
[Integrate Fortify with GitHub Action](https://github.com/marketplace/actions/fortify-ast-scan)

## Fortify Official Documentation and Usage Guides

FCLI - Fortify Command Line Interface
- [FCLI GitHub](https://github.com/fortify/fortify-cli)
- [FCLI's Installation & Usage](https://fortify.github.io/fcli/latest/)
- [FCLI's Manual Page](https://fortify.github.io/fcli/latest/manpage/fcli.html)

MCP - Model Context Protocol
- [Blog on FCLI MCP Launch](https://community.opentext.com/cybersec/b/cybersecurity-blog/posts/fortify-cli-3-9-0-your-appsec-tools-meet-ai-agents)
- [FCLI MCP Manual](https://fortify.github.io/fcli/latest/manpage/fcli-util-mcp-server-start.html)

Fortify Software Security Center (SSC)
- [Fortify Software Security Center Documentation](https://www.microfocus.com/documentation/fortify-software-security-center/)
- [Fortify Software Security Center Usage Guide](https://www.microfocus.com/documentation/fortify-software-security-center/2520/ssc-ugd-html-25.2.0/index.htm)


Fortify SAST - Static Code Analyzer (sourceanalyzer) and ScanCentral SAST (sc-sast/ scsast)
- [Fortify ScanCentral SAST Usage Guide](https://www.microfocus.com/documentation/fortify-software-security-center/2520/sc-sast-ugd-html-25.2.0/index.htm)
- [Fortify SAST Documentation](https://www.microfocus.com/documentation/fortify-static-code-analyzer-and-tools/)
- [Fortify SAST Usage Guide](https://www.microfocus.com/documentation/fortify-static-code-analyzer-and-tools/2520/sast-ugd-html-25.2.0/index.htm)

Fortify DAST - WebInspect / ScanCentral DAST (sc-dast/ scdast)
- [Fortify DAST Documentation](https://www.microfocus.com/documentation/fortify-ScanCentral-DAST/)
- [Fortify DAST Usage Guide](https://www.microfocus.com/documentation/fortify-ScanCentral-DAST/2520/sc-dast-ugd-html-25.2.0/index.htm)

Fortify Aviator
- [Fortify SAST Aviator Usage Guide](https://www.microfocus.com/documentation/fortify-static-code-analyzer-and-tools/2520/sast-aviator-ugd-html-25.2.0/index.htm)

Fortify on Demand (FoD)
- [Fortify on Demand Documentation](https://www.microfocus.com/documentation/fortify-on-demand/)
- [Fortify on Demand Usage Guide](https://www.microfocus.com/documentation/fortify-on-demand/253/core-appsec-ugd-25.3-en.pdf)

Vulncat - taxonomy of software security errors
- [Vulncat](https://vulncat.fortify.com/en)