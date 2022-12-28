# Virtual Environment

Every project has a different set of requirements & different sets of python packages might be required to support it. The versions of each package can differ or break with each python or dependent packages versions update, so it is important to isolate every project within an enclosed virtual environment.

## Anaconda

[Anaconda](https://www.anaconda.com) is a python installation that bundles all essential packages for a data science project, hence by default most people will have this installed. It comes with its own virtual environment. Here are the basic commands to create & remove.

```bash
conda create -n <yourenvname> python=3.7
pip install -r requirements.txt
conda activate <yourenvname>
conda deactivate
# in some IDE, like in WSL2 ubuntu, have to use source instead of conda for activation

conda env list
conda env remove -n <yourenvname>
```

## VENV

venv is my next favourite, as it is a default library shipped with python, and very easy to setup. The libraries will be installed in the directory you are in, so remember to add the folder in `.gitignore`. To remove the libraries, just delete the folder that was created.

The downside of venv is that you cannot install a particular python version.

```bash
cd <projectfolder>
python -m venv <name>
source <name>/bin/activate
pip install -r requirements.txt
deactivate

# check env; via its path
pip -V
```

## Jupyter Notebook

Sometimes we need to experiment with certain code or data, and not ready to convert into python scripts; hence the use of jupyter notebooks. To change the environment or kernel (as named in jupyter), we first setup our virtual env using venv or conda, and install the necessary library as follows.

```bash
pip install ipykernel
python -m ipykernel install --user --name=<env_name>
```

In the jupyter notebook, we then select the kernel option from the top right and switch to the virtual env we launched earlier.

<figure>
  <img src="https://github.com/mapattacker/ai-engineer/blob/master/images/jupyter.png?raw=true" width="90%" />
  <figcaption></figcaption>
</figure>


Sourced from this [article](https://janakiev.com/blog/jupyter-virtual-envs/)

## Pip

pypip is one of the main python package manager, another being conda. To see what are the packages being install in pip, we can use the command `pip freeze`.

To check if a specified package is being installed, we can use `pip show <package-name>`, e.g. `pip show flask`

```bash
Name: Flask
Version: 1.1.2
Summary: A simple framework for building complex web applications.
Home-page: https://palletsprojects.com/p/flask/
Author: Armin Ronacher
Author-email: armin.ronacher@active-4.com
License: BSD-3-Clause
Location: /Users/siyang/opt/anaconda3/lib/python3.8/site-packages
Requires: itsdangerous, Werkzeug, click, Jinja2
Required-by: locust, Flask-BasicAuth
```