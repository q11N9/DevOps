# Docker
# 1. What is Docker? 
Docker is an open platform for developing, shipping, and running applications. [[1]](https://docs.docker.com/get-started/docker-overview/)
This is sound generally. But you can imagine that, when programmer codes an app or a program, they need to import many packages, libraries, ... to work. But until implementing to server and then transfer them to users, they cannot do that on user's devices. So, docker was born. 

Docker helps us to create a "virtual environment", let us run applications without affecting our local settings. 

# 2. Docker architecture
Docker uses a client-server architecture

![image](https://hackmd.io/_uploads/HkNMFG2Dyg.png)

You can connect `Docker clients` to a remote `Docker daemon`, which does the heavy of building, running and distributing our `Docker container`. 
For connection between client and daemon, using a REST API, over UNIX sockets or a network interface. Another `Docker client` is `Docker Compose`
## Docker Daemon (dockerd)
It listens for requests and manages objects (images, containers, networks and volumes)
## Docker client (docker)
- Is the primary way that many Docker users interact with Docker
- Commands use `Docker API`
- They can communicate with more that one daemon
## Docker registries
- Stores Docker images 
## Docker objects
### Images
An images is a **read-only** template with instructions for creating a Docker container
### Containers
- A container is a runnable instance of an image
- We can create, start, stop, move or delete them

The relationship between containers and images likes relationship between super class and sub class in Java OOP. 

# 3. How to use Docker
## 3.1 Small Python Web App
Here is a small project to containerize a Python Web App
`Dockerfile`

```dsl!
# Build stage
FROM python:3.9-slim as builder
WORKDIR /app
COPY requirements.txt requirements.txt
RUN pip install --no-cache-dir -r requirements.txt

# Runtime stage
FROM python:3.9-slim
WORKDIR /app
COPY --from=builder /usr/local /usr/local
COPY . .
ENV PATH=/root/.local/bin:$PATH
CMD ["python", "app.py"]
```
In this `Dockerfile`: 
- Base image is minimal Python 3.9 image
- Our work dir is in `/app`
- Copy the `requirements.txt` file from the host machine to the container
we will install dependencies for our app. 
`docker-compose.yml`

```yaml!
version: '3.8'
services:
  web:
    build: .
    ports:
      - "5000:5000"
    environment:
      - REDIS_HOST=redis
    depends_on:
      - redis

  redis:
    image: redis:alpine
    volumes:
      - redis_data:/data
volumes:
  redis_data:
```
Explaination: 
- Define Docker Compose version is `3.8`
- Web Application: 
    - Build `Dockerfile` in our current directory
    - Map the port **5000** of the container to port **5000** in the host
    - Set an environment variable `REDIS_HOST=redis`
    - Ensure that the `redis` service start before `web`, but it does not wait until Redis is fully ready
- Redis Service 
    - `image`: Use the lightweight Alpine-based Redis image
    - `volumes`: Mount `redis_data:/data`
- Lastly, defines `redis_data` so data will not be lost even if the container is removed.


`requirements.txt`

```
redis
flask
```

`app.py`

```python!
from flask import Flask
import redis
import os

app = Flask(__name__)
redis_host = os.getenv("REDIS_HOST", "localhost")
r = redis.Redis(host=redis_host, port=6379, decode_responses=True)

@app.route('/')
def hello():
    r.incr('hits')
    return f'This page has been viewed {r.get("hits")} times.'

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```
## 3.2 Dockerize a Front-end ReactJS.
You can download the source code in [here](https://devopsedu.vn/wp-content/uploads/2024/02/todolist.zip)
`Dockerfile`

```
## Build stage ##
FROM node:18.18-alpine AS build

WORKDIR /app
COPY . .
RUN npm install
RUN npm run build

## Run stage ##
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```
- **Build stage**: 
    - This step will pull an image NodeJS from Docker Hub, then run our own project on there. 
    - Working dir is in `/app`
    - It copies everything from the current directory (on the host) to the current directory (inside the container)
    - Install dependencies.
- **Run stage**:
    - Pull a Nginx Alpine Image from Docker Hub
    - COPY the file created after build stage (`dist`) to /usr/share/nginx/html (the default working directory of Nginx container)
    - The container will open in port 80 for mapping. 
    - Run Nginx as the main process in the container. 

Finally, we just need to run 
```bash!
docker run --name todolist-v1 -dp 6868:80 todolist:v1
```

to run our project on host port 6868.

## 3.3 Dockerize a Back-end JavaSpringBoot. 
With this [project](https://devopsedu.vn/wp-content/uploads/2024/02/shoeshop-ecommerce.zip), you can use this Dockerfile: 
```
## Build stage ##
FROM maven:3.5.3-jdk-8-alpine AS build

WORKDIR /app
COPY . .
RUN mvn install -DskipTests=true

## Run stage ##

FROM alpine:3.19

RUN adduser -D shoeshop
RUN apk add openjdk8

WORKDIR /run
COPY --from=build /app/target/shoe-ShoppingCart-0.0.1-SNAPSHOT.jar /run/shoe-ShoppingCart-0.0.1-SNAPSHOT.jar

RUN chown -R shoeshop:shoeshop /run

USER shoeshop

EXPOSE 8080

ENTRYPOINT java -jar /run/shoe-ShoppingCart-0.0.1-SNAPSHOT.jar
```
And `docker run` the same as in front-end project.

# 4. Docker Registry
A **Docker Registry** is a storage and distribution system for** Docker images**. It allows users to **push**, **pull**, and **manage** container images efficiently.

Types of Docker Registries:
- **Public Registry** – Open to anyone (e.g., Docker Hub).
- **Private Registry** – Self-hosted for internal use (e.g., Docker Private Registry, AWS ECR, GCP Artifact Registry).

Popular Docker Registries:
- **Docker Hub** (hub.docker.com) – Default public registry.
- **AWS Elastic Container Registry** (ECR) – Private registry in AWS.
- **Google Container Registry** (GCR) – Managed by Google Cloud.
- **Harbor** – Open-source enterprise registry with security features.

## 4.1 How to setup Docker registry (Free)
### Using Docker Hub
First, you need to login with your Docker Hub account using: 

```bash!
docker login 
```

Rename (it's actually make a new image from original image but with different name)

```bash=
docker tag <original_image_name> <name_on_repository> 
```

The new name will be displayed on Docker Hub. It should be `<your_username>/<name_of_image>`

And then use 

```bash=
docker push <new_image_name>
```
To push it on your Docker Hub's repository.

### Using Private Registry
You need a server which has installed Docker. **It should be independent from other server(Gitlab, ...)** for avoiding overload. 

Next, we need to create a certification with our own signature. We will use **openssl** to create `certs`. 

The command is

```bash=
openssl req -newkey rsa:4096 -nodes -sha256 -keyout certs/domain.key -subj "/CN=<host_address>" -addext "subjectAltName = DNS:<host_address>,IP:<host_address>" -x509 -days 365 -out certs/domain.crt
```

This will create 2 files in `certs`: `domain.cert` and `domain.key`


After having our own `certs`, we need a registry image to run our registry on it. 
`docker-compose.yml`

```yaml!
version: '3'

services:
  registry:
    image: registry:2
    restart: always
    container_name: registry-server
    ports:
      - "5000:5000"
    volumes:
      - ./data:/var/lib/registry
      - ./certs:/certs
    environment:
      REGISTRY_HTTP_TLS_CERTIFICATE: /certs/domain.crt
      REGISTRY_HTTP_TLS_KEY: /certs/domain.key
```

Explaination: 
- Our version of Docker Compose file format is 3.
- Use the official Docker registry image version 2. 
- Automatically restarts.
- Name: `registry-server`
- Port: `5000(host):5000(container)`
- Mount local folder `/data` in current directory to `var/lib/registry` inside the container
- Mount local folder `/certs` in current directory to `/cert` inside the container. This is where **TLS(SSL)** certificate file are stored.
- `REGISTRY_HTTP_TLS_CERTIFICATE` → Specifies the S**SL certificate file** (`domain.crt`), used to enable HTTPS.
- `REGISTRY_HTTP_TLS_KEY` → Specifies the **SSL private key** (domain.key), used for encryption.

To start Docker registry: 

```bash
docker-compose up -d
```

Our server is running now. You can check in **https://<host_address>:5000** (It would be blank).For more readable, you can check in `/v2/_catalog`

The registry is running but cannot login. To do that, follow this instruction.  

Copy file `domain.cert` to `/etc/docker/cert.d` under the name `ca.cert`

![image](https://hackmd.io/_uploads/rJd3lSPFke.png)

And restart Docker:

```bash
systemctl restart docker
```

After rest
Now you can use: 

```bash!
docker login <host_address>
```

(any users could login into). 
So now just do the same as we did with Docker Hub before: 

![image](https://hackmd.io/_uploads/S1NQNHDt1e.png)

Checking our website(in `/v2/_catalog`): 

![image](https://hackmd.io/_uploads/Hy24VSPYye.png)

# 5. Using Docker to establish CI/CD pipeline
The principle of this part is to create an imange of our project, then push it to our registry we established before (if you have money, i recommended to use Harbor with AWS Cloud). After that, we pulled it to our local machine and run it. 

We need first create a **Dockerfile** in current branch first. For example, I created one in branch `staging`: 

`Dockerfile`

```
## Build stage ##
FROM maven:3.5.3-jdk-8-alpine AS build

WORKDIR /app
COPY . .
RUN mvn install -DskipTests=true

## Run stage ##

FROM alpine:3.19

RUN adduser -D shoeshop
RUN apk add openjdk8

WORKDIR /run
COPY --from=build /app/target/shoe-ShoppingCart-0.0.1-SNAPSHOT.jar /run/shoe-ShoppingCart-0.0.1-SNAPSHOT.jar

RUN chown -R shoeshop:shoeshop /run

USER shoeshop

EXPOSE 8080

ENTRYPOINT java -jar /run/shoe-ShoppingCart-0.0.1-SNAPSHOT.jar

```

Next, modify our `.gitlab-ci.yml`: 

```yaml!
variables: 
    DOCKER_IMAGE: ${REGISTRY_URL}/${REGISTRY_PROJECT}/${CI_PROJECT_NAME}:${CI_COMMIT_TAG}_${CI_COMMIT_SHORT_SHA}
    DOCKER_CONTAINER: ${CI_PROJECT_NAME}
stages: 
    - buildandpush
    - deploy 
    - showlog

buildandpush:
    stage: buildandpush
    variables: 
        GIT_STRATEGY: clone
    before_script:
        - docker login ${REGISTRY_URL} -u ${REGISTRY_USER} -p ${REGISTRY_PASSWORD}
    script:
        - docker build -t $DOCKER_IMAGE .
        - docker push $DOCKER_IMAGE
    tags:
        - devops
    only:
        - tags
deploy:
    stage: deploy
    variables:
        GIT_STRATEGY: none
    before_script:
        - docker login ${REGISTRY_URL} -u ${REGISTRY_USER} -p ${REGISTRY_PASSWORD}
    script:
        - docker pull $DOCKER_IMAGE
        - docker rm -f $DOCKER_CONTAINER
        - docker run --name $DOCKER_CONTAINER -dp 8080:8080 $DOCKER_IMAGE
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
        - docker logs $DOCKER_CONTAINER
    tags: 
        - devops
    only:
        - tags
```

Some variables you need to define first: 
- `$REGISTRY_URL`: The registry's url that you have created(mine is `192.168.240.100:5000`)
- `$REGISTRY_PROJECT`: Our registry's project we want to push and pull image from. 
- `$CI_PROJECT_NAME`: The default project name in our Gitlab. 
- `$CI_COMMIT_TAG`: Our tag we create to trigger the pipeline. 
- `$CI_COMMIT_SHORT_SHA`: Return the first 6 characters of SHA hashing. 

You can add your variables in **Settings --> CI/CD --> Variables**. 

Those docker commands are pretty detailed, like the process I told before. 