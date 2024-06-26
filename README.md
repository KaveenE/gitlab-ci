[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)


# Gitlab CI/CD
A **zero to production** tutorial using Gitlab for CI/CD management. In this tutorial the setup enviroment will be:

- GitLab `(Code repository)`
- Digital Ocean `(Infrastructure)`
- Docker `(Containers)`

### Note

> **You don't need to use Gitlab as your code repository exactly, you also can use Github, check this link if you want to use Gitlab only for manage your CI/CD https://docs.gitlab.com/ee/ci/ci_cd_for_external_repos/github_integration.html**

## Summary
1. [Create a droplet](#digital-ocean)
    1. [Create a public key](#ssh)
2. [Configure your droplet](#config)
3. [Create a gitlab runner](#runner)
    1. [Install gitlab runner](#runner-install)
    2. [Register a runner](#runner-register)
4. [Create a docker image](#docker)
5. [Gitlab CI](#ci)
    1. [Enviroment Variables](#ci-ssh)
    2. [Create a gitlab-ci.yml](#ci-yml)
6. [Deploying your application](#deploy)

<div id='digital-ocean'/>

### Create a droplet

After made a account on [Digital Ocean website](https://www.digitalocean.com/), at the initial page you will click to create a `new project` on Digital Ocean:

![Projects page](img/project.png)

Now you have a project on Digital Ocean, that basically serves for centralize one, or more, droplets in a same group. Click on `Create a droplet` to choose a type of droplet and your size. Because we are focused  on create a enviroment using Docker for deploy automatization, we'll also choose a droplet that fits with that environment.

![Docker droplet](img/docker-droplet.png)

Choosing a docker droplet, the `Digital Ocean` will give us a VM that already contains docker installed and also a firewall configuration that allows us to access the standard docker port remotely.

### Note

> **If you intent to use this tutorial for deploy a Spring Boot Application, it's better to choose at least the U$10/mo machine, because the 1GB RAM isn't enough to build the project, you'll face memory overflow issues.**

<div id='ssh'/>

#### SSH Access

Add your public ssh key to `Digital Ocean`. You need to create a ssh from the machine that you want to access remotely your droplet. 

![SSH](img/ssh-key.png)

If you haven't already have a public ssh key, following this steps:

```
$ ssh-keygen -o
Generating public/private rsa key pair.
Enter file in which to save the key (/home/you/.ssh/id_rsa):
Created directory '/home/you/.ssh'.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/you/.ssh/id_rsa.
Your public key has been saved in /home/you/.ssh/id_rsa.pub.
```

And get your public ssh key: 

```
$ cat ~/.ssh/id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAasd24cZ/124zcA...you@you-pc
```
Copy and paste that on `Digital Ocean`, and that's it.

And after all, click on `Create`. 

<div id='config'/>

### Configure your droplet
After the creation of your droplet, you will able to see some information about the droplet, and the most important : `IP Address`

![IP](img/droplet.png)

Now, open a terminal and type

```
$ ssh root@your.ip.address
```
<p align="center">
  <img src="img/terminal.png" alt="Terminal Image"/>
</p>

### Disclaimer
- SSH as root is dangerous
- Create another user in droplet with superuser privileges and ssh as that  
```
# Replace 'username' with the desired username for the new account
sudo adduser username

sudo usermod -aG sudo username

su -i username

echo "{thePublicKeyYouGeneratedPreviously}" > .ssh/authorized_keys
```

### Note

> **If you can't connect remotely with some error like Connection Refused, maybe you should use the Digital Ocean Console Launcher for add manually your public ssh key on this  folder: ~/.ssh/authorized_keys**

Checking your firewall, you will see what ports are available to remote access. On the initial configuration you only have `SSH and Docker ports.`

```
$ ufw status
Status: active
To                         Action      From
--                         ------      ----
22/tcp                     ALLOW       Anywhere
2375/tcp                   ALLOW       Anywhere
2376/tcp                   ALLOW       Anywhere
22/tcp (v6)                ALLOW       Anywhere (v6)
2375/tcp (v6)              ALLOW       Anywhere (v6)        
2376/tcp (v6)              ALLOW       Anywhere (v6)
```

### Note

> **After you start a deploy your applications on this droplet, you should change this firewall configuration or also configure a proxy for remote access to your applications.**

<div id='runner'/>

### Create a gitlab runner
#### What is it

GitLab Runner is the open source project that is used to run your jobs and send the results back to GitLab. It is used in conjunction with GitLab CI, the open-source continuous integration service included with GitLab that coordinates the jobs.

<div id='runner-install'/>


#### Install

Follow these steps for install `gitlab runner` on your droplet:

Simply download one of the binaries for your system (replace `latest` with some version if you need to match with your Gitlab Instance, eg: `v16.9.1`):

```
# Linux x86-64
sudo curl -L --output /usr/local/bin/gitlab-runner "https://s3.dualstack.us-east-1.amazonaws.com/gitlab-runner-downloads/latest/binaries/gitlab-runner-linux-amd64"

# Linux x86
sudo curl -L --output /usr/local/bin/gitlab-runner "https://s3.dualstack.us-east-1.amazonaws.com/gitlab-runner-downloads/latest/binaries/gitlab-runner-linux-386"

# Linux arm
sudo curl -L --output /usr/local/bin/gitlab-runner "https://s3.dualstack.us-east-1.amazonaws.com/gitlab-runner-downloads/latest/binaries/gitlab-runner-linux-arm"

# Linux arm64
sudo curl -L --output /usr/local/bin/gitlab-runner "https://s3.dualstack.us-east-1.amazonaws.com/gitlab-runner-downloads/latest/binaries/gitlab-runner-linux-arm64"

# Linux s390x
sudo curl -L --output /usr/local/bin/gitlab-runner "https://s3.dualstack.us-east-1.amazonaws.com/gitlab-runner-downloads/latest/binaries/gitlab-runner-linux-s390x"

# Linux ppc64le
sudo curl -L --output /usr/local/bin/gitlab-runner "https://s3.dualstack.us-east-1.amazonaws.com/gitlab-runner-downloads/latest/binaries/gitlab-runner-linux-ppc64le"

# Linux x86-64 FIPS Compliant
sudo curl -L --output /usr/local/bin/gitlab-runner "https://s3.dualstack.us-east-1.amazonaws.com/gitlab-runner-downloads/latest/binaries/gitlab-runner-linux-amd64-fips"

```

Give it permissions to execute:

```
$ sudo chmod +x /usr/local/bin/gitlab-runner
```

Install and run as service:

```
$ sudo gitlab-runner install --user={yourUserName}
$ sudo gitlab-runner start
```

Verify if it is up:

```
$ sudo systemctl status gitlab-runner
● gitlab-runner.service - GitLab Runner
     Loaded: loaded (/etc/systemd/system/gitlab-runner.service; enabled; vendor preset: enabled)
     Active: active (running) since Sun 2024-04-14 01:45:01 UTC; 12s ago
   Main PID: 41017 (gitlab-runner)
      Tasks: 6 (limit: 1013)
     Memory: 12.6M
        CPU: 164ms
     CGroup: /system.slice/gitlab-runner.service
             └─41017 /usr/local/bin/gitlab-runner run --working-directory /home/developer --config /etc/gitlab-runner/config.toml --service gitlab-runner --user developer
```


<div id='runner-register'/>


#### Register a runner
Go back on your Gitlab project, on the lateral menu: `Settings > CI/CD > Runners (Expand)`

![image](https://github.com/KaveenE/gitlab-ci/assets/59110376/9f8a3c74-9be3-4fd8-abf7-95e2bb9c4c6a)

1. Toggle off `Enable Instance Runners for this project`
2. Register a runner with relevant configuration.
![image](https://github.com/KaveenE/gitlab-ci/assets/59110376/93830ccc-607b-4913-903c-4259f715ed26)
3. You will be redirected to a page like the below.
![image](https://github.com/KaveenE/gitlab-ci/assets/59110376/ad125408-df4b-40e0-a0e5-486307825a6b)

4. Copy the command from that page and paste into droplet terminal  
   4.1. Then go back to your droplet terminal and:

```
$ sudo gitlab-runner register  --url https://gitlab.com  --token {theTokenYouGotFromTheRedirectedPageAbove}
```

5. Enter your GitLab instance URL:

```
 Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com )
 https://gitlab.com
```
6. Enter the Runner executor:

```
 Please enter the executor: ssh, docker+machine, docker-ssh+machine, kubernetes, docker, parallels, virtualbox, docker-ssh, shell:
 docker
```

7. If you chose Docker as your executor, you’ll be asked for the default image to be used for projects that do not define one in `.gitlab-ci.yml`:

```
 Please enter the Docker image (eg. ruby:2.1):
 alpine:latest
```
#### Confirm runner is up
And that's it, if you refresh your gitlab runner's page on your repository, you will see:

<p align="center">
  <img src="https://github.com/KaveenE/gitlab-ci/assets/59110376/8a917983-fbd4-422f-a5de-3dbe22a2bea2" alt="Runners Image"/>
</p>

<div id='docker'/>

### Create a docker image

Before we start to build our `.gitlab-ci.yml`, you should create your `Dockerfile` to create a docker image.

#### What's docker

Docker is a computer program that performs operating-system-level virtualization, also known as "containerization".

#### Dockerfile

Here we have a example of a Dockerfile for a Spring Boot Application, but you can create a image that will fit with your application, but remender `your runner should have the same base image.`

```Dockerfile
FROM openjdk:10.0.1-slim
VOLUME /tmp
COPY build/libs/*.jar app.jar
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
```

### Note

> **You can see the official docker documentation to help you to create your own https://docs.docker.com/engine/reference/builder/**

<div id='ci'/>

### Gitlab CI

<div id='ci-ssh'/>

#### Enviroment Variables
1. Before we create the `.gitlab-ci.yml`, you need to generate a `private ssh key` on your **droplet**. It's simple, it's the same steps that we did before, but now in your droplet.

```
$  ssh-keygen -t ed25519 -C "DigitalOcean-Server-Pfp-Be-Dev"
Generating public/private ed25519 key pair.
Enter file in which to save the key (/root/.ssh/id_ed25519):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /root/.ssh/id_ed25519
Your public key has been saved in /root/.ssh/id_ed25519.pub
```

2. Get your **private ssh key**: 

```
$ cat < ~/.ssh/id_ed25519
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAACmFlczI1Ni1jdHIAAAAGYmNyeXB0AAAAGAAAABCfvf30Kq
0gV8WocQrgTrVNAAAAEAAAAAEAAAAzAAAAC3NzaC1lZDI1NTE5AAAAIF5EGh8VtNswD+qS
N8A7sD5VeP7v5x0LH0J5Bj5Q9wCzAAAAsCnNrHg5kv5kShgHk3Akjg8fI9f0sf6KbzycrV
Wo7yIG/YcePRxDRsFNzGG0pc86Nn9RsNN/fzeptd3Eh+dV94WGolqAzEnOSy7Y7M7b9p3+
+CDJdEvh2d8BEVPRshNR/gZTPIdqwlSerQEXF7vnYqtNdSwtczLDfDFhbWfrxGsNtFAqvY
eBtIJghCr+UnzHJjYMsMtj13sccf5ozolr2p+0+j4/Gyq9/sLapS5buBRv
-----END OPENSSH PRIVATE KEY-----
```

3. Base64 encode it (Necessary for next step as Gitlab)
```
cat < ~/.ssh/id_ed25519 | base64 -w0
LS0tLS1CRUdJTiBPUEVOU1NIIFBSSVZBVEUgS0VZLS0tLS0KYjNCbGJuTnphQzFyWlhrdGRqRUFBQUFBQ21GbGN6STFOaTFqZEhJQUFBQUdZbU55ZVhCMEFBQUFHQUFBQUJDZnZmMzBLcQowZ1Y4V29jUXJnVHJWTkFBQUFFQUFBQUFFQUFBQXpBQUFBQzNOemFDMWxaREkxTlRFNUFBQUFJRjVFR2g4VnROc3dEK3FTCk44QTdzRDVWZVA3djV4MExIMEo1Qmo1UTl3Q3pBQUFBc0NuTnJIZzVrdjVrU2hnSGszQWtqZzhmSTlmMHNmNktienljclYKV283eUlHL1ljZVBSeERSc0ZOekdHMHBjODZObjlSc05OL2Z6ZXB0ZDNFaCtkVjk0V0dvbHFBekVuT1N5N1k3TTdiOXAzKworQ0RKZEV2aDJkOEJFVlBSc2hOUi9nWlRQSWRxd2xTZXJRRVhGN3ZuWXF0TmRTd3RjekxEZkRGaGJXZnJ4R3NOdEZBcXZZCmVCdElKZ2hDcitVbnpISmpZTXNNdGoxM3NjY2Y1b3pvbHIycCswK2o0L0d5cTkvc0xhcFM1YnVCUnYKLS0tLS1FTkQgT1BFTlNTSCBQUklWQVRFIEtFWS0tLS0tCg==
```
4. Go to `Settings > CI/CD > Variables (Expand)`  
  4.1. Create a variable called **SSH_PRIVATE_KEY** or any name of your choosing with relevant configuration and paste the base64 encoded version  
  4.2 Gitlab has some constraints on what can be pasted as value. Thus, base64 was needed

![image](https://github.com/KaveenE/gitlab-ci/assets/59110376/752f552c-5742-43d9-8ac0-588c6902ecfc)


That will allow us to send the files and execute commands inside of `.gitlab-ci.yml.`

<div id='ci-yml'/>

#### Create a gitlab-ci.yml

```yml
image: openjdk:10.0.1-slim

stages:
  - build
  - deploy_test
```

**image:** Defines a docker image that you're using.
**stages:** Defines a job stage (ex.: build_app).

> These stages names not necessary should be equals to your tags name, that was defined on your `gitlab runner.`

```yml
variables:
  SERVICE_NAME: 'service-name'
  DEPLOY_PATH: '/home/projects/$SERVICE_NAME'
  SERVICE_DIST_FILE_PATH: 'build/libs/*.jar'
  APP_PORT: 8080
  DISTRIBUTION: '0.0.1-SNAPSHOT'
```

**variables:** Define job variables on a job level.

```yml
before_script:
  - echo "Before script..."
  - 'which ssh-agent || ( apt-get update -y && apt-get install openssh-client -y )'
  - apt-get -y install zip unzip
  - eval $(ssh-agent -s)
  - echo "$SSH_PRIVATE_KEY" | tr -d '\r' | ssh-add - > /dev/null
  - mkdir -p ~/.ssh
  - chmod 700 ~/.ssh
```

**before_script:** Override a set of commands that are executed before job

> Here we're installing ssh to can connect to our droplet on Digital Ocean.

```yml
build:
  stage: build
  script:
  - echo "Initiliaze build task..."
  - ./gradlew clean build
  - echo "Finalized build task..."
  artifacts:
    untracked: true
  tags:
    - build
```

**script:** Defines a shell script which is executed by Runner.

**artifacts:** Define list of [job artifacts](https://docs.gitlab.com/ee/ci/yaml/#artifacts)

**tags:** Defines a list of tags which are used to select Runner.

> Here defined a simple pipeline stage called `build`, that have a single responsability: **guarantee that application it's building**. If no error ocurred, then we'll get the artifacts created to our next stage.

```yml
deploy_test:
  variables:
    DEPLOY_HOST: 'root@your-ip-addres'
    DEPLOY_PATH: 'deploy-path/$SERVICE_NAME'
    DIST_FILE: '$SERVICE_NAME-$DISTRIBUTION.jar'
    DOCKER_IMAGE_NAME: '$SERVICE_NAME/java'
  stage: deploy_test
  tags:
    - deploy_test
  dependencies:
  - build
```

**dependencies:** Define other jobs that a job depends on so that you can pass artifacts between them

And finally, here we have the script that deploy our application, in a `Docker container` inside a `Digital Ocean` droplet.

```yml
script:
  - echo "Deploy prod starting..."
  - ssh -o StrictHostKeyChecking=no $DEPLOY_HOST 'exit'
  - ssh $DEPLOY_HOST "rm -rf $DEPLOY_PATH/*"
  - ssh $DEPLOY_HOST "mkdir -p $DEPLOY_PATH/"
  - zip -r $DIST_FILE Dockerfile $SERVICE_DIST_FILE_PATH
  - scp $DIST_FILE $DEPLOY_HOST:$DEPLOY_PATH/
  - ssh $DEPLOY_HOST "unzip $DEPLOY_PATH/$DIST_FILE -d $DEPLOY_PATH/"
  - ssh $DEPLOY_HOST "rm $DEPLOY_PATH/$DIST_FILE"
  - ssh $DEPLOY_HOST "docker system prune -f"
  - ssh $DEPLOY_HOST "cd $DEPLOY_PATH/ && docker build -f Dockerfile -t $DOCKER_IMAGE_NAME ."
  - ssh $DEPLOY_HOST "docker stop $SERVICE_NAME || true && docker rm $SERVICE_NAME || true"
  - ssh $DEPLOY_HOST "docker run -e APP_PORT=$APP_PORT -p $APP_PORT:$APP_PORT --name $SERVICE_NAME -d $DOCKER_IMAGE_NAME"
  - echo "Deploy prod finished"
```

#### Comments about deploy script

* StrictHostKeyChecking=no allows you to disable strict host key checking

* Then we delete the previous folder that contains your app and create a new one.

* Then we zip our files and send them to the droplet for unzip there.

* Remove all unused containers, networks, images (both dangling and unreferenced), and optionally, volumes with `docker system prune -f`.

* We use `cd command` to go for the folder that have the files and them create a docker image using the `Dockerfile`.

* Them stop previous docker container running your service, with pipe true because you couldn't have a docker container with your `$SERVICE_NAME`.

* And finally use docker run for running a new container with the image that you created.

> **You can check a complete documentation guide on the official gitlab page https://docs.gitlab.com/ee/ci/yaml/** 


<div id='deploy'/>


### Deploying your application

It's simple, you add a `.gitlab-ci.yml` and a `Dockerfile` in your project root folder, commit and that's it.

On your gitlab repository, in `CI/CD` menu you will see something like that:

![Deploy CI/CD](img/deploy.png)

And that's it. 

If you have any doubt about this tutorial, just ask me!

### License
Apache License. [Click here for more information.](LICENSE)
