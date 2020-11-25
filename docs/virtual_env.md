# Virtual Environment

Every project has a different set of requirements & different sets of python packages might be required to support it. The versions of each package can differ or break with each python or dependent packages versions update, so it is important to isolate every project within an enclosed virtual environment.

## Anaconda

[Anaconda](https://www.anaconda.com) is a python installation that bundles all essential packages for a data science project, hence by default most people will have this installed. It comes with its own virtual environment. Here are the basic commands to create & remove.

```bash
conda create -n <yourenvname> python=3.7
pip install -r requirements.txt
conda activate <yourenvname>
conda deactivate

conda env list
conda env remove -n <yourenvname>
```

## VENV

venv is my next favourite, as it is a default library shipped with python, and very easy to setup. The libraries will be installed in the directory you are in, so remember to add the folder in `.gitignore`. To remove the libraries, just delete the folder that was created.

```bash
cd <projectfolder>
python -m venv <name>
source <name>/bin/activate
pip install -r requirements.txt
deactivate
```