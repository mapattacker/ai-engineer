# Data Version Control

[DVC](https://dvc.org) is a library for ML versioning of blobs, which can include data and/or models. For this documentation, we will be using an example where we store the data versions into AWS S3 bucket. To start off, install the dvc library, together with AWS S3's accompanying library 

```bash
pip install dvc[s3]
```

## DVC INIT

`dvc init` will add a `.dvcignore` file, and more importantly, a `.dvc` folder. Within the latter, it contains: 
    
 * `cache` folder: stores the data versions & corresponding hashes, which will be uploaded to the S3. These will not be uploaded to the repository, as there is a `.gitignore` file automatically created, that excludes this folder.
 * `config` file: stores the s3 bucket URLs

## Add S3 Keys

To allow DVC to push & pull data via S3 bucket, we need to set the AWS Access & Secret Keys, either through AWS CLI `aws configure`, or save them as environment variables in the OS.

```bash
export AWS_SECRET_ACCESS_KEY="xxx"
export AWS_ACCESS_KEY_ID="xxx"
```

## Add Remote Link

We then add the remote links, by giving it a name, and the S3 URL. You can see that we are classifying the links as folders within an S3 bucket

```bash
dvc remote add babystroller s3://images/babystroller
dvc remote add rubbishbin s3://images/rubbishbin
# add as a default remote
dvc remote add -d safetycone s3://images/safetycone
```

This will update the `.dvc/config` file as shown below.

```
[core]
    remote = safetycone
['remote "babystroller"']
    url = s3://images/babystroller
['remote "safetycone"']
    url = s3://images/safetycone
['remote "rubbishbin"']
    url = s3://images/rubbishbin
```

## Push Data to S3

We create a folder and add the datasets in there. This will create a version of the data. The details are as follows:

 * Add folder <foldername> with data
 * `dvc add <foldername>`
 * a file called `<foldername>.dvc` will be created
 * `.dvc/cache` folder containing the data version & hashes will be created
 * the `<foldername>` is automatically added to `.gitignore` so that the data will not be uploaded to the repository
 * Push data version to S3: `dvc push -r <remote name> <foldername>.dvc`

We need to be mindful to set the remote name to the correct link else it will just use the default (core) link.


## Update Git

We then git commit the `<foldername>.dvc`, `.gitignore` & `.dvc/config` (if there are any additions of the remote links) to the repository. The git commit message/hash will be the basis to access the data version.

## Retrieve Data from S3

We can pull from specific dvc folders by using `dvc pull -r <remote name> <foldername>.dvc`

## Full Code Example

```bash
# add remote link
dvc remote add rubbishbin s3://images/rubbishbin

# assume folder called rubbishbin is added with data inside
# store data version to S3
dvc add rubbishbin
dvc push -r rubbishbin rubbishbin.dvc

# git commit
git add rubbishbin.dvc .dvc/config .gitignore
git commit -m "Add init rubbishbin dataset"
git push

# pull data
dvc pull -r rubbishbin rubbishbin.dvc
```