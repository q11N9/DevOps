# CI/CD

Today we will enter a new chapter using Gitlab, which we have installed in [Git](https://hackmd.io/eUu_Kw0qR0inFN9HQehsAA), it's CI/CD. 

# 1. What is CI/CD?
When developers finished their jobs, they need to check if it's worked or not. And us, DevOps, cannot deploy it to server and then give them result by ourself. So **CI/CD** was born. 

It will automatically deploy codes to server and give the result back to the developers. 

**CI/CD** stands for 2 things: **Continuous Integration/ Continuous Deployment or Continuous Delivery**
- **Continuous Integration**: This step will setup everthing, from tools, scripts, testing, ... to integrate to the source code, prepare for the next step. 
- **CD**: 
    - **Continuous Deployment**: automatically deploy the project.
    - **Continuous Delivery**: It's not fully automatic. It can be added a step `confirm`, or it can be more handy.

There are 2 common tools that being used to make a **CI/CD** pipeline: Gitlab and Jenkins. Today we will use Gitlab to create our own pipeline. 

# 2. Gitlab CI/Continuous Deployment

This project will be praticed on our first server (not the Gitlab one). 

First, you need to install [Gitlab runner](https://docs.gitlab.com/runner/install/linux-repository.html)

After installed, choose any project on Gitlab you want to deploy, go to **Settings --> CI/CD --> Runners**. I use Shoeshop Project for example. 

![image](https://github.com/user-attachments/assets/a460d49a-55c5-45d8-b322-4a2a14586e14)

When you expand the **Runners**, you will see in the part **Specific runners**, it has specific runners for our chosen project. 

![image](https://github.com/user-attachments/assets/1079775f-b50f-4232-b441-51f227e72104)

Now we need to configure our runner on the server (not the Gitlab server) by using. 

```bash
gitlab-runner register
```

And then paste the url and token. 

![image](https://github.com/user-attachments/assets/b12a53f9-2b8e-4623-b0b2-11246c103dd8)

The description and tags should be the same as your runner's name. The tags will be used to specify which runner would be used to run pipeline of the project. 

Next is the executor:

![image](https://github.com/user-attachments/assets/137c8b4c-58ee-4836-b22a-179904ba51f0)

This executor will be the environment where our pipeline setup runs on. In this section, I use **shell** for start. 

After completed, you can configure your runner in the file they give 

```bash
Configuration (with the authentication token) was saved in "/etc/gitlab-runner/config.toml"
```

![image](https://github.com/user-attachments/assets/d1687e30-68d8-4426-83fa-7a2a0322371e)

I will let `concurrent` equals to 4, instead of 1 so our runner can run 4 jobs at a time. .

Moreover, in Gitlab website, you can see your runner on it. 

![image](https://github.com/user-attachments/assets/5b1d319f-9625-4bc6-b7b6-0268cedb162f)

in the **Available specific runners**. 

Click the pencil symbol to modify settings of runner. 

![image](https://github.com/user-attachments/assets/c6328e3a-bfb6-40e2-87fc-771f1f329a1e)

You can read the descriptions for more details and follow this setting. 

Now back to our project **shoeshop**. We need a file called `.gitlab-ci.yml` to setup our pipeline. We will use this file to automatically build, deploy it for us, like we have done it by hand in the **Deploy** note.

Before going to create that file, we need to grant permission sudo of some commands for user `shoeshop`. Use `visudo` to edit. 

![image](https://github.com/user-attachments/assets/cda70912-e22f-444d-aed3-36c40d29f2f7)

Now it will run all commands start with `cp`, `chown` and `su shoeshop` without password (Because we want our pipeline is automatic)

This is my yaml file: 

```yaml!
variables: 
    projectname: shoe-ShoppingCart
    version: 0.0.1-SNAPSHOT
    projectuser: shoeshop
    projectpath: /datas/shoeshop/
stages: 
    - build 
    - deploy 
    - showlog

build:
    stage: build
    variables: 
        GIT_STRATEGY: clone
    script:
        - mvn install -DskipTests=true
    tags:
        - devops
    only:
        - tags
deploy:
    stage: deploy
    variables:
        GIT_STRATEGY: none
    script:
        - sudo cp target/$projectname-$version.jar $projectpath
        - sudo chown -R $projectuser. $projectpath
        - sudo su $projectuser -c "kill -9 $(ps -ef| grep shoe-ShoppingCart-0.0.1-SNAPSHOT.jar | grep -v grep | awk '{print $2}')"
        - sudo su $projectuser -c "cd $projectpath; nohup java -jar $projectname-$version.jar > nohup.out 2>&1  &"
    tags: 
        - devops
    only:                                               
        - tags
showlog:
    stage: showlog
    variables:
        GIT_STRATEGY: none
    script:
        - sleep 20
        - sudo su $projectuser -c "cd $projectpath; tail -n 10000 nohup.out"
    tags: 
        - devops
    only:
        - tags

```

Explaination: 
- **variables**: Defines some global variablese to easier reading and setting up. 
- **stages**: Lists our process, including **build, deploy, showlog (optional but recommended)**.
- **build** stage (I let it has the same name as in **stages**, because we are configuring **build** stage): 
    - **stage**: Specify which stage we are configuring. 
    - **variables:** Set `GIT_STRATEGY = clone`, so this will clone the code from the web to our local. **So it will delete all non-exist file in local**.
    - **script**: Get tools (maven)
    - **tags**: Uses for which runners we use to run this. 
    - **only**: Only the changes with tags can be run this pipeline. 
- **deploy** and **showlog**: The uses of their fields are the same as in the **build**. 

Save it and create a tags in here: 

![image](https://github.com/user-attachments/assets/171b2ccd-c865-4730-87ed-e4fba6ae7106)

And our pipeline will run. You can watch it in `CI/CD`:

![image](https://github.com/user-attachments/assets/c67152ec-b121-4786-945b-aa0b0db63e83)

![image](https://github.com/user-attachments/assets/20a9c613-664b-4020-b17f-e544b651e341)

Click into any part and you can see your result, it has the same as our work before: 

![image](https://github.com/user-attachments/assets/eea8ddde-6687-44ec-b6e6-7472554be351)

We have successfully deployed our project!

# 3. Gitlab CI/Continuous Delivery
What about Continuous Delivery? For this one, you can customize to manually run your deploy, or any stage you want. I.e, after fixing, our `.gitlab-ci.yml` becomes: 

```yaml!
variables: 
    projectname: shoe-ShoppingCart
    version: 0.0.1-SNAPSHOT
    projectuser: shoeshop
    projectpath: /datas/shoeshop/
stages: 
    - build 
    - deploy 
    - showlog

build:
    stage: build
    variables: 
        GIT_STRATEGY: clone
    script:
        - mvn install -DskipTests=true
    tags:
        - devops
    only:
        - tags
deploy:
    stage: deploy
    variables:
        GIT_STRATEGY: none
    when: manual
    script:
        - >
            if ["$GITLAB_USER_LOGIN" == "fucalors"]; then
                sudo cp target/$projectname-$version.jar $projectpath
                sudo chown -R $projectuser. $projectpath
                sudo su $projectuser -c "kill -9 $(ps -ef| grep shoe-ShoppingCart-0.0.1-SNAPSHOT.jar | grep -v grep | awk '{print $2}')"
                sudo su $projectuser -c "cd $projectpath; nohup java -jar $projectname-$version.jar > nohup.out 2>&1  &"
            else
                echo "Permission denied"
                exit 1
            fi
    tags: 
        - devops
    only:                                               
        - tags
showlog:
    stage: showlog
    variables:
        GIT_STRATEGY: none
    when: manual
    script:
        - sleep 20
        - sudo su $projectuser -c "cd $projectpath; tail -n 10000 nohup.out"
    tags: 
        - devops
    only:
        - tags

```

I added some new features: 
- **when**: this is used to control which part we can start it. I use it for **deploy** and **showlog**. 
- Adding new condition, so only admin or specific users could start **deploy** or **showlog**. 

![image](https://github.com/user-attachments/assets/11d66560-ea7d-4efe-9553-9257ce2e14a0)

For more information, you can read in [here](https://docs.gitlab.com/ee/topics/build_your_application.html)
