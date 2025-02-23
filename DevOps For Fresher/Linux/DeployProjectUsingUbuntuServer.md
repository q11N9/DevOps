# Deploying simple projects with ubuntu server
## How to deploy a project?

- Which **tools** does the project use? 
- The **configuration file** is the most important.
- There are 2 steps to deploy a project: **build** and **run**. 

Notes: 
- Each project has its own folder.
- Each project has an unique user. 

## Prequisites
- An Ubuntu Server with static IP address. 
## Deploy a frontend project. 
### 1. Vue.js project - "To-do list"

#### a. Deploy project with a normal way
You can download file project in [here](https://devopsedu.vn/wp-content/uploads/2024/02/todolist.zip)

Unzip it and we have a folder `todolist`.
First, we need to create a user: 

```bash
adduser todolist
```

Then, change the owner and group of folder `todolist` to `todolist:todolist`

```bash
chown -R todolist. todolist
```

Set permission for folder `todolist`:

```bash
chmod -R 750 todolist 
```

Because this is a vue project, we need to install **Nodejs**. For more information, you can see more in [here](https://vuejs.org/guide/quick-start)

:::warning
Note: Remember, the version of your tools need to be higher than project needs. 
:::

After install npm and Nodejs, we will install dependencies for our project.

```bash
npm install
```
There is a file `package.json` that guide us to build the project: 

![image](https://github.com/user-attachments/assets/8545e27a-dce8-469c-a779-425ffe4c7a0b)

The `"serve"` command will deploy our project after we `"build"` it. 

Now, if you run `npm run build`, an error occurs: 

![image](https://github.com/user-attachments/assets/1c457435-751c-4624-90d0-b4f1aa445786)

because the version of our tool is older than the project's.

You can install Nodejs 18 in [here](https://www.stewright.me/install-nodejs-18-on-ubuntu-22-04/)

Now we can build our project: 

![image](https://github.com/user-attachments/assets/d301ce11-4de8-459e-a29e-2fb027124fca)

A `dist` directory is created, and we will use it to deploy our project. 

Deploy our project: 

```bash
npm run serve
```

![image](https://github.com/user-attachments/assets/7f75c09b-437a-426c-ac01-21633341084c)

You can direct to those addresses to check the result. But this way is still not optimize. 

#### Deploy project with Nginx

When we install Nginx, the default port is 80. 

![image](https://github.com/user-attachments/assets/513eb0a4-1dd8-4a0e-96a4-e98bd2b5af83)

Like I told, there must be a configuration file somewhere. Nginx's is saved in `/etc/nginx`

![image](https://github.com/user-attachments/assets/601acea2-7e92-4098-b3dc-6a1af5b34d61)

The default setting of Nginx is in `sites-available/default`. 

After changing default port of Nginx to 8999: 

![image](https://github.com/user-attachments/assets/27977b48-f3ac-4bf7-a01d-00112db1314f)

![image](https://github.com/user-attachments/assets/49f77247-230e-41c4-a211-2a71dfa928e8)

Now it is listening in port 8999. 

Next, we create a file config for our project in `conf.d/todolist.conf`. 

![image](https://github.com/user-attachments/assets/a58a0d32-2540-4127-8802-f100c5ba3a7c)

with content:

```nginx
server {
  listen 8081;
  root /home/fucalors/projects/todolist/dist/;
  index index.html;
  try_files $uri $uri/ /index.html;
}
```

And restart Nginx to see the result: 

```bash
systemctl restart nginx
```

In the future, there are more projects that run with Nginx, so instead of restart entire Nginx, we can use 

```bash
nginx -s reload
```

to reload your specific project. 

![image](https://github.com/user-attachments/assets/a7582d6b-3f21-48d7-905b-1d808bd4a5b5)

> If you have `500 Internal Server Error`, check the owner and group of the file `conf.d/todolist`.  
> Then, add user `www-data` (You can check it in `/etc/nginx/nginx.conf`) to group `todolist`.

### 2. React project - "Vision"
This will use service of systemd to deploy project. 

The process is the same as **Vue.js** project, until `npm install`. 

Next, we create a file service in `/lib/systemd/system/vision.service`

```INI
[Service]
Type=simple
User=vision
Restart=on-failure
WorkingDirectory=/home/fucalors/projects/vision/
ExecStart=npm run start -- --port=3000
```

Explaination:
- We build a simple service.
- User `vision`.
- If it is down, it will restart again.
- Working directory.
- Background running command. 

To start our project, we use: 

```bash
systemctl deamon-reload
systemctl start vision
```

There are more ways to deploy frontend project: Docker; Vercel, Netlify, Firebase; .... You can see in this board: 



| Method | Use case | 
| -------- | -------- | 
| **Nginx**         |   Best for static sites (React, Vue, Angular, Svelte)       |
|   **systemd**       |   Keeping a local frontend server running       |
|   **Docker**        |  Portable, scalable, works with Kubernetes        |
|  **Vercel, Netlify, Firebase**        |   Fastest, best for static frontend       |
|   **PM2**       |   For running Node.js-based frontend frameworks       |
|    **Apache HTTP Server**      |   Alternative to Nginx       |
| **CI/CD Pipelines**     | Automate deployments    | 

## Deploy a backend project
### Java spring - "shoeshop"
This project needs 2 parts: Create database server and deploy project. 

The process creating user and set permission, ... are the same as I did before. 

You can see in the folder `shoeshop`, there's a file called `pom.xml`. So we know that the tool we need is [Maven](https://spring.io/guides/gs/spring-boot)

Checking configuration file `pom.xml`: 

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
        <modelVersion>4.0.0</modelVersion>
        <parent>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-parent</artifactId>
                <version>2.2.5.RELEASE</version>
                <relativePath/> <!-- lookup parent from repository -->
        </parent>
        <groupId>com.example</groupId>
        <artifactId>shoe-ShoppingCart</artifactId>
        <version>0.0.1-SNAPSHOT</version>
        <name>shoe-ShopingCart</name>
        <description>shoe Shopping Cart</description>

        <properties>
                <java.version>1.8</java.version>
        </properties>
    
    ...
```

So we need a java version higher than 1.8. 

Required: [Java](https://www.rosehosting.com/blog/how-to-install-java-17-lts-on-ubuntu-20-04/), [Maven](https://phoenixnap.com/kb/install-maven-on-ubuntu)

The next step is configuring our database. You can configure it in an independent server(recommended). 

There is a sql file in our folder and we will use [Mariadb](https://www.digitalocean.com/community/tutorials/how-to-install-mariadb-on-ubuntu-20-04)

After installation, we have a socket called `mysqld` that opens in port 3306: 

![image](https://github.com/user-attachments/assets/86d16e41-737a-470b-a87a-f0945065a39f)

Because i have installed before and configure it, it would display as `0.0.0.0`. But in the start, it will be our localhost, which is `127.0.0.1`. 

To configure it, first we need to stop **Mariadb** and then config the file `/etc/mysql/mariadb.conf.d/50-server.cnf`. 

![image](https://github.com/user-attachments/assets/4dc7f103-b644-400d-b7f9-b7a85de786f8)

There is a `bind-address` there. Change it to `0.0.0.0` and then re-check your netstat after `systemctl restart mariadb`. 

Next step, we need to import our database. To do that, run: 

```bash
$ mysql -u root
```

And then type some sql commands:

```sql!
show databases; --to see the current databases
create database shoeshop; --create our database for our project
create user 'shoeshop'@'%' indentified by '123'; -- this will create a user shoeshop:123 that can access to all databases. 
grant all privileges on shoeshop.* to 'shoeshop'@'%'; -- grant privileges for user shoeshop
flush privileges; --apply changes
```

Next, we need to import database using user `shoeshop`. 

```bash
$ mysql -h 192.168.240.99 -P 3306 -u shoeshop -p
```

and then enter your previous password. 

![image](https://github.com/user-attachments/assets/839387c1-54a2-428a-90fa-1b74d7c39de4)

To import our database: 

```sql!
use shoeshop; 
show tables; 
source /home/fucalors/projects/shoeshop/shoe_shopdb.sql; --path to your database file 
show tables; -- check the result
```

![image](https://github.com/user-attachments/assets/ea4bcaef-cefd-4e3c-853b-34b12fc9c06d)

Now we need to import our database server into configuration file, which is in `src/main/resources/application.properties`. Replace `<>` to our information. 

```java
# ===============================
# DATABASE
# ===============================

spring.datasource.driver-class-name=com.mysql.jdbc.Driver

spring.datasource.url=jdbc:mysql://192.168.240.99:3306/shoeshop
spring.datasource.username=shoeshop
spring.datasource.password=123
```

Login to user `shoeshop` and install dependencies:

```bash
$ mvn install -DskipTests=true
```
![image](https://github.com/user-attachments/assets/e4f775fa-f012-4f9f-9f10-00bd1bbdf909)

It will create a folder `target`.

Finally, we can run our project:

```bash
$ nohup java -jar target/shoe-ShoppingCart-0.0.1-SNAPSHOT.jar 2>&1 &
```
This command will run our project in background and all outputs will be saved in `nohup.out`. 

You can stop project with `kill -9 <PID>`

![image](https://github.com/user-attachments/assets/818e75c4-9f59-44d2-9583-7fdc4a7e33e1)



