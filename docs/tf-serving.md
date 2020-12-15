# Tensorflow Serving

Tensorflow Serving, developed by Google, allows fast inference using gRPC (and also REST). It eliminates the need for web server, and talks directly to the model. Some of the other advantages, stated from the official [github](https://github.com/tensorflow/serving) site includes: 

 * Can serve multiple models, or multiple versions of the same model simultaneously
 * Exposes both gRPC as well as HTTP inference endpoints
 * Allows deployment of new model versions without changing any client code
 * Supports canarying new versions and A/B testing experimental models
 * Adds minimal latency to inference time due to efficient, low-overhead implementation
 * Features a scheduler that groups individual inference requests into batches for joint execution on GPU, with configurable latency controls
 * Supports many servables: Tensorflow models, embeddings, vocabularies, feature transformations and even non-Tensorflow-based machine learning models

Much of the learnings of this page came from a course from Coursera called [TensorFlow Serving with Docker for Model Deployment](https://www.coursera.org/projects/tensorflow-serving-docker-model-deployment). Do sign up for this free course for a better sense of things.

## Save Model as Protobuf

We need to use `tensorflow.save_mode.save()`, or tf.keras's `model.save(filepath=file_path, save_format='tf')` API to save the trained model in a protobuf format, e.g. `model.pb`.

```python
import os
import time

import tensorflow as tf

base_path="amazon_review/"
path = os.path.join(base_path, str(int(time.time())))
tf.saved_model.save(model, path)
```

This is how a model directory & its contents look like, with each model version stored in a time-stamped folder. With the timestamp, it allows automated canary deployment when a new version is created.

```bash
├── amazon_review
│ ├── 1600788643
│ │ ├── assets
│ │ ├── saved_model.pb
│ │ └── variables
```

## TensorFlow Serving with Docker

It is easiest to serve the model with docker, as described from the official [website](https://www.tensorflow.org/tfx/serving/docker).

Below is an example, where we link the model to the dockerised tensorflow-serving image, and expose both gRPC & REST ports.

```bash
docker pull tensorflow/serving
docker run -p 8500:8500 \
           -p 8501:8501 \
           --mount type=bind,\
           source=/path/to/model_folder/,\
           target=/models/model_folder \
           -e MODEL_NAME=model_name \
           -t tensorflow/serving
           --name amazonreview
```

| CMD | Desc |
|-|-|
| `-p 8500:8500` | expose gRPC port |
| `-p 8501:8501` | expose REST port |
| `--mount type=bind,source=/path/to/model_folder/,target=/models/model_folder` | copy model from local folder to docker container folder |
| `-e MODEL_NAME=model_name` | name of the model, also used to define serving endpoint |
| `--name amazonreview` | name of docker container |


## REST API

As with all REST APIs, we can use python, CURL or Postman to send our requests. However, we need to be aware that, by default:

 * The request JSON is `{"instances": [model_input]}`, with model_input as a list
 * The endpoint is `http://{HOST}:{PORT}/v1/models/{MODEL_NAME}:{VERB}`

 | CMD | Desc |
|-|-|
| `HOST` | domain name or IP address |
| `-p 8501:8501` | default 8501 |
| `MODEL_NAME` | name of model defined in docker instance |
| `VERB` | model signature. either `predict`, `classify`, or `regress` |

Below is an example using CURL

```bash
curl -d '{"instances": [1.0, 2.0, 5.0]}' \
    -X POST http://localhost:8501/v1/models/amazon_review:predict
```

Below is an example using python. More [here](https://neptune.ai/blog/how-to-serve-machine-learning-models-with-tensorflow-serving-and-docker)

```python
import json
import requests
import sys

def get_rest_url(model_name, host='127.0.0.1', port='8501', verb='predict', version=None):
    """ generate the URL path"""
    url = "http://{host}:{port}/v1/models/{model_name}".format(host=host, port=port, model_name=model_name)
    if version:
        url += 'versions/{version}'.format(version=version)
    url += ':{verb}'.format(verb=verb)
    return url


def get_model_prediction(model_input, model_name='amazon_review', signature_name='serving_default'):
    url = get_rest_url(model_name)
    data = {"instances": [model_input]}
    
    rv = requests.post(url, data=json.dumps(data))
    if rv.status_code != requests.codes.ok:
        rv.raise_for_status()
    
    return rv.json()['predictions']

if __name__ == '__main__':

    url = get_rest_url(model_name='amazon_review')
    model_input = "This movie is great! :D"
    model_prediction = get_model_prediction(model_input)
    print(model_prediction)
```


## gRPC Client

To use gRPC for tensorflow-serving, we need to first install it via `pip install grpc`. There are certain requirements needed for this protocol, namely:

 * Prediction data has to be converted to the Protobuf format
 * Request types have designated types, e.g. float, int, bytes
 * Payloads need to be converted to base64
 * Connect to the server via gRPC stubs


Below is an example of a gRPC implementation in python.

```python
import sys
import grpc
from grpc.beta import implementations
import tensorflow as tf
from tensorflow_serving.apis import predict_pb2
from tensorflow_serving.apis import prediction_service_pb2, get_model_metadata_pb2
from tensorflow_serving.apis import prediction_service_pb2_grpc


def get_stub(host='127.0.0.1', port='8500'):
    channel = grpc.insecure_channel('127.0.0.1:8500') 
    stub = prediction_service_pb2_grpc.PredictionServiceStub(channel)
    return stub


def get_model_prediction(model_input, stub, model_name='amazon_review', signature_name='serving_default'):
    request = predict_pb2.PredictRequest()
    request.model_spec.name = model_name
    request.model_spec.signature_name = signature_name
    request.inputs['input_input'].CopyFrom(tf.make_tensor_proto(model_input))
    response = stub.Predict.future(request, 5.0)  # 5 seconds
    return response.result().outputs["output"].float_val


def get_model_version(model_name, stub):
    request = get_model_metadata_pb2.GetModelMetadataRequest()
    request.model_spec.name = 'amazon_review'
    request.metadata_field.append("signature_def")
    response = stub.GetModelMetadata(request, 10)
    # signature of loaded model is available here: response.metadata['signature_def']
    return response.model_spec.version.value

if __name__ == '__main__':
    print("\nCreate RPC connection ...")
    stub = get_stub()
    while True:
        print("\nEnter an Amazon review [:q for Quit]")
        if sys.version_info[0] <= 3:
            sentence = raw_input() if sys.version_info[0] < 3 else input()
        if sentence == ':q':
            break
        model_input = [sentence]
        model_prediction = get_model_prediction(model_input, stub)
        print("The model predicted ...")
        print(model_prediction)
```