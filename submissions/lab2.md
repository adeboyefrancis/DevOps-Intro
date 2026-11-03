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
