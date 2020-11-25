# Security Scans

If your work moves to production, you will definitely want to ensure that you have done your part to identify & resolve any security vulnerabilities present in the source code. We can use various open-sourced libraries to help out. They should, ideally be placed in a precommit, or added to the repository's CI automated pipeline.


## Source Code Scan

Also known as Static Application Security Test (SAST), the most popular python src scanning is [bandit](https://bandit.readthedocs.io/en/latest/). Each vulnerability detected is classified by their severity & confidence level.

 * Installation: `pip install bandit`
 * Single File: `bandit your_file.py`
 * Directory: `bandit -r ~/your_repos/project`
 * Display only High Severities: `bandit -r ~/your_repos/project -lll`
 * Json Output: `bandit --format json --recursive project/ -l --output bandit.json`
 * Skip Certain Vulnerabilities, by placing `.bandit` file at directory to check

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

This scans for vulnerabilities in python libraries. [safety](https://github.com/pyupio/safety) is decent open-sourced library with the free database being updated once a month. However, it does not classify vulnerabilities by severity levels.

 * Installation: `pip install safety`
 * Check installed packages in VM: `safety check`
 * Check requirements.txt, does not include dependencies: `safety check -r requirements.txt`
 * Full Report: `safety check --full-report`
 * Json Output: `safety check --json --output insecure_report.json`


## Secrets Scan

[detect-secrets](https://github.com/Yelp/detect-secrets) scans for the presence of a variety of secrets & passwords. It creates a json output, with the key "results" filled if any secrets are detected.

 * Installation: `pip install detect-secrets`
 * Directory: `detect-secrets scan directory/*`