# mlops-taxi-prediction

## Requirements in the developer machine

- Python 3.9 (recommend to install Anaconda)
- Docker
- Git Bash
- Ggithub account with aws secrets set-up in the repository
- docker-compose

- AWS account and tools

  - AWS account with permissions to create infrastructure
  - AWS access key (id and secret)
  - AWS cli: Download and install AWS cli
    - https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html
    - Windows
      ```
      msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi
      aws --version
      aws-cli/2.4.24 Python/3.8.8 Windows/10 exe/AMD64 prompt/off
      ```
  - Terraform
    - Download Terraform executable: https://www.terraform.io/downloads
    - Save it to ./infrastructure directory

## Setup

  - AWS cli
      ````bash
      aws configure
      AWS Access Key ID [None]: [your aws key id]
      AWS Secret Access Key [None]: [your asw secret access key]
      Default region name [None]: eu-west-1
      Default output format [None]:
      aws sts get-caller-identity
      ```

  - Create terraform backend bucket to keep Terraform state
    Note: bucket names shall be unique. Choose your location accordingly.
    Note: bucket is private, objects  but anyone with appropriate permissions can grant public access to objects.
    ```bash
    aws s3api create-bucket --bucket [your bucket name] --create-bucket-configuration LocationConstraint=eu-west-1
    ```

### Fork and clone repo

Go to:
- https://github.com/MarcosMJD/mlops-chicago-taxi

And fork the repo
Then clone the forked repo

### Infrastructure

Build staging infrastructure

Edit main.tf file in infrastructure directory and modify
  backend "s3" {
    bucket  = "chicago-taxi-tfstate-mmjd" <- Use your own bucket name
to use that created previously

Required: Use Big Bash in this step

cd infrastructure
./terraform.exe init
terraform plan --var-file=stg.tfvars
terraform apply --var-file=stg.tfvars
yes

Keep the outputs for later

### Dependencies

Go to sources directory and run:

pip install --upgrade pip
pip install pipenv
pipenv install --dev
pipenv shell

### Pre-commit hooks
Go to the root directory of the repo and run

pre-commit install
git add .pre-commit-config.yaml

This last step is needed because .git folder is not cloned and pre-commits live there

## Access to servers

From the Terraform output, get the outputs and run:

prefect config set PREFECT_API_URL="http://<external-ip>:8080/api"
export MLFLOW_TRACKING_URI="http://<external-ip>:8080"
export PROJECT_NAME="chigaco_taxi"
export PROJECT_ID_HYPHENS="chicago-taxi"
export BUCKET_NAME="<s3-bucket-name>"
<s3-bucket-name> is the one made by Terraform in the IaC stage

## ML project cifecycle

### Developing

Go to sources/development directory and run
Jupyter notebook
Go to your browser, load model_development.ipynb
And execute the notebook

Check your experiments and models in the mlflow server url
http://<external-ip>:8080

### ML pipeline

Trainning Pipeline: manual

Go to sources/development and run
python trainning_pipeline.py
Check your flow run in prefect server url
http://<external-ip>:8080

Deploy trainning pipeline

Run:
python prefect_deployment.py

Go to Prefect server url and check the deployment, block and queue

Run trainning pipeline by agent

To start the agent, run:
prefect agent start chicago-taxi

To launch the run of the deployment, on another shell execute:
In sources directory, run pipenv shell
Run prefect config set PREFECT_API_URL="http://<external-ip>:8080/api"
run prefect deployment run main-flow/chicago-taxi-deployment

### Model Deployment

Use CI/CD to deploy new server/model.

CI

E.g. <new-branch> = "Feature1"
git checkout -b ＜new-branch＞ develop
Modify any file in the sources directory
git add -A
git commit -m 'test ci/cd'
git push
go to github.com to your forked repo and
go to Pull requests
Click on New pull request
Select base:develop
Compare: <new-branch>
Click Create pull request
Click Create pull request
Go to Actions
Check CD test sucessfully passed

CD



### Monitoring

## Continue
Set model env vars to lambda in CI/CD
- Prefect Agent
- Monitoring
- Quality checks

## Notes
Since we use tag latest and the same image name and same ECR,
by simply terraform apply will make and push de image, but not update
lambda. Because lambda parameters does not change.
In CD, we build and push the image, but not update the lambda image.
What we do is update the function configuration with the environment vars to
update the model.
To update lambda image, we need to use update-function-code --image-uri
Use mlflow model registry to get stg model
  Check how to do also in CD.


## ToDo
- Pass parameters to prefect deployment
- Separate creation of s3 bucket, mlflow and prefect servers from the rest to avoid recreation of these in CD because of random generation number. Use random number generation again

- Use pipelines or save dv as an artifact
- Manage passwords (e.g. database) in aws
  - mlflow https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/setup-credentials.html
- Make user_data persistent, so that after reboot the ec2, it still works
- Check lib versions in pipfiles
- Use logging in prefect
- Use S3 to store datasets
- Check no cache when using pipenv in Dockerfile
- Improve paths in the tests. Use current python script file to import other modules,
  Similar to .sh files.
- Check why aws config initialization fails when github actions if profile default is set in main.tf
- Deployment en Makefile?
- Test localstack aws gateway + ECR + lambda + S3
- In dev system... maybe script:
  - Set mlflow env var for the server
  - Set prefect to use prefect server api
  - Modify model and preprocessor to use pipeline or model
  - ignore files in prefect
  - Unit test lambda is loading S3 model actually. Find a way to avoid this.

Bugs
 - Nothing







## Usefull commands and snippets

### Shell script

Header of a sh script
"#!/usr/bin/env bash"

Returns the directory where the Bash script file is saved
"$(dirname "$0")"

Get last error code
ERROR_CODE=$?

### Prefect
o create programmatically an storage block in prefect:
from prefect.filesystems import S3
block = S3(bucket_path="chicago-taxi-fc4rdz8d")
block.save("example-block")

Then to build a deployment
prefect deployment build trainning_pipeline.py:main_flow --name test --tag test --storage-block s3/example-block -q test
This will upload the .py files to S3 bucket

Or with object:

To create the deployment and queue (in prefect orion server), and also upload to S3 the yaml file:
prefect deployment apply .\main_flow-deployment.yaml

To start an agent
prefect agent start -q 'test'

To run a flow
prefect deployment run <FLOW_NAME>/<DEPLOYMENT_NAME>

## aws cli and aws-api

Download object from aws s3
aws s3api get-object --bucket stg-chicago-taxi-fc4rdz8d --key mlflow/2/e4ff37b7254a408c86826fb2a25573a9/artifacts/model/conda.yaml ./conda.yaml

Download folder from aws s3
aws s3 cp s3://stg-chicago-taxi-fc4rdz8d/mlflow/2/e4ff37b7254a408c86826fb2a25573a9/artifacts/model ./model --recursive

Get latest RUN_ID from latest S3 partition. In practice, this is generally picked up from a tool like MLflow or a DB

export RUN_ID=$(aws s3api list-objects-v2 --bucket ${MODEL_BUCKET_DEV} \
--query 'sort_by(Contents, &LastModified)[-1].Key' --output=text | cut -f2 -d/)

Copy between buckets
aws s3 sync s3://${MODEL_BUCKET_DEV} s3://${MODEL_BUCKET_PROD}

Update lambda env vars
https://docs.aws.amazon.com/lambda/latest/dg/configuration-envvars.html
aws lambda update-function-configuration --function-name ${LAMBDA_FUNCTION} --environment "Variables=${variables}"


## Pre-commit

Create a default precommit hook:
  pre-commit sample-config > .pre-commit-config.yaml
  pre-commit install
  git add .pre-commit-config.yaml

Pytest only adds to sys.path directories where test files are, so you need to add the sources directory with:
export PYTHONPATH=. fro sources directory

You alternatively can run (not checked)
python -m pytest

## Docker-compose
Stop services only
docker-compose stop

Stop and remove containers, networks..
docker-compose down

Down and remove volumes
docker-compose down --volumes

Down and remove images
docker-compose down --rmi <all|local>
