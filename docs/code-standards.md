# Code Standards

## requirements.txt

`requirements.txt` contains the list of python libraries & their versions that is needed for the application to work. This file must be present for every repository. To see how it works together in a virtual environment, refer to the earlier [section](https://mapattacker.github.io/ai-engineer/virtual_env/).

Use the library `pip install pipreqs` to auto-generate using the command `pipreqs directory_path`. Note to test with `pip install -r requirements.txt`, to ensure installation works. Put the python version in the first line in the file (and comment out). e.g., # python=3.8. This will ensure what python version to use.

Below is an example of how different types of libraries should be placed. By default pipreqs can only auto-populate libraries that are pip installed. For others which are installed via github or a website, you will need to specify as below.

```python
# python=3.7
pika==1.1.0
scipy==1.4.1
scikit_image==0.16.2
numpy==1.18.1
# package from github, not present in pip
git+https://github.com/cftang0827/pedestrian_detection_ssdlite
# wheel file stored in a website
--find-links https://dl.fbaipublicfiles.com/detectron2/wheels/cu101/index.html
detectron2
--find-links https://download.pytorch.org/whl/torch_stable.html
torch==1.5.0+cu101
torchvision==0.6.0+cu101
```

## DocStrings

DocStrings should be present for every function & method of a class. For primary functions, ensure that it provides 1) Description of the function, 2) Argument name, data type, and description, 3) Return description & data type.

For secondary functions, they should minimally contain the function description.


```python
def yolo2coco_bb(size, yolo):
    """convert yolo format to coco bbox

    Args
    ----
    size (tuple): (width, height)
    yolo (tuple): (ratio-bbox-centre-x, ratio-bbox-centre-y, ratio-bbox-w, ratio-bbox-h)

    Returns
    -------
    coco: xmin, ymin, boxw, boxh
    """
    width = size[0]
    height = size[1]

    centrex = yolo[0] * width
    centrey = yolo[1] * height
    boxw = yolo[2] * width
    boxh = yolo[3] * height

    halfw = boxw / 2
    halfh = boxh / 2

    xmin = centrex - halfw
    ymin = centrey - halfh

    coco = xmin, ymin, boxw, boxh
    return coco
```


## Isort

Sorts by alphabetical order the imported libraries, while splitting python base libraries as first order, with the 3rd party libraries as second order, and local script imports as the third.

 * Installation: `pip install isort`
 * Single File: `isort your_file.py`


## Black

Black **auto-formats** python files to adhere to PEP8 format as well as other styles that the team felt is useful. This includes removing whitespaces, new lines, change single to double quotes, and etc.

 * Installation: `pip install black`
 * Format Files
    * Single File: `black my_file.py`
    * Directory: `black directory_name`
 * Check Changes w/o Formating
    * `black --diff my_file.py`


## Flake8

A wrapper of 3 libraries that **checks (but does not change)**, against python standard  styling (PEP8), programming errors (like “library imported but unused” and “Undefined name”) and to check cyclomatic complexity.

 - E***/W***: pep8 errors and warnings
 - F***: PyFlakes codes (see below)
 - C9**: McCabe complexity plugin mccabe
 - N8**: Naming Conventions plugin pep8-naming

<br>

 * Installation: `pip install flake8`
 * Current Project: `flake8`
 * Single File (and all imported scripts): `flake8 my_file.py`

<hr>

## Pre-Commit

Pre-commit is a git hook that you preconfig to run certain scripts, in this case, the above ones before committing to git. A useful compilation is done by [laac.dev](https://www.laac.dev/blog/automating-convention-linting-formatting-python/).

 * Installation: `pip install pre-commit`
 * create config file at root of project: `.pre-commit-config.yaml`
 * Installation (into git hook): `pre-commit install`
 * Uninstallation (from git hook): `pre-commit uninstall`
 * Add Files for Commit: `git add files.py`
 * Run Commit, and Precommit will autorun: `git commit -m 'something'`
 * Skip Hook: `SKIP=flake8 git commit -m "something"`

Here is an example of the `.pre-commit-config.yaml`

```yml
default_language_version:
  python: python3
repos:
  - repo: https://github.com/psf/black
    rev: stable
    hooks:
      - id: black
  - repo: https://gitlab.com/pycqa/flake8
    rev: 3.8.3
    hooks:
      - id: flake8
  - repo: https://github.com/PyCQA/bandit
    rev: 1.6.2
    hooks:
      - id: bandit
  - repo: https://github.com/timothycrosley/isort
    rev: 4.3.21
    hooks:
      - id: isort
```