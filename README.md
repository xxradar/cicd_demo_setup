# cicd_demo_setup
## Pre-requisites
### VM
### Infrastructure
- AWS EC2 Ubuntu
- t3 large
- 50 GB Storage
- SSH key
#### Networking
- Assign an EIP
#### Install Docker
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
