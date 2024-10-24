
# CI/CD for NodeJS

### Main Server Setup

#### Prerequisites:
Ensure that you have installed the following on your server/local machine:
- **AWS CLI**
- **Terraform**
- **Docker Engine**
- **kubectl**

### Steps

#### 1. Run Docker using the following command:
```sh
docker run -d --name jenkins-dind \
    -p 8080:8080 -p 50000:50000 \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v $(which docker):/usr/bin/docker \
    -u root -e DOCKER_GID=$(getent group docker | cut -d: -f3) \
    jenkins/jenkins:lts
```

#### 2. Exec into the container and run:
```sh
docker exec -it DOCKER_PROCESS bash

groupadd -for -g $DOCKER_GID docker
usermod -aG docker jenkins
```

#### 3. Setup Jenkins on **localhost:8080**. Configure the Jenkins server and install the **NodeJS** plugin.

#### 4. Restart Docker from CLI:
```sh
docker restart jenkins-dind
```

#### 5. Under **Manage Jenkins**:
Navigate to **Tools** → **NodeJS Installations**. Set the **Name** & **Version** (Name should match the one used in the Jenkinsfile tool section).

### SonarQube Setup

#### 6. Find SonarQube on Docker Hub and fulfill the "Docker Host Requirements" by running:
```sh
sysctl -w vm.max_map_count=524288
sysctl -w fs.file-max=131072
ulimit -n 131072
ulimit -u 8192
```

#### 7. Run SonarQube as a Docker container:
```sh
docker run -d --name sonarqube-dind \
    -e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true \
    -p 9000:9000 sonarqube:latest
```

#### 8. SonarQube default credentials:
- **Username:** admin
- **Password:** admin

Change the password to `Sonarqube@123`. Install the **Sonarqube Scanner** and **Sonar Quality Gates** plugins on Jenkins.

#### 9. On the SonarQube webpage:
- Create a local project.
- Provide "Project display name" and "Project Key".
- Go to **Global Settings**.

#### 10. Create a SonarQube token:
Generate a token of type **Global Analysis Token**. Example:
```
sqa_6zxcvbb59ad051e32589676b877f83c8a32574305
```

#### 11. Add Credentials in Jenkins:
In Jenkins, add credentials for SonarQube as **secret text**, and name it as `cicd-project-token`.

#### 12. Configure SonarQube in Jenkins:
- Go to **Manage Jenkins** → **System** → **SonarQube servers**.
- Set **Name**, **Server URL**, and the **Authentication token**.

#### 13. Configure the SonarQube Scanner in Jenkins:
- Go to **Manage Jenkins** → **Tools** → **SonarQube Scanner installations**.
- Set **Name** as `SonarQubeScanner` and save.

#### 14. Networking Between Jenkins and SonarQube:
Since Jenkins and SonarQube containers are not on the same network, create a Docker network:
```sh
docker network create dind-network
docker network connect dind-network jenkins-dind
docker network connect dind-network sonarqube-dind
```

Verify they are in the same network by pinging the SonarQube server from the Jenkins container:
```sh
apt-get update
apt install iputils-ping
docker exec -it jenkins-dind ping sonarqube-dind
```

### Docker Image Build

#### 1. Install Docker Pipeline, Docker, and Docker API Plugins on Jenkins.

#### 2. Create a Docker Hub repository:
Create a repo on Docker Hub named **CICD-Project-1** and ensure it's public.

#### 3. Update the Jenkinsfile and build the project.

#### 4. Install Trivy for image scanning:
Install Trivy on Jenkins by running:
```sh
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin v0.18.3
```

Restart Jenkins after installing Trivy:
```sh
docker restart jenkins-dind
```

Update the Jenkinsfile and build the pipeline again.

### Pushing the Docker Image to ECR:

#### 1. Install AWS CLI on the Jenkins Docker container:
```sh
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

#### 2. Create an IAM user with ECR full access and retrieve the access key:
- **AWS Access Key ID:** AKIAEXAMPLESEE
- **AWS Secret Access Key:** a$secret6E7EqCMAccess3ZbKey6gXXXXcs9w/0f
- **Default region name:** us-east-x

#### 3. Create an ECR repository and get the URI:
```sh
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com
```
Create ECS Cluster using [Terraform](https://github.com/pratiksinha-kol/EKS-with-Terraform). 

#### 4. Add AWS Plugins to Jenkins:
Install **AWS ECR Plugin** and **AWS SDK: ALL** in Jenkins.

#### 5. Configure AWS Credentials in Jenkins.

#### 6. Update the Jenkinsfile:
Set `REPO_NAME=name_of_repo`, `REPO_REGISTRY`, and `IMAGE_TAG`, then build the pipeline.

### Deploying the Application to ECS:

#### 1. Create an ECS Cluster and deploy the Docker image.

#### 2. Configure the ECS Task Definition:
Use the `ECR_IMAGE_URI`. The application listens on port 3000, so configure the container port to 3000.

#### 3. Create an ECS Service:
- Create a new security group (open ports 80, 3000) and a new ALB.
- For the health check, use `/hello`.

#### 4. Confirm the deployment:
Access the application via:
```
DNS_NAME/hello
```
