# Flask

Flask is a micro web framework written in Python. It is easy and fast to implement. How is it relevant to AI? Sometimes, it might be necessary to run models in the a server or cloud, and the only way is to wrap the model in a web application. 

The input containing the data for predicting & the model parameters, aka `Request` will be send to this API containing the model which does the prediction, and the `Response` containing the results will be returned. Flask is the most popular library for such a task.


## Simple Flask App

Below is a simple flask app to serve an ML model's prediction. Assuming this app is named `serve_http.py`, we can launch this flask app locally via `python serve_http.py`. The API can be accessed via `http://localhost:5000/`.

It is important to set the `host="0.0.0.0"`, so that it binds to all network interfaces of the container, and will be callable from the outside.

```python
"""flask app for model prediction"""
import traceback

from flask import Flask, request

from predict import detectObj
from utils_serve import array2json, from_base64

app = Flask(__name__)


@app.route("/", methods=["POST"])
def get_predictions():
    """Returns pred output in json"""
    try:
        req_json = request.json

        # get image array
        encodedImage = req_json["requests"][0]["image"]["content"]
        decodedImage = from_base64(encodedImage)

        # get input arguments
        features = req_json["requests"][0]["features"][0]
        min_height = features.get("min_height") or 0.03
        min_width = features.get("min_width") or 0.03
        maxResults = features.get("maxResults") or 20
        score_th = features.get("score_th") or 0.3
        nms_iou = features.get("nms_iou") # using .get will return None if request does not include key-value

        # get pred-output
        pred_bbox = detectObj(decodedImage, 
                            min_height, 
                            min_width, 
                            maxResults, 
                            score_th, 
                            nms_iou)

        # format to response json output
        json_output = array2json(pred_bbox, class_mapper)
        return json_output

    except Exception as e:
        tb = traceback.format_exc()
        app.logger.error(tb)
        return {"errorMessages": tb.replace("\n","")}


if __name__ == '__main__':
    app.run(host="0.0.0.0")
```

## Async

From `Flask>=2.0` onwards, it supports async syntax with the installation of `pip install Flask[async]`. It should be noted to use async only in I/O bound tasks which takes less than a few seconds to process. An excellent description from [here](https://testdriven.io/blog/flask-async/). It works similarly & with the same response time as multi-threading.

### Synchronous 

Take for example this synchronous REST app that is requesting data from multiple urls. The response time is 1.024s.

```python
import json

import requests
from flask import Flask, request

app = Flask(__name__)

api_urls = \
    {"sml": "http://localhost:5001/test",
     "asc": "http://localhost:5002/test",
     "trd": "http://localhost:5003/test",
     "psn": "http://localhost:5004/test"}


def call_recommender_sync(rre, api_urls):
    """call individual recommenders & get predictions"""
    url = api_urls[rre]
    data = {"resultSize": 1}
    prediction = requests.post(url, json=data).content
    prediction = json.loads(prediction)
    return (prediction, rre)

@app.post("/recommendation")
def fusion_api():
    # synchronous, 1.024s ---------
    concat_list = []
    for rre in api_urls.keys():
        predictions = call_recommender_sync(rre, api_urls)
        concat_list.append(predictions)

    return {"result": str(concat_list)}

if __name__ == "__main__":
    app.run(port=5000, debug=True)
```

### Asynchronous

To write in asynchronous code to send multiple requests, we need to use `aiohttp` in place of `requests`, and `aiohttp` to gather the results in a list. `aiohttp` is a **non-blocking** program, which allow other threads to continue running while it's waiting. The appropriate `async` and `await` syntax needs to be added too. The response time is around 0.356s, x2.87.

```python
import asyncio
import json

import requests
from aiohttp
from flask import Flask, request

app = Flask(__name__)

api_urls = \
    {"sml": "http://localhost:5001/test",
     "asc": "http://localhost:5002/test",
     "trd": "http://localhost:5003/test",
     "psn": "http://localhost:5004/test"}


async def call_recommender_async(rre, api_urls):
    """call individual recommenders & get predictions"""
    url = api_urls[rre]
    data = {"resultSize": 1}
    async with aiohttp.ClientSession() as session:
        async with session.post(url, json=data) as resp:
            prediction = await resp.json()
    return (prediction, rre)

@app.post("/recommendation")
async def fusion_api():
    # asynchronous, 0.356s, x2.87 ---------
    concat_list = []
    for rre in api_urls.keys():
        predictions = asyncio.create_task(call_recommender_async(rre, api_urls))
        concat_list.append(predictions)
    concat_list = await asyncio.gather(*concat_list)
    return {"result": str(concat_list)}

if __name__ == "__main__":
    app.run(port=5000)
```


### Multi-Threading

We can use `concurrent.futures` to send requests using multi-threading. The response time is similar, 0.359s, x2.85 faster. 

This is because of Python's Global Interpretor Lock (GIL), whereby only one thread can run one time. Python uses thread-switching to change to another thread to start another task, rendering multi-threading as an async, not parallel process.

```python
import json
from concurrent.futures import ThreadPoolExecutor, as_completed

import requests
from flask import Flask, request

app = Flask(__name__)

api_urls = \
    {"sml": "http://localhost:5001/test",
     "asc": "http://localhost:5002/test",
     "trd": "http://localhost:5003/test",
     "psn": "http://localhost:5004/test"}

def call_recommender(rre, api_urls):
    """call individual recommenders & get predictions"""
    url = api_urls[rre]
    data = {"resultSize": 1}
    prediction = requests.post(url, json=data).content
    prediction = json.loads(prediction)
    return (prediction, rre)

def multithread(api_urls):
    futures = []
    with ThreadPoolExecutor(max_workers=4) as executor:
        for rre in api_urls.keys():
            futures.append(executor.submit(call_recommender, rre, api_urls))
        return [future.result() for future in as_completed(futures)]

@app.post("/recommendation")
def fusion_api():
    # multi-threading, 0.359s, x2.85 ---------
    concat_list = multithread(api_urls)
    return {"result": str(concat_list)}

if __name__ == "__main__":
    app.run(port=5000, debug=True)
```



## Gunicorn

Flask as a server is meant for development, as it tries to remind you everytime you launch it, giving the message `WARNING: This is a development server. Do not use it in a production deployment. Use a production WSGI server instead.`

It has very limited parameters to manage the server, but luckily wrappers are available to connect Flask to a feature rich web server. One of the best is [Gunicorn](https://gunicorn.org); a mature, fully featured server and process manager. It allows automated worker production and management through simple configurations.

### Command

```bash
# gunicorn --bind <flask-ip>:<flask-port> <flask-script>:<flask-app>
gunicorn --bind 0.0.0.0:5000 serve_http:app
```

### Config File

Rather than entering all the configs when launching gunicorn, we can hard code some of them in a config file `gunicorn.conf.py`. With this, we can adjust the workers based on the machine's cores. Gunicorn's documentation recommend the number of workers to be set as `(total-cpu * 2) + 1`. One of the most important config besides workers is the `preload`, this allows the preloading of your model in memory and shared among all the workers. If not, each of your worker will load the model separately and consume a lot, if not all the RAM in the machine.

```python
# gunicorn.conf.py
# to see all flask stdout, we can change the log level to debug

import multiprocessing

workers = (multiprocessing.cpu_count() * 2) + 1
preload_app = True
log_level = info
timeout = 10
```

We can also use the `gevent` or `gthread` to implement concurrency if there are significant I/O blocking bottlenecks. For the latter, this is done by “monkey patching” the code, mainly replacing blocking parts with compatible cooperative counterparts from gevent package. See this article for more [information](https://dev.to/lsena/gunicorn-worker-types-how-to-choose-the-right-one-4n2c).


### Logging

Using the default logging library does not automatically appear in the gunicorn logs. To do that, you have to use `print("<something>", flush=True)`.

## Testing

### Python Requests

The python `requests` library provides a convenient function to test your model API. Its function is basically just a one-liner, e.g. `response = requests.post(url, headers=token, json=json_data)`, but below provides a complete script on how to use it with a JSON request to send over an image with model parameters.

```python
"""python template to send request to ai-microservice"""

import base64
import json

import requests


json_template = \
{
  "requests": [
    {
      "features": [
        {
         "score_th": None, 
         "nms_iou": None
        }
      ],
      "image": {
        "content": None
      }
    }
  ]
}

def send2api(url, token, image, score_th=0.35, nms_iou=0.40):
    """Sends JSON request to AI-microservice and recieve a JSON response"""
    base64_bytes = base64.b64encode(image.read()).decode("utf-8")

    json_template["requests"][0]["image"]["content"] = base64_bytes
    json_template["requests"][0]["features"][0]["score_th"] = score_th
    json_template["requests"][0]["features"][0]["nms_iou"] = nms_iou

    token = {"X-Api-Token": token}
    response = requests.post(url, headers=token, json=json_template)
    json_response = response.content

    j = json.loads(json_response)
    return j



if __name__ == "__main__":
    url = "http://localhost:5000/"
    token = "xxx"
    image_path = "sample_images/20200312_171718.jpg"
    image = open(image_path, 'rb')
    j = send2api(url, token, image)
    print(j)
```

### Postman

[Postman](http://postman.com) is a popular GUI to easily send requests and see the responses. 

![](https://github.com/mapattacker/ai-engineer/blob/master/images/postman.png?raw=true)

### CURL

We can also use CURL in the terminal for commandline sending of requests.

Here’s a simple test to see the API works, without sending the data.

```bash
curl --request POST localhost:5000/api
```

Here’s one complete request with data

```bash
curl --header "Content-Type: application/json" \
    --request POST \
    --data '{"username":"xyz","password":"xyz"}' \
    http://localhost:5000/api
```

To run multiple requests in parallel for stress testing

```bash
curl --header "Content-Type: application/json" \
    --request POST \
    --data '{"username":"xyz","password":"xyz"}' \
    http://localhost:5000/api &
curl --header "Content-Type: application/json" \
    --request POST \
    --data '{"username":"xyz","password":"xyz"}' \
    http://localhost:5000/api &
curl --header "Content-Type: application/json" \
    --request POST \
    --data '{"username":"xyz","password":"xyz"}' \
    http://localhost:5000/api &
wait
```

## OpenAPI

OpenAPI is a standard API documentation specification in ymal or json format, originated from Swagger. It usually comes with a user interface, the most popular being [SwaggerUI](https://petstore.swagger.io/?_ga=2.260024972.744460072.1616040410-1862236881.1616040410).

It provides all the information required about the API, with also an ability to test the API itself.

There are three ways to go about this. 

 1. generated separately as a single ymal file, and hosted using [connexion](https://github.com/zalando/connexion/blob/master/docs/index.rst)
 2. generate as docstrings or individual ymal file for each endpoint using [flasgger](https://github.com/flasgger/flasgger)
 3. auto-generated using defined schemas, and adding additional info within the Flask app using [Flask-Pydantic-Spec](https://github.com/turner-townsend/flask-pydantic-spec)

Personally, I believe the first is the most realistic, as the API specs are usually defined before the Flask app is created, and that doc can be sent to others for verification without creating a Flask app.

Below is an example script for point 1. 

```python
import connexion
from flask import request

from predict import prediction

app = connexion.App(__name__, specification_dir='.')


@app.route("/predict", methods=["POST"])
def predict():
    JScontent = request.json
    img = JScontent["image"]
    response = prediction(img)
    return response


if __name__ == "__main__":
    app.add_api('openapi.yml')
    app.run(host="0.0.0.0")
```

And point 3. 


```python
"""flask server with pydantic validation & openapi integration"""

from typing import List

from flask import Flask, request
from flask_pydantic_spec import FlaskPydanticSpec, Request, Response
from pydantic import BaseModel, Field, confloat

from predict import prediction

app = Flask(__name__)
api = FlaskPydanticSpec("flask", title="Objection Detection", version="v1.0.0")


class RequestSchema(BaseModel):
    maxResults: int = Field(None, example=20, description="Maximum detection result to return")
    min_height: float = Field(None, example=0.3, description="Score")
    min_width: float = Field(None, example=0.3, description="Score")
    score_th: float = Field(None, example=0.3, description="Score")
    nms_iou: float = Field(..., example=0.4, description="Non-max suppression, intersection over union")
    type: str = Field(..., example="safetycone", description="name of object to detect")
    image: str = Field(..., description="base64-encoded-image")

class _normalizedVertices(BaseModel):
    x: float = Field(..., example=5.12, description="X-coordinate")
    y: float = Field(..., example=20.56, description="Y-coordinate")
    width: int = Field(..., example=500, description="width in pixel")
    height: int = Field(..., example=600, description="height in pixel")
    score: confloat(gt=0.0, lt=1.0) = Field(..., example=0.79, description="confidence score")

class ResponseSchema(BaseModel):
    normalizedVertices: List[_normalizedVertices]


@app.route("/predict", methods=["POST"])
@api.validate(
    body=Request(RequestSchema),
    resp=Response(HTTP_200=ResponseSchema),
    tags=["API Name"]
)
def get_predictions():
    """Short description of endpoint

    Long description of endpoint"""
    JScontent = request.json
    img = JScontent["image"]
    response = prediction(img)
    return response


if __name__ == "__main__":
    api.register(app)
    app.run(host="0.0.0.0")
```

You can refer to my [repo](https://github.com/mapattacker/Flask-OpenAPI) for the full example.