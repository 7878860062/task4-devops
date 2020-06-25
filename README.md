# Task 4


Task Overview:

Create A dynamic Jenkins cluster and perform this task.

Steps to proceed as:

1. Create a container image that has Linux and other basic configuration required to run Slave for Jenkins. ( example here we require kubectl to be configured )

2. When we launch the job it should automatically start a job on slave based on the label provided for a dynamic approach.

3. Create a job chain of job1 & job2 using the build pipeline plugin in Jenkins

4. Job1: Pull the Github repo automatically when some developers push the repo to Github and perform the following operations as:

1. Create the new image dynamically for the application and copy the application code into that corresponding docker image

2. Push that image to the docker hub (Public repository)

( Github code contain the application code and Dockerfile to create a new image )

5. Job2 ( Should be run on the dynamic slave of Jenkins configured with Kubernetes kubectl command): Launch the application on the top of Kubernetes cluster performing following operations:

1. If launching the first time then create a deployment of the pod using the image created in the previous job. Else if deployment already exists then do a rollout of the existing pod making zero downtime for the user.

2. If Application created the first time, then Expose the application. Else don’t expose it.

Let's Start:

For performing this task we require 3 VM

1 - Where Jenkins is running

2 - Where the docker engine is running

3 - Where k8s(Kubernetes) running

Abstract Idea:

We start Jenkins service from VM1 and then here we configure a cloud service that is linked with VM2 where our docker engine is running. We create an image for dynamic Kubernetes in VM2 which will help us to run our Kubernetes cluster on VM3.

Before proceeding to do this step in your VM2:

Open /usr/lib/systemd/system/docker.service file and add -H tcp://0.0.0.0:3456
(This command will help you to access the docker service of this VM from another machine[this process is called socket binding]).
![0](https://user-images.githubusercontent.com/64473684/85508003-dd7e5f00-b610-11ea-9ba5-c759a475d1b6.png)



and then reload and restart your docker services

```javascript
systemctl daemon-reload
systemctl restart docker
```

Now go to your VM1 and start your Jenkins service and install the "docker" plugin

```javascript
systemctl start jenkins 
```

![13](https://user-images.githubusercontent.com/64473684/85508296-67c6c300-b611-11ea-8228-b4c155e1fd07.png)


and then configure your cloud as I showed below:

```javascript
Jenkins->Manage Jenkins -> Manage nodes and clouds-> configure cloud-> add a new cloud-> docker
```

![2](https://user-images.githubusercontent.com/64473684/85508622-10752280-b612-11ea-9097-9f08b9479820.PNG)

In the Yellow section type your VM2 IP where your docker engine is running...

And rest the column fill as I filled.

POST-COMMIT file for triggering:
```javascript
#!/bin/bash

echo "Auto Push Enabled"

git push


curl --user "admin:redhat" http://192.168.43.140:8080//job/Job1_image_build/build?token=devopss

```
GitHub Repo:

JOB1:
![21](https://user-images.githubusercontent.com/64473684/85511707-a612b100-b616-11ea-8968-dbd445b0af67.PNG)
![16](https://user-images.githubusercontent.com/64473684/85510380-deb18b00-b614-11ea-9809-14df8e4c1520.PNG)
![15](https://user-images.githubusercontent.com/64473684/85510415-e96c2000-b614-11ea-9ef5-98985175f670.PNG)
![17](https://user-images.githubusercontent.com/64473684/85510429-effa9780-b614-11ea-81c8-4769f0e8925d.PNG)

Note:

1.In the Cloud column type, the name of the cloud is configured earlier.
2. In Registry Credential column give your docker hub credential.
3. Your image name always starts with your username.
JOB2(Prerequisites):

Before going to JOB2 we again need to perform some steps...

Create a cloud... Create a Kubernetes cluster in VM2 which will help you to contact minikube.

Creating a Kubernetes Image:

Dockerfile:

```javascript
FROM centos
RUN yum install java-11-openjdk.x86_64 -y
RUN yum install openssh-server -y
RUN yum install sudo -y
RUN echo "root:redhat" | chpasswd
RUN curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl

RUN chmod +x ./kubectl
RUN mv ./kubectl /usr/local/bin/kubectl
RUN mkdir /root/.kube
COPY client.key /root/.kube
COPY client.crt /root/.kube
COPY ca.crt /root/.kube
COPY config /root/.kube
RUN mkdir /root/devops
COPY deploy.yml /root/devops
RUN ssh-keygen -A
CMD ["/usr/sbin/sshd", "-D"]

```
For Running this file create a new directory<any_name>

Inside this directory add your client.key, client.crt, ca.crt file

I will get this file from  **"C:\Users\<account_name>\.minikube" and "C:\Users\<Account_name>\.minikube\profiles\minikube" **

and then create a config file in this directory( Code attached below):

```javascript
apiVersion: v1
kind: Config

clusters:
- cluster:
    server: https:// 192.168.43.14:8443     #Add Your Minikube IP here
    certificate-authority: ca.crt
  name: chatpc

contexts:
- context:
    cluster: chatpc
    user: shrikant

users:
- name: shrikant
  user:
    client-key: client.key
    client-certificate: client.crt

```
create a deploy.yml file(Same directory)

```javascript
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dev-deploy
spec:
  replicas: 2
  selector:
    matchLabels:
        env: prod

  template:
    metadata:
     name: dev-pod
     labels:
       env: prod
    spec:
     containers:
      - name: dev-con
        image: chatpc99/webui
        ```
        Now lastly, create a Kubernetes cluster image using dockerfile
        
        ```javascript
        docker build -t kube:latest .
```
Now go and configure cloud for your JOB2
![61](https://user-images.githubusercontent.com/64473684/85514618-db6cce00-b619-11ea-971b-5f90434c525d.PNG)
![62](https://user-images.githubusercontent.com/64473684/85514630-de67be80-b619-11ea-8416-965ef7e6fd78.PNG)
JOB2:
![63](https://user-images.githubusercontent.com/64473684/85514645-e162af00-b619-11ea-8c69-a25789323e3a.PNG)
![65](https://user-images.githubusercontent.com/64473684/85514654-e3c50900-b619-11ea-86ec-69d7e28e2991.PNG)

That's It...

Now come to the testing phase.

Just go and commit from directory everything will be automated just from one commit...

You will access your website from a browser using minikube_ip:<port>

I print the whole Kubernetes details so you can see the port from there...

and for checking minikube IP use minikube ip command inside window cmd prompt

OUTPUTS:


![103](https://user-images.githubusercontent.com/64473684/85520732-ee36d100-b620-11ea-8ab8-f196568880f3.PNG)

![104](https://user-images.githubusercontent.com/64473684/85525449-ba5eaa00-b626-11ea-8e17-feacc8f9bc89.png)


![105](https://user-images.githubusercontent.com/64473684/85524398-c39b4700-b625-11ea-8cdb-b73d43d0d2a2.png)

![201](https://user-images.githubusercontent.com/64473684/85524418-c7c76480-b625-11ea-967e-c6aedfc229a5.PNG)

![0 (6)](https://user-images.githubusercontent.com/64473684/85524518-e0377f00-b625-11ea-8105-c5ac07e6470d.png)



![85372237-2b329300-b54f-11ea-9fc9-d6b6bdd34dda](https://user-images.githubusercontent.com/64473684/85526411-9bace300-b627-11ea-8a0f-895c81ef4dea.png)







