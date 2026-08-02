# Engineering Standards & Repository Architecture Policy
**COEP Quant Finance Club**  
*Target Audience:* Student Coordinators, Technical Heads, Developers, and Maintainers.  
*Document Version:* 2.1.0 (Production-Ready)

---

## 1. Executive Summary & Philosophy

As the COEP Quant Finance Club scales its engineering operations—spanning high-frequency algorithmic trading engines, heavy quantitative research pipelines, and production web infrastructure—we must maintain strict architectural hygiene. 

Unchecked repository sprawl and mixed concerns lead to broken builds, untracked dependencies, and security vulnerabilities. This document outlines our mandatory engineering standards, repository governance structure, and CI/CD protocols. **Compliance with this policy is mandatory for all code merged into organization repositories.**

---

## 2. Organization Architecture: Multi-Repo with Topic Monorepos

We utilize a **Hybrid Multi-Repo with Topic Monorepos** structure. 

*   **The Multi-Repo Layer:** Distinct domains reside in separate GitHub repositories under the organization account (e.g., `quant-research`, `low-latency-HFT`, `webdev`).
*   **The Topic Monorepo Layer:** Within each domain repository, multiple independent projects live in separate subfolders.

### Visual Directory Layout Matrix

```text
coep-quant-club/low-latency-HFT/                  <-- Domain Repository (1 .git root)
├── .github/
│   └── workflows/
│       ├── hft-engine-ci.yml                    <-- Path-filtered workflow
│       └── orderbook-ci.yml                     <-- Path-filtered workflow
├── .githubfiles/
│   └── CODEOWNERS                               <-- Automated PR review mapping
├── projects/
│   ├── core-matching-engine/                    <-- Isolated Subfolder Project A
│   │   ├── include/
│   │   ├── src/
│   │   ├── CMakeLists.txt                       <-- Isolated Build Manifest
│   │   └── README.md
│   └── python-gateway/                          <-- Isolated Subfolder Project B
│       ├── src/
│       ├── requirements.txt                     <-- Isolated Python Manifest
│       └── README.md
├── .gitignore
└── README.md

3. Core Architectural Mandates
A. CRITICAL WARNING: Forbidding Nested Git Repositories
[!CAUTION]
NEVER initialize a git repository inside a subfolder of an existing repository.
You must have EXACTLY ONE .git folder tracking engine located strictly at the absolute root of the repository.

Why? Creating sub-repositories (nested .git folders) creates Git Submodules or detached pointer histories accidentally. This breaks GitHub’s UI, hides dependency trees from security scanners, and causes build pipelines to fail silently because the root runner cannot track nested working trees.

Enforcement: Pull requests containing nested .git directories will be automatically rejected by pre-commit hooks and administrative checks.

B. Dependency Isolation Strategy
Each subfolder project must be completely self-contained regarding its dependencies and build configurations.

Never aggregate dependencies at the repository root (e.g., do not put a monolithic requirements.txt or package.json at the root of a topic monorepo).

Mandatory Manifests: Every subfolder must house its own tool-specific native manifest (requirements.txt, pyproject.toml, Cargo.toml, or CMakeLists.txt).

C. Path-Filtered CI/CD Workflows
To avoid running unnecessary heavy builds across the entire monorepo when only one project changes, we leverage GitHub Actions paths filtering.

Below is a production-ready GitHub Actions workflow template (.github/workflows/hft-engine-ci.yml) demonstrating path filtering, targeted directory execution, and dependency installation:

name: HFT Engine CI Pipeline

on:
  push:
    branches: [ "main" ]
    paths:
      - 'projects/core-matching-engine/**'
  pull_request:
    branches: [ "main" ]
    paths:
      - 'projects/core-matching-engine/**'

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ./projects/core-matching-engine
        
    steps:
      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Install System Dependencies (CMake & Build Essentials)
        run: |
          sudo apt-get update
          sudo apt-get install -y cmake build-essential

      - name: Configure CMake
        run: cmake -B build -DCMAKE_BUILD_TYPE=Release

      - name: Build Project
        run: cmake --build build --config Release

      - name: Run Test Suite
        run: ctest --test-dir build --output-on-failure


D. Security & Vulnerability Auditing
Native security engines (such as GitHub Dependabot and Snyk) natively crawl through directory trees looking for isolated manifest files (requirements.txt, package-lock.json, Cargo.lock). Because we enforce dependency separation inside subfolders, native scanning tools automatically detect outdated or vulnerable packages per project without requiring manual root configuration.

4. Governance & Production Guardrails
To prevent coordinators, tech leads, or contributors from accidentally pushing broken builds or bypassing code reviews directly to production, the following governance rules are rigidly enforced.

A. Branch Protection Rules (main)
All topic repository main branches must enforce the following settings via GitHub Repository Settings > Branches:

Require a pull request before merging: Direct pushes to main are hard-blocked for all users, including administrators.

Require status checks to pass before merging: The CI/CD workflows (e.g., HFT Engine CI Pipeline) must report a passing state before the merge button becomes active.

Do not allow bypassing the above settings: Enforce restrictions across all administrative users to maintain systemic integrity.

Require linear history: Disable merge commits where appropriate; enforce squash-merging to keep the commit log clean.

B. Automated Code Ownership Management (CODEOWNERS)
To guarantee domain-specific peer review, repositories must house a .github/CODEOWNERS file. This file automatically assigns designated domain coordinators or technical heads as mandatory reviewers depending on the paths modified in a Pull Request.

Create .github/CODEOWNERS with the following template structure:

Plaintext
# COEP Quant Finance Club - Global Code Owners
# These owners are default reviewers for any unassigned files.
*   @coep-quant/tech-execs

# Domain-Specific Subfolder Owners
/projects/core-matching-engine/    @coep-quant/hft-lead @coep-quant-c++-head
/projects/python-gateway/          @coep-quant/backend-lead
/quant-research/                   @coep-quant/research-heads
5. Summary Checklist for Submitting Code
Before opening a Pull Request to any COEP Quant Finance Club repository, ensure you have verified:

[ ] No nested .git directories exist inside any subfolder.

[ ] Dependencies are cleanly scoped inside the project's subfolder manifest (requirements.txt, CMakeLists.txt, etc.).

[ ] Path filters in .github/workflows/ cover your specific subfolder.

[ ] All tests pass locally and code follows established style conventions.
