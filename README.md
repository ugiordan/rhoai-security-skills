# rhoai-security-skills

AI skill definitions for the [rhoai-security-scanner](https://github.com/ugiordan/rhoai-security-scanner) skill chain engine. This repo is cloned at build time into the scanner's Docker image at `/app/skills`.

SAST tools (semgrep, gitleaks, trivy, etc.) are not here. They're managed by `tool_registry.py` and `scan-repo.sh` in the scanner repo. This repo only contains AI skills that run after SAST completes.

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

All defined in `skills.yaml`. Three categories:

### Scanning (3 skills, tool use via Agent SDK)

| Skill | Tier | Tools | Description |
|-------|------|-------|-------------|
| semantic-scan | heavy | Read, Glob, Grep | AI security scan for business logic flaws, LLM risks |
| adversarial-review | heavy | Read, Glob, Grep | Multi-specialist adversarial code review |
| prodsec-audit | medium | Read, Glob, Grep | Line-by-line security audit with context building |

### Triage (3 skills, reasoning only)

| Skill | Tier | Description |
|-------|------|-------------|
| claude-direct-merge | medium | Single-prompt finding deduplication and risk ranking |
| adversarial-triage | heavy | Multi-specialist triage with cross-validation |
| fp-check | medium | 6-gate false-positive verification framework |

### CVE (3 skills)

| Skill | Tier | Description |
|-------|------|-------------|
| cve-analyze | medium | CVE exploitability assessment in application context |
| cve-triage | medium | CVE true/false positive verification |
| cve-fix | heavy | Automated CVE remediation with PR creation |

## Chains

Chains compose skills into multi-step pipelines. Defined in `chains.yaml`.

| Chain | Description | Steps |
|-------|-------------|-------|
| **full-scan** | AI analysis + triage (runs after SAST) | ai-scan -> merge-triage |
| **ai-scan** | AI-powered analysis (parallel, max 3 concurrent) | semantic-scan, adversarial-review, prodsec-audit |
| **merge-triage** | Finding dedup and triage (sequential) | claude-direct-merge -> adversarial-triage |
| **cve** | CVE analysis and remediation | cve-analyze -> cve-triage -> cve-fix (optional) |

## Adding a New Skill

1. Add the skill entry to `skills.yaml` with all metadata fields
2. Create `<path>/prompt.md` and optionally `<path>/schema.json`
3. Reference it in the appropriate chain in `chains.yaml`
4. Run `validate-skills.py` from the scanner repo to verify

## How the Scanner Uses This Repo

The scanner's Dockerfile clones this repo at build time:

```dockerfile
RUN git clone --depth 1 https://github.com/ugiordan/rhoai-security-skills.git /app/skills
```

The skill engine reads `skills.yaml` and `chains.yaml` from `/app/skills/` at startup. To update skills in production, rebuild the scanner image.
