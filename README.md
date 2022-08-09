# Automating Docker container deployment with Jenkins
---
## Part I: Installing Java8 :coffee: and Jenkins :woman_cook:

> Full installation instructions [here](https://www.macminivault.com/installing-jenkins-on-macos/)

1. Install Jenkins on MacOS using Homebrew package manager :beers: 
> If homebrew already installed (run in Terminal `brew -v` to check), skip to the next step 
```ba
/usr/bin/ruby -e /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

2. Check if Homebrew requires any recommendations:
```bash
brew doctor
```

3. Before installing Jenkins, we need to install Java8
```bash
brew cask install java8
```
**Troubleshooting error**:
```bash
Cask 'java8' is unavailable: No Cask with this name exists.
java --version
java 14.0.1 2020-04-14
Java(TM) SE Runtime Environment (build 14.0.1+7)
Java HotSpot(TM) 64-Bit Server VM (build 14.0.1+7, mixed mode, sharing)
``` 

Jenkins requires Java8. Click [here](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html) to install.

Check all installed java version
```bash
/usr/libexec/java_home -V
```
**which should return**
```bash
Matching Java Virtual Machines (2):
    14.0.1, x86_64:	"Java SE 14.0.1"	/Library/Java/JavaVirtualMachines/jdk-14.0.1.jdk/Contents/Home
    1.8.0_251, x86_64:	"Java SE 8"	/Library/Java/JavaVirtualMachines/jdk1.8.0_251.jdk/Contents/Home
```

Pick `1.8.0_251, x86_64` to set as the default then:
```bash
export JAVA_HOME=`/usr/libexec/java_home -v 1.8.0_251, x86_64`
```

When you run java -version you will see:
```bash
java version "1.8.0_251"
```
Install and configure kubectl in jenkins to perform deployment in eks from jenkins.

4. To make the Jenkins web interface accessible from anywhere, not just local machine, open up the config file:
```bash
sudo nano /usr/local/opt/jenkins-lts/homebrew.mxcl.jenkins-lts.plist
```

5. Find this line:
```bash
<string>--httpListenAddress=127.0.0.1</string>
```

6. Change it to:
```bash
<string>--httpListenAddress=0.0.0.0</string>
```
To exit out of nano, press `Ctrl+X`, hit `Y` to save Changes, hit `Enter`

7. Start or restart Jenkins
```bash
brew services start jenkins-lts
brew services restart jenkins-lts
```

8. Open the browser and type in the following:
```bash
http://localhost:8080/
```

9. To check whether jenkins is running:
```bash
brew services list
```

---
## Part II: Configuring Jenkins:
1. Navigate to Manage Jenkins > Manage Plugins > Available > Install **all** Docker plugins, `docker-build-step`, `Docker Compose Build Step`, `Docker build plugins` and `Github`

**List of plugins used:**
```bash
Blue Ocean
Credentials Plugin
Docker Plugin
CloudBees Docker Hub/Registry Notification
CloudBees Docker Build and Publish
Email Extension
Github Plugin
NodeJS Plugin
Oracle Java SE Development Kit Installer Plugin
Pipeline Plugin
Timestamper
```

**Tip:** Install `Blue Ocean` plugin for better UI 

---

## Part III: Setting up CI build on Jenkins :woman_cook:
1. Create a freestyle job on Jenkins and call it `Docker_Pipeline_Integration_Test` with the following configurations:
    - **General** -> `Discard Old Builds` -> `Max # of builds to keep` -> 3
    - **Github Project URL** -> Insert URL for Github Repository
    - **Source Code Management** -> `Git` ->  Insert `Repository URL` and `Credentials` (To learn how to add credentials click [here](https://sharadchhetri.com/how-to-setup-jenkins-credentials-for-git-repo-access/)) -> `Branches to build` -> `Branch Specifier` -> `*/dev*`
    - **Build Triggers** -> `GitHub hook trigger for GITScm polling`
    - **Build Environment** -> `Add Timestamps` -> `Provide Node & npm bin/folder to PATH` -> Choose default NodeJs Installation (go to step 3 if NodeJS plugin requires activation) -> npmrc file ` - use system default - ` - Cache location `Default`
    - **Build** -> `Execute Shell` -> Go to Step 5 
    - **Post-build Actions** -> `Push Only if Build Succeeds` -> `Merge Results` -> `Branches` -> Branch to push: `master` -> Target remote name: `origin`
    -  Click `Apply` and `Save`

> **Note**: Webhooks only work with a public IP. You will need to forward your local port [http://localhost:8080/](http://localhost:8080/) to the Internet/public using an SSH server like [Serveo](https://medium.com/automationmaster/how-to-forward-my-local-port-to-public-using-serveo-4979f352a3bf), Ngrok or [SocketXP](https://www.socketxp.com/download). 
---
> To set up Github Webhooks, Jenkins, and Ngrok for Local Development click [here](https://medium.com/@developerwakeling/setting-up-github-webhooks-jenkins-and-ngrok-for-local-development-f4b2c1ab5b6)
---
**Commands to forward local port to public IP with** `SocketXP`:
```bash
sudo su
sudo curl -O https://portal.socketxp.com/download/darwin/socketxp && chmod 777 socketxp && sudo mv socketxp /usr/local/bin
socketxp login "authentication_token_goes_here"
socketxp connect http://localhost:8080
Connected.
Public URL -> https://anwarmah-z012h3op.socketxp.com
```
---
2. On Github, navigate to your [repository](https://github.com/anwarmah/Docker_Jenkins_Pipeline/tree/development) -> Go to `Settings` -> `Webhooks` -> in Payload URL, enter Jenkins URL e.g:
```bash
http://anwarmah-z012h3op.socketxp.com/github-webhook/
```
-> Enable `SSL verification` -> `Update webhook` -> `Redeliver`

3. Go back to Jenkins and make sure `nodejs` plugin is installed
4. To activate `nodejs` plugin, go to `Manage Jenkins` > `System Configuration` > `Global Tool Configuration` > `NodeJS` > `Add NodeJS` > Give it a name e.g. `Node` > Save and Apply
5. Execute shell
```bash
npm install 
npm test
```
6. Once saved, make changes on your IDE on a new branch and push to Github -> Jenkins will listen to incoming `POST` requests to the Payload URL used on Github and automatically merge changes from the new branch to the master branch if the tests pass. 
7. Go to console output to check if the build was successful (indicated by the blue circle :large_blue_circle:)

---
## Part IV: Setting up CD build on Jenkins with Dockerfile 	:whale:
Once our CI build is successful, create another build on Jenkins which will listen to the CI build which we named `Docker_Pipeline_Integration_Test` and automatically build a Docker image if it successfully passed the tests and merges the code to the master branch. 
1. Make sure `docker pipeline plugin` is installed
2. Create a [Dockerhub](https://hub.docker.com/) account
3. Once logged in, click on `Create` -> `Create Repository` -> Type in a name for your Docker repository e.g anwarmah/Docker_Jenkins_Pipeline
4. After the Docker repository has been created, go back to Jenkins and navigate to `Credentials` -> `System` -> `Global Credentials` -> `Add Credentials`
5. Go back to Jenkins home page and click `New Item`, select `Pipeline` and name it `docker-deployment-test-v1` and provide it with the following configurations:
    - **General** -> Github project -> Insert Project URL
    - **Build Triggers** -> Select `Build after other projects are built` -> Projects to Watch (select the CI build created in [Part III](#part-iii-setting-up-ci-build-on-jenkins)): `Docker_Pipeline_Integration_Test` -> Trigger only if build is stable
    - **Pipeline** -> Add the following script (scripts are based on the Groovy programming language):
```bash
pipeline {
    agent any
    environment {
    dockerImage = ''
    PATH = "$PATH:/usr/local/bin"
}

    agent {
        'docker'}
    stages {
            stage('Cloning our Git') {
                steps {
                git 'https://github.com/anwarmah/Docker_Jenkins_Pipeline.git'
                }
            }

            stage('Building Docker Image') {
                steps {
                    script {
                        dockerImage = docker.build("backendapi")
                    }
                }
            }

            stage('Deploying Docker Image to ECR') {
                steps {
                    script {
                        docker.withRegistry('https://720766170633.dkr.ecr.us-east-2.amazonaws.com', 'ecr:us-east-2:aws-credentials') {
                        dockerImage.push("${env.BUILD_NUMBER}")
                        dockerImage.push("latest")
                        }
                    }
                }
            }

            stage('Deploy') {
                steps{
                  sh "kubectl apply -f deployment.yml"
                }
            }
        }
    }
```