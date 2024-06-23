
## Goals

Create a Python package, test it, deploy Databricks bundle with
- Wheel build
- Workflow with a job to run a notebook that uses the wheel
- Run the job.

# steps


1. Create .gitignore file

```txt
.databricks/
build/
dist/
__pycache__/
*.egg-info
.venv/
.pytest_cache/
```

2. Initialise your git repository and commit code

```bash
git init
git add .
git commit -m "Initial commit"
```

3. Create a folder to hold the Python package.

```bash
├── src
│   └── python_package
│       ├── __init__.py
│       └── module.py
```

4. Write your package code.

for __init__.py

```python
__version__ = "0.0.1"
```

for module.py

```python
def hello():
    return "Hello, World!"
```

5. Create a virtual environment

```bash
python -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
```

6. install the package dependencies.

file requirements-dev.txt

```txt
pytest
setuptools
wheel
```

```bash
pip install -r requirements-dev.txt
```

7. Create a setup.py file

```python

from setuptools import setup, find_packages

import sys
sys.path.append('./src')

from datetime import datetime, timezone
import python_package

setup(
    name="python_package",
    version=python_package.__version__ + "+" + datetime.now(timezone.utc).strftime("%Y%m%d.%H%M%S"),
    author="pedro.junqueira@agile-analytics.com.au",
    description="wheel file example based on python_package/src",
    packages=find_packages(where='./src'),
    package_dir={'': 'src'},
    install_requires=[
        "setuptools"
    ],
)
```
8. install your package in interactive (editable) mode

```bash
pip install -e .
```

9. Write tests for your package.

create a  test folder in the root of the project.

file test_my_module.py

```python
from python_package import module

def test_hello():
    assert module.hello() == "Hello, World!"
 
```

10. Run the tests

```bash
python -m pytest
```

11. Create a databricks bundle configuration file

databricks.yaml in the root of the project

```yaml
bundle:
  name: python_package

include:
  - resources/*.yml

artifacts:
  default:
    type: whl
    path: .

targets:
  dev:
    mode: development
    default: true
    workspace:
      host: https://<your-workspace>
```

12. Create a databricks resorce files in the folder resources

```yaml
resources:
  jobs:
    python_package_job:
      name: python_package_job
      tasks:
        - task_key: notebook_task
          job_cluster_key: job_cluster
          notebook_task:
            notebook_path: ../src/notebook.ipynb
          job_cluster_key: job_cluster
          libraries:
            - whl: ../dist/*.whl

      job_clusters:
        - job_cluster_key: job_cluster
          new_cluster:
            spark_version: 13.3.x-scala2.12
            node_type_id: Standard_D3_v2
            autoscale:
                min_workers: 1
                max_workers: 2
```
13. Create a notebook for the job inside the src folder

```python
from python_package import module

result = module.hello()

print(result)
```

14. Validate the bundle

```bash
databricks bundle validate
```

15. Deploy the bundle to your Databricks workspace

```bash
databricks bundle deploy -t dev
```

Check if the following artifacts were created

Under Workspace folder -> Users/your_user/.bundle/python_package/dev/artifacts/.internal

you should see the wheel file

under workflows you should see the job called python_package_job with a prefix [dev your user name] in the name


16. the state of your project should be like this

```bash
.
├── .gitignore
├── setup.py
├── README.md
├── databricks.yaml
├── requirements-dev.txt
├── resources
│   └── python_package.yml
├── src
│   ├── python_package
│   │   ├── __init__.py
│   │   └── module.py
│   └── notebook.ipynb
└── test
    └── test_module.py
```

17. run the job

```bash
databricks bundle run python_package_job
```
go to Databricks workspace in Workflows and check if job run successfully.

## Useful Commands

build python wheel file

```bash
python setup.py bdist_wheel
```

Distrubuting package

https://packaging.python.org/en/latest/guides/distributing-packages-using-setuptools/

# Adding CI/CD to your project with GitHub Actions

The goal is to deploy the bundle artifacts to production when code is merged to a master branch.

In a CI/CD and in production you should to it using a service principal, which is a system account that can be used to authenticate to Azure services and deploy and run production workloads.

Here are the steps to add CI/CD to your project with GitHub Actions.

1. Create an Azure Service Principal and contributor role to the resource group scope

```bash
RESOURCE_GROUP_ID=$(az group show --name "YourResourceGroupName" --query id -o tsv)
az ad sp create-for-rbac --name "YourServicePrincipalName" --role "Contributor" --scopes $RESOURCE_GROUP_ID
```
the command line will print the following output

```json
{
  "appId": "UUID",
  "displayName": "<name-foryour-sp>",
  "password": "system-generated-password",
  "tenant": "<your-tenant-id>"
}
```
Save this information in a safe place.

2. Add service principal to your Databricks Workspace and grant access to your workspace

You need to be Azure Account Admin to do this

You can follow the steps (Steps 3, 4 and 5)[https://learn.microsoft.com/en-us/azure/databricks/dev-tools/auth/oauth-m2m#step-3-add-the-service-principal-to-your-azure-databricks-workspace]

4. Create a Databricks personal token to your Azure Service Principal using the databricks cli


Your cannot use the UI to create a Databricks personal token for a service principal, you need to use the databricks cli.

[details here](https://learn.microsoft.com/en-us/azure/databricks/admin/users-groups/service-principals)

First to autenticate and create a token your service principal needs to be in the Databricks cli configuration file located in `~/.databrickscfg`

Open the file with VS code

```bash
code ~/.databrickscfg
```

add this to your configuration file

```txt
[your-sp-profile-name]
host          = https://<host>.azuredatabricks.net
client_id     = <appId>
client_secret = <service principle Oauth Token>
```

check if profile is working

```bash
databricks auth profiles
databricks auth env --profile dab-sp
```

also validate bundle if it is on the right profile

```bash
databricks bundle validate --profile dab-sp -t dev
```
finally creater a PAT (personal access token)

```bash
databricks tokens create --comment dab -p your-sp-profile-name
```

it will return a json with the following format

```json
{
  "token_info": {
    "comment":"dab",
    "creation_time":1718442556929,
    "expiry_time":-1,
    "token_id":"uuid"
  },
  "token_value":"dapi-number-long-string"
}
```
save token_value in a safe place.

5. Add the following secrets to your GitHub repository

SP_TOKEN=token_value

[details](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions)

6. Create a GitHub Actions workflow file

Create a folder .github/workflows in the root of your project

create a file called prod-deployment.yml

```yaml
name: "Prod deployment"

concurrency: 1

on:
  push:
    branches:
      - master

jobs:
  test_package:
    name: "Checkout, Install, and Test"
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install package in editable mode
        run: pip install -e .

      - name: Install dependencies
        run: pip install -r requirements-dev.txt

      - name: Run tests
        run: python -m pytest

  deploy:
    name: "Deploy bundle"
    runs-on: ubuntu-latest
    needs:
      - test_package

    steps:
      - uses: actions/checkout@v3

      - uses: databricks/setup-cli@main

      - run: databricks bundle deploy
        working-directory: .
        env:
          DATABRICKS_TOKEN: ${{ secrets.SP_TOKEN }}
          DATABRICKS_BUNDLE_ENV: prod
```

8. Add a prod target to your databricks.yaml file

```yaml
targets:
  dev:
    mode: development
    default: true
    workspace:
      host: <workspaceurl-dev>

  prod:
    mode: production
    workspace:
      host: <workspaceurl-dev>
      root_path: /Users/${workspace.current_user.userName}/.bundle/${bundle.name}/${bundle.target}
    run_as:
      service_principal_name: <service-principal-uuid>
```

9. commit and push your code to the master branch

```bash
git add . && git commit -m "test cicd" && git push origin master
```

10. Check the GitHub Actions tab in your repository to see the workflow running.

11. Check the Databricks workspace to see if the job was created and run successfully.

12. Check the artifacts in the production workspace