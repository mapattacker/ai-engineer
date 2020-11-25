# Flask

Flask is a micro web framework written in Python. It is easy and fast to implement. How is it relevant to AI? Sometimes, it might be necessary to run models in the a server or cloud, and the only way is to wrap the model in a web application. 

The input containing the data for predicting & the model parameters, aka `Request` will be send to this API containing the model which does the prediction, and the `Response` containing the results will be returned. Flask is the most popular library for such a task.


## Simple Flask App

Below is a simple flask app to serve an ML model's prediction. Assuming this app is named `serve_http.py`, we can launch this flask app locally via `python serve_http.py`. The API can be accessed via `http://localhost:5000/`.


```python
"""flask app for model prediction"""
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
        nms_iou = features.get("nms_iou") or 0.4

        # get pred-output
        pred_bbox = detectObj(
            decodedImage, min_height, min_width, maxResults, score_th, nms_iou
        )

        # format to response json output
        json_output = array2json(pred_bbox, class_mapper)
        return json_output

    except Exception as e:
        print(e)


if __name__ == '__main__':
    app.run()
```


## Gunicorn

Flask as a server is meant for development, as it tries to remind you everytime you launch it. It has very limited parameters to manage the server, but luckily wrappers are available to connect Flask to a feature rich web server. Examples include timeout, and multiple workers.

[Gunicorn](https://gunicorn.org) is one of the most popular, and also probably the easiest to use.

```bash
# gunicorn -w 2 flaskscript:flaskapp
# it uses port 8000 by default, but we can change it
gunicorn --bind 0.0.0.0:5000 -w 2 serve_http:app
```

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
    """Sends JSON request to AI-microservice and recieve a JSON response

    Args
    ----
    url (str): URL of server where AI-microservice is hosted
    image (image file): opened image file
    score_th (float): Minimum prediction score for bounding box to be accepted
    nms_ious (float): IOU threshold for non-max suppression

    Rets
    ----
    (dict): JSON response
    """
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

![](https://github.com/mapattacker/computer-vision-python/raw/master/images/postman.png)