# Data Version Control

[DVC](https://dvc.org) is a popular data version control library that implements data versioning together with Git. 

The purpose of data versioning in MLOps is that we can trace for each training run, what is the data version used.


## TL;DR

```bash
# setting up -------
dvc init
dvc remote add <remote-name> <s3-bucket-url>

git add .dvc/
git commit -m "add dvc config"
git push


# store a data version -------
dvc add -r <remote-name> <data-directory>
dvc push

git add <data-directory>.dvc
git commit -m "data-version-commit-message"
git push


# pull a data version -------
git pull # OR
git checkout <git-commit-hash>

dvc pull -r <remote-name> <data-directory>.dvc
```


## Setting Up

### Install

```bash
pip install dvc[s3]
```

### Initialisation

```
cd <repository-root>
dvc init
```

Initialisation creates the following:

 * `.dvcignore` file: exclude files from DVC
 * `.dvc` folder
    * `cache` folder: stores data version and corresponding hash keys. They will not be uploaded to git as a `.gitignorefile` is added to this folder to exclude
    * `config` file: stores all dvc remote urls (e.g. s3 bucket links)


### Add Remote Link

We then add the remote links, by giving it a name, and the S3 URL. You can see that we are classifying the links as folders within an S3 bucket

```bash
# dvc remote add <remote-name> <s3-bucket-link>
dvc remote add tp1 s3://rre/data/tp1
```

This will update the `.dvc/config` file as shown below.

```bash
['remote "tp1"']
    url = s3://rre/data/tp1
```

To assign a default remote, add a `-d` option

```bash
dvc remote add -d tp2 s3://rre/data/tp2
```

The updated config will be

```bash
[core]
    remote = tp2
['remote "tp1"']
    url = s3://rre/data/tp1
['remote "tp2"']
    url = s3://rre/data/tp2
```

Commit the config file to your git repo

```bash
git add .dvc/
git commit -m "Add dvc config"
git push
```

## Store Data Version

DVC manages data versioning using two ways

 1. Publishing a data version to a remote (i.e. S3) using *DVC commands*
 2. Committing the data version’s hash key to a remote git repository using *Git commands*

<figure>
  <img src="https://github.com/mapattacker/ai-engineer/blob/master/images/dvc-store1.png?raw=true" style="width:100%" />
  <figcaption></figcaption>
</figure>

<figure>
  <img src="https://github.com/mapattacker/ai-engineer/blob/master/images/dvc-store2.png?raw=true" style="width:100%" />
  <figcaption></figcaption>
</figure>

### Committing a Data Version

 * Place your first version of your data to a directory, e.g., /data
 * Add this data version to staging

```bash
dvc add <data-directory>/
```

 * A file `<data-directory>.dvc` is created, containing the hash key of this data version
 * Push the data version to remote

```bash
# if you have set a default remote
# and only one <data-directory>.dvc file in directory
dvc push

# more specific command
dvc push -r <remotelink-name> <data-directory>.dvc
```

### Committing a Data Version Hash

 * We need to commit the <data-directory>.dvc file to git so that we can retrieve the hash key for the data version based on the git commit message, or any tags

```bash
git add <data-directory>.dvc
git commit -m "data version 1"
git push
```

## Pull Data Version

To extract a data version, we need to

 1. pull the appropriate `<data-directory>.dvc` which contains the data version hash, using `git`
 2. pull the data version from S3, using `dvc`

<figure>
  <img src="https://github.com/mapattacker/ai-engineer/blob/master/images/dvc-pull1.png?raw=true" style="width:100%" />
  <figcaption></figcaption>
</figure>

<figure>
  <img src="https://github.com/mapattacker/ai-engineer/blob/master/images/dvc-pull2.png?raw=true" style="width:100%" />
  <figcaption></figcaption>
</figure>

```bash
# pull current data version hash of <data-directory>.dvc 
git pull

# pull previous data version hash of <data-directory>.dvc
git checkout <commit-hash>

# pull the data version
dvc pull
# or more specific
dvc pull -r <remotelink-name> <data-directory>.dvc
```

## Read Data Version

The python API allows you to read **single files (only)** directly from the DVC remote. This enables you to easily integrate DVC to your data/training pipeline script.

See this [link](https://dvc.org/doc/api-reference/read) for full list of arguments.

```python
import pickle
import dvc.api

text_file = dvc.api.read(path="<path-to-file-relative-to-repo-root>")

pickle_file = dvc.api.read(path="<path-to-file-relative-to-repo-root>", model="rb")
pickle_file = pickle.loads(pickle_file)
```