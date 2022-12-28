# Model Version

The process of model development usually involves various iterations. This can be due to changes in the data, model type, and model hyperparameters, all of which can influence the final evaluation metrics selected.

Therefore, we need a system to track each of the model version we trained, as well as log all the different information we used for each training cycle.

Together with model deployment & monitoring tools, this comprises of the entire ML lifecycle, also known as MLOps.

## MLFlow

There are a number of different model version platforms, like [AWS Sagemaker](https://aws.amazon.com/sagemaker/), and [DVC Pipelines](https://dvc.org/doc/start/data-pipelines) + [CML](https://cml.dev/). However, one of most popular open-source platform is [MLFlow](https://mlflow.org/), which we will demostrate here.

To start, we install MLFlow with `pip install mlflow`, and to launch the user interface, `mlflow ui`. 

For a more verbose specification, to indicate the database, artifacts folder and the server ip (and port) we can do as below.

```bash
cd <project-directory>

mlflow server \
    --backend-store-uri sqlite:///mlflow.db \
    --default-artifact-root ./artifacts \
    --host 0.0.0.0 -p 5000
```


### Python Code

The mlflow tracking and model APIs are very easy to use. It involves the following.

__Starting__

| CMD | Desc |
|-|-|
| `mlflow.set_tracking_uri("http://127.0.0.1:5000")` | point to mlflow server |
| `mlflow.set_experiment("iris")` | create experiment if not exist |
| `mlflow.start_run(run_name="test run")` | give a name for each run (optional) |

__Logging Params, Metrics, Artifacts__

| CMD | Desc |
|-|-|
| `log_param("n_estimators", n_estimators)` | log some parameter |
| `log_metric("f1", f1)` | log evaluation metric |
| `log_artifacts("logs")` | log any files |

Note that we can also (log metrics based on epochs)[https://www.mlflow.org/docs/latest/tracking.html#performance-tracking-with-metrics] of a training so that we can plot the loss over each epoch. This is done by including the step argument in `mlflow.log_metrics(key="loss", value=loss, step=epoch)`.


__Logging Model__

We also also auto-save the model using the model "flavor" API. The model signature, which is the input and output names and format, can also be recorded. All these information will also be stored under artifacts.

```python
signature = infer_signature(X_train, y_predict)
mlflow.sklearn.log_model(
    sk_model=model,
    artifact_path="artifacts",
    registered_model_name="iris-randomforest-classifier",
    signature=signature
)
```

__Code__

```python
import os
from statistics import mean

import matplotlib.pyplot as plt
import mlflow
import mlflow.sklearn
import pandas as pd
from mlflow import log_artifacts, log_metric, log_param
from mlflow.models.signature import infer_signature
from sklearn.datasets import load_iris
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import (f1_score, plot_confusion_matrix,
                             precision_score, recall_score)
from sklearn.model_selection import train_test_split



iris = load_iris()
X = pd.DataFrame(iris["data"], columns=iris["feature_names"])
y = iris.target
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.3, random_state=0
)


def train(n_estimators):
    model = RandomForestClassifier(n_estimators=n_estimators)
    model = model.fit(X_train, y_train)
    return model


def evaluation(model, y_predict):
    f1 = mean(f1_score(y_test, y_predict, average=None))
    precision = mean(precision_score(y_test, y_predict, average=None))
    recall = mean(recall_score(y_test, y_predict, average=None))
    confusion = plot_confusion_matrix(model, X_test, y_test, cmap=plt.cm.Blues)
    plt.savefig("logs/confusion_metrics.png")
    return acc, f1, precision, recall, confusion


def mlflow_logs(n_estimators, acc, f1, precision, recall):
    log_param("n_estimators", n_estimators)
    log_metric("f1", f1)
    log_metric("precision", precision)
    log_metric("recall", recall)
    log_artifacts("logs")


if __name__ == "__main__":
    # start mlflow
    mlflow.set_tracking_uri("http://127.0.0.1:5000")
    mlflow.set_experiment("iris")
    mlflow.start_run(run_name="test run")
    n_estimators = 99

    # train, predict, evaluate
    model = train(n_estimators)
    y_predict = model.predict(X_test)
    acc, f1, precision, recall, confusion = evaluation(model, y_predict)
    
    # mlflow logging
    mlflow_logs(n_estimators, acc, f1, precision, recall)
    signature = infer_signature(X_train, y_predict)
    mlflow.sklearn.log_model(
        sk_model=model,
        artifact_path="artifacts",
        registered_model_name="iris-randomforest-classifier",
        signature=signature
    )
```

### MLFlow UI

This is the interface seen when MLFlow launches.

<figure>
  <img src="https://github.com/mapattacker/ai-engineer/blob/master/images/mlflow-main.png?raw=true" />
  <figcaption>Main UI</a></figcaption>
</figure>

Clicking on a specific run leads to its details.

<figure>
  <img src="https://github.com/mapattacker/ai-engineer/blob/master/images/mlflow-run.png?raw=true" />
  <figcaption>Run Details</a></figcaption>
</figure>

And any artifacts stored. This can include any file, including images.

<figure>
  <img src="https://github.com/mapattacker/ai-engineer/blob/master/images/mlflow-artifacts.png?raw=true" />
  <figcaption>Artifacts</a></figcaption>
</figure>

Or the model.

<figure>
  <img src="https://github.com/mapattacker/ai-engineer/blob/master/images/mlflow-model.png?raw=true" />
  <figcaption>Model artifact & signature</a></figcaption>
</figure>

If we registered the model, it will be recorded in the model registry.

<figure>
  <img src="https://github.com/mapattacker/ai-engineer/blob/master/images/mlflow-modelregistry.png?raw=true" />
  <figcaption>Model registry</a></figcaption>
</figure>