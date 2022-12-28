# MLOps

A machine learning lifecycle doesn't follow the typical software development lifecycle, though the essential parts of **automation**, **traceability**, and **monitoring** are still key to its success. This is where a modified form of CI/CD for machine learning comes in, i.e. MLOps.

## Maturity Model

A maturity model is a tool that helps people assess the current effectiveness of a person or group and supports figuring out what capabilities they need to acquire next in order to improve their performance. 

We can use it as a metric for establishing the progressive requirements needed to measure the maturity of a machine learning production environment and its associated processes.

Both [Google](https://cloud.google.com/architecture/mlops-continuous-delivery-and-automation-pipelines-in-machine-learning) and [Azure](https://docs.microsoft.com/en-us/azure/architecture/example-scenario/mlops/mlops-maturity-model) provide a good detail description of an MLOps maturity model. 

I have taken part of Azure's and summarised it below.

<div class="md-typeset__scrollwrap">

<table>
    <thead>
        <tr>
            <th>-</th>
            <th>1: NO MLOps</th>
            <th>2: DevOps, No MLOps</th> 
            <th>3: Auto Training</th>
            <th>4: Auto Deployment</th>
            <th>5: Auto Retraining</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <th>Highlights</th>
            <th>Entire lifecycle is manual</th> 
            <th>Automated app builds and unit-tests</th>
            <th>Training is centrally managed & traceable</th>
            <th>Full traceability from deployment back to original data</th>
            <th>Full system automation and monitoring</th>
        </tr>
    </tbody>
</table>