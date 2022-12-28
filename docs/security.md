# Security Scans

If your work moves to production, you will definitely want to ensure that you have done your part to identify & resolve any security vulnerabilities present in the source code. We can use various open-sourced libraries to help out. They should, ideally be placed in a precommit, or added to the repository's CI automated pipeline.

Note that a lot of such scans are now available as plugins if you are running in a CI/CD environment, e.g. Gitlab-CI or github workflow.


## Source Code Scan

Also known as Static Application Security Test (SAST), the most popular python src scanning is [bandit](https://bandit.readthedocs.io/en/latest/). Each vulnerability detected is classified by their severity & confidence level.

| Desc | CMD |
|-|-|
| Installation | `pip install bandit` |
| Single File | `bandit your_file.py` |
| Directory | `bandit -r ~/your_repos/project` |
| Display only High Severities | `bandit -r ~/your_repos/project -lll` |
| Json Report | `bandit --format json --recursive project/ -l --output bandit.json` |


We can skip certain vulnerabilities, by placing `.bandit` file at directory to check with the contents as such.

```
[bandit]
skips: B104,B101
```

Here is a bash script to detect the presence of high vulnerabilities.

```bash
bandit -r ../$module -lll
bandit --format json --recursive ../$module --output bandit.json
high_count=$(cat bandit.json | jq '[.results [] | select(.issue_severity=="HIGH")]' | jq '. | length')
```

## Dependency Scan

This scans for vulnerabilities in python libraries. [safety](https://github.com/pyupio/safety) is decent open-sourced library with the free database being updated once a month. However, it does not classify vulnerabilities by severity levels, nor does it scan for libraries which have their own open-source dependencies.

| Desc | CMD |
|-|-|
| Installation | `pip install safety` |
| Check installed packages in VM | `safety check` |
| Check requirements.txt, does not include dependencies | `safety check -r requirements.txt` |
| Full Report | `safety check` |
| Json Report | `safety check --json --output insecure_report.json` |


## Secrets Scan

[detect-secrets](https://github.com/Yelp/detect-secrets) scans for the presence of a variety of secrets & passwords. It creates a json output, with the key "results" filled if any secrets are detected. There are other libraries that does the same job, like [aws's git-secrets](https://github.com/awslabs/git-secrets).

| Desc | CMD |
|-|-|
| Installation | `pip install detect-secrets` |
| Directory | `detect-secrets scan directory/*` |

<hr>

## License Scan

Even if we are using open-sourced libraries, we must be aware of the licensing of each library, especially if we are using them for commercial purposes. There are various scanners out there that can help us to compile the license and if you are able to use or modify them freely.

## IaC Scan

Infrastructure as Code (IaC) Scanning scans your IaC configuration files for known vulnerabilities. [Kics](https://kics.io), an open-source library, supports scanning of configuration files for Terraform, Ansible, Docker, AWS CloudFormation, Kubernetes, etc.


## Pre-Commit

Pre-commit is a git hook that you preconfig to run certain scripts, in this case, the above ones before committing to git. This prevents the hassle of committing sensitive info, or avoid the hassle of cleaning up the git history later.

A useful compilation is done by [laac.dev](https://www.laac.dev/blog/automating-convention-linting-formatting-python/).


 * Installation: `pip install pre-commit`
 * create config file at root of project: `.pre-commit-config.yaml`
 * Installation (into git hook): `pre-commit install`
 * Uninstallation (from git hook): `pre-commit uninstall`
 * Add Files for Commit: `git add files.py`
 * Run Commit, and Precommit will autorun: `git commit -m 'something'`
 * Skip Hook: `SKIP=flake8 git commit -m "something"`

Here is an example of the `.pre-commit-config.yaml`

```yml
default_language_version:
  python: python3
repos:
  - repo: https://github.com/PyCQA/bandit
    rev: 1.7
    hooks:
      - id: bandit
        args: [-lll]
  - repo: https://github.com/Yelp/detect-secrets
    rev: v0.13.0
    hooks:
      - id: detect-secrets
        args: [--no-base64-string-scan]
```

Another alternative which I prefer is using tox to compile all the testing and security scans together in a single file. See my section on [testing](https://mapattacker.github.io/ai-engineer/testing1/#tox) for more.


## Synk Advisor

[Synk Advisor](https://snyk.io/advisor/) is a great site to check the security of open-source libraries. Besides checking using known security vulnerabilities as a metric, it also establishes an overall health score using the library's popularity, maintenance, and community.

![](https://github.com/mapattacker/ai-engineer/blob/master/images/security-synk.png?raw=true)