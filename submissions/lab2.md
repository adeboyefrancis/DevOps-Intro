# Lab 2 Submission

```bash
# Outputs
git rev-parse HEAD  #----> dee1800de7a56922e14841fa5fd53bb41f8309b7
git cat-file -t HEAD #----> commit (Object Type)
git cat-file -p HEAD #----> retrieve SHA of tree,parent,author,gpgsign private key
git cat-file -p HEAD | grep -i tree | awk '{print $2}' #---> query and explicity prints tree SHA cfdc8893963c102d6bd5b1b36e387026d349f77a
git cat-file -p cfdc889 | grep blob #---> drill down into tree to fetch  blob SHA using the first few char of tree sha
git cat-file -p 4137b #----> View contents of blob object, similar to linux cat README.md - 100=first3 prefix type of file and 644=last 3 suffix indicates the filetype (owner=RW, group=R , other=R) using Octal permission type, This can be verified by running stat app/README.md
```

## Explore Internals File Structure.git/

```bash
ls -la .git/ # 1) TLD , can also user unix/linux utility `tree -d or tree -a` for drill down into file structure
# Output
.rw-r--r--@  128 adeboyefrancis  3 Nov 21:39 󰊢 COMMIT_EDITMSG
.rw-r--r--@  468 adeboyefrancis  3 Nov 21:44 󱁻 config
.rw-r--r--@   73 adeboyefrancis  2 Nov 21:56 󰡯 description
.rw-r--r--@   98 adeboyefrancis  3 Nov 22:14 󰡯 FETCH_HEAD
.rw-r--r--@   21 adeboyefrancis  3 Nov 21:43 󰡯 HEAD  #--> 2) Points to main branch: cat .git/HEAD
drwxr-xr-x@    - adeboyefrancis  2 Nov 21:56  hooks
.rw-r--r--@ 3.8k adeboyefrancis  3 Nov 21:44 󰡯 index
drwxr-xr-x@    - adeboyefrancis  2 Nov 21:56  info
drwxr-xr-x@    - adeboyefrancis  2 Nov 21:56  logs
drwxr-xr-x@    - adeboyefrancis  3 Nov 22:14  objects #---> 4) Core Object subdir (TREE,BLOB,COMMIT)+SHAs : ls .git/objects/ | head  or ls .git/objects/ | tail  depending on sorting order
.rw-r--r--@   41 adeboyefrancis  3 Nov 21:44 󰡯 ORIG_HEAD
.rw-r--r--@  112 adeboyefrancis  2 Nov 21:56 󰡯 packed-refs
drwxr-xr-x@    - adeboyefrancis  2 Nov 21:56  refs #---> 3) Shows branches: ls ./git/refs/heads

# 5) Count objects using `find , pipe , wc -l` -> Output 55
find .git/objects -type f | wc -l
```

## Simulate a disaster and recovery using Reflog

```bash
# Reflog - Last Resort
git reflog
#Output
8e578ae (HEAD -> feature/lab2) HEAD@{1}: reset: moving to HEAD~2
6085ccb HEAD@{2}: commit: WIP(test-lab2): start
21f508b (origin/main, origin/HEAD, main) HEAD@{3}: checkout: moving from main to feature/lab2
21f508b (origin/main, origin/HEAD, main) HEAD@{4}: pull: Fast-forward
8e578ae (HEAD -> feature/lab2) HEAD@{5}: checkout: moving from feature/misc-notes to main
5987ca8 (origin/feature/misc-notes) HEAD@{6}: commit: docs(notes): Added Mental Model Diagrams and Quick Reference sheet
8e578ae (HEAD -> feature/lab2) HEAD@{7}: checkout: moving from main to feature/misc-notes
8e578ae (HEAD -> feature/lab2) HEAD@{8}: pull: Fast-forward
dee1800 (tag: v0.1.0-lab2-cr258419) HEAD@{9}: pull: Fast-forward
24e29cb HEAD@{10}: checkout: moving from feature/signed-commit to main
010ce7e HEAD@{11}: commit: docs: POC signed commit from Home Lab
24e29cb HEAD@{12}: checkout: moving from main to feature/signed-commit
24e29cb HEAD@{13}: clone: from github.com:adeboyefrancis/DevOps-Intro.git

# Recovery the prodigal commit
git reset --hard 8e578ae
HEAD is now at 8e578ae Merge pull request #3 from adeboyefrancis/feature/lab2

# Garbage Collection( git gc ) would have lead to total lost of data/work during
# reset and recovery of commit.

```

## Annotate and Tag Release (Semantic Versioning)

```bash
git tag -v "v0.1.1-lab2-${USER}"
object 21f508beef3e08b7c9243c36a2628914f44a0075
type commit
tag v0.1.1-lab2-adeboyefrancis
tagger Francis Adeboye <adeboye.francis@icloud.com> 1793746370 +0000

Lab 2 milestone — version control deep dive
Good "git" signature for adeboye.francis@icloud.com with ED25519 key SHA256:kc4diaZZtW4t65rcLjTxRm/KDAN9GDbuJnIVE5UHul0
```
