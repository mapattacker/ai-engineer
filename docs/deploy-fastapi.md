# FastAPI

[FastAPI](https://fastapi.tiangolo.com) is one of the next generation python web framework that uses ASGI (asynchronous server gateway interface) instead of the traditional WSGI. It also includes a number of useful functions to make API creations easier.

## Uvicorn

FastAPI uses Uvicorn as its ASGI. We can configure its settings as described [here](https://www.uvicorn.org/settings/). We can also specify it in the fastapi python app script, or at the terminal when we launch uvicorn.

For the former, with the below specification, we can just execute python app.py to start the application.

```python
# app.py
from fastapi import FastAPI
import uvicorn

app = FastAPI()

if __name__ == "__main__":
    uvicorn.run('app:app', host='0.0.0.0', port=5000)
```

If we run from the terminal, with the app residing in example.py.

```bash
uvicorn example:app --host='0.0.0.0' --port=5000
```

The documentation recommends that we use gunicorn which have richer features to better control over the workers processes.


```bash
gunicorn app:app --bind 0.0.0.0:5000 -w 1 --log-level debug -k uvicorn.workers.UvicornWorker
```

<hr>

## Request-Response Schema

FastAPI uses the pydantic library to define the schema of the request & response APIs. This allows the auto generation in the OpenAPI documentations, and for the former, for validating the schema when a request is received.

For example, given the json:

```json
{
    "boundingPoly": {
        "normalizedVertices": [
                {
                "x": 0.406767,
                "y": 0.874573,
                "width": 0.357321,
                "height": 0.452179,
                "score": 0.972167
                },
                {
                "x": 0.56781,
                "y": 0.874173,
                "width": 0.457373,
                "height": 0.452121,
                "score": 0.982109
                }
            ]
        },
    "name": "Cat"
}
```

We can define in pydantic as below, using multiple basemodels for each level in the JSON.

 * If there are no values input like `y: float`, it will listed as a required field
 * If we add a value like `y: float = 0.8369`, it will be an optional field, with the value also listed as a default and example value
 * If we add a value like `x: float = Field(..., example=0.82379)`, it will be a required field, and also listed as an example value

More attributes can be added in `Field()`, that will be populated in OpenAPI docs.

```python
class _normalizedVertices(BaseModel):
    x: float = Field(..., example=0.82379, description="X-coordinates"))
    y: float = 0.8369
    width: float
    height: float
    score: float

class _boundingPoly(BaseModel):
    normalizedVertices: List[_normalizedVertices]

class ResponseSchema(BaseModel):
    boundingPoly: _boundingPoly
    name: str = "Human"
```


We do the same for the request schema and place them in the routing function.

```python
from fastapi import FastAPI
from pydantic import BaseModel, Field
from typing import List

import json
import base64
import numpy as np

@app.post('/api', response_model = ResponseSchema)
async def human_detection(request: RequestSchema):

    JScontent = json.loads(request.json())
    encodedImage = JScontent['requests'][0]['image']['content']
    npArr = np.fromstring(base64.b64decode(encodedImage), np.uint8)
    imgArr = cv2.imdecode(npArr, cv2.IMREAD_ANYCOLOR)
    pred_output = model(imgArr)

    return pred_output
```

<hr>

## Open-API

OpenAPI documentations of Swagger UI or Redoc are automatically generated. You can access it at the endpoints of `/docs` and `/redoc`.

First, the title, description and versions can be specified from the initialisation of fastapi. The request-response pydantic schema and examples will be added after its inclusion in a POST/GET request routing function.


```python
from fastapi import FastAPI

app = FastAPI(title="Human Detection API",
              description="Submit Image to Return Detected Humans in Bounding Boxes",
              version="1.0.0")

@app.post('/api', response_model= RESPONSE_SCHEMA)
def human_detection(request: REQUEST_SCHEMA):
    do something
    return another_thing
```