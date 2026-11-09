# Security Findings Register — Week 5 DevSecOps Activity

Owner for all rows: Victor (HelloWorldVictor). Dates: 2026-11-09.

## Findings

| # | Scanner (Drill) | Finding | Severity | Decision | Justification / Notes | Revisit |
|---|---|---|---|---|---|---|
| 1 | Bandit (Drill 2) | B105 hardcoded_password_string — Django `SECRET_KEY` hardcoded in `campus_notifier/settings.py:27` | Low (Medium confidence) | **Remediated** | Key now loaded from `DJANGO_SECRET_KEY` env var via `.env` (gitignored) + `load_dotenv()`, reusing the pattern already used for DB config. **Important: the old key is still in git history, so "removed from the file" is not "rotated".** It was a generated `django-insecure-` dev placeholder, so real-world exposure is low; for a genuine leak the fix would be history scrub + real rotation. | Closed |
| 2 | Trivy (Drill 3) | CVE-2025-64459 — Django 5.2.7 SQL injection, fix published: 5.2.8 | **CRITICAL** | **Remediated** | Bumped `Django==5.2.7` → `Django==5.2.19` in `requirements.txt` (stayed on the 5.2 LTS line). The Drill 5 gate (exit-code 1 on CRITICAL) failed the build on this CVE, then went green after the bump. | Closed |
| 3 | Trivy (Drill 3) | 5 further Django HIGHs in 5.2.7 cleared by the same bump: CVE-2025-64458 (Windows DoS), CVE-2026-1207 (SQLi via RasterField), CVE-2026-25673 (DoS), CVE-2026-33034 (DoS), CVE-2026-15307 (GeoDjango RCE) | HIGH | **Remediated** | Same action as row 2 (`Django==5.2.19` ≥ every listed fixed version). | Closed |
| 4 | Trivy (Drill 3) | CVE-2026-1287 — Django SQL injection via crafted column aliases | HIGH | **Accept as documented risk** | No fixed version published as of 2026-11-09 — nothing to upgrade to. The app uses the ORM with standard querysets only (see `events/views.py`); gate blocks CRITICAL, this is HIGH. | On next Django security release, or 2026-12-09 |
| 5 | Trivy (Drill 3) | CVE-2026-3902 — Django header spoofing via ambiguous header mapping | HIGH | **Accept as documented risk** | No fixed version published as of 2026-11-09. Dev container is not deployed behind an untrusted proxy chain, so the ambiguous-header path is not reachable today. | On next Django security release, or 2026-12-09 |
| 6 | Trivy (Drill 3) | 49 HIGH Debian 13.7 OS CVEs (util-linux family, libacl1, etc.) in the `python:3.12-slim` base image | HIGH | **Accept as documented risk** | Every finding is status `affected` — Debian has published **no fixed packages**, so `apt upgrade` cannot remediate. Remedy: rebuild the image on a fresh base regularly so patched packages are picked up the day Debian ships them; the container is a short-lived, non-production dev artifact. | Every rebuild, latest 2026-12-09 |
| 7 | Checkov (Drill 4) | CKV_AWS_136 — ECR repository not encrypted with KMS | — | **Suppressed with justification (accepted risk)** | Inline `#checkov:skip=CKV_AWS_136` comment in `terraform/main.tf`: ECR is already encrypted at rest with the default AES256; KMS adds key-management cost and complexity for a class-demo registry that never leaves testing. | Before any production use |

## Gate policy (Drill 5)

| Job | Policy | Reason |
|---|---|---|
| Secret scan (gitleaks) | **Blocks** | Any real secret, ever, no threshold. (Unchanged — it already blocked.) |
| Image scan (trivy) | **Blocks on CRITICAL** | `severity: CRITICAL`, `exit-code: "1"`: a CRITICAL with a fix published fails the build. Blocking on HIGH too was deemed the stricter-but-noisier policy; HIGHs are triaged in the register instead. Evidence: run failed on CVE-2025-64459, went green after the Django bump. |
| IaC scan (checkov) | **Blocks** | `soft_fail: false`: terraform that opens something to the internet fails the build; the accepted-risk finding stays suppressed with its written justification. |
| Code scan (bandit) | **Warn** (deliberate choice) | Today's only finding class is Low severity (B105 heuristic). A gate that fails the build on Low-severity heuristic hits is the gate the team disables by Thursday. The check still goes visibly red on the PR when findings exist (it did in Drill 2), so nothing gets silently ignored. Revisit the threshold if Medium/High findings appear. |

## Notes from the drills

- The pre-commit gitleaks hook only scans what is staged *now*; CI gitleaks scans full history (`fetch-depth: 0`). That's why CI re-runs it: the hook is convenience, CI is enforcement — and it also covers anyone who never ran `pre-commit install`.
- Bandit flagged the `django-insecure-` key that gitleaks passed: Bandit understands code semantics (a password-shaped string assigned to a well-known variable), while gitleaks matches entropy/patterns and deliberately allowlists the `django-insecure-` placeholder.
