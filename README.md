This module supports three use-cases:

1. **GitHub -> S3 (build artifact) -> Elastic Beanstalk (running application stack)**.
The module gets the code from a ``GitHub`` repository (public or private), builds it by executing the ``buildspec.yml`` file from the repository, pushes the built artifact to an S3 bucket,
and deploys the artifact to ``Elastic Beanstalk`` running one of the supported stacks (_e.g._ ``Java``, ``Go``, ``Node``, ``IIS``, ``Python``, ``Ruby``, etc.).
    - http://docs.aws.amazon.com/codebuild/latest/userguide/sample-maven-5m.html
    - http://docs.aws.amazon.com/codebuild/latest/userguide/sample-nodejs-hw.html
    - http://docs.aws.amazon.com/codebuild/latest/userguide/sample-go-hw.html


2. **GitHub -> ECR (Docker image) -> Elastic Beanstalk (running Docker stack)**.
The module gets the code from a ``GitHub`` repository, builds a ``Docker`` image from it by executing the ``buildspec.yml`` and ``Dockerfile`` files from the repository,
pushes the ``Docker`` image to an ``ECR`` repository, and deploys the ``Docker`` image to ``Elastic Beanstalk`` running ``Docker`` stack.
    - http://docs.aws.amazon.com/codebuild/latest/userguide/sample-docker.html


3. **GitHub -> ECR (Docker image)**.
The module gets the code from a ``GitHub`` repository, builds a ``Docker`` image from it by executing the ``buildspec.yml`` and ``Dockerfile`` files from the repository,
and pushes the ``Docker`` image to an ``ECR`` repository. This is used when we want to build a ``Docker`` image from the code and push it to ``ECR`` without deploying to ``Elastic Beanstalk``.
To activate this mode, don't specify the ``app`` and ``env`` attributes for the module.
    - http://docs.aws.amazon.com/codebuild/latest/userguide/sample-docker.html


## Usage


**IMPORTANT:** The `master` branch is used in `source` just as an example. In your code, do not pin to `master` because there may be breaking changes between releases.
Instead pin to the release tag (e.g. `?ref=tags/x.y.z`) of one of our [latest releases](https://github.com/cloudposse/terraform-aws-cicd/releases).


Include this repository as a module in your existing terraform code:

```hcl
module "build" {
    source              = "git::https://github.com/cloudposse/terraform-aws-cicd.git?ref=master"
    namespace           = "eg"
    stage               = "staging"
    name                = "app"

    # Enable the pipeline creation
    enabled             = true

    # Elastic Beanstalk
    app                 = "<(Optional) Elastic Beanstalk application name>"
    env                 = "<(Optional) Elastic Beanstalk environment name>"

    # Application repository on GitHub
    github_oauth_token  = "(Optional) <GitHub Oauth Token with permissions to access private repositories>"
    repo_owner          = "<GitHub Organization or Person name>"
    repo_name           = "<GitHub repository name of the application to be built and deployed to Elastic Beanstalk>"
    branch              = "<Branch of the GitHub repository>"

    # http://docs.aws.amazon.com/codebuild/latest/userguide/build-env-ref.html
    # http://docs.aws.amazon.com/codebuild/latest/userguide/build-spec-ref.html
    build_image         = "aws/codebuild/standard:2.0"
    build_compute_type  = "BUILD_GENERAL1_SMALL"

    # These attributes are optional, used as ENV variables when building Docker images and pushing them to ECR
    # For more info:
    # http://docs.aws.amazon.com/codebuild/latest/userguide/sample-docker.html
    # https://www.terraform.io/docs/providers/aws/r/codebuild_project.html
    privileged_mode     = true
    aws_region          = "us-east-1"
    aws_account_id      = "xxxxxxxxxx"
    image_repo_name     = "ecr-repo-name"
    image_tag           = "latest"

    # Optional extra environment variables
    environment_variables = [{
      name  = "JENKINS_URL"
      value = "https://jenkins.example.com"
    },
    {
      name  = "COMPANY_NAME"
      value = "Amazon"
    },
    {
      name = "TIME_ZONE"
      value = "Pacific/Auckland"
   }]
}
```




## Examples

### Example: GitHub, NodeJS, S3 and EB

This is an example to build a Node app, store the build artifact to an S3 bucket, and then deploy it to Elastic Beanstalk running ``Node`` stack


``buildspec.yml`` file

```yaml
version: 0.2

phases:
  install:
    commands:
      - echo Starting installation ...
  pre_build:
    commands:
      - echo Installing NPM dependencies...
      - npm install
  build:
    commands:
      - echo Build started on `date`
  post_build:
    commands:
      - echo Build completed on `date`
artifacts:
  files:
    - node_modules/**/*
    - public/**/*
    - routes/**/*
    - views/**/*
    - app.js
```

### Example: GitHub, NodeJS, Docker, ECR and EB

This is an example to build a ``Docker`` image for a Node app, push the ``Docker`` image to an ECR repository, and then deploy it to Elastic Beanstalk running ``Docker`` stack

``buildspec.yml`` file

```yaml
version: 0.2

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - $(aws ecr get-login --region $AWS_REGION)
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - docker build -t $IMAGE_REPO_NAME .
      - docker tag $IMAGE_REPO_NAME:$IMAGE_TAG $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$IMAGE_REPO_NAME:$IMAGE_TAG
  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker image to ECR...
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$IMAGE_REPO_NAME:$IMAGE_TAG
artifacts:
  files:
    - '**/*'
```

``Dockefile``

```dockerfile
FROM node:latest

WORKDIR /usr/src/app

COPY package.json package-lock.json ./
RUN npm install
COPY . .

EXPOSE 8081
CMD [ "npm", "start" ]

```



## Makefile Targets
```
Available targets:

  help                                Help screen
  help/all                            Display help for all targets
  help/short                          This help short screen
  lint                                Lint terraform code

```
## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|:----:|:-----:|:-----:|
| attributes | Additional attributes (e.g. `policy` or `role`) | list(string) | `<list>` | no |
| aws_account_id | AWS Account ID. Used as CodeBuild ENV variable when building Docker images. [For more info](http://docs.aws.amazon.com/codebuild/latest/userguide/sample-docker.html) | string | `` | no |
| branch | Branch of the GitHub repository, _e.g._ `master` | string | - | yes |
| build_compute_type | `CodeBuild` instance size.  Possible values are: ```BUILD_GENERAL1_SMALL``` ```BUILD_GENERAL1_MEDIUM``` ```BUILD_GENERAL1_LARGE``` | string | `BUILD_GENERAL1_SMALL` | no |
| build_image | Docker image for build environment, _e.g._ `aws/codebuild/standard:2.0` or `aws/codebuild/eb-nodejs-6.10.0-amazonlinux-64:4.0.0` | string | `aws/codebuild/standard:2.0` | no |
| buildspec | Declaration to use for building the project. [For more info](http://docs.aws.amazon.com/codebuild/latest/userguide/build-spec-ref.html) | string | `` | no |
| codebuild_cache_bucket_suffix_enabled | The cache bucket generates a random 13 character string to generate a unique bucket name. If set to false it uses terraform-null-label's id value | bool | `true` | no |
| delimiter | Delimiter to be used between `name`, `namespace`, `stage`, etc. | string | `-` | no |
| elastic_beanstalk_application_name | Elastic Beanstalk application name. If not provided or set to empty string, the ``Deploy`` stage of the pipeline will not be created | string | `` | no |
| elastic_beanstalk_environment_name | Elastic Beanstalk environment name. If not provided or set to empty string, the ``Deploy`` stage of the pipeline will not be created | string | `` | no |
| enabled | Enable ``CodePipeline`` creation | bool | `true` | no |
| environment_variables | A list of maps, that contain both the key 'name' and the key 'value' to be used as additional environment variables for the build | object | `<list>` | no |
| force_destroy | Force destroy the CI/CD S3 bucket even if it's not empty | bool | `false` | no |
| github_oauth_token | GitHub Oauth Token | string | - | yes |
| image_repo_name | ECR repository name to store the Docker image built by this module. Used as CodeBuild ENV variable when building Docker images. [For more info](http://docs.aws.amazon.com/codebuild/latest/userguide/sample-docker.html) | string | `UNSET` | no |
| image_tag | Docker image tag in the ECR repository, e.g. 'latest'. Used as CodeBuild ENV variable when building Docker images. [For more info](http://docs.aws.amazon.com/codebuild/latest/userguide/sample-docker.html) | string | `latest` | no |
| name | Solution name, e.g. 'app' or 'jenkins' | string | - | yes |
| namespace | Namespace, which could be your organization name, e.g. 'eg' or 'cp' | string | `` | no |
| poll_source_changes | Periodically check the location of your source content and run the pipeline if changes are detected | bool | `true` | no |
| privileged_mode | If set to true, enables running the Docker daemon inside a Docker container on the CodeBuild instance. Used when building Docker images | bool | `false` | no |
| region | AWS Region, e.g. `us-east-1`. Used as CodeBuild ENV variable when building Docker images. [For more info](http://docs.aws.amazon.com/codebuild/latest/userguide/sample-docker.html) | string | `` | no |
| repo_name | GitHub repository name of the application to be built (and deployed to Elastic Beanstalk if configured) | string | - | yes |
| repo_owner | GitHub Organization or Person name | string | - | yes |
| stage | Stage, e.g. 'prod', 'staging', 'dev', or 'test' | string | `` | no |
| tags | Additional tags (e.g. `map('BusinessUnit', 'XYZ')` | map(string) | `<map>` | no |

## Outputs

| Name | Description |
|------|-------------|
| codebuild_badge_url | The URL of the build badge when badge_enabled is enabled |
| codebuild_cache_bucket_arn | CodeBuild cache S3 bucket ARN |
| codebuild_cache_bucket_name | CodeBuild cache S3 bucket name |
| codebuild_project_id | CodeBuild project ID |
| codebuild_project_name | CodeBuild project name |
| codebuild_role_arn | CodeBuild IAM Role ARN |
| codebuild_role_id | CodeBuild IAM Role ID |
| codepipeline_arn | CodePipeline ARN |
| codepipeline_id | CodePipeline ID |




