# Virtual Environments

## Virtual Environment

Every project has a different set of requirements and different set of python packages to support it. The versions of each package can differ or break with each python or dependent packages update, so it is important to isolate every project within an enclosed virtual environment.

### Anaconda

Anaconda is a python installation that bundles with all essential packages for a data science project, hence by default most people will have this installed. It comes with its own virtual environment. Here are the basic commands to create 

```bash
conda create -n <yourenvname> python=3.7
pip install -r requirements.txt
conda activate <yourenvname>
conda deactivate

conda env list
conda env remove -n <yourenvname>
```

### VENV

venv is my next favourite, as it is a default library shipped with python, and very easy to setup. The libraries will be installed in the directory you are in, so remember to 
To remove the libraries, just delete the folder that was created.

```bash
cd <projectfolder>
python -m venv <name>
source <name>/bin/activate
pip install -r requirements.txt
deactivate
```