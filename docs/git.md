# Git

Git is the most popular source code version control system. Popular software development hosts like Github, Gitlab, Bitbucket all uses git. It is essential for two main purposes:

 1. __Branches__: allows different components to be developed concurrently and merged to a single branch later
 2. __Version Control__: allows rollback of code

<hr>

## Access from Dev Env to Git Host

 1. In your terminal > `ssh-keygen`
 2. Accept all defaults
 3. Copy the public key `cat ~/.ssh/id_rsa.pub`
 4. Go to Github/Gitlab Settings > SSH Keys, and paste the keys here

## Remote URL

When using the first git clone to pull the repository to your local machine, we need to specify to use https or ssh. To me it is better for the latter to avoid multiple logins whenever we want to push changes to the remote repository.

```bash
# git repository via ssh
git clone git@github.com:mapattacker/ai-engineer.git
# show remote url
git remote -v
# switch remote url to https
git remote set-url origin https://github.com/mapattacker/ai-engineer.git
```

<hr>

## Branches

Branches plays a key role in git. This allows multiple users to coordinate in a project, which each working on a separate branch. As a rule of thumb, we do not push any codes directly to the master branch. The owner of the repository will usually disable this option.

### Commands

```bash
# check which branch you are at
git branch
# pull updates from remote
git fetch
# change to dev branch
git checkout dev
# create a new local branch
git checkout -b <branchname>

# delete local branch
git branch -d <branchname>
# delete remote branch
git push origin -d <branchname>
```

### Workflow

Below is a workflow typical in a project team. Image sourced from [Buddy](https://buddy.works/blog/5-types-of-git-workflows).

![](https://github.com/mapattacker/ai-engineer/blob/master/images/gitflow.png?raw=true)

 1. Create a new branch for a new feature
 2. Work on the new feature
 3. Commit & push a small working component
 4. Send a merge request to e.g. dev branch
 5. Tech lead approve, code is merged to dev branch

### Merge Request

Also known as Pull Request, this is an option in e.g., Github GUI, to merge the code from one branch to another, e.g. dev to master. The branch owner will validate the code changes, give comments, and accept/reject the request.

<hr>

## Version Control

Prior to commit our code, we should check the changes we made.

```bash
# check for any changes for all files
git status
# check for code changes in a file
git diff <filename>
```

### Commands

To commit your code, we first want to resolve any potential conflicts from the branch we want to merge to later (in case, new updates occurred). Then, we add the file or all files in a folder, followed by a commit msg, and then push to the remote (hosting service like github).

```bash
git merge <branchtomergelater>
git add <fileOrfolder>
git commit -m "your commit msg"
git push
```

It is essential to commit small, bite-size code changes to prevent the reviewer from being overwhelmed when validating your code during a merge request.

### Resets

Sometimes, we might accidentally add or commit our changes erroneously. To reset them, we need to use the `reset` command.

```bash
# put latest commit history to staging
git reset --soft HEAD~1

# remove all staging files
git reset
# remove specific staging file
git reset HEAD <file.py>
```

### Revert

If we want to revert commits in the remote, we can use the following.

```bash
# git push -f origin <commit-hash>:<branch-name>
git push -f origin 4a30214b819:master
```

### Workflow

The various git commands and how they interact at different stages are as illustrated below.

![](https://github.com/mapattacker/ai-engineer/blob/master/images/git.png?raw=true)

<hr>

## Release Tags

We can add tags, usually for release versions, so that it is easy to revert back to a specific version within a branch.

| CMD | Desc |
|-|-|
| `git tag -a v1.0.0 -m "1st prod version"` | tag in local |
| `git tag -a v1.0.0 -m "1st msg" -m "2nd msg"` | tag multiple msgs in new lines |
| `git push origin v1.0.0` | push to remote |
| `git tag -d v1.0` | delete local tag ONLY |
| `git push --delete origin v1.0.0` | delete remote tag ONLY |
| `git tag` | list tags |

## Remove last Commit

At times, we want to remove the last commit from remote. Use the following to do a hard reset of the head and force push to the origin.

```bash
git reset --hard HEAD^
git push origin -f
```

## Squash all Commits

If, for whatever reason, we wish to squash all commits to the head commit, e.g. to clear all commits for a template repo. 

```bash
git reset $(git commit-tree HEAD^{tree} -m "squash all commits to head")
git push --force
```

<hr>

## Delete from Git History

[BFG Repo-Cleaner](https://rtyley.github.io/bfg-repo-cleaner/) is a 3rd party java file used to remove files that are not accidentally uploaded, e.g., passwords, blobs. Download their file from the website and follow the instructions there, or from the example below.

We cannot delete specific files and folders based on path, the reasoning given by the developer of BFG in [stackoverflow](https://stackoverflow.com/questions/21142986/remove-filenames-from-specific-path).

| CMD | Desc |
|-|-|
| `git clone --mirror git@gitlab.com:project/repo-name.git` | clone only the .git |
| `java -jar bfg-<version>.jar --delete-files "*.{png,jpg,gif}" repo-name.git` | delete certain file extensions |
| `java -jar bfg-<version>.jar --delete-folders <foldername> repo-name.git` | delete all folders with the folder model |
| `cd repo-name.git` | go into git directory |
| `git reflog expire --expire=now --all && git gc --prune=now --aggressive` | delete old files |
| `git push --force` | push updated git to remote |

After uploading the new git history to the repository, it is important to not push any of you or your team's legacy git history back to the repo.