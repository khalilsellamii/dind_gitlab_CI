# This is a step by step guide to creating our first working pipeline using Gitlab CI/CD

## Architecture

<p align="center">
<img src="https://github.com/khalilsellamii/dind_gitlab_CI/blob/main/azeaz.png" alt="Alt text" width="800" height="400">
</p>  

## 1. Using Docker in Docker image without tls verification

> PS: we need to setup the pipeline environment to contain :

DOCKER_HOST = `tcp://docker:2375`  
DOCKER_TLS_CERTDIR = `""`

### 1.0 Setup our local gitlab runner in which our pipeline will be executed

```
$  curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
```

after installing the runner, we need to register it :
```
$ sudo gitlab-runner register 
```				

> To get the registration token, go to Settings > CI CD > runners > registartion token				

### 1.1 Build our pipeline				
```
stages:
  - test
  - build
  - deploy

run_tests:
  stage: test
  tags: 
   - testing
  image: python:3.9-slim-buster
  before_script:
    - apt-get update && apt-get install make
  script:
    - make test



build_image:
  stage: build
  tags: 
   - testing
  image: docker
  services:
    - docker:dind
  variables:
    DOCKER_TLS_CERTDIR: ""
    DOCKER_HOST: tcp://docker:2375
  before_script:
    - echo "before_script_checkpoint"
    - echo "$DOCKERHUB_PASS" | docker login -u khalil1234 --password-stdin
  script:
    - echo "successfully connected to docker hub"
    - docker build -t khalil1234/python_gitlab_ci_cd:v1.0 .
    - docker push khalil1234/python_gitlab_ci_cd:v1.0

    
deploy_application:
  stage: deploy
  tags: 
   - testing
  before_script:
    - chmod 600 $SSH_PRIVATE_KEY
  script:
    - ssh -o StrictHostKeyChecking=no -i $SSH_PRIVATE_KEY khalil@23.96.89.172 "
       sudo su -c '
       docker ps -aq | xargs docker stop | xargs docker rm &&
       docker run -d -p8080:5000 khalil1234/python_gitlab_ci_cd:v1.0'"
```

<p align="center">
<img src="https://github.com/khalilsellamii/dind_gitlab_CI/blob/main/pipeline_succesed.png" alt="Alt text" width="1000" height="200">
</p>  

## 2. Using Docker in Docker image with tls verification
It’s pretty much the same thing, however, we need to change some variables and options

```
	sudo nano /etc/gitlab-runner/config.toml
```

<p align="center">
<img src="https://github.com/khalilsellamii/dind_gitlab_CI/blob/main/image.png" alt="Alt text" width="600" height="400">
</p> 

> As we see, we added `/certs/client` as a volume in the volumes section of the runner’s configuration.

We just modify the DOCKER_HOST variable to use 2376 instead of 2375.

Also, wemodify the DOCKER_TLS_CERTDIR to be “/certs” and not null because this is the variable that control the dind image to use tls.

```
build_image:
  stage: build
  tags: 
   - testing
  image: docker
  services:
    - docker:dind

  variables:
    DOCKER_TLS_CERTDIR: "/certs"
    DOCKER_HOST: tcp://docker:2376 
```



`[+]` Link to the application source code repositroy on gitlab : https://gitlab.com/khalilsellamii/gitlab-cicd-crash-course/-/blob/main/.gitlab-ci.yml?ref_type=heads
