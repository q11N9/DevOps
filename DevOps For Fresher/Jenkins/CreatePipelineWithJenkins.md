# Jenkins
# I. What is Jenkins?
**Jenkins** is the leading open soruce **automation server**. It provides hundreds of plugins to support building, deploying and **automating any project**. 
# II. How to install Jenkins. 
You can follow this instruction: [How to install Jenkins](https://www.digitalocean.com/community/tutorials/how-to-install-jenkins-on-ubuntu-20-04)
But instead of install java 11, you need to install java 17. Follow this: [How to install java 17](https://www.rosehosting.com/blog/how-to-install-java-17-lts-on-ubuntu-20-04/)

Next step, you need to modify `/etc/hosts`, add a new with IP and domain: 

![image](https://hackmd.io/_uploads/H1Z6WB3YJg.png)

On your windows, add it to your file `hosts` in `C:\Windows\System32\drivers\etc` too. 

Allow firewall in port 8080 with 

```bash
ufw allow 8080
```

Now you can access into your Jenkins's domain in port 8080. The password is in `/var/lib/jenkins/secrets/initialAdminPassword`: 

![image](https://hackmd.io/_uploads/rkAAzS3F1g.png)

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

![image](https://hackmd.io/_uploads/SyPiFjGqkx.png)

Next step, a new node is required on Jenkins for running our pipeline. This node is connected to our project server(`devops`). Click in **Dashboard --> Manage Jenkisn -> Nodes --> New Node**. Your node's name should be the same as your server. 

:::info
A node is a machine (physical or virtual) that is part of the Jenkins environment. It can be **controller(master)** or **agent(slave)**
:::

![image](https://hackmd.io/_uploads/Bk1E3jM9yl.png)

There are some categories here: 

![image](https://hackmd.io/_uploads/Syrv3iG9kg.png)

![image](https://hackmd.io/_uploads/r1EOnizcyx.png)

**Number of executors**: How many build jobs that node can run concurrently

**Remote root directory**: This will be in `/var/lib/jenkins` (you need to create one in your project server). On jenkins server, this will be like this: 

![image](https://hackmd.io/_uploads/B1ItajMqke.png)

Remember to grant permission for `jenkins` user using 

```bash!
chown jenkins. /var/lib/jenkins
```

Next, a port for TCP for agent inbound need to add in. Go to **Manage Jenkins --> Security --> Agents**: 

![image](https://hackmd.io/_uploads/ryRE0oM91l.png)

This port will use to communicate between agent and jenkins server. You should choose an unused port on jenkins server. I choose port 8999, checking `netstat`:

![image](https://hackmd.io/_uploads/HkroRiz9yg.png)

A TCP port is open on port 8999.

Come back and create a node. And an instruction to guide to run an agent appears. 

![image](https://hackmd.io/_uploads/H1RZknzqkx.png)

I will use **Run from agent command line, with the secret stored in a file: (Unix)**

```bash!
echo d750ca7240f9fde097652c6e47e3ec008ee50ba094d7de9b05a222e88a9fc182 > secret-file
curl -sO http://jenkins.fucalors.tech:8080/jnlpJars/agent.jar
java -jar agent.jar -url http://jenkins.fucalors.tech:8080/ -secret @secret-file -name devops -webSocket -workDir "/var/lib/jenkins"
```

Switch our directory to `/var/lib/jenkins`, `su` to `jenkins` user and run those command. 

![image](https://hackmd.io/_uploads/r1fvghf5Jx.png)

Now our agent should be working now

![image](https://hackmd.io/_uploads/r15OxnG91x.png)

:::info
There is a note here. To automatically run your agent, convert it into a service so it will start after you turn on server. 
Create and configure a file `/etc/systemd/system/jenkins-agent.service` with: 

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

Reload daemon and start jenkins agent: 

```bash
systemctl daemon-reload
systemctl start jenkins-agent
```

:::

Go to **Dashboard --> New Item --> Folder**. Create a folder named **Action_in_lab** to test our `shoeshop` project pipeline. 

To connect to Gitlab, go to **Dashboard --> Manage Jenkins --> Plugins --> Available plugins**. Find for `gitlab` and `blue ocean`. **Blue Ocean** plugin will give use better appearance when monitoring process. 

![image](https://hackmd.io/_uploads/HksaZ3z9yl.png)

![image](https://hackmd.io/_uploads/HyJkfnfc1e.png)

After downloading plugins, we must configure to completely connect to Gitlab. Go to **Dashboard --> Manage Jenkins --> System**. Scroll down until you found `Gitlab` header. 

![image](https://hackmd.io/_uploads/HkSOG2f5kl.png)

Enter required information about your Gitlab. About credentials:
- Go to Gitlab, create a new user `jenkins` with access level `Admin`
- Log in to user `jenkins`, **Edit profiles --> Access tokens**

![image](https://hackmd.io/_uploads/Syx442Mq1x.png)

Create a name and tick into `api` box then create one api token. Then Gitlab will give you a token. Save it to somewhere because it will appear when you leave this page. 

Back to Jenkins, choose **Add credentials** and choose to add your Gitlab API Token

![image](https://hackmd.io/_uploads/BkaUQpGqkx.png)

Now your Jenkins is connected to Gitlab. 
To run a pipeline, create a pipeline `shoeshop` in folder **Action_in_lab** we created before. 
In **Configure** of our pipeline, there are some fields that we need to tick: 

![image](https://hackmd.io/_uploads/HkjW4aGc1x.png)

![image](https://hackmd.io/_uploads/H1rGEpGq1l.png)

- **Discard old build**: Choose it and keep how many builds you want to keep 

![image](https://hackmd.io/_uploads/H11LNaM91g.png)

- **Build when a change is pushed to GitLab. GitLab webhook URL: (URL)**

![image](https://hackmd.io/_uploads/HJvMSTfckx.png)

- **Pipeline**:

![image](https://hackmd.io/_uploads/rkBwBTG5ye.png)

This one you need to choose the Git in `SCM`, for the Git repo, use yours, for the credentials, use user `jenkins` we created before. 

**Branch to build**: 

![image](https://hackmd.io/_uploads/HkZkU6f9yx.png)

I will build on my branch `develop`. You can add more branches too. Save and you can see your pipeline is working

![image](https://hackmd.io/_uploads/rymB86z5yg.png)

But we're not done yet. To automatically notify an external service (Jenkins, Slack, ...) you need to configure Webhook on Gitlab.
:::info
A GitLab Webhook is a mechanism that allows GitLab to automatically notify an external service (like Jenkins, Slack, or a deployment server) whenever a specific event occurs in a repository.
:::

Before that, go to **Settings** of the Admin, choose to **Network --> Outbound Request**. Tick to the box **Allow requests to the local network from web hooks and services**:

![image](https://hackmd.io/_uploads/SJtFPaMqJl.png)

Choose your project on Gitlab, go to **Settings --> Webhook**

![image](https://hackmd.io/_uploads/r1NJP6z9Jg.png)

Our format for URL will be: 

```!
http://<user on jenkins>:<token of that user on jenkins>@<jenkins's address>/project/<path to project on jenkins> 
```

About token, click in your profile picture on Jenkins, choose **Security** or **Configure** if your system is the older version.

![image](https://hackmd.io/_uploads/H1Szd6Gckg.png)

Create a token: 

![image](https://hackmd.io/_uploads/rki8_aGcJe.png)

My final format of URL is: 

```!
http://admin:<token>@jenkins.fucalors.tech/project/Action_in_lab/shoeshop
```
:::warning
Remember to add host on gitlab server

![image](https://hackmd.io/_uploads/rkM1K6f5kx.png)

:::

Choose which events will trigger the pipeline

![image](https://hackmd.io/_uploads/SkqItaMcJg.png)

And disable SSL verification

![image](https://hackmd.io/_uploads/B1TwK6M5ye.png)

Now you will get your own webhook:

![image](https://hackmd.io/_uploads/Bk1tK6f9yx.png)

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

![image](https://hackmd.io/_uploads/ByMbsaGqke.png)

Go to Blue Ocean for better appearance: 

![image](https://hackmd.io/_uploads/S1JBi6Gq1l.png)

Here you can what happened in each stage. 

Check result: 

![image](https://hackmd.io/_uploads/rymFsaf5yg.png)

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

:::info
A Jenkins parameter allows users to dynamically provide input values when triggering a build. This makes Jenkins jobs more flexible and customizable.
:::
This part, you need to have knowledge about Groovy. Before that, install `Active Choice` plugin: 

![image](https://hackmd.io/_uploads/B1xwCYm51x.png)

This will be more flexible for your project. 
Configure our pipeline with this option (before installing plugin):

![image](https://hackmd.io/_uploads/HkJAhtX9Jl.png)

There are some option here for us to choose
- **Boolean parameter**: Checkbox (true/false).
- **Choice Parameter**: Dropdown menu with predefined values.
- **File Parameter**: Allows users to upload a file during build.
- **String Parameter**: Accepts a text string as input.
- **Password Parameter**: A hidden input field for sensitive data.
- **Run Parameter**: Select a previous build from another job.

After installing plugin, it would be like this 

![image](https://hackmd.io/_uploads/SyanAtm5yx.png)

Choose **Active Choices Parameter** and modify like this 

![image](https://hackmd.io/_uploads/HySuJqm5ye.png)

Save and you will see this in your pipeline

![image](https://hackmd.io/_uploads/HJhY1qQc1e.png)

Add some parameters to prepare: 

![image](https://hackmd.io/_uploads/B1RDFiH9Je.png)

![image](https://hackmd.io/_uploads/rkbtZ9m9ke.png)

![image](https://hackmd.io/_uploads/rkDdZhS9Jl.png)

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

![image](https://hackmd.io/_uploads/rk3GN5Xqyg.png)

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

![image](https://hackmd.io/_uploads/B1H0PcBqkx.png)

Function `startProcess` is labeled with stage `start`. If there is no previous processName, it will return `cannot start process`.  
If it worked, the pipeline will echo out a message: 

![image](https://hackmd.io/_uploads/BkBB_9Sc1l.png)

You can check in Blue Ocean or in the main page of project. 

![image](https://hackmd.io/_uploads/Hk80PiBqyl.png)

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

![image](https://hackmd.io/_uploads/Sk4YijBqyg.png)

After pulled from Gitlab, we build it with maven then copy it to our deploy folder, change permission and run project. 

![image](https://hackmd.io/_uploads/ByHHpsHckx.png)

4. Rollback

First, we create 2 another folder for run project and backup (remember to create it under `shoeshop` user)


And change our `folderDeploy`, also add 2 more variables

![image](https://hackmd.io/_uploads/r1faAjHcyg.png)

And we need to approve our code that we created the `rollback_version` parameter before: 

![image](https://hackmd.io/_uploads/SkjdynSc1l.png)

![image](https://hackmd.io/_uploads/Hy4nbhScye.png)

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

![image](https://hackmd.io/_uploads/HkQfd2H5kl.png)

And run any upcode process to see the result:

![image](https://hackmd.io/_uploads/SyEE_hrcyx.png)
