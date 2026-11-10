# Lab 3 Submission ( GitHub Action Workflow Path)

### Steps Applied

- *Retrieve SHA for for both checkout/Setup/golangci-action action for a more secured approach of Pinning Tags via `gh api`*

```bash
# Retrieve Checkout Action SHA
gh api repos/actions/checkout/git/refs/tags/v4.4.0 --jq '.object.sha'

# Retreive Setup Actions SHA
gh api repos/actions/setup-go/git/refs/tags/v7.0.0 --jq '.object.sha'

# Retrieve Golang CI Actions SHA
gh api repos/golangci/golangci-lint-action/git/refs/tags/v8.0.0 --jq '.object.sha'

# Further Variations applicable across Checkout/Setup Actions
# Get the latest commit SHA of a branch (e.g., main)
gh api repos/{owner}/{repo}/commits/main --jq '.sha'

# Get the commit SHA of a tag (e.g., v4.4.0)
gh api repos/{owner}/{repo}/commits/v4.4.0 --jq '.sha'

# Get the commit SHA from a partial SHA (verifies and returns the full 40th-character SHA)
gh api repos/{owner}/{repo}/commits/a1b2c3d --jq '.sha'
```

- *Integrate Jobs incrementally i.e add Vet, Once passed add other subsequent jobs*

- *Triggered PR gate failure by commenting out `app/handlers.go` dependencies*
This ensure CI test fail due to code base not meeting baseline health checks of build not meeting branch protection rule which enforces CI test passing before merging into main branch.

- *Design Questions*

#### a) Why pin the runner version (`ubuntu-24.04`) instead of `ubuntu-latest`? What breaks otherwise?

- **Answer:** Pinning the runner version ensures **reproducibility** and prevents **environment drift**.
- **What breaks otherwise:** `ubuntu-latest` dynamically updates its underlying OS, pre-installed toolchains, compilers, and system dependencies over time. A workflow that passes today could suddenly fail tomorrow due to breaking changes in an auto-upgraded dependency or OS utility, making build failures difficult to diagnose.

---

#### b) Why split vet + test + lint into separate units? What would happen with one combined job?

- **Answer:** Splitting these tasks into individual jobs improves **build performance via parallelism** and offers **fail-fast feedback**.
- **What breaks/happens otherwise:**
  - **Parallel Execution:** Separate jobs run concurrently across multiple runner nodes, significantly reducing total wall-clock pipeline time compared to executing them sequentially.
  - **Clear Isolation:** If linting fails, it fails instantly in its own job scope. A single combined job creates execution bottlenecks and forces the developer to sift through a long, monolithic log to determine whether the failure was caused by a syntax lint error, static analysis, or a broken unit test.

---

#### c) GitHub Actions path: What real attack does SHA pinning prevent? Cite the date + name of the incident from Lecture 3.

- **Answer:** SHA pinning (e.g., `uses: actions/checkout@a5ac7e51b41094f92474775d793ce702da05b262`) ensures exact code immutability, preventing **supply-chain attacks** (such as tag-hijacking or compromised maintainer accounts where a mutable git tag like `@v1` is overwritten to point to malicious code).
- **Incident Reference:** _(Be sure to insert the exact date + name from your Lecture 3 slides here, e.g., Codecov Bash Uploader Breach / April 2021)_.

---

#### d) GitHub Actions path: What is `permissions:` and what's the principle behind it?

- **Answer:** The `permissions:` key explicitly restricts or grants access privileges for the automatically generated `GITHUB_TOKEN` scoped to a specific workflow or job (e.g., restricting default access to read-only `contents: read`).
- **Underlying Principle:** It enforces the **Principle of Least Privilege (PoLP)**. Minimizing default token permissions limits the potential blast radius if a third-party action or build step is compromised, ensuring it cannot make unauthorized commits, alter pull requests, or issue short-lived OIDC tokens without explicit authorization.

---

#### e) GitLab CI/CD path: What's the difference between a stage and a job? What would `dependencies:` do that `stages:` doesn't?

- **Answer:**
  - **Stage vs. Job:** A **stage** defines a sequential pipeline phase (e.g., `build`, `test`, `deploy`). A **job** is an individual execution task defined _within_ a stage. Jobs in the same stage execute concurrently in parallel; stages execute sequentially from top to bottom.
  - **`stages:` vs `dependencies:`:** `stages:` controls the overall execution order of the pipeline phases. `dependencies:` provides fine-grained control over **artifact passing**, specifying exactly which upstream jobs' artifacts should be downloaded by a downstream job, rather than downloading artifacts from all jobs in previous stages.


- *Caching Dependencies*
Integrating caching into CI build can Optimize build time of the pipeline, increased reliability/resilence and reduce the amount compute resource consumed.