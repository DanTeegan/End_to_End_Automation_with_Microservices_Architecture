# Automating A Full Pipeline With Containerisation


## Note that we will be using 16.04 ubuntu AMI's to create our EC2 instances

## Installing Jenkins

```
wget -q -O - https://pkg.jenkins.io/debian/jenkins-ci.org.key | sudo apt-key add -
echo "deb https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list
sudo apt-get update -y
sudo apt-get install jenkins -y
sudo systemctl start jenkins
sudo systemctl status jenkins
```

## Then opened jenkins on our browser using our public ip colon 8080

### Logging in using the password

- Below command will give us the admin password that we must insert into jenkins
```
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
- the username will be admin

- Then installed the recommended plugins


## Creating a jenkins slave node
- We then created a slave node through jenkins and made another EC2 instance which we called slave

- The slave is an environment to runo ur tests on

- We first enter our opt folder and create a folder named jenkins

```
sudo mkdir jenkins
sudo wget <INSERT LINK TO AGENT.JAR FILES>
```

- This downloads the agent.jar file
- We can now run the agent.jar file which will connect us to our master node

- Before we do this we need to create a user called jenkins and switch to it

```
sudo adduser jenkins
sudu su jenkins
```

- Now we run the agent.jar file using the command given on the Jenkins dashboard
- 
```
java -jar agent.jar -jnlpUrl <YOUR IP>/computer/Jenkins-Slave/slave-agent.jnlp -secret 813e64a1fdd2e97f1128ffc6fa4378cb4359464c4a439e4eeb6b4ecda1d99ef3

```

- Now our node is connected to the master effectively



- Now if we run a build we will get the error 'npm not found', therefore we will download npm on our jenkins slave


```
sudo apt-get install python-software-properties
curl -sL https://deb.nodesource.com/setup_10.x | sudo -E bash -
sudo apt-get install nodejs -y
```

- When we install node, it comes with npm
- in order for our server to run the tests we must first have npm
- If we rerun our build it should work succesfully

## Creating a Docker Instance

- On EC2 we now create a Docker instance

- Now we can download docker with the following commands

```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt-get update
apt-cache policy docker-ce
sudo apt-get install -y docker-ce
sudo systemctl status docker
```

## Giving docker access to sudo

- In order for us to run Docker commands within jenkins we need to allow sudo access for Docker, this can
be done running the lines of code below

```
sudo usermod -aG docker ${USER}
newgrp docker
sudo service docker restart
sudo systemctl restart docker
sudo chmod 666 /var/run/docker.sock

```


## Running a Container within our instance

- We will now pull an image from a docker repository to check if we can see the application running on a browser

```
sudo docker run -d -p 3000:3000 ibbocus/jenkins-docker-pipeline
```

- We can then enter our app IP into the browser with the port being 3000

`http://176.34.149.206:3000/`

- And should see the app successfully running


## Creating our Continuous Deployment job

- Now that we have created an instance to hold our containers we can now automate the deployment phase with our CD pipeline job

- We must install docker pipeline plugin

- When creating the job, we want it to be triggered if our CI job is successful

- For this pipeline to be succesfull we must add a credential which allows us to interact with our docker repository
  
1)To do this from the dashboard we click manage jenkins
2 Then manage credentials
1) Click on the jenkins link found under 'stores scoped to jenkins'
2) Click global credentials
3) Then click add credentials on the left hand side
4) We will then add the username, password and id (the string we will uses to reference that credential within the pipeline)


## Creating the Docker Repository

- Before we create the pipeline we want to create a repo that we will send the image
- This can be done on docker hub
- On this instance we will call the repo ''automation-with-docker''


#### Adding the pipeline script

```
pipeline {
  environment {
    registry = "<>"
    registryCredential = 'dockerhub'
    dockerImage = ''
  }
  agent any
  stages {
    stage('Cloning Git') {
      steps {
        git 'https://github.com/aosborne17/Automating-A-Full-Pipeline-With-Containerisation'
      }
    }
    // stage('Build') {
    //   steps {
    //      sh 'npm install'
    //   }
    // }
    // stage('Test') {
    //   steps {
    //     sh 'npm test'
    //   }
    // }
    stage('Building image') {
      steps{
        script {
          dockerImage = docker.build registry + ":$BUILD_NUMBER"
        }
      }
    }
    stage('Deploy Image') {
      steps{
         script {
            docker.withRegistry( '', registryCredential ) {
            dockerImage.push()
          }
        }
      }
    }
    stage('Remove Unused docker image') {
      steps{
        sh "docker rmi $registry:$BUILD_NUMBER"
      }
    }
  }
}

```


### Successful Pipeline build


- Each of the build should pass successfully
  


- We will also be able to see that a push has been made to our Docker Hub



### Overcoming Obstacles




## Continuous Deployment Job

- We must install the 'SSH agent' plugin to be able to ssh into our Virtual Machines
  
  - Now in our builds we click on the SSH agent and add credentials,
  - we then change kind to ''SSH Username with private key'' and click private key
  - We will then enter our gitbash and enter the contents of our DevOpsStudents.pem file, copying everything into the jenkins credentials


### Security Groups Access

- In our security groups we must allow our master jenkins agent to SSH into our Docker App and run the commands

![](/images/Allowing-Jenkins-To-Enter-Docker-App.png)


- We will also trigger this build only if our CD pipeline builds succesfully, this job can then pull the most recently created image from our docker hub and run it from within our EC2 instance

- Our execute shell would look like so:

```
ssh -o "StrictHostKeyChecking=no" ubuntu@176.34.149.206 <<EOF
            docker run -d -p 3000:3000 aosborne17/automation-with-docker:22  
EOF
```

- With these configurations, the build is successful however whenever I check the contaiener status they are ran and then destroyed almost instantly
