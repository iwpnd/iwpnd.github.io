---
layout: post
title: How to continuously deploy a fastAPI to AWS Lambda with AWS SAM
tags: [python, fastapi, aws, aws lambda, aws sam]
categories: python fastapi aws
date:   2020-01-21 13:37:00 +0200
toc: true
github_comments_issueid: "1"
---

My last [article about fastAPI](https://iwpnd.pw/articles/2020-01/opinion-on-fastapi) was supposed to be an article about how to deploy a fastAPI on a budget, but instead turned out to be an opinion on fastAPI and I left it at that. Let's change that.

FastAPIs [documentation](https://fastapi.tiangolo.com/) is exhaustive on all accounts. It gets you started real quick, takes you by the hand if it gets more complicated and even describes features in details when it doesn't really have to. I like it. Where it falls short however is when it comes to [deployment](https://fastapi.tiangolo.com/deployment/). That's probably due to the fact that there are gazillions of ways to deploy an API. The documentation proposes to use [Docker](https://docker.com/) and while I understand that this is the way to go for most companies and most applications, I don't see why I would want to deploy a small application with a few undemanding endpoints to a Docker Swarm or even Kubernetes cluster. Those come with a (hefty) price tag and are not interesting for the private person who just wants to get his hand dirty on fastAPI a little.

So let's use this article to start over and learn how to setup a basic continuous deployment pipeline for a fastAPI app on a budget. We will be using [AWS API Gateway](https://aws.amazon.com/api-gateway/), [AWS Lambda](https://aws.amazon.com/lambda/) the serverless computing services by AWS and [Travis](https://travis-ci.com/). Both AWS Lambda and AWS API Gateway are billed per API call and by the amount of data that you transfer. If you're still eligible for the free tier of AWS you can use those two services completely free for the scope of this tutorial. If not refer to the pricing example of [AWS API Gateway](https://aws.amazon.com/api-gateway/pricing/#Pricing_Examples) and [AWS Lambda](https://aws.amazon.com/lambda/pricing/). Take a particular detailed look on the AWS Lambda pricing as this depends on the time your application is running, and the memory size that is provisioned to the AWS Lambda. Luckily AWS finally shed some transparency on that matter with a proper [calculator](https://aws.amazon.com/lambda/pricing/#Calculator).

## prerequisites
Let's assume that you already have registered an [AWS account](https://aws.amazon.com/free), setup a user other than `root` and have installed and configured [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html) properly. The latter is not requirement, as you do everything we do in the AWS console, yet it's not a bad idea to learn the AWS CLI anyways.

### Setup a new role for AWS Lambda to assume
First we setup a role that the AWS Lambda will assume within our AWS account. Roles are AWSs way to enforce the [principle of least privilege](https://en.wikipedia.org/wiki/Principle_of_least_privilege) when it comes to the resources an AWS Lambda can execute or have access to. Go to the [IAM console](https://console.aws.amazon.com/iam/), select *Roles* and *Create*. That'll take you into the *Role creation* screen where you will choose a *AWS service* as the trusted entity and select *Lambda* as the service that will use the role we're going to create. Press next and it'll take you to the *Permissions* tab. Here you can either create a permission policy from scratch or select one of the existing ones. Keep in mind that the permissions vary depending on the purpose of the Lambda function. We'll take an existing permission for now. Search for `lambda` and select the `AWSLambdaBasicExecutionRole` which only allows our Lambda to write logs to AWS Cloudwatch. Press next, give it a tag or don't and press *Review*. Now you're prompted to give it a name. Choose a meaningful name (e.g. fastapilambdarole) and continue to create the role, which takes you pack to *Roles* where you click your newly created role and mark down the Role ARN ([Amazon Resource Name](https://docs.aws.amazon.com/general/latest/gr/aws-arns-and-namespaces.html)). 

### Setup an AWS S3 bucket
For small applications that only use vanilla python without external libraries one could quickly copy and paste the code into the AWS Lambda console. Bigger applications that use third-party libraries however will either be uploaded as a zip file, or in our case we will provide the location of our deployment package as an AWS S3 bucket. If you have AWS CLI properly setup you can create a bucket with:

```
aws s3api create-bucket \
--bucket my-travis-deployment-bucket \
--region eu-west-1 \
--create-bucket-configuration LocationConstraint=eu-west-1
```
Otherwise you navigate to to the [S3 console](https://s3.console.aws.amazon.com/s3/) and create one there.

### Setup a Travis User in AWS
Next up we create a new AWS user that Travis can use to perform actions on AWS resources on our behalf. We could just use our own encrypted credentials and secret, but a) spider-senses intensify b) we follow the principle of least privilege again. If Travis does not need the rights to summon the mighty [p3dn.24xlarge](https://aws.amazon.com/ec2/instance-types/p3/), Travis will stick to creating AWS Lambdas.  
First we will create a new policy. This time we have to be more specific because we need Travis to have access to AWS S3 for the deployment package, AWS Cloudformation to build our API stack, API Gateway and AWS Lambda. You can do it in the AWS console or with AWS CLI ([see here](https://docs.aws.amazon.com/cli/latest/reference/iam/create-policy.html)). Use this policy for the example: 

<details>
<summary>

```json  
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowListBucket",
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            [...]
```
</summary>
 
```json  
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowListBucket",
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::my-travis-deployment-bucket"
            ]
        },
        {
            "Sid": "AllowPassLambdaRole",
            "Effect": "Allow",
            "Action": [
                "iam:PassRole"
            ],
            "Resource": [
                "arn:aws:iam::<your-account-id-here>:role/fastapilambdarole"
            ]
        },
        {
            "Sid": "AllowS3Actions",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObjectAcl",
                "s3:GetObject",
                "s3:DeleteObject",
                "s3:PutObjectAcl"
            ],
            "Resource": "arn:aws:s3:::my-travis-deployment-bucket/*"
        },
        {
            "Sid": "AllowLambda",
            "Effect": "Allow",
            "Action": [
                "lambda:*"
            ],
            "Resource": "*"
        },
        {
            "Sid": "AllowListPolicies",
            "Effect": "Allow",
            "Action": [
                "iam:ListPolicies"
            ],
            "Resource": "*"
        },
        {
            "Sid": "AllowApiGateway",
            "Effect": "Allow",
            "Action": [
                "apigateway:*"
            ],
            "Resource": "*"
        },
        {
            "Sid": "AllowCloudFormation",
            "Effect": "Allow",
            "Action": [
                "cloudformation:*"
            ],
            "Resource": "*"
        }
    ]
}
 
```
</details>
  

Continuing, you go to the [IAM console](https://console.aws.amazon.com/iam/), but this time you will create a user with *programmatic access* and a meaningful name such as `travisdeploymentuser` or the likes. Now we attach the policy we just created to the Travis user. We get prompted with the `AWS-ACCESS-KEY-ID` and the `AWS-SECRET-ACCESS-KEY` of the user. Note those down for now. Done. 

## The example application
For the sake of this tutorial I created a [Github repository](https://github.com/iwpnd/fastapi-aws-lambda-example) with an example application that you can use as a first step and to/or built on top.

### Application structure
Inspired by the [fastapi-realworld-example-app](https://github.com/nsidnev/fastapi-realworld-example-app), I neatly separated the pydantic models, the configuration, the endpoints and the routers.

```
.
├── Dockerfile
├── LICENSE
├── README.md
├── example_app
│   ├── __init__.py
│   ├── api
│   │   ├── __init__.py
│   │   └── api_v1
│   │       ├── __init__.py
│   │       ├── api.py
│   │       └── endpoints
│   │           ├── __init__.py
│   │           └── example.py
│   ├── core
│   │   ├── __init__.py
│   │   ├── config.py
│   │   └── models
│   │       ├── input.py
│   │       └── output.py
│   └── main.py
├── requirements.txt
├── scripts
│   └── example.ipynb
├── setup.py
├── template.yml
├── .travis.yml
├── .pre-commit-config.yaml
├── tests
│   ├── __init__.py
│   ├── test_example_endpoint.py
│   └── test_ping.py
```
For simplicities sake we have exactly two endpoints. One is `/ping` in `main.py` and the other is `/api/v1/example` that takes two integer values and returns their product. If you want you can expand this functionality with your own pedantic models and additional routes.  
I also already included a `.pre-commit-configuration.yaml` for you to start using [pre-commit](https://iwpnd.pw/articles/2020-01/pre-commit-to-the-rescue) right away. It comes with pre configured hook for [black](https://iwpnd.pw/articles/2020-01/black-python-code-formatter).  

### Test the application locally
To test the example application locally we have a couple of options. One is by cloning the [repository](https://github.com/iwpnd/fastapi-aws-lambda-example) and starting it locally with uvicorn. The other is to build a docker image from the `Dockerfile` in the repository and expose the app from within a container.

```
git clone https://github.com/iwpnd/fastapi-aws-lambda-example
cd fastapi-aws-lambda-example
# create and activate a virtual environment
pip install -e .
pip install uvicorn # or anything else that can handle ASGI
pytest . -v
uvicorn example_app.main:app --host 0.0.0.0 --port 8080 --reload
```

Or

```
docker build -t example_app_image .
docker run -p 8080:8080 -name example-app-container example_app_image
```

No matter what you choose you can now go to your browser and check the applications documentation via [http://localhost:8080/docs](http://localhost:8000/docs) and test the API through the Swagger UI right there.

## Wrap the application with Mangum
In order for this application to run with AWS Lambda & AWS API Gateway, we have to wrap it with [Mangum](https://github.com/erm/mangum). Mangum works as an adapter for [ASGI applications](https://asgi.readthedocs.io/en/latest/introduction.html) like the ones you can create with fastAPI, so that they can send and receive information from API Gateway to Lambda and vice versa.

```python
from fastapi import FastAPI
from example_app.api.api_v1.api import router as api_router
from example_app.core.config import API_V1_STR, PROJECT_NAME
from mangum import Mangum

app = FastAPI(
    title=PROJECT_NAME,
)


app.include_router(api_router, prefix=API_V1_STR)


@app.get("/ping")
def pong():
    """
    Sanity check.

    This will let the user know that the service is operational.

    And this path operation will:
    * show a lifesign

    """
    return {"ping": "pong!"}


handler = Mangum(app, enable_lifespan=False)
```

### The AWS Lambda handler
The handler is necessary in AWS Lambda for it is the function that AWS Lambda can invoke when the services executes your code. It follows a simple syntax:

```python
from your_module import Yourclass
from your_database import database

db = database.connect()

def handler(event, context):
    msg = Yourclass(
        text=event["message"],
        connection=db.connection
        )
    msg.build()
    return msg.transformed
```

You can import your own libraries, like `your_module` or `your_database` and you can create variables or database connections. Everything outside of the `handler` function will execute when the AWS Lambda is provisioned. After that you can use it within the `handler` that AWS Lambda will use on consecutive calls.   
The `event` is what AWS Lambda uses to pass in event data to the `handler`. The `context` on the other hand provides information about the invocation, function, and execution environment (see [docs](https://docs.aws.amazon.com/lambda/latest/dg/python-context-object.html) for more details). 

### Mangum as the handler for event and context
A fastapi application does not have a handler, so that's what [Mangum](https://github.com/erm/mangum) is for. It wraps the `app`, therefore will receive `event` and `context` in a AWS Lambda execution environment and will pass those on to the `app` itself.  
For this to work we have to setup AWS API Gateway proxy integration to pass the raw request to the AWS Lambda, and let the `app` decide on how to process the information and what to return, including 404s etc. This is what allows this setup in the first place.

## Deploy with AWS SAM
To deploy the AWS Lambda function we have now built, we will use the AWS Serverless Application Model ([AWS SAM](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/what-is-sam.html), an open source framework to build serverless applications. As an extension to [AWS Cloudformation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html) it integrates nicely with all the other AWS services we need and lets us build our infrastructure from code - the `template.yml` in the [repository]([repository](https://github.com/iwpnd/fastapi-aws-lambda-example)).

```yaml
Resources:
    FastapiExampleGateway:
        Type: AWS::Serverless::Api
        Properties:
            StageName: prod
            OpenApiVersion: '3.0.0'

    FastapiExampleLambda:
        Type: AWS::Serverless::Function
        Properties:
            Events:
                ApiEvent:
                    Properties:
                        RestApiId:
                            Ref: FastapiExampleGateway
                        Path: /{proxy+}
                        Method: ANY
                    Type: Api
            FunctionName: fastapi-lambda-example
            CodeUri: ./
            Handler: example_app.main.handler
            Runtime: python3.7
            Timeout: 300 # timeout of your lambda function
            MemorySize: 128 # memory size of your lambda function
            Description: fastAPI aws lambda example
            # other options, see -> docs
            Role: !Sub arn:aws:iam::${AWS::AccountId}:role/fastapilambdarole

```

There are some things we have to unpack here. What we do is, we tell AWS Cloudformation to provision resources on our behalf and to deploy them in a stack. In the *Resources* section you see the API Gateway first, then the Lambda function we want to build from the code in the *CodeUri* with the `handler` in *Handler*. We define the *Runtime* of the Lambda, as well as *MemorySize* and *Timeout*. It is important for you to attach the proper role in the *Role* section, that we have created earlier. In the *Events* section we tell AWS Cloudformation to use `FastapiExampleGateway` as the API Gateway with `{proxy+}` integration, because as you recall that's what makes this setup work in the first place.
Check out the official [Template Anatomy](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-specification-template-anatomy.html) to get a better understanding of other options available.

Once setup, you can now deploy the [fastapi-aws-lambda-example application]([repository](https://github.com/iwpnd/fastapi-aws-lambda-example)) from your local machine, through the `template` that tells AWS Cloudformation to build a stack and provision the resources necessary.

### 1. Stage: Validate the SAM template
First thing we do is to validate the SAM template to check if the yaml we provide is valid.

```
sam validate
```
```
2020-01-21 10:28:39 Found credentials in environment variables.
/path/to/fastapi-aws-lambda-example/template.yml is a valid SAM Template
```

### 2. Stage: Build the deployment package
Next up, we build the deployment package. If your application depends on packages that have natively compiled programs you pass `--use-container` and SAM will attempt to build the application in a Docker container using based on [LambCI](https://github.com/lambci). Optionally you can see what's happening in the container if you also pass the `--debug` flag.

```
sam build --use-container --debug
```

This will build the deployment package and store it in `.aws-sam/build` along with a new `template.yaml` that now also contains the values we `!Sub` 'ed or substituted like so `${AWS::AccountId}`, in the initial template.

```
Starting Build inside a container
Building resource 'FastapiExampleLambda'

Fetching lambci/lambda:build-python3.7 Docker container image......
Mounting /path/to/fastapi-aws-lambda-example as /tmp/samcli/source:ro,delegated inside runtime container

Build Succeeded

Built Artifacts  : .aws-sam/build
Built Template   : .aws-sam/build/template.yaml

Commands you can use next
=========================
[*] Invoke Function: sam local invoke
[*] Package: sam package --s3-bucket <yourbucket>

Running PythonPipBuilder:ResolveDependencies
Running PythonPipBuilder:CopySource
```

### 3. Stage: Package the application
Up next, packaging. As stated in the beginning, when an application in an AWS Lambda exceeds a certain size or has dependencies, we have to package the application and either upload it in the AWS console, or prepare an intermediate AWS S3 bucket and let AWS Lambda get the application package from there. AWS SAM requires you to do the latter, or better, does it for you if you provision a bucket and pass it with `--s3-bucket my-travis-deployment-bucket`.

```
sam package --s3-bucket my-travis-deployment-bucket --output-template-file out.yml --region eu-west-1
```
Which returns:

```
2020-01-21 10:48:12 Found credentials in environment variables.
Uploading to 2adfa5ddb62b541b7cf323cda43ee394  8523862 / 8523862.0  (100.00%)  
Successfully packaged artifacts and wrote output template to file out.yml.
Execute the following command to deploy the packaged template
sam deploy --template-file /path/to/fastapi-aws-lambda-example/out.yml --stack-name <YOUR STACK NAME>
```
Now the application is packaged and a final template has been created that will now be used to tell AWS Cloudformation where the package is.

### 4. Stage: Deploy the application
```
sam deploy --template-file out.yml --stack-name example-stack-name --region eu-west-1 --no-fail-on-empty-changeset --capabilities CAPABILITY_IAM
```

```
Waiting for stack create/update to complete
Successfully created/updated stack - example-stack-name
```

This last step will finally deploy the application `--stack-name` to a `--region`. Important to note here is the option `--no-fail-on-empty-changeset`. Deploying a new version of your application code does not change the stack itself. So running the command without this option will result in failure. With this you can however push consecutive updates of your codes to the same stack.

If you now go to the [API Gateway Console](https://eu-west-1.console.aws.amazon.com/apigateway/main/apis?region=eu-west-1) you will see your API deployed to the `prod` stage at [https://xxxxxxxxxx.execute-api.eu-west-1.amazonaws.com/prod]().

## Continuous deployment with Travis
Now for the last part, the continuous deployment through Github and Travis. The idea is that any code commit that passes an automated testing phase is automatically released into the production environment, and is accessible by the user. This means that the stages we laid out above, will no longer be executed manually by you, but instead in a Travis CI pipeline. Aight, let'se go.

1. Go to [https://travis-ci.com/](https://travis-ci.com/) and sign up with your github account.
2. Accept the Authorization of Travis CI and you'll be redirected to GitHub.
3. Click on your profile picture in the top right of your Travis Dashboard, click the green Activate button, and select the repositories you want to use with Travis CI. 
4. Select the repository of your application
5. Encrypt your `AWS-ACCESS-KEY-ID` and your `AWS-SECRET-ACCESS-KEY` of the `travisdeploymentuser` we created at the beginning like [this](https://docs.travis-ci.com/user/encryption-keys#usage), and put those in the .travis.yml file
6. Commit the .travis.yml file to your repository and check your build process in the [Travis dashboard](https://travis-ci.com/dashboard).

```yaml
language: python
cache: pip
python:
- '3.7'
install:
- pip install awscli
- pip install aws-sam-cli
jobs:
  include:
    - stage: test
      script:
        - pip install pytest
        - pip install -e .
        - pytest . -v
    - stage: deploy
      script:
        - sam validate
        - sam build --debug
        - sam package --s3-bucket my-travis-deployment-bucket --output-template-file out.yml --region eu-west-1
        - sam deploy --template-file out.yml --stack-name example-stack-name --region eu-west-1 --no-fail-on-empty-changeset --capabilities CAPABILITY_IAM
      skip_cleanup: true
      if: branch = master
notifications:
  email:
    on_failure: always
env:
  global:
  - AWS_DEFAULT_REGION=eu-west-1
  - secure: your-encrypted-aws-access-key-id
  - secure: your-encrypted-aws-secret-access-key
```


From now on every time you commit changes to your repository `master` branch, it will be immediately be deployed to AWS Lambda.

## Fazit
1. We learned how to create a new user and policies in AWS IAM
2. We now have a basic application we can build upon
3. We learned about AWS API Gateway and AWS Lambda
4. We learned how [Mangum](https://github.com/erm/mangum) works
5. We learned about AWS SAM and how to deploy an application from your own machine
6. We learned how to use Travis for continuous deployment of said application instead of doing it manually

If you have any questions feel free to reach out to me in the [example repository](https://github.com/iwpnd/fastapi-aws-lambda-example) or via mail.
