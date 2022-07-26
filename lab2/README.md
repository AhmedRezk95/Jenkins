## Q1: configure Jenkins image to run docker commands on your host docker daemon

- what we want to install docker inside Jenkins, we will install docker client and then we will make our Jenkins container user our local ”my pc” docker demon via docker volume

- Create Dockerfile for this task

```bash
FROM jenkins/jenkins:lts
USER root
# Install docker client
RUN apt-get update -qq
# Install dependencies
RUN apt-get install -qq apt-transport-https ca-certificates curl gnupg2 software-properties-common
# Add Docker’s GPG Key (remember it is debian not linux ) / don't set it as sudo !
RUN curl -fsSL https://download.docker.com/linux/debian/gpg | apt-key add -
# Install the Docker Repository "remember it is debian not linux)"
RUN add-apt-repository \
    "deb [arch=amd64] https://download.docker.com/linux/debian \
    $(lsb_release -cs) \
    stable"
# Update Repositories  &  Install Latest Version of Docker
RUN apt-get update -qq \
    && apt-get install docker-ce -y
# Add the user jenkins to the group docker on the system 
RUN usermod -aG docker jenkins
```

- We build a new image based on it

```bash
docker build -t jenkins-with-docker .
```

- Run a container based on this image **(set volume of my local docker daemon to reflect on Jenkins docker daemon )**

```bash
docker run -d -p 8089:8080 -v /var/run/docker.sock:/var/run/docker.sock --name JenkinsWithDocker jenkins-with-docker
```

## Q2: create CI/CD for this repo [https://github.com/mahmoud254/jenkins_nodejs_example.git](https://github.com/mahmoud254/jenkins_nodejs_example.git)

First: you have to create a repo inside [dockerhub.com](http://dockerhub.com)

- log-in inside dockerhub “ we will need username and password for your credientials”
- create public repo “Jenkins”

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/57190d49-fdf5-41fc-8ef7-16ebefc069e3/Untitled.png)

```bash

```

Second: based on Q1, we will use a Jenkins with docker installed and its daemon pointed to my local docker daemon

- set my dockerhub credentials inside Jenkins using Manage Credentials
- set username, password and ID in order to use it in the stage

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/bd58d79b-502e-46be-82cc-b4df532d8f1d/Untitled.png)

Third: Create a pipeline for the application, set the code and build it 

- pipeline code
- you will use **withCredentials** plugin in order to login to dockerhub and push the image
- we have 3 stages, 1- preparation: to git the code, 2- CI: build a docker image and login and push this image to my dockerhub repo, 3- CD: run the container

```bash
pipeline {
    agent any

    stages {
        
        stage('preparation') {
            steps {
                // Get some code from GitHub repository
                git 'https://github.com/mahmoud254/jenkins_nodejs_example.git'
            }
        }
            
        stage('ci') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) {
                    sh """ 
                    docker build . -f dockerfile -t rizk95/jenkins
                    docker login -u ${USERNAME} -p ${PASSWORD}
                    docker push rizk95/jenkins
                    """
                }
            }
        }

        stage('cd') {
            steps {
                sh "docker run -d -p 3000:3000 rizk95/jenkins"
            }
        }
    }
}
```

Finally Build your pipeline (It will take a lot of time to push to your dockerhub as per your internet speed connection but it will work at the end)

## RESULTS

![Screenshot from 2022-07-25 13-50-26.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/5d55027e-ded8-487a-9256-6e4a218c46ae/Screenshot_from_2022-07-25_13-50-26.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/fca67518-a2f1-4c59-a72d-08f9f3f3ea50/Untitled.png)

## Q3: create docker file to build image for jenkins slave

- i generated a key called “docker_rsa” for the purpose of the lab and assign and reallocate it in docker container

```bash
FROM ubuntu
USER root

################ FIRST CONFIGURE IMAGE TO BE SLAVE #################
# create a directory to set all jenkins projects
RUN mkdir -p jenkins_home
RUN chmod 777 jenkins_home
# setup the required packages for creating a jenkins slave: 
# openjdk ,openssh: for allowing ssh 
RUN apt-get update -qq
RUN apt-get install openjdk-11-jdk -qq
RUN apt-get install openssh-server -qq
# create a user called jenkins
RUN useradd -ms /bin/bash jenkins

################ SETUP DOCKER CLIENT INSIDE  SLAVE #################
# Install docker client
RUN apt-get update -qq
# Install dependencies
RUN apt-get install -qq apt-transport-https ca-certificates curl gnupg2 software-properties-common
# Add Docker’s GPG Key (remember it is ubuntu based ) / don't set it as sudo !
RUN curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
# Install the Docker Repository "remember it is ubuntu based)"
RUN add-apt-repository \
    "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
    $(lsb_release -cs) \
    stable"
# Update Repositories  &  Install Latest Version of Docker
RUN apt-get update -qq \
    && apt-get install docker-ce -y
# Add the user jenkins to the group docker on the system 
RUN usermod -aG docker jenkins

# Create shh folder in home directory and set the access for it
RUN mkdir -p /home/jenkins/.ssh
COPY docker_rsa.pub /home/jenkins/.ssh/authorized_keys
RUN chown -R jenkins:jenkins /home/jenkins/.ssh
RUN chmod 700 /home/jenkins/.ssh
RUN chmod 644 /home/jenkins/.ssh/authorized_keys

# log-in as jenkins user
USER jenkins
# cd jenkins_home
WORKDIR /home/jenkins/jenkins_home

EXPOSE 22
# run it 
CMD ["/bin/bash"]
```

```bash
docker build . -f slave_dockerfile -t jenkins-slave
```

## Q4: create container from this image and configure ssh and from jenkins master create new node with the slave container

- From the previous question
- remember to assign docker demon from your local pc via docker volume in /var/run/docker.sock

```bash
docker run -d -it -v /var/run/docker.sock:/var/run/docker.sock --name r-slave jenkins-slave
```

- Very Important ( for our dockerfile we use Ubuntu image which its openssh service is not automatically opened) , you have to log in as root user and start the service
    
    ![Screenshot from 2022-07-26 11-50-23.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/f3d5e612-a220-496d-9071-ca9ff5263f10/Screenshot_from_2022-07-26_11-50-23.png)
    

### To create New node inside Jenkins

- we first go to ****Manage nodes and clouds → New Node****

![Screenshot from 2022-07-26 12-27-45.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/e1e8d5d0-2c2d-4b94-951a-1e41ff7f64bc/Screenshot_from_2022-07-26_12-27-45.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/74d15337-e34d-44e0-bf62-68253398d1e2/Untitled.png)

- set remote root directory / set labels
- **Usage:** only build

![Screenshot from 2022-07-26 11-47-08.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/3f17ec4a-28b6-4bda-8f88-a1f190a9bc94/Screenshot_from_2022-07-26_11-47-08.png)

- get container IP Address

```bash
docker inspect <container ID>
```

![Screenshot from 2022-07-26 11-51-41.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/ecd189ec-140d-432e-9b4c-24a3f7b4ae17/Screenshot_from_2022-07-26_11-51-41.png)

- set your IP host

![Screenshot from 2022-07-26 12-32-28.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/84f85e2b-d73e-44bf-ab10-082b444b1f30/Screenshot_from_2022-07-26_12-32-28.png)

- set credentials:
    - use as SSH
    - id: set the id that you will use in pipeline in the future
    - [Very important]username: the user inside the container “we created a jenkins user”
    - Private Key: copy and paste your created private key that you generated before
    - save and save your node

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/dda5faf1-97a1-4f31-98d1-1cce86603a5d/Untitled.png)

![Screenshot from 2022-07-26 11-55-58.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/0f5a2f1c-1fd1-4e99-8c65-439ec8f87158/Screenshot_from_2022-07-26_11-55-58.png)

## RESULTS

- get inside the node logs
    
    ![Screenshot from 2022-07-26 12-38-28.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/9c670650-69ff-4061-a08b-23b25741bafa/Screenshot_from_2022-07-26_12-38-28.png)
    

## Q5: Integrate slack with Jenkins and send slack message when stage in your pipeline is successful

![Screenshot from 2022-07-26 12-46-20.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/bdb0ce92-b3e9-44c1-b166-d8ba7b284fec/Screenshot_from_2022-07-26_12-46-20.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/fcf0cbb9-b658-410e-9af4-cbb6bd7c5f08/Untitled.png)

 

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/86fdd0aa-db7b-426a-802c-884278e98baf/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/3281e954-b201-4b9d-ac70-1bcfcf242d31/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/d645bbca-f984-40fa-a51e-b31d4a705191/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/128d514c-29e9-43df-bb1b-c388c972c140/Untitled.png)

- 
- I used the previous pipeline to test slack integration using post plugin

```bash
pipeline {
    agent any

    stages {
        
        stage('preparation') {
            steps {
                // Get some code from GitHub repository
                git 'https://github.com/mahmoud254/jenkins_nodejs_example.git'
            }
        }
            
        stage('ci') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) {
                    sh """ 
                    docker build . -f dockerfile -t rizk95/jenkins
                    docker login -u ${USERNAME} -p ${PASSWORD}
                    docker push rizk95/jenkins
                    """
                }
            }
        }

        stage('cd') {
            steps {
                sh "docker run -d -p 3000:3000 rizk95/jenkins"
            }
        }
    }
    # for slack notification
     post {
            success {
                slackSend (message:"Build deployed successfully - ${env.JOB_NAME} ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)")
            }
            failure {
                slackSend (message:"Build failed  - ${env.JOB_NAME} ${env.BUILD_NUMBER} (<${env.BUILD_URL}|Open>)")
            }
        }
}
```

### RESULTS

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/9d5d2be9-8a7f-4c11-bd73-09a916ac3d38/Untitled.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/0f05c6f3-d26c-4c65-8aea-a5ea722cc3ea/Untitled.png)

## Q5: install audit logs plugin and test it

***Manage plugins → audit log → install → restart the docker container***

## RESULTS

![Screenshot from 2022-07-26 13-26-07.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/546a071a-6aab-491d-baad-094743d7b52c/Screenshot_from_2022-07-26_13-26-07.png)

![Screenshot from 2022-07-26 13-30-49.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/a12f69dd-67ca-4cee-b69d-2b14c86c3bc4/Screenshot_from_2022-07-26_13-30-49.png)

![Screenshot from 2022-07-26 13-31-03.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/e56aabc3-9628-43d4-ad2d-9a0317893c05/Screenshot_from_2022-07-26_13-31-03.png)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/2f07bbfa-2f5a-4410-898b-2ee78f1d8273/Untitled.png)
