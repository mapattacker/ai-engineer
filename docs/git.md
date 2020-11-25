# Git

Git is the most popular source code version control system. Popular software development hosts like Github, Gitlab, Bitbucket all uses git. It is essential for two main purposes:

 1. __Version Control__: allows rollback of code
 2. __Branches__: allows different components to be developed concurrently and merged to a single branch later

<br>

The process works roughly like this:

 1. Create a new branch
 2. Work on a new feature
 3. Commit & push a small working component
 4. Send a merge request to e.g. dev branch
 5. Tech lead approve, code is merged to dev branch


## Branches

Branches plays a key role in git. This allows multiple users to coordinate in a project, which each working on a separate branch. As a rule of thumb, we do not push any codes directly to the master branch. The owner of the repository will usually disable this option.

```bash
# check which branch you are at
git branch
# pull updates from remote
git fetch
# change to dev branch
git checkout dev
# create a new local branch
git checkout -b <featurename>
```

## Commit Code

Prior to commit our code, we should check the changes we made.

```bash
# check for any changes for all files
git status
# check for code changes in a file
git diff <filename>
```

To commit your code, we first want to resolve any potential conflicts from the branch we want to merge to later (in case, new updates occurred). Then, we add the file or all files in a folder, followed by a commit msg, and then push to the remote (hosting service like github).

```bash
git merge <branchtomergelater>
git add <fileOrfolder>
git commit -m "your commit msg"
git push
```

It is essential to commit small, bite-size code changes to prevent the reviewer from being overwhelmed when validating your code during a merge request.


## Merge Request

Also known as Pull Request, this is an option in e.g., Github GUI, to merge the code from one branch to another, e.g. dev to master. The branch owner will validate the code changes, give comments, and accept/reject the request.

