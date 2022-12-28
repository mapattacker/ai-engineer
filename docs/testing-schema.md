# Schema Validation

There are quite a number of popular schema validation libraries available, but I will just stick to [Pydantic](https://pydantic-docs.helpmanual.io/), for the reasons that it is popular, touted to be the [fastest](https://pydantic-docs.helpmanual.io/benchmarks/), and used natively in FastAPI.


## Config

The config file is one of the most frequently changed file in the repository, and for that reason, it is important to include in our unit-tests. 

The below shows an example using a config yaml file & pydantic, with a custom validator.

```yml
# config.yml
modeldir: model
modelname: nameofname
format: dataframe # graph or dataframe
```

```python
import pytest
import yaml
from pydantic import BaseModel, ValidationError, validator


class configSchema(BaseModel):
    modeldir: str
    modelname: str
    format: str

    @validator('format')
    def format_list(cls, v):
        if v not in ["dataframe", "graph"]:
            raise ValueError("must be either 'dataframe or 'graph'")
        return v


def test_config():
    """test for all key-values in config file"""
    cf = yaml.safe_load(open("foldername/config.yml"))
    try:
        out = configSchema(**cf)
        print(out)
    except ValidationError as e:
        pytest.raises(e) 
```

## API Request

Pydantic can also be used in Flask to validate all incoming requests. To validate the request schema before passing to the code, we can write a decorator function.


```python
import logging
from functools import wraps
from typing import List, Optional

from flask import Flask, abort, jsonify, make_response, request, logging as flog
from pydantic import BaseModel, ValidationError, confloat, conint, validator


app = Flask(__name__)


class _weightage(BaseModel):
    recommender: str
    weight: confloat(ge=0, le=1)

    @validator("recommender")
    def recommender_list(cls,v):
        if v not in recommender_list:
            raise ValueError("Recommender is not part of {}".format(str(recommender_list)))

class RequestSchema(BaseModel):
    productSKU: List[str]
    storeId: str
    weightage: List[_weightage]
    customerId: Optional[str]
    preceding_time_window: Optional[str]
    resultSize: Optional[conint(ge=1, le=50)] = 20

    @validator("weightage")
    def weight_sum(cls, v):
        sumw = 0
        for i in range(len(v)):
            weight = v[i].weight
            sumw = weight + sumw
        if sumw != 1:
            raise ValueError("sum of weightage is not equals to 1")


def validate_request(requestschema):
    """decorator to validate request schema"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            try:
                requestschema(**request.json)
            except ValidationError as e:
                app.logger.error('{} - {}'.format(e, [422]))
                err_json = json.loads(e.json())
                abort(make_response(jsonify(err_json), 422))
            return func(*args, **kwargs)
        return wrapper
    return decorator


@app.route("/recommendation", methods=["POST"])
@validate_request(RequestSchema)
def fusion_api():
    req_content = request.json    
    # do something
    return predicted_results
```

## JSON

While pydantic works fine for JSON validation and we can fine-tune to test at quite a grandular level, it can be hard to grasp at start. Using something like `pytest-schema` (`pip install pytest-schema`) makes writing test cases much easier in `pytest`.

```python
from pytest_schema import schema

my_schema = {
    "key1": int
    "key2": float
    "key3": {
        "key4": str,
        "key5": str
    }
}

response = {
    "key1": 111
    "key2": 0.1
    "key3": {
        "key4": "test",
        "key5": "test
    }
}

def test_schema():
    assert schema(my_schema) == response


```

## Pandas

Ok, I take back what I said on Pydantic. For validating pandas dataframes, it is slightly difficult to use that library, hence we will use a library heavily inspired by Pydantic, called [pandera](https://pandera.readthedocs.io/en/stable/index.html).

```python
from pandera import DataFrameSchema, Column, Check

schema = DataFrameSchema({
    "antecedents": Column("category"),
    "consequents": Column(object),
    "antecedent support": Column("float32"),
    "consequent support": Column("float32"),
    "support": Column("float32", checks=Check(lambda x: 0 <= x <= 1, \
                                                element_wise=True, \
                                                error="range checker [0, 1]")),
    "confidence": Column("float32", checks=Check(lambda x: 0 <= x <= 1, \
                                                element_wise=True, 
                                                error="range checker [0, 1]")),
    "lift": Column("float32"),
    "leverage": Column("float32"),
    "conviction": Column("float32"),
    "storeid": Column("category"),
})

try:
    validated_df = schema(df, lazy=True)
except Exception as e:
    print(e)
```

The `validated_df` is the same dataframe that can be used to continue coding. If we are validating in pytest, we can add a try, except to catch the error in a graceful way.

The argument `lazy=True` should added to ensure that it captures all errors before giving a validation error report as shown below.

```python
A total of 1 schema errors were found.

Error Counts
------------
- schema_component_check: 1

Schema Error Summary
--------------------
                                                   failure_cases  n_failure_cases
schema_context column      check                                                 
Column         consequents pandas_dtype('float64')      [object]                1

Usage Tip
---------

Directly inspect all errors by catching the exception:

try:
    schema.validate(dataframe, lazy=True)
except SchemaErrors as err:
    err.failure_cases  # dataframe of schema errors
    err.data  # invalid dataframe
```