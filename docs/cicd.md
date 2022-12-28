# CI/CD

Continuous Integration and Continuous Delivery/Deployment is an important part of DevOps, where a standard mode of operations are automated in a pipeline frequently, and reliably.

<figure>
  <img src="https://github.com/mapattacker/ai-engineer/blob/master/images/cicd.png?raw=true" />
  <figcaption>Differences between CI-CD-CD.<a href="https://www.atlassian.com/continuous-delivery/principles/continuous-integration-vs-delivery-vs-deployment/"> Source</a></figcaption>
</figure>

This process is usually integrated with a version control platform like Github and Gitlab. Both popular platforms provide in-built CI/CD automation called [Github Actions](https://github.com/features/actions) and [Gitlab-CI](https://docs.gitlab.com/ee/ci/). They are largely free, with some paid add-on tools. Other popular ones include [CircleCI](https://circleci.com) and [TravisCI](https://travis-ci.com/plans).


## Gitlab-CI

### Basics

Gitlab CI is run by "runners", which are essentially google servers spun up when a __job__ is activated. A series of jobs make up a __pipeline__, while certain jobs of a specific attribute are assigned to a __stage__. By default, all jobs in a stage needs to finish before the next stage's jobs starts.

A typical pipeline workflow consists of the various stages.

 1. Test: security scans, unit-tests, integration tests
 3. Build: building a docker image
 4. Deploy: deployment of image to dev/staging/production

<figure>
  <img src="https://github.com/mapattacker/ai-engineer/blob/master/images/ci-pipelines.png?raw=true" />
  <figcaption>A pipeline with various stages, and individual jobs, shown in Gitlab's interface</a></figcaption>
</figure>

### Yaml File

Gitlab-CI will auto run if a file called `.gitlab-ci.yml` is stored in the root of the repository. Inside the ymal file, containing mostly shell commands that tells runner what to execute. It can also refer to scripts stored in the repo so as to keep the ymal file short and concise.


## Github Actions

### Yaml File

Github Actions will auto run if one or more ymal file is located in the folder `.github/workflows/`.