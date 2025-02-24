# Jenkins
# I. What is Jenkins?
**Jenkins** is the leading open soruce **automation server**. It provides hundreds of plugins to support building, deploying and **automating any project**. 
# II. How to install Jenkins. 
You can follow this instruction: [How to install Jenkins](https://www.digitalocean.com/community/tutorials/how-to-install-jenkins-on-ubuntu-20-04)
But instead of install java 11, you need to install java 17. Follow this: [How to install java 17](https://www.rosehosting.com/blog/how-to-install-java-17-lts-on-ubuntu-20-04/)

Next step, you need to modify `/etc/hosts`, add a new with IP and domain: 

![image](https://github.com/user-attachments/assets/96bd262f-5457-479a-b26e-b8fcf15f2da6)

On your windows, add it to your file `hosts` in `C:\Windows\System32\drivers\etc` too. 

Allow firewall in port 8080 with 

```bash
ufw allow 8080
```

Now you can access into your Jenkins's domain in port 8080. The password is in `/var/lib/jenkins/secrets/initialAdminPassword`: 

![image](https://github.com/user-attachments/assets/232e5e0f-6470-4e18-8ad9-767e8d01892c)

Install default plugins after. 

To run without adding default port 8080 after, you need to install Nginx and then edit file 
`/etc/nginx/conf.d/jenkins.fucalors.tech.conf` 
with content: 

```conf
server {
  listen 80;
  server_name jenkins.fucalors.tech;
  location / {
    proxy_pass http://jenkins.fucalors.tech:8080;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection keep-alive;
    proxy_set_header Host $host;
    proxy_cache_bypass $http_upgrade;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
```

restart nginx and you can remove port 8080. 

# III. Implement Jenkins CI/CDeployment
Like Gitlab, we need a independent user for Jenkins: 

```bash
adduser jenkins
```

On our server used to deploy project (mine is **"devops"**), install Java 17 as well. 

![image](https://github.com/user-attachments/assets/65959c10-c093-418a-bb21-db3d86454d42)

Next step, a new node is required on Jenkins for running our pipeline. This node is connected to our project server(`devops`). Click in **Dashboard --> Manage Jenkisn -> Nodes --> New Node**. Your node's name should be the same as your server. 

> A node is a machine (physical or virtual) that is part of the Jenkins environment. It can be **controller(master)** or **agent(slave)**

![image](https://github.com/user-attachments/assets/b9d0f318-c142-4266-93c0-c68ddbf5b682)

There are some categories here: 

![image](https://github.com/user-attachments/assets/68a8070d-069f-4087-9cf0-b29dfe12a6f1)

![image](https://github.com/user-attachments/assets/fc52220b-4840-4549-903e-5acd1e8e28f2)

**Number of executors**: How many build jobs that node can run concurrently

**Remote root directory**: This will be in `/var/lib/jenkins` (you need to create one in your project server). On jenkins server, this will be like this: 

![image](https://github.com/user-attachments/assets/1d9141e3-6006-4696-8b89-2370249b56fa)

Remember to grant permission for `jenkins` user using 

```bash!
chown jenkins. /var/lib/jenkins
```

Next, a port for TCP for agent inbound need to add in. Go to **Manage Jenkins --> Security --> Agents**: 

![image](https://github.com/user-attachments/assets/01a5ab59-93c7-42a0-95ad-38e0216944b8)

This port will use to communicate between agent and jenkins server. You should choose an unused port on jenkins server. I choose port 8999, checking `netstat`:

![image](https://github.com/user-attachments/assets/5ebf14d7-860d-4bea-aec9-659d57a7adba)

A TCP port is open on port 8999.

Come back and create a node. And an instruction to guide to run an agent appears. 

![image](https://github.com/user-attachments/assets/b5e75026-83e0-4eff-b0d6-a507c6a21a68)

I will use **Run from agent command line, with the secret stored in a file: (Unix)**

```bash!
echo d750ca7240f9fde097652c6e47e3ec008ee50ba094d7de9b05a222e88a9fc182 > secret-file
curl -sO http://jenkins.fucalors.tech:8080/jnlpJars/agent.jar
java -jar agent.jar -url http://jenkins.fucalors.tech:8080/ -secret @secret-file -name devops -webSocket -workDir "/var/lib/jenkins"
```

Switch our directory to `/var/lib/jenkins`, `su` to `jenkins` user and run those command. 

![image](https://github.com/user-attachments/assets/6362b065-ff19-4d33-96f8-386006f3759f)

Now our agent should be working now

![image](https://github.com/user-attachments/assets/3a888489-5dc8-494d-99b9-537c6c762c0d)


> There is a note here. To automatically run your agent, convert it into a service so it will start after you turn on server. 
> Create and configure a file `/etc/systemd/system/jenkins-agent.service` with: 

```!
[Unit]
Description=Jenkins Agent Service
After=network.target

[Service]
Type=simple
WorkingDirectory=/var/lib/jenkins
ExecStart=/bin/bash -c 'java -jar agent.jar -url http://jenkins.fucalors.tech:8080/ -secret @secret-file -name devops -webSocket -workDir "/var/lib/jenkins"'
User=jenkins
Restart=always

[Install]
WantedBy=multi-user.target
```

> Reload daemon and start jenkins agent: 

```bash
systemctl daemon-reload
systemctl start jenkins-agent
```

Go to **Dashboard --> New Item --> Folder**. Create a folder named **Action_in_lab** to test our `shoeshop` project pipeline. 

To connect to Gitlab, go to **Dashboard --> Manage Jenkins --> Plugins --> Available plugins**. Find for `gitlab` and `blue ocean`. **Blue Ocean** plugin will give use better appearance when monitoring process. 

![image](https://github.com/user-attachments/assets/da78041c-f9d8-4288-ad82-061298b1c6d5)

![image](https://github.com/user-attachments/assets/2e468a09-84ad-4711-90c4-51d175d7d9ec)

After downloading plugins, we must configure to completely connect to Gitlab. Go to **Dashboard --> Manage Jenkins --> System**. Scroll down until you found `Gitlab` header. 

![image](https://github.com/user-attachments/assets/e085722e-deec-4327-ad1c-53c453fbc29b)

Enter required information about your Gitlab. About credentials:
- Go to Gitlab, create a new user `jenkins` with access level `Admin`
- Log in to user `jenkins`, **Edit profiles --> Access tokens**

![image](https://github.com/user-attachments/assets/3b706348-6127-432f-9d6d-75cea03e7c2f)

Create a name and tick into `api` box then create one api token. Then Gitlab will give you a token. Save it to somewhere because it will appear when you leave this page. 

Back to Jenkins, choose **Add credentials** and choose to add your Gitlab API Token

![image](https://github.com/user-attachments/assets/09e1753e-cfbd-4bf3-aefb-a9ed49426b31)

Now your Jenkins is connected to Gitlab. 
To run a pipeline, create a pipeline `shoeshop` in folder **Action_in_lab** we created before. 
In **Configure** of our pipeline, there are some fields that we need to tick: 

![image](https://github.com/user-attachments/assets/228a83ef-319b-4259-b7b4-265edaf54b7e)

![image](https://github.com/user-attachments/assets/d250aa73-6898-47bf-9d66-8b6d6e9e3952)

- **Discard old build**: Choose it and keep how many builds you want to keep 

![image](https://github.com/user-attachments/assets/b282318e-395e-4f4d-b573-bef14f215f85)

- **Build when a change is pushed to GitLab. GitLab webhook URL: (URL)**

![image](https://github.com/user-attachments/assets/f807c3e8-790f-4b08-9b66-7763337ce5c1)

- **Pipeline**:

![image](https://github.com/user-attachments/assets/f7c65fcb-2e02-4e5d-b216-eafa369dee3e)

This one you need to choose the Git in `SCM`, for the Git repo, use yours, for the credentials, use user `jenkins` we created before. 

**Branch to build**: 

![image](https://github.com/user-attachments/assets/7c946f03-58a8-42c3-a178-7e147ee45b3a)

I will build on my branch `develop`. You can add more branches too. Save and you can see your pipeline is working

![image](https://github.com/user-attachments/assets/c7dba3da-af2d-4ee6-9811-0266237120f7)

But we're not done yet. To automatically notify an external service (Jenkins, Slack, ...) you need to configure Webhook on Gitlab.

> A GitLab Webhook is a mechanism that allows GitLab to automatically notify an external service (like Jenkins, Slack, or a deployment server) whenever a specific event occurs in a repository.

Before that, go to **Settings** of the Admin, choose to **Network --> Outbound Request**. Tick to the box **Allow requests to the local network from web hooks and services**:

![image](https://github.com/user-attachments/assets/9c05f332-aaf5-4bff-b2af-eb73b0a918a8)

Choose your project on Gitlab, go to **Settings --> Webhook**

![image](https://github.com/user-attachments/assets/6c16f466-adb2-4809-8e82-7017b64c8c5b)

Our format for URL will be: 

```!
http://<user on jenkins>:<token of that user on jenkins>@<jenkins's address>/project/<path to project on jenkins> 
```

About token, click in your profile picture on Jenkins, choose **Security** or **Configure** if your system is the older version.

![image](https://github.com/user-attachments/assets/def9abe1-24ca-4a40-b981-c3c3c12dd0c1)

Create a token: 

![image](https://github.com/user-attachments/assets/2ef4b178-e7f6-4853-98f3-d58e60add111)

My final format of URL is: 

```!
http://admin:<token>@jenkins.fucalors.tech/project/Action_in_lab/shoeshop
```

> Remember to add host on gitlab server

![image](https://github.com/user-attachments/assets/3d165603-a871-4a8b-8846-b020cca9c2c6)

Choose which events will trigger the pipeline

![image](https://github.com/user-attachments/assets/c90e3d12-21ed-4de8-8e6b-8a0c06f9663c)

And disable SSL verification

![image](https://github.com/user-attachments/assets/02c1a6ee-1761-4602-879c-2cfa14a2a42f)

Now you will get your own webhook:

![image](https://github.com/user-attachments/assets/3dc298f8-58d8-4068-ad2f-3e8a8f28ea74)

To successfully run your pipeline, we need to create a Jenkinfile in branch we want to run pipeline. I will create one in branch `develop`: 

```groovy!
pipeline {
    agent {
        label 'devops'
    }
    environment {
        appUser = "shoeshop"
        appName = "shoe-ShoppingCart"
        appVersion = "0.0.1-SNAPSHOT"
        appType = "jar"
        processName = "${appName}-${appVersion}.${appType}"
        folderDeploy = "/datas/${appUser}"
        buildScript = "mvn clean install -DskipTests=true"
        copyScript = "sudo cp target/${processName} ${folderDeploy}"
        permisScript = "sudo chown -R ${appUser}. ${folderDeploy}"
        killScript = "sudo kill -9 \$(ps -ef | grep ${processName} | grep -v grep | awk '{print \$2}')"
        runScript = 'sudo su ${appUser} -c "cd ${folderDeploy}; java -jar ${processName} > nohup.out 2>&1 &"'
    }
    stages {
        stage('build') {
            steps {
                sh(script: """ ${buildScript} """, label: "build with maven")
            }
        }
        stage('deploy') {
            steps {
                sh(script: """ ${copyScript} """, label: "copy the .jar file into deploy folder")
                sh(script: """ ${permisScript} """, label: "set permission folder")
                sh(script: """ ${killScript} """, label: "terminate the running process")
                sh(script: """ ${runScript} """, label: "run the project")
            }
        }
    }
}

```

Those are nearly the same as the `.gitlab-ci.yml` we created before but different in syntax. Commit and see if our pipeline works or not: 

![image](https://github.com/user-attachments/assets/1b38a43a-32c2-415b-b891-bfb33f0c0cff)

Go to Blue Ocean for better appearance: 

![image](https://github.com/user-attachments/assets/af48522a-9e0a-47ac-9dde-a9e7867bc61d)

Here you can what happened in each stage. 

Check result: 

![image](https://github.com/user-attachments/assets/e7693844-f322-49db-b1f2-2dd864ec1674)

# IV. Implement Jenkins CI/CDelivery

Now, we modify a little bit to manual deploy: 

```groovy!
pipeline {
    agent {
        label 'lab-server'
    }
    environment {
        appUser = "shoeshop"
        appName = "shoe-ShoppingCart"
        appVersion = "0.0.1-SNAPSHOT"
        appType = "jar"
        processName = "${appName}-${appVersion}.${appType}"
        folderDeploy = "/datas/${appUser}"
        buildScript = "mvn clean install -DskipTests=true"
        copyScript = "sudo cp target/${processName} ${folderDeploy}"
        permsScript = "sudo chown -R ${appUser}. ${folderDeploy}"
        killScript = "sudo kill -9 \$(ps -ef| grep ${processName}| grep -v grep| awk '{print \$2}')"
        runScript = 'sudo su ${appUser} -c "cd ${folderDeploy}; java -jar ${processName} > nohup.out 2>&1 &"'
    }
    stages {
        stage('build') {
            steps {
                sh(script: """ ${buildScript} """, label: "build with maven")
            }
        }
        stage('deploy') {
            steps {
                script {
                    try {
                        timeout(time: 5, unit: 'MINUTES') {
                            env.useChoice = input message: "Can it be deployed?",
                                parameters: [choice(name: 'deploy', choices: 'no\nyes', description: 'Choose "yes" if you want to deploy!')]
                        }
                        if (env.useChoice == 'yes') {
                            sh(script: """ ${copyScript} """, label: "copy the .jar file into deploy folder")
                            sh(script: """ ${permsScript} """, label: "set permission folder")
                            sh(script: """ ${killScript} """, label: "terminate the running process")
                            sh(script: """ ${runScript} """, label: "run the project")
                        }
                        else {
                            echo "Do not confirm the deployment!"
                        }
                    } catch (Exception err) {

                    }
                }

            }
        }
    }
}

```

We use `try-catch` and `env.useChoice` to get the confirmation of user. Other steps are the same. 

# V. Jenkins CI/CD advanced for production deployment using Jenkins parameters


> A Jenkins parameter allows users to dynamically provide input values when triggering a build. This makes Jenkins jobs more flexible and customizable.

This part, you need to have knowledge about Groovy. Before that, install `Active Choice` plugin: 

![image](https://github.com/user-attachments/assets/61db9ec1-9538-4f5a-9fde-d4d750861441)

This will be more flexible for your project. 
Configure our pipeline with this option (before installing plugin):

![image](https://github.com/user-attachments/assets/ce1f17eb-687f-4ab0-b4c5-6d2128a0fb4c)

There are some option here for us to choose
- **Boolean parameter**: Checkbox (true/false).
- **Choice Parameter**: Dropdown menu with predefined values.
- **File Parameter**: Allows users to upload a file during build.
- **String Parameter**: Accepts a text string as input.
- **Password Parameter**: A hidden input field for sensitive data.
- **Run Parameter**: Select a previous build from another job.

After installing plugin, it would be like this 

![image](https://github.com/user-attachments/assets/72246125-30df-47a5-a21a-3400b5a61f05)

Choose **Active Choices Parameter** and modify like this 

![image](https://github.com/user-attachments/assets/f8c458e0-9b2b-40d9-b8e0-ba0ab1a216be)

Save and you will see this in your pipeline

![image](https://github.com/user-attachments/assets/8b763dd6-dea4-4a63-b9e8-0a402937ecf0)

Add some parameters to prepare: 

![image](https://github.com/user-attachments/assets/25c649b2-c15c-4e4c-8594-e83d60ab927c)

![image](https://github.com/user-attachments/assets/7d1a3607-cb67-4d44-87bf-d000df4f1ae0)

![image](https://github.com/user-attachments/assets/1a2f21e2-ece8-452b-bb5d-b6996b6fb4cf)

Script:
```groovy!
import jenkins.model.*
import hudson.FilePath
backupPath = '/datas/shoeshop/backups/'
def node = Jenkins.getInstance().getNode(server)
def remoteDir = new FilePath(node.getChannel(), "${backupPath}")

def files = remoteDir.list()
def nameFile = files.collect{it.name}

if (action == "rollback"){
  return nameFile
}
```

![image](https://github.com/user-attachments/assets/8d25e5c2-4efc-4edc-9750-acab4b3e6085)

Now for our Pipeline script. I will go step by step.
1. We build a script to run project and kill previous process before running: 

```groovy!
appUser = "shoeshop"
appName = "shoe-ShoppingCart"
appVersion = "0.0.1-SNAPSHOT"
appType = "jar"
processName = "${appName}-${appVersion}.${appType}"
folderDeploy = "/datas/${appUser}"
buildScript = "mvn clean install -DskipTests=true"
copyScript = "sudo cp target/${processName} ${folderDeploy}"
permsScript = "sudo chown -R ${appUser}. ${folderDeploy}"
killScript = "sudo kill -9 \$(ps -ef | grep ${processName} | grep -v grep | awk '{print \$2}')"
runScript = "sudo su ${appUser} -c 'cd ${folderDeploy}; java -jar ${processName} > nohup.out 2>&1 &'"

def getProcessId(processName){
    def processId = sh(returnStdout: true, script: """ ps -ef | grep ${processName} | grep -v grep | awk \'{print \$2}\' """, label: "get process ID")
}

def startProcess(){
    stage('start'){
        sh(script: """ ${runScript}  """, label: "run the project")
        sleep 5
        def processId = getProcessId("${processName}")
        if ("${processId}" == "")
            error("cannot start process")
    }
    echo("${appName} with server " + params.server + " started")
}

node(params.server){
    currentBuild.displayName = params.action
    if (params.action == "start"){
        startProcess()
    }
}
```

We defined 2 functions: `getProcessId` and `startProcess`. Variable `processId` in function `getProcessId` took the stdout of next command, with label for better reading in Blue Ocean: 

![image](https://github.com/user-attachments/assets/5a01b064-9df4-49ea-82d8-e22e8b16321d)

Function `startProcess` is labeled with stage `start`. If there is no previous processName, it will return `cannot start process`.  
If it worked, the pipeline will echo out a message: 

![image](https://github.com/user-attachments/assets/539d8cae-cc57-4d8f-859f-08ac0e59577b)

You can check in Blue Ocean or in the main page of project. 

![image](https://github.com/user-attachments/assets/55ef4308-4538-481d-94e1-67d899f865c6)

2. Stop Process

```groovy!
appUser = "shoeshop"
appName = "shoe-ShoppingCart"
appVersion = "0.0.1-SNAPSHOT"
appType = "jar"
processName = "${appName}-${appVersion}.${appType}"
folderDeploy = "/datas/${appUser}"
buildScript = "mvn clean install -DskipTests=true"
copyScript = "sudo cp target/${processName} ${folderDeploy}"
permsScript = "sudo chown -R ${appUser}. ${folderDeploy}"
killScript = "sudo kill -9 \$(ps -ef | grep ${processName} | grep -v grep | awk '{print \$2}')"
runScript = "sudo su ${appUser} -c 'cd ${folderDeploy}; java -jar ${processName} > nohup.out 2>&1 &'"

def getProcessId(processName){
    def processId = sh(returnStdout: true, script: """ ps -ef | grep ${processName} | grep -v grep | awk \'{print \$2}\' """, label: "get process ID")
}

def startProcess(){
    stage('start'){
        sh(script: """ ${runScript}  """, label: "run the project")
        sleep 5
        def processId = getProcessId("${processName}")
        if ("${processId}" == "")
            error("cannot start process")
    }
    echo("${appName} with server " + params.server + " started")
}

def stopProcess(){
    stage('stop'){
        def processId = getProcessId("${processName}")
        if ("${processId}" != "")
            sh(script: """ sudo kill -9 ${processId} """, label: "kill process" )
        echo("${appName} with server " + params.server + " stop")
    }
}
node(params.server){
    currentBuild.displayName = params.action
    if (params.action == "start"){
        startProcess()
    }
    if (params.action == "stop"){
        stopProcess()
    }
}
```

3. Process 'upcode'

This will add a process that pull our code with commit hash from gitlab: 

```groovy!
appUser = "shoeshop"
appName = "shoe-ShoppingCart"
appVersion = "0.0.1-SNAPSHOT"
appType = "jar"
processName = "${appName}-${appVersion}.${appType}"
folderDeploy = "/datas/${appUser}"
buildScript = "mvn clean install -DskipTests=true"
copyScript = "sudo cp target/${processName} ${folderDeploy}"
permsScript = "sudo chown -R ${appUser}. ${folderDeploy}"
killScript = "sudo kill -9 \$(ps -ef | grep ${processName} | grep -v grep | awk '{print \$2}')"
runScript = "sudo su ${appUser} -c 'cd ${folderDeploy}; java -jar ${processName} > nohup.out 2>&1 &'"
gitLink = "http://git.fucalors.tech/shoeshop/shoeshop.git"

def getProcessId(processName){
    def processId = sh(returnStdout: true, script: """ ps -ef | grep ${processName} | grep -v grep | awk \'{print \$2}\' """, label: "get process ID")
}

def startProcess(){
    stage('start'){
        sh(script: """ ${runScript}  """, label: "run the project")
        sleep 5
        def processId = getProcessId("${processName}")
        if ("${processId}" == "")
            error("cannot start process")
    }
    echo("${appName} with server " + params.server + " started")
}

def stopProcess(){
    stage('stop'){
        def processId = getProcessId("${processName}")
        if ("${processId}" != "")
            sh(script: """ sudo kill -9 ${processId} """, label: "kill process" )
        echo("${appName} with server " + params.server + " stop")
    }
}

def upcodeProcess(){
    // Pull code from gitlab
    stage('checkout'){
        if (params.hash == "")
            error("require hashing for code update")
        checkout([$class: 'GitSCM', branches: [[ name : params.hash]], 
            userRemoteConfigs: [[ credentialsId: 'jenkins-gitlab-user-account', url: gitLink ]]])
    }
    stage('build'){
        sh(script: """ ${buildScript}  """, label: "build with maven")
    }
    stage('config'){
        sh(script: """ ${copyScript}  """, label: "copy .jar file into the deploy folder")
        sh(script: """ ${permsScript}  """, label: "assign project permissions")
    }
}
node(params.server){
    currentBuild.displayName = params.action
    if (params.action == "start"){
        startProcess()
    }
    if (params.action == "stop"){
        stopProcess()
    }
    if (params.action == "upcode"){
        // clone source code --> build --> deploy
        currentBuild.description = "server " + params.server + " with hash " + params.hash
        stopProcess()
        upcodeProcess()
        startProcess()
    }
}
```

`checkout` function takes 3 parameters: where you want to take the code from, which branches, and which credentials for remote configure. 

And also, we add a description for our process with `currentBuild.description`. 
For instance, I use latest hash code commit: 

![image](https://github.com/user-attachments/assets/b7a532bf-2ca3-47ba-9fdb-044ff27c094b)

After pulled from Gitlab, we build it with maven then copy it to our deploy folder, change permission and run project. 

![image](https://github.com/user-attachments/assets/c97c8272-6086-47d1-873f-6a464cca1f83)

4. Rollback

First, we create 2 another folder for run project and backup (remember to create it under `shoeshop` user)

And change our `folderDeploy`, also add 2 more variables

![image](https://github.com/user-attachments/assets/1c93f80e-a5b2-4963-8230-dde117141803)

And we need to approve our code that we created the `rollback_version` parameter before: 

![image](https://github.com/user-attachments/assets/38035224-2b17-4434-9351-67345d7f9b6f)

![image](https://github.com/user-attachments/assets/d2162a46-9c70-4c4c-b190-ea784ea9629d)

(Choose action `rollback` and F5 in page Credentials)

Now we add 2 more function for backup and rollback

```groovy!
def backupProcess(){
    stage('backup'){
        // Format: shoe-ShoppingCart_<backup day>_<time>_<hash code>.zip
        def timeStamp = new Date().format("ddMMyy_HHmm")
        def zipFileName = "${appName}_${timeStamp}" + ".zip"
        //Compress project 
        sh(script: """ sudo su ${appUser} -c "cd ${folderMain}; zip -jr ${folderBackup}/${zipFileName} ${folderDeploy}"  """, label: "backup old version")
    
        
    }
}

def rollbackProcess(){
    stage('rollback'){
        
        sh(script: """ sudo su ${appUser} -c "cd ${folderDeploy};rm -rf *"  """, label: "delete the current version")
        sh(script: """ sudo su ${appUser} -c "cd ${folderBackup};unzip ${params.rollback_version} -d ${folderDeploy}"  """, label: "rollback process")
    
    }
}
```

Modify a little in `node{}` function:

```groovy!
if (params.action == "upcode"){
        // clone source code --> backup --> build --> deploy
        currentBuild.description = "server " + params.server + " with hash " + params.hash
        backupProcess()
        stopProcess()
        upcodeProcess()
        startProcess()
    }
    if (params.action == "rollback"){
        stopProcess()
        rollbackProcess()
        startProcess()
    }
```

So before it upcode, it will backup and then do the rest of works. 

When we choose action rollback, rollback versions will appear: 

![image](https://github.com/user-attachments/assets/1698d590-1b81-44ea-9a27-0b12c6a7bab8)

And run any upcode process to see the result:

![image](https://github.com/user-attachments/assets/5520ae84-1883-455f-b8a6-30955e0672e9)
