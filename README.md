![build-status](https://img.shields.io/docker/build/alessandrob/kops-kubectl.svg)
![build-automated](https://img.shields.io/docker/automated/alessandrob/kops-kubectl.svg)
# aws-codebuild-git 
base image for AWS codebuild

### What is it?
A simple docker image to facilitate deployments on AWS codebuild. It has already ssh-client installed, so that it can be used with a [deploy key](https://developer.github.com/v3/guides/managing-deploy-keys/) to push tags on successful merge of a pull request. It also conveniently add [aws-codebuild-extras](https://github.com/alessandrobologna/aws-codebuild-extras), forked from the [original](https://github.com/thii/aws-codebuild-extras)  to generate convenient environment variables for your deployment script. 

### Usage
Create an ssh deploy key for your repo called _repo_name_:

```bash
ssh-keygen -t rsa -N "" -b 4096 -C "codebuild@example.com" -f repo-name_rsa
```
This will generate a public (identified with `.pub` extension) and a private key.
Please note that each repository will need their own keys, unless you use a personal key or a "machine user" key (see the github explanation in the link above).

Then, in your repository settings, click "add deploy key":

![deploy](https://developer.github.com/assets/images/add-deploy-key.png)

You will have to use the public key that was generated above, and make sure you are giving write access to it.

Now, you can safely store the private key in [AWS Systems Manager Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-paramstore.html)

```bash
aws ssm put-parameter --name 'repo-name_rsa' --type "SecureString" --value "$(cat repo-name_rsa)"
```

An example buildspec.yaml can be as follow:

```yaml
version: 0.2
env:
parameter-store:
    DEPLOY_KEY: "repo-name_rsa"
phases:
  install:
    commands:
      - . /usr/local/bin/codebuild-extras
  pre_build:
    commands:
  build:
    commands:
      - /bin/bash deploy
  post_build:
    commands:
      - /usr/local/bin/git-tagger "${DEPLOY_KEY}"
  ```
The `deploy` script would be part of your project, and would do whatever you need to build, test, and ship the code, based on the commit information available as environment variables thanks to `aws-codebuild-extras`.
The `git-tagger` script will take the deploy key from the parameter store, and push a tag (based on the current timestamp) if a pull request is merged to master.

As an example workflow, you could use a similar script to the one below for deploying to a DEV, QA and PROD environment,  based on the event type:

```bash
#!/bin/bash

if [[ "master" = "$CODEBUILD_GIT_BRANCH" ]] 
then
	if [[ "$CODEBUILD_GIT_TAG" =~ ^v[0-9]{8}-[0-9]{6} ]]
	then
		echo "==> Deploying $CODEBUILD_GIT_TAG to PROD"
	else 
	  echo "==> Merged to master, will just tag this release"
	fi
elif [[ "$CODEBUILD_PULL_REQUEST" == "false" ]]
then
  echo "==> Deploying branch $CODEBUILD_GIT_BRANCH to DEV"
else
  echo "==> Deploying PR/$CODEBUILD_PULL_REQUEST to QA"
fi

```
This script will:

- build and deploy to DEV for a push to a branch
- build and deploy to QA if the current branch is part of a pull request
- tag a relase if a pull request is merged to master
- finally, deploy to PROD if a tag is pushed that matches the regex for CODEBUILD_GIT_TAG (so after a PR merge)

### Customization
By default, `git-tagger` will tag builds out the master branch, but this can be overriden defining `MAIN_BRANCH` to be another branch. Also, the format of the git tag is by default `v$(date "+%Y%m%d-%H%M%S")` but it can be overridden setting `GIT_TAG`
