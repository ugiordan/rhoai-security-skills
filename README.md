# rhoai-security-skills

Skill definitions for the [rhoai-security-scanner](https://github.com/ugiordan/rhoai-security-scanner) skill chain engine. This repo is cloned at build time into the scanner's Docker image at `/app/skills`.

## Structure

```
skills.yaml                          # Skill registry (all metadata lives here)
chains.yaml                          # Chain definitions (how skills compose into pipelines)
scanning/
  semantic-scan/
    prompt.md                        # AI prompt for semantic security analysis
    schema.json                      # JSON Schema for skill output validation
```

## Skills

Two categories of skills, all defined in `skills.yaml`:

### SAST Tools (15)

Bash-runner skills that wrap existing scanning tools. No model tier, no prompts. Executed via `scan-repo.sh`.

| Skill | Category | Description |
|-------|----------|-------------|
| semgrep | SAST | Multi-language SAST with custom rules |
| gosec | SAST | Go security checker |
| gitleaks | Secrets | Secret detection in git history and code |
| trufflehog | Secrets | High-signal secret scanner with verification |
| shellcheck | Linting | Shell script static analysis |
| hadolint | Linting | Dockerfile linting |
| kube-linter | Kubernetes | K8s manifest linting for security and best practices |
| trivy | SCA | Vulnerability scanner for containers and filesystems |
| grype | SCA | Vulnerability scanner for container images |
| osv-scanner | SCA | OSV database vulnerability scanner |
| govulncheck | SCA | Go vulnerability checker using call graph analysis |
| pip-audit | SCA | Python dependency vulnerability scanner |
| actionlint | CI/CD | GitHub Actions workflow linter |
| zizmor | CI/CD | GitHub Actions security scanner |
| cppcheck | SAST | C/C++ static analysis |
| flawfinder | SAST | C/C++ security flaw finder |

### AI Skills (9)

LLM-powered skills executed via AgentRunner (tool use) or LLMPromptRunner (reasoning only).

| Skill | Tier | Tools | Description |
|-------|------|-------|-------------|
| semantic-scan | heavy | Read, Glob, Grep | AI security scan for business logic flaws, LLM risks |
| adversarial-review | heavy | Read, Glob, Grep | Multi-specialist adversarial code review |
| prodsec-audit | medium | Read, Glob, Grep | Line-by-line security audit with context building |
| claude-direct-merge | medium | none | Single-prompt finding deduplication and risk ranking |
| adversarial-triage | heavy | none | Multi-specialist triage with cross-validation |
| fp-check | medium | none | 6-gate false-positive verification framework |
| cve-analyze | medium | none | CVE exploitability assessment in application context |
| cve-triage | medium | none | CVE true/false positive verification |
| cve-fix | heavy | Read, Write, Bash, Grep, Glob | Automated CVE remediation with PR creation |

## Chains

Chains compose skills into multi-step pipelines. Defined in `chains.yaml`.

| Chain | Description | Steps |
|-------|-------------|-------|
| **full-scan** | Complete security scan | sast -> ai-scan -> merge-triage -> report |
| **sast** | Static analysis (parallel, 15 tools, max 5 concurrent) | All SAST tools |
| **ai-scan** | AI-powered analysis (parallel, max 3 concurrent) | semantic-scan, adversarial-review, prodsec-audit |
| **merge-triage** | Finding dedup and triage (sequential) | claude-direct-merge -> adversarial-triage |
| **cve** | CVE analysis and remediation | cve-analyze -> cve-triage -> cve-fix (optional) |
| **report** | Report generation | per-tool -> consolidated -> comparison |

## Adding a New Skill

1. Add the skill entry to `skills.yaml` with all metadata fields
2. If it's an AI skill, create `<path>/prompt.md` and optionally `<path>/schema.json`
3. Reference it in the appropriate chain in `chains.yaml`
4. Run `validate-skills.py` from the scanner repo to verify

## How the Scanner Uses This Repo

The scanner's Dockerfile clones this repo at build time:

```dockerfile
RUN git clone --depth 1 https://github.com/ugiordan/rhoai-security-skills.git /app/skills
```

The skill engine reads `skills.yaml` and `chains.yaml` from `/app/skills/` at startup. To update skills in production, rebuild the scanner image.
