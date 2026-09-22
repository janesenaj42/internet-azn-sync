# Internet–AZN Git Sync

*Correct as of 22 September 2026*

---

## 1. Version Control & Security Scanning: AZN vs Internet

Scanning on Internet runs on free/OSS substitutes in place of AZN's paid enterprise stack.

| Area | AZN (On-Prem) | Internet | 
| --- | --- | --- | 
| Version control | GitLab Premium | GitLab Free* | 
| Lint (frontend) | ESLint | ESLint | 
| SAST | Fortify, Parasoft | Semgrep CE | 
| SCA | Scantist | Trivy | 
| Artifact / container registry | JFrog | GitLab Package/Container Registry | 

*capped at 5 seats, in midst of migrating to github

**What Lint, SAST and SCA are:**
- **Lint (ESLint)** checks code style and structure — catching formatting issues, unused variables, and common bug patterns — as source code is written. It's a code-quality check, not a security scan, and runs alongside SAST/SCA in CI.
- **SAST (Static Application Security Testing)** scans your own source code — without running it — to catch security bugs like injection flaws, hardcoded secrets, or unsafe API usage before the code ships.
- **SCA (Software Composition Analysis)** scans your third-party dependencies (npm/Maven packages, base images, etc.) against known vulnerability databases (CVEs) to catch risky or outdated libraries you've pulled in.

---
 
## 2. Synchronization Methodology & Workflow

> [!NOTE] Core strategy: one-way downstream sync only (Internet → AZN)
> - Code is developed and reviewed on Internet, then pulled into AZN at release time. 
> - No reverse sync exists — AZN never pushes code back to Internet.
 
Each repo now carries **two CI config files**: `.gitlab-ci.internet.yml` (runs on Internet, online) and `.gitlab-ci.yml` (runs on AZN, offline).
 

```mermaid
flowchart TD
    classDef ci fill:#eaf5f2,stroke:#2f8f82,color:#16233f
    classDef human fill:#eef1fb,stroke:#5b6fbd,color:#1f2a5c,stroke-dasharray:4 4

    P["push / merge request"]:::human
    P --> Internet
    subgraph Internet["🌐 Internet — .gitlab-ci.internet.yml"]
        direction TB
        I0["Lint + unit tests + build"]:::ci
        I1["Semgrep CE<br/>SAST scan"]:::ci
        I2["Trivy<br/>SCA scan"]:::ci
        I0 --> I3["Build image<br/>push to registry"]:::ci
        I1 --> I3
        I2 --> I3
        I3 -- If SemVer tag --> I4["Git bundle"]:::ci
        I4 --> I5["Attach bundle to<br/>GitLab Release"]:::ci
    end
 
    I5 --> H1["Manual transfer via FG from Internet to AZN<br/>(air gap)"]:::human
    H1 --> H2["Run airgap-release-import.sh on AZN"]:::human
    H2 --> A0
 
    subgraph Anzen["🔒 Anzen — .gitlab-ci.yml"]
        direction TB
        A0["Lint"]:::ci --> A1["Fortify / Parasoft<br/>SAST scan"]:::ci
        A1 --> A1b["Scantist<br/>SCA scan"]:::ci
        A1b --> A2["Build<br/>Docker / npm / Maven"]:::ci
        A2 --> A3["Build image<br/>push to registry"]:::ci
    end
 
    subgraph Legend["Legend"]
        direction LR
        LCI["CI (automated)"]:::ci
        LHuman["Human action"]:::human
    end
    style Legend fill:#f2f2f2,stroke:#9a9a9a,color:#333333
```
 
**What `airgap-release-import.sh` does:** 
- verifies the bundle, commit, and tag validity
- merges and fast-forwards the validated release into the AZN repository
- **It is repo-agnostic.** There is one copy, in `webcore-compose/scripts/`.
  ```
  airgap-release-import.sh -C ~/repos/<any-repo> release-1.2.0.bundle
  ```
     - The one fixed assumption is the tag pattern `^v?[0-9]+\.[0-9]+\.[0-9]+$`, which matches the pattern the release job uses. Releases must stay on plain SemVer tags for the import to work.
 
### Why the Git Trees between Internet and AZN Look Different
 
Internet carries full branch history. AZN stays a linear, read-only mirror advanced only by verified fast-forward merges.
 
**Internet — full branching:**
```
main
├── feature/*        (active dev)
├── release/v1.4     (release prep)
└── tags: v1.4.0     (triggers release-bundle)
```
 
**AZN — linear mirror only:**
```
main
└── fast-forward only
      (no feature branches,
       no local commits)
```
 
AZN only ever receives fast-forwarded commits from a verified bundle — it never grows its own branches. Internet keeps the full feature/release model needed for day-to-day development and review.
 
---
 
## 3. Current Status
 
GC3 is currently developing on internet.


