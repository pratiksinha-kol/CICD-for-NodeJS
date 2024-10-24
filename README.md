# CICD for NodeJS

### Main Server

#### Ensure that you have installed AWS CLI, Terraform, Docker Engine and kubectl on the server.  

1. **Run Docker using the given command:**
```sh
docker run -d --name jenkins-dind \
    -p 8080:8080 -p 50000:50000 \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v $(which docker):/usr/bin/docker \
    -u root -e DOCKER_GID=$(getent group docker | cut -d: -f3) \
    jenkins/jenkins:lts
```

2. **Exec into the pod and run the following command:**
```sh
docker exec -it DOCKER_PROCESS bash

groupadd -for -g $DOCKER_GID docker
usermod -aG docker jenkins
```

3. Setup Jenkins on ***localhost:8080*** must be complete now. Configure the Jenkins server and add NodeJS plugin to use.

4. **Restart Docker from CLI, not from the GUI:**
```sh
docker restart jenkins-dind
```

5. **Under Manage Jenkins:**

 - Navigate to **Tools** → **NodeJS Installations**

 - Set the **Name & Version** (Name should be the same as used in Jenkinsfile tool section)

6. **Find SonarQube on Docker Hub** and ensure "Docker Host Requirements" are fulfilled. Run the commands:
```sh
sysctl -w vm.max_map_count=524288
sysctl -w fs.file-max=131072
ulimit -n 131072
ulimit -u 8192
```

7. **Run SonarQube as Docker:**
```sh
docker run -d --name sonarqube-dind \
    -e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true \
    -p 9000:9000 sonarqube:latest
```

8. **Default SonarQube credentials:**

  - **Default Username:** admin

  - **Default Password:** admin

Change password to _Sonarqube@123_. Install **"Sonarqube Scanner"** and **"Sonar Quality Gates"** plugin on Jenkins.

9. **On SonarQube webpage:**

  - Create a local project

  - Provide "Project display name" and "Project Key"

  - Go to Global Settings

10. **Create a SonarQube token** of type "Global Analysis Token":

    _Token example: sqa_6zxcvbb59ad051e32589676b877f83c8a32574305_

11. **Add Credentials on Jenkins** for SonarQube as "secret text" and name it as _cicd-project-token_

12. **Go to Manage Jenkins:**

  - System → SonarQube servers

  - Set Name (SonarQube), Server URL, and Authentication token.

13. **Go to Manage Jenkins:**

  - Tools → SonarQube Scanner installations

  - Set Name (SonarQubeScanner) → Save & Apply.

14. Since, Jenkins and SonarQube containers are **NOT** on the **same network**, it will create a issue with the pipeline.

    **Create Docker network for Jenkins and SonarQube:**
    ```sh
    docker network create dind-network
    docker network connect dind-network jenkins-dind
    docker network connect dind-network sonarqube-dind
    ```
    Confirm if they are in the same network :-->
    
    Login into Jenkins container to ping SonarQube server to ensure they are in the same network:
    ```sh
    apt-get update
    apt install iputils-ping
    docker exec -it jenkins-dind ping sonarqube-dind
    ```

### Docker Image Build

1. **Install Docker Pipeline, Docker, and Docker API Plugins** on Jenkins server/docker.

2. On Docker Hub, create a repo named **CICD-Project-1** and ensure it is public.

3. Update the **[Jenkinsfile](Jenkinsfile)** and build the project.

4. **Install Trivy on Jenkins server** for image scanning:

   ```sh
   curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin v0.18.3
   ```

  Restart the container after installing Trivy:
  ```sh
  docker restart jenkins-dind
  ```

  Update the [Jenkinsfile](Jenkinsfile) and build the pipeline again.

### Push Image to ECR:

1. **Install AWS CLI on Jenkins Docker:**

```sh
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

2. **Create IAM user with EC2 Registry full access and get the access key:**

  - AWS Access Key ID: AKIAEXAMPLESEE

  - AWS Secret Access Key: a$secret6E7EqCMAccess3ZbKey6gXXXXcs9w/0f

  - Default region name: us-east-x

3. **Go to ECR, create a repo** and get the URI: ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/name_of_repo. Fetch the push command:
```sh
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com
```

4. Add the given plugins --> **"AWS ECR Plugin", "AWS SDK: ALL"** on Jenkins server.

5. Under **Credentials**, select AWS Credentials and configure the access keys.

6. Update the Jenkinsfile with variables `REPO_NAME=name_of_repo`, `REPO_REGISTRY`, `IMAGE_TAG`, and build the pipeline.

7. **Create ECS Cluster** to deploy the application with the image built.

8. Configure **task definition** for the ECS cluster, use the `ECR_IMAGE URI`. The application listens on port 3000, so set the container port to 3000.

9. **Create ECS Service:**

  - Go back to the ECS cluster

  - In the service, create a new security group (open port 80, 3000) and new ALB

  - In the health check, use `/hello`

10. Confirm using `DNS_NAME/hello`

