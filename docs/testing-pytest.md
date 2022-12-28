# Pytest

## Testing Overview

Testing is an important aspect for any software system. The real value of testing occurs with system changes. Done correctly, each tests reduces uncertainty when analysing a change to the system

A traditional testing pyramid from Google includes the type of tests as well as their quantity for each test as illustrated below.

<figure>
  <img src="https://github.com/mapattacker/ai-engineer/blob/master/images/hierarchy-tests.png?raw=true" width="450" />
  <figcaption>Software engineering test pyramid.<a href="https://sre.google/sre-book/testing-reliability/"> Source</a></figcaption>
</figure>


With a machine learning system being more complex, as it does not just contain __code__, but also __data__ and __model__. We will also need to test for these.

<figure>
  <img src="https://github.com/mapattacker/ai-engineer/blob/master/images/test-pyramid.png?raw=true" />
  <figcaption>Martin Fowler's test pyramid for ML.<a href="https://martinfowler.com/articles/cd4ml.html"> Source</a></figcaption>
</figure>


## Pytest Basics

[Pytest](https://docs.pytest.org/en/stable/) is one of the python libraries used for automated testing your code. Refactoring & improvements can thus be easier to validate, debug, and also included in your CI/CD pipeline. 

An good tutorial on using pytest can be found [here](https://towardsdatascience.com/unit-testing-for-data-scientists-dc5e0cd397fb) 

A few basic conventions of pytest includes:

 * Test scripts must start or end with the word `test`, e.g. `test_data.py`
 * Functions must start with the word `test`, e.g. `def test_filepath_exits()`
 * Pytest uses `assert` to provide more context in the test result

Here are a few commands in the cli after running `pytest`

| CMD | Desc |
|-|-|
| `-v` | report in verbose, but truncated mode |
| `-vv` | report in verbose, non-truncated mode |
| `-q` | report in quiet mode, useful when there are hundreds of tests |
| `-s` | show print statements in console |
| `-ss` | show more print statements in console |
| `-ignore` | ignore the specified path when discovering paths |
| `-maxfail` | stop test after specified no. of failures |
| `<test_example.py>` | run only tests in this test script |
| `-m <name>` | run only tests marked with `<name>` |


For the last command, we can mark each of our test functions, e.g. `pytest.mark.integration` so that we can call them later using `pytest -m integration`. To run other unmarked test, we use `pytest -m 'not integration'`

Note that `pytest` runs from the existing python library, and can ignore your virtual environment setup. As a result, you might face import errors even though the libraries are already installed in the virtual env. To ask it to run in the virtual env, use `python -m pytest`.

## Directory Structure

Our repository directory is usually partitioned as follows below, with the testing and source code directory in separate folders. This is because the deployment project code is independent of the testing code.

```
.
├── project
│   ├── app.py
│   └── func.py
└── tests
    ├── unit_tests
    │   ├── __init__.py
    │   └── test_func.py
    └── integration_tests
        ├── __init__.py
        └── test_docker_api.py
```

## Importing Function to Test Scripts

To launch pytest, we will need to launch from the root, so that the test script can access the app.py

```bash
pytest test/unit_test/test_app.py -v
```

In the test script we then append the project folder to it. This will make the import more straightforward, especially when there are many cross imports between scripts in the project folder.

```python
# test_app.py
sys.path.append("project")
from app import function_a
```

An easier, and recommended method is to add an `__init__.py` to each of the test folders, which redirects to the modules in the project folders.

```python
# __init__.py
import sys
from os.path import dirname, join, normpath

THIS_DIR = dirname(__file__)
PROJ_DIR = normpath(join(THIS_DIR, '..', '..', 'project'))
sys.path.append(PROJ_DIR)
```

We can also create `setup.py` at the root, and `pip install --editable .`. This will create a folder `project.egg.info` like all the libraries installed using pip, and make all imports under the project folder to be imported anywhere from the root. 

```python
# setup.py
from setuptools import setup, find_packages
setup(name="project", packages=find_packages())
```

```python
# test_app.py
from project.app import function_a
```


## INI File

We can add more configurations to pytest through a `pytest.ini` file. The commands can also be placed in a `tox.ini` file if using tox to launch pytest.

```ini
[pytest]
junit_family=xunit1
filterwarnings =
    ignore::DeprecationWarning
    ignore::RuntimeWarning
    ignore::UserWarning
    ignore::FutureWarning
```

## Class

We can place our classes in classes to better organise them. Note that the class name must start with a `Test_*` for pytest to register, and each method name must have `test_`. Below is an example

```python
class Test_personalised:
    
    def test_ucf_present_no_customer_id(self):
        try:
            out = RequestSchema(**data1)
        except Exception as e:
            msg = ast.literal_eval(e.json())[0]["msg"]
        assert msg == "error msg 1"

    def test_adl_present_no_customer_id(self):
        try:
            out = RequestSchema(**data2)
        except Exception as e:
            msg = ast.literal_eval(e.json())[0]["msg"]
        assert msg == "error msg 2"
```

## Fixtures

When creating test scripts, we often need to run some common code before we create the test case itself. Instead of repeating the same code in every test, we create fixtures that establish a baseline code for our tests. Some common fixtures include data loaders, or initialize database connections.

Fixtures are define in the file `conftest.py` so that tests from multiple test modules in the directory can access the fixture function.

```python
# conftest.py
import pytest

@pytest.fixture
def input_value():
    return 4


# test_module.py
def test_function(input_value):
    subject = square(input_value)
    assert subject == 16
```

The test fixture have a scope argument, e.g. `@pytest.fixture(scope="session")` which set when the fixture will be destroyed. For example, the session scope is destroyed at the end of the test session (i.e. all the tests). By default, the scope is set to `function`, which mean the fixture is destroyed at the end of the test function when it was called.

## Parameterize

Parameterizing allows multiple inputs to be iterate over a test function. For the example below, there will be 3 tests being run.

```python
import pytest

@pytest.mark.parametrize("inputs", [2, 3, 4.5])
def test_function(inputs):
    subject = square(inputs)
    assert isinstance(subject, int)
```

If we have multiple variables for each iteration, we can do the following.

```python
@pytest.mark.parametrize("image, expect", [
     ("cone_dsl_00009.jpg", 2),
     ("cone_dsl_00016.jpg", 3),
     ("cone_dsl_00017.jpg", 2),
     ("cone_dsl_00018.jpg", 3)])
def test_predict_validation(image, expect):
    """test model with a few key test images"""
    img_path = os.path.join("tests/data/images", image)
    img_arr = cv2.imread(img_path)
    prediction = detectObj(img_arr)
    detection_cnt = len(prediction)
    assert detection_cnt == expect
```

## Function Annotation

While not a pytest functionality, __Function Annotations__ are useful and easy to implement for checking the input and return types in pytest.

```python
def foo(a:int, b:float=5.0) -> bool:
    return "something"

foo.__annotations__
# {'a': <class 'int'>, 'b': <class 'float'>, 'return': <class 'bool'>}
```

## Mocking

Mocking is often used in unit tests where we can mock or "fake" a library, function, return value etc., as we only want to test the unit/function itself, not other related stuff that are bundled with it.

### Patching

Using the in-built `unittest.mock.patch` we can easily mock functions and their return values. One useful note is that we need to patch *where the object/function is looked up* rather than *where it is defined*.


```python
# project/app.py
import time

def api_run():
    time.sleep(2)
```

```python
# project/func.py
def function_a():
    api_run_a()
    return 1

def return_a_value():
    return 1

def function_b():
    y = return_a_value()
    return y
```

In this sample unittest script, the `api_call` has a latency of 2 sec. After mocking it, we can see that the test runs instantly.

We can also call patch without a decorator, in case we have other decorators that made the test function's arguments confusing.

Last, we can also have a return value for the patched function.

```python
import sys
from unittest.mock import patch

sys.path.append("project")
from func import function_a, function_b


def test_no_mock():
    ret = function_a()
    assert 1 == ret

@patch('func.api_run')
def test_mock_func(mock_api_run):
    ret = function_a()
    assert 1 == ret

def test_mock_func_no_deco():
    with patch('func.api_run') as mock_api_run:
        ret = function_a()
    assert 1 == ret

@patch('func.return_a_value', return_value=5)
def test_mock_func_return(mock_return_a_value):
    """api_run_b will return 5 to function b"""
    ret = function_b()
    assert 5 == ret
```

We can also stack the patches, but note that **first argument in the test function correspondings to the last patch decorator** given.

```python
@patch("app.load_model_item2vec_sku")
@patch("app.query_sku")
@patch("app.sort_by_cos_sim", return_value="abc")
def test_api(sort_by_cos_sim, query_sku, load_model_item2vec_sku):
    expected = sort_by_cos_sim.return_value
    with app.app.test_client() as client:
        response = client.post('/product-similarity', 
                        json={"resultSize":10,"sku": "x"}).json
    assert response=={"sku": expected}
```

To mock a variable in the function, we can use the same patch. We don't need to pass any arguments to the test function.

```python
from unittest.mock import patch

from app import function

@patch("app.variable_name", "new_variable_value")
def test_script():
    result = call_function()
    assert "new_variable_value" == result
```

To have a cleaner interface without multiple `with` or decorators, we can instead (and recommended) use the pytest extension of mocking `pip install pytest-mock`. 


```python
from pipeline import pipeline

def test_mocker(mocker):
    mocker.patch('pipeline.preprocess')
    predict = mocker.patch('pipeline.training', return_value=5)

    expected = predict.return_value

    output = pipeline(5)
    assert output == expected
```

### Global Variables & Functions

Sometimes, we might need to have global variables or functions defined in a script. An example is shown below, whereby we are opening a model file that is not present in the repository.

```python
# func.py
import pickle

f = open("model.pkl", "r")
model = pickle.load(f)

def function_a():
    # do something
    return 1
```

Hence, if we use the test script below, a `FileNotFoundError: [Errno 2] No such file or directory: 'model.pkl'.` will be prompted. We **cannot mock the global functions (or variables)**; they will only be mocked if they are called within the function/method tested, in this case is `function_a`.

```python
# test_func.py
from func import function_a

def test_function_a():
    ret = function_a()
    assert 1 == ret
```

There are a few ways to overcome this:

*  Use `if "pytest" not in sys.modules:`

```python
import pickle
import sys

if "pytest" not in sys.modules:
    f = open("model.pkl", "r")
    model = pickle.load(f)

def function_a():
    # do something
    return 1
```

* Place the global file calling under `if __name__ == "__main__"`

```python
import pickle
import sys

def function_a():
    # do something
    return 1

if __name__ == "__main__":
    f = open("model.pkl", "r")
    model = pickle.load(f)
```
* Have a `prod` (and/or `dev`) and `test` config variables, the latter which includes the path to a dummy test model.

```python
# ------ in test_func.py ------
import os

os.environ["ENV"] = TEST
from func import function_a

def test_function_a():
    ret = function_a()
    assert 1 == ret


# ------ in config.py ------
conf = \
{"ENV": {
    {"PROD":
        {"MODEL_NAME": "model.pkl"}
        {"MODEL_PATH": "./"}
    },
    {"TEST":
        {"MODEL_NAME": "model.pkl"}
        {"MODEL_PATH": "../tests/unit_tests/data"}
    }
}}

# ------ func.py ------
import os
import pickle
import config


env = os.environ["ENV"]
if not env: # if $ENV not given, set to PROD
    env = "PROD"
conf = conf[env]

model_path = os.path.join(conf["MODEL_PATH"], conf["MODEL_NAME"])
f = open(model_path, "r")
model = pickle.load(f)

def function_a():
    # do something
    return 1
```

* Set the paths and names via environment variables

<hr>


## Tox

[Tox](https://tox.readthedocs.io/en/latest/index.html) is a generic virtualenv for commandline execution, and is particularly useful for running tests. Think of it as a CI/CD running in your laptop. Tox can be installed via `pip install tox`

A file called `tox.ini` is created and set at the root, with the contents as follows. To run pytest using tox, just run `tox`. To reinstall the libraries, run `tox -r`. To run a specific environment, run `tox -e integration`.

Note that `skipsdist = True` has to be added if you don't have a `setup.py` file present.

```ini
[tox]
skipsdist = True
envlist = unit, integration


[testenv:unit]
deps = 
    pytest==5.3.5
    pytest-cov==2.10.1
    -r{toxinidir}/tests/unit_tests/requirements.txt

commands = pytest tests/unit_tests/ -v --cov=project --junitxml=report.xml


[testenv:integration]
deps = 
    pytest==5.3.5
    Cython==0.29.17 
    numpy==1.19.2

commands =
    pip install -r {toxinidir}/project/requirements-serve.txt 
    pytest tests/integration_tests/ -v --junitxml=report.xml

# side tests -----------------------------

[testenv:lint]
deps =
    flake8
commands = flake8 --select E,F,W association

[testenv:bandit]
deps = 
    bandit

commands = bandit -r -ll association

[testenv:safety]
deps = 
    safety
    -r{toxinidir}/association/requirements.txt
commands = safety check


[pytest]
junit_family=xunit1
filterwarnings =
    ignore::DeprecationWarning
    ignore::RuntimeWarning
    ignore::UserWarning
    ignore::FutureWarning
```

<hr>

## Test Coverage

A common question is how much should I test? An analogy would be the wearing of armor to cover oneself. Too much armor will weight down a person, while too little will be very vulnerable. A right balance needs to be struck.

A test coverage report show how much of your code base is tested, and allows you to spot any potential part of your code that you have missed testing. We can use the library `pytest-cov` for this, and it can be installed via `pip install pytest-cov`. 

Once test cases are written with pytest, we can use it to generate the coverage report. An example, using the command `pytest --cov=<project-folder> <test-folder>` is shown below.

```python
pytest --cov=project tests/

-------------------- coverage: ... ---------------------
Name                 Stmts   Miss  Cover
----------------------------------------
myproj/__init__          2      0   100%
myproj/myproj          257     13    94%
myproj/feature4286      94      7    92%
----------------------------------------
TOTAL                  353     20    94%
```

To ignore certain files or folders, add a `.coveragerc` file to where you call pytest, and add the following.

```ini
[run]
omit = project/train.py
```

Various report formats can be produced, and are as stated in their [documentation](https://pytest-cov.readthedocs.io/en/latest/reporting.html).


