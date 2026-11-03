### Quick Reference Cheatsheet

[Git Internals](https://cheatsheets.davidveksler.com/git-internals.html)

```bash
# Useful Plumbing Commands
git cat-file -t SHA	# Show object type: blob, tree, commit, or tag.
git hash-object -w FILE	# Compute an object ID and optionally write a blob.
git cat-file -p SHA	# Pretty-print object contents.
git update-index	# Manipulate the index at the plumbing level.
git write-tree	# Create tree objects from the current index.
git commit-tree	# Create a commit object from a tree and parent(s).
git ls-files --stage	# Show index entries and conflict stages.
git rev-parse HEAD	# Resolve a revision or ref to an object ID.
git show-ref	# List refs and the object IDs they point to.
git symbolic-ref HEAD	# Read or update symbolic refs such as HEAD.
git merge-base A B	# Find the common ancestor used for a three-way merge or rebase.
git verify-pack -v	# Inspect objects and deltas inside a pack index.
git fsck	# Verify object database integrity and find unreachable objects.
git reflog <ref>	# Show previous values of a local ref.
```
