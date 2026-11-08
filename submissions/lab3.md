# Lab 3 Submission ( GitHub Action Workflow Path)

### Steps Applied

- Retrieve SHA for for both checkout/Setup/golangci-action action for a more secured approach of Pinning Tags via `gh api`

```bash
# Retrieve Checkout Action SHA
gh api repos/actions/checkout/git/refs/tags/v4.4.0 --jq '.object.sha'

# Retreive Setup Actions SHA
gh api repos/actions/setup-go/git/refs/tags/v7.0.0 --jq '.object.sha'

# Retrieve Golang CI Actions SHA
gh api repos/golangci/golangci-lint-action/git/refs/tags/v6.5.2 --jq '.object.sha'

# Further Variations applicable across Checkout/Setup Actions
# Get the latest commit SHA of a branch (e.g., main)
gh api repos/{owner}/{repo}/commits/main --jq '.sha'

# Get the commit SHA of a tag (e.g., v4.4.0)
gh api repos/{owner}/{repo}/commits/v4.4.0 --jq '.sha'

# Get the commit SHA from a partial SHA (verifies and returns the full 40th-character SHA)
gh api repos/{owner}/{repo}/commits/a1b2c3d --jq '.sha'
```

- Integrate Jobs incrementally i.e add Vet, Once passed add other subsequent jobs
