# cicd_demo_setup
## Pre-requisites
### 1. Infrastructure
- AWS EC2 Ubuntu
- t3 large
- 50 GB Storage
- SSH key
### 2. Networking
- Assign an EIP
### 3.Install Docker
Docker quick install
```
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh ./get-docker.sh 
```
```
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
```
Verify Docker install
```
docker run hello-world
```
## Install Jenkins
#### 1. Make sure Docker is OK
```
docker version
docker ps
```
### 2. Create the Docker network
Jenkins and the inner Docker daemon need to "see" each other by name.
```
docker network create jenkins
```
This creates a bridge network called jenkins. 

### 3. Start the Docker-in-Docker daemon (this is where TLS certs are created)
```
docker run --name jenkins-docker --rm --detach \
  --privileged \
  --network jenkins --network-alias docker \
  --env DOCKER_TLS_CERTDIR=/certs \
  --volume jenkins-docker-certs:/certs/client \
  --volume jenkins-data:/var/jenkins_home \
  --publish 2376:2376 \
  docker:dind --storage-driver overlay2
```
Sanity check that dind is healthy:
```
docker logs jenkins-docker
```
You should see logs mentioning something like generating certs, listening on port 2376, etc. If you see errors about certificates or permissions here, that is the first place to fix things.

### 4. Build the Jenkins image with Docker CLI inside
Create a Dockerfile in an empty folder
```
mkdir jenkins_build && cd jenkins_build
```
```
cat > Dockerfile <<'EOF'
FROM jenkins/jenkins:2.528.2-jdk21

USER root
RUN apt-get update && apt-get install -y lsb-release ca-certificates curl && \
    install -m 0755 -d /etc/apt/keyrings && \
    curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc && \
    chmod a+r /etc/apt/keyrings/docker.asc && \
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
    https://download.docker.com/linux/debian $(. /etc/os-release && echo \"$VERSION_CODENAME\") stable" \
    | tee /etc/apt/sources.list.d/docker.list > /dev/null && \
    apt-get update && apt-get install -y docker-ce-cli && \
    apt-get clean && rm -rf /var/lib/apt/lists/*
USER jenkins
RUN jenkins-plugin-cli --plugins "blueocean docker-workflow json-path-api"
EOF
```
Then build it:
```
docker build -t myjenkins-blueocean:2.528.2-1 .
```
This gives you a Jenkins image that has the Docker CLI and plugins you need. 

### 5. Run Jenkins and connect it to the Docker daemon with TLS
Now run the Jenkins container:
```
docker run --name jenkins-blueocean --restart=on-failure --detach \
  --network jenkins \
  --env DOCKER_HOST=tcp://docker:2376 \
  --env DOCKER_CERT_PATH=/certs/client \
  --env DOCKER_TLS_VERIFY=1 \
  --publish 8080:8080 --publish 50000:50000 \
  --volume jenkins-data:/var/jenkins_home \
  --volume jenkins-docker-certs:/certs/client:ro \
  myjenkins-blueocean:2.528.2-1
```
### 6. Check that the certs are actually there

Before trying Jenkins, you can verify the cert setup with a simple test container:
```
docker run --rm \
  --network jenkins \
  --env DOCKER_HOST=tcp://docker:2376 \
  --env DOCKER_CERT_PATH=/certs/client \
  --env DOCKER_TLS_VERIFY=1 \
  --volume jenkins-docker-certs:/certs/client:ro \
  docker:cli version
```
If this prints server and client version, your TLS and certs are correct. If it fails with x509 or "certificate signed by unknown authority", something is wrong with the mount or env vars.

### 7. Access Jenkins UI
First, grab the initial admin password:
```
docker exec -it jenkins-blueocean cat /var/jenkins_home/secrets/initialAdminPassword
```
Then connect to the UI via port-forwarding. <br>
```
ssh -i <SSH_KEY> ubuntu@EIP -L 8080:localhost:8080
```
```
http://localhost:8080
```
Paste the initial password in the browser and finish the setup wizard, install suggested plugins, create admin user, etc.

## Lacework Integration
### Generate `.lacework/codesec.yaml`
Make sure there is a configured `.lacework/codesec.yaml` in the repository.<br>
https://docs.fortinet.com/document/forticnapp/latest/administration-guide/975371/leveraging-the-codesec-yaml-file <br>
```
lacework iac config generate
```
Make sure this `.lacework/codesec.yaml` is pushed to the  repo.
<br>
### Configure the credentials and a Jenkins Job
Follow the instructions at
https://docs.fortinet.com/document/forticnapp/latest/administration-guide/127315/jenkins-integration

Create a job:
- configure the git repo
- make sure the env variables are set
- configure a build step
- configure the `LW_ACCOUNT`
- Source Code Management -> Git -> set repo
- Source Code Management -> Git -> Advanced -> set branch specifier
- Create a Build Step

```
#!/bin/bash
## Provide Lacework credentials
echo "LW_ACCOUNT=fortinet-cse-international" > env.list 
echo "LW_API_KEY=${LW_API_KEY}" >> env.list
echo "LW_API_SECRET=${LW_API_SECRET}" >> env.list 

## Provide Jenkins build details
env | grep '^BRANCH_\|^CHANGE_\|^TAG_\|^BUILD_\|^JOB_\|^JENKINS_\|^GIT_' >> env.list

## Run Codesec against the workspace
docker run --rm \
  --env-file env.list \
  -v "$PWD:/app/src" \
  lacework/codesec:stable \
  lacework iac scan --directory=/app/src
```
