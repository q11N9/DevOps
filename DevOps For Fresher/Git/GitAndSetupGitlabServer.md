# Git
## What is Git? 
Git is a DevOps tool used for source code management. It is used to tracking changes in the source code, enabling multiple developers to work together on non-linear development. 

### The three states
Git has three main states that your files can reside in: **modified, staged, committed**. 

- **Modified** means you have **changed** the file but have not committed it to your database yet.
- **Staged** means that you have marked a modified file in its current version to go into your next commit snapshot.
- **Committed** means that the data is safely stored in your local database.

![image](https://github.com/user-attachments/assets/a81759cd-293f-4568-b186-71d1f9fb78d7)

This is an image of Git architecture: 

![image](https://github.com/user-attachments/assets/267dae51-a8da-4842-82fd-bcd202d8bed8)

# Install Gitlab server

To download Gitlab, you can choose your favourite version in [here](https://packages.gitlab.com/gitlab/gitlab-ee). In this article, I will use version 14. 

Install Gitlab on Ubuntu: 

```bash
$ curl -s https://packages.gitlab.com/install/repositories/gitlab/gitlab-ee/script.deb.sh | sudo bash
$ sudo apt-get install gitlab-ee=14.4.1-ee.0
```

Now we want to access our Gitlab through a local domain. First, configure file `/etc/hosts` by adding a new line with syntax: 

```
<Gitlab server IP> <Gitlab's domain> 
```

Then configure file `/etc/gitlab/gitlab.rb`, change our `external_url` to our domain. 

![image](https://github.com/user-attachments/assets/f37fc630-5a06-48c5-8433-49efe011cf61)

And run 

```bash
gitlab-ctl reconfigure
```

to reconfigure our changes. 

So we're done in our Ubuntu server. To access through our Windows, we first backup the file `hosts` in 

```
C:\Windows\System32\drivers\etc
```

Then replace it with a new file with content:

```
<Gitlab server IP> <Gitlab's domain> 
```

Congrats! You successfully installed your Gitlab server.

![image](https://github.com/user-attachments/assets/fc29be32-cec4-4170-9c41-2d0025c76de6)

To login, you can use user `root` with password saved in 
`/etc/gitlab/initial_root_password`

# Common Git commands

As you know, there are 3 states of Git. This example, I will use `test-project`, with username `Nguyen Van Ngoc Quy` and email `fucalors@fucalors.tech`. 
First, we need to setup Git global using: 

```
git config --global user.name "Nguyen Van Ngoc Quy"
git config --global user.email "fucalors@fucalors.tech"
```

To create a new repository: 

```
git clone git@git.fucalors.tech:fucalors/test-project.git
cd test-project
git switch -c main
touch README.md
git add README.md
git commit -m "add README"
git push -u origin main
```

Push an existing folder: 

```cd existing_folder
git init --initial-branch=main
git remote add origin git@git.fucalors.tech:fucalors/test-project.git
git add .
git commit -m "Initial commit"
git push -u origin main
```

Push an existing Git repository

```
cd existing_repo
git remote rename origin old-origin
git remote add origin git@git.fucalors.tech:fucalors/test-project.git
git push -u origin --all
git push -u origin --tags
```

# Deploy Gitworkflow
This is a picture of Git workflow

![image](https://github.com/user-attachments/assets/792c7d04-c4a8-4b1a-be04-9f338f339175)

- Branch `main`: Contains user environment codes. 
- Branch `develop`: Contains development environment codes. 
- Branch `feature`: Created from branch `develop`, contains feature codes. 
- Branch `release`: Contains testing environment codes. 
- Branch `hotfix`: Created from branch `main`. 

As a DevOps, you need to restrict the push and merge on protected branches. 

