# Main course
*Continuous Delivery using Docker*


## Setting up Git server
Steps to setup gitlab and clone repo's


### Setting up GitLab - docker-compose
- The containers that we need for our continuous delivery environment will be managed by Docker Compose. Later we will add more.
- You will find a docker-compose.yml file in $HOME/atos-workshop/continuous. For now this file only contains one container definition for GitLab, our Git server.

```
git:
  image: 172.31.21.24:5000/gitlab/gitlab-ce:8.0.4-ce.1
  volumes:
    - ./.data/gitlab/config:/etc/gitlab
    - ./.data/gitlab/logs:/var/log/gitlab
    - ./.data/gitlab/data:/var/opt/gitlab
  ports:
    - "2222:22"
    - "80:80"
```


### Setting up GitLab - Start
- Execute the following commands in the shell on your AWS instance.

```
# navigate to atos-workshop/continuous
docker-compose up -d

# to inspect the logs
docker-compose logs git

# Wait until gitlab server has started
# Look for 'The server is now ready to accept connections'
# Use ctrl-c to quit the logging
```


### Setting up GitLab - Configure
- Start a webbrowser and browse to: `http://< ip-aws-instance >/`
- Login with the default user and password.
  - username: root
  - password: 5iveL!fe
- Next you have to set a new password. After setting the password login again.


### Setting up GitLab - Configure
- Go to create new project, fill in the form as follows:
  - Project path: jfall
  - Import project from: "git Any repo by URL" `https://bitbucket.org/jcz-jfall/stickynote-service.git`
  - Visibility Level: Public
- Create project, now you have your own Git server running.


### Setting up Jenkins - Dockerfile
- Jenkins is available as Docker image but for this workshop we build our own Jenkins image. We use the Jenkins base image.
- Go to the `atos-workshop/continuous/jenkins` directory and add a file named `Dockerfile` with the following content.

```
FROM 172.31.21.24:5000/jenkins:1.609.2

USER root
RUN apt-get update && apt-get install \  
     wget curl apt-transport-https -y \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

USER jenkins

COPY plugins.txt /usr/share/jenkins/plugins.txt
RUN /usr/local/bin/plugins.sh /usr/share/jenkins/plugins.txt
```


### Setting up Jenkins - plugins
- Since we try to automate most of the things we are doing, we also automate and manage our jenkins plugins by specifying them in the plugins.txt file.
- We have already added a file containing most of the plugins but some are missing. Add the following plugins to the file `plugins.txt`.  

```
docker-build-step:1.33
docker-commons:1.2
git-client:1.19.0
git:2.4.0
gradle:1.24
delivery-pipeline-plugin:0.9.7
```


### Setting up Jenkins - docker-compose
- The containers for our continuous delivery environment are managed by Docker Compose, as seen for GitLab. Now we will add the Jenkins container to the configuration.
- Create a dir for jenkins data

```
# Navigate to the atos-workshop/continuous directory first
sudo mkdir -p .data/jenkins && sudo chmod 777 .data/jenkins
```
- Edit the file `docker-compose.yml`, add the following content at the end.

```
jenkins:
  build: ./jenkins
  volumes:
    - ./.data/jenkins:/var/jenkins_home
  ports:
    - "8080:8080"
  extra_hosts:
    dockerhost: 172.17.0.1
    mavenrepository: 172.31.21.24
```


### Jenkins - Add link to gitlab container
- We need to make the GitLab container accessible from the jenkins container.
- We do that by adding a link in the jenkins container configuration.
- Add the `links:` section and `git:git` to that section, so the jenkins container can refer to the git host, using the name "git".

```
jenkins:
  build: ./jenkins
  volumes:
     - ./.data/jenkins:/var/jenkins_home
  ports:
    - "8080:8080"
  extra_hosts:
    ... ...
  links:
    - git:git
```


### Setting up Jenkins - docker-compose
- We have defined our jenkins container in a docker-compose file.
  - The container will be build based on a Dockerfile in the directory `./jenkins`.
  - The container will mount the `jenkins_home` directory to the directory `.data/jenkins` on the host.
  - The container maps port `8080` to `8080` on the localhost.
  - The container will have an extra entry in the host file to resolve the dockerhost for building images.
- Start the docker container.
  ```
  docker-compose up -d
  ```
- Open Jenkins in a browser (http://< ip-aws-instance >:8080).


### Jenkins - Global configuration
- First we have to set some global configurations in jenkins.
  - Open the Jenkins application in your browser
  - Go to: Manage Jenkins -> Configure system
  - Search for: `Docker Builder` and provide the url: `http://dockerhost:2375`


### Jenkins - Setting up delivery pipeline
Now we are ready to set up our delivery pipeline. The pipeline is an aggregation of several small builds to build, test and deliver our service.

![logo](images/pipeline-complete.png)


### Jenkins - Job main (1/2)

Clone repo and setting build variables for the delivery pipeline

![logo](images/pipeline-main.png)


### Jenkins - Job main (2/2)
- Create new item
- Item name: `service-main`
- Type: `Free style project`
- Delivery pipeline configuration: `Stage Name=build, Task Name=clean`
- Source Code Management Git: http://git/root/jfall.git (remember the linked container named git)
- Add post build action: Trigger parameterized build on other projects
  - projects to build: `service-build` (ignore non-existing error)
  - Add parameters: predefined parameters
  ```
  SOURCE_BUILD_NUMBER=${BUILD_NUMBER}
  SOURCE_BUILD_WORKSPACE=${WORKSPACE}
  VERSION=1.0.${SOURCE_BUILD_NUMBER}
  ```
- Save


### Jenkins - Job build (1/2)

Compile the sources and run the unit tests for our service.

![logo](images/pipeline-build.png)


### Jenkins - Job build (2/2)
- Create new item
- Item name: `service-build`
- Type: `Free style project`
- Delivery pipeline: `Stage Name=build, Task Name=build`
- Advanced Project Options
  - Use custom workspace directory: `${SOURCE_BUILD_WORKSPACE}`
- Add build step: Invoke Gradle script
  - Use gradle wrapper
  - Tasks: `build`
- Save, and test by starting the `service-main` job.


### Jenkins - Job package (1/3)

Create a docker container for our service

![logo](images/pipeline-package.png)


### Jenkins - Job package (2/3)
- Create new item
- Item name: `service-package`
- Type: `Copy service-build`
- Delivery pipeline: `Stage Name=build, Task Name=package`
- Change build step: Invoke Gradle script
  - change: Tasks: `buildImage`
- Save


### Jenkins - Job package (3/3)
- Configure job `service-build` to add a trigger.
  - Add post build action: Trigger parameterized build on other projects
    - projects to build: `service-package`
    - Add parameters: current build parameters
- Save and test by starting the main project.
- If you like to see how we build our docker image, you can inspect the `build.gradle` file in the repository (GitLab server).


### Adding docker ui
- In the next part we are creating many docker containers. Just for the fun we install a dockerui on our host.
- Edit the docker-compose.yml (in atos-workshop/continuous) and add the dockerui section

```
dockerui:
  image: 172.31.21.24:5000/dockerui/dockerui
  privileged: true
  ports:
    - "9090:9000"
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock
```
- Use `docker-compose up -d` to start the dockerui container.
- Browse to http://< ip-aws-instance >:9090 to see which containers are running.


### Jenkins - Job service-start (1/5)

Creating and starting docker containers for our service and mongo database.

![logo](images/pipeline-start.png)


### Jenkins - Job service-start (2/5)
- Create new item
- Item name: `service-start`
- Type: `Copy service-build`
- Delivery pipeline: `Stage Name=QA, Task Name=start`
- Remove build step gradle.
- Add build step: Execute docker command
  - Docker command: pull image
  - Image name: `172.31.21.24:5000/mongo`
  - Tag: `3.0.6`
- Add build step: Execute docker command
  - Docker command: create container
  - Image name: `172.31.21.24:5000/mongo:3.0.6`
  - Container name: `test_db`


### Jenkins - Job service-start (3/5)
- Add build step: Execute docker command
  - Docker command: create container
  - Image name: `stickynote/stickynote-service:latest`
  - Container name: `test_service`
  - Advanced - Links: `test_db:mongodb`
- Add build step: Execute docker command
  - Docker command: start container(s)
  - Container ID(s): `test_db, test_service`
  - Advanced - Port bindings: `8888:8080`
  - Advanced - Wait for ports:
  ```
  test_db 27017
  test_service 8080
  ```


### Jenkins - Job service-start (4/5)
- Change post build action: Trigger parameterized build on other projects
  - projects to build: `service-stop`
  - Don't mind the error about non-existing project, we'll fix that later.
- Save


### Jenkins - Job service-start (5/5)  
- Configure job `service-package` to add a trigger.
    - Add post build action: Trigger parameterized build on other projects
      - projects to build: `service-start`
      - Add parameters: current build parameters
- Save


### Jenkins - Job service-stop (1/3)

Stopping and removing the earlier created containers.

![logo](images/pipeline-stop.png)


### Jenkins - Job service-stop (2/3)
- Create new item
- Item name: `service-stop`
- Type: `Copy service-start`
- Delivery pipeline: `Stage Name=QA, Task Name=stop`
- Remove all existing build steps.
- Remove Trigger from Post-build Actions.


### Jenkins - Job service-stop (3/3)
- Add build step: Execute docker command
  - Docker command: Stop container(s)
  - Container ID(s): `test_service, test_db`
- Add build step: Execute docker command
  - Docker command: Remove container(s)
  - Container ID(s): `test_service, test_db`
  - Advanced - Ignore if not found to TRUE
  - Advanced - Remove volumes to TRUE
- Save and test by starting the main project.


### Jenkins - Job service-tests (1/3)

Perform tests against our running container.

![logo](images/pipeline-tst.png)


### Jenkins - Job service-tests (2/3)
- Create new item
- Item name: `service-tests`
- Type: `Copy service-build`
- Delivery pipeline: `Stage Name=QA, Task Name=tests`
- Change build step: Invoke Gradle script
  - Use gradle wrapper
  - Tasks (multiline!)
    - line 1: `integrationTest`
    - line 2: `jmeterRun`
- Change post build action: Trigger parameterized build on other projects
  - projects to build: `service-stop`
  - Add parameters: current build parameters
- Save


### Jenkins - Job service-tests (3/3)
- Configure job `service-start` to change the post build trigger.
  - Change post build action: Trigger parameterized build on other projects
    - projects to build: `service-tests`
    - Add parameters: current build parameters
- Save and test by starting the main project.


### Jenkins - Job service-deploy (1/5)

Deploy our service.

![logo](images/pipeline-deploy.png)


### Jenkins - Job service-deploy (2/5)
- Create new item
- Item name: `service-deploy`
- Type: `Copy service-start`
- Delivery pipeline: `Stage Name=Release, Task Name=deploy`
- Remove all build steps (4).
- Remove post build action: Trigger parameterized build on other projects
- Add build step: Execute docker command
  - Docker command: Tag image
  - Name of image: `stickynote/stickynote-service:latest`
  - Target repository of the new tag: `stickynote/stickynote-service`
  - The tag to set: `${VERSION}`


### Jenkins - Job service-deploy (3/5)
- Add build step: Execute docker command
  - Docker command: Remove container(s)
  - Container ID(s): `demo_db, demo_service`
  - Advanced - Ignore if not found to TRUE
  - Advanced - Remove volumes to TRUE
  - Advanced - Force remove to TRUE
- Add build step: Execute docker command
  - Docker command: create container
  - Image name: `172.31.21.24:5000/mongo:3.0.6`
  - Container name: `demo_db`


### Jenkins - Job service-deploy (4/5)
- Add build step: Execute docker command
  - Docker command: create container
  - Image name: `stickynote/stickynote-service:${VERSION}`
  - Container name: `demo_service`
  - Advanced - Links: `demo_db:mongodb`
- Add build step: Execute docker command
  - Docker command: start container(s)
  - Container ID(s): `demo_db, demo_service`
  - Advanced - Port bindings: `8887:8080`
  - Advanced - Wait for ports:
  ```
  demo_db 27017
  demo_service 8080
  ```
- Save


### Jenkins - Job service-deploy (5/5)
- Configure `service-stop` job
- Add post build action: Trigger parameterized build on other projects
  - projects to build: `service-deploy`
  - Add parameters: current build parameters
- Save and test by starting the main project.


## Delivery pipeline view
- Click on the (+) to add an extra tab.
- Choose Delivery Pipeline view
- Pick a name for your view
- Components - Add
- Choose a Name
- Initial Job `service-main`
- Click OK to finish


### Jenkins - Sonar (1/4)

Adding Sonar to keep track of our quality.

![logo](images/pipeline-sonar.png)


### Jenkins - Sonar (2/4)
Optional step, if you have less then 20 minutes left, continue with Dessert!
- Create new item
- Item name: `service-sonar`
- Type: `Copy service-build`
- Delivery pipeline: `Stage Name=QA, Task Name=sonar`
- Change build step: Invoke Gradle script
  - Use gradle wrapper
  - Tasks: sonarRunner
- Save


### Docker - Sonar
First we stop all of our docker services, and add the Sonar docker images.
- Stop the docker services using docker-compose (in atos-workshop/continuous)
```
docker-compose stop
```


### Docker - Sonar
- Edit the docker-compose.yml file.
- Add a container for the database:

```
sonardb:
  image: 172.31.21.24:5000/postgres:9
  environment:
    - POSTGRES_USER=sonar
    - POSTGRES_PASSWORD=secret
  volumes:
    - ./.data/sonardb/data:/var/lib/postgresql/data
```


### Docker - Sonar
- Add a container for sonar:

```
sonar:
  image: 172.31.21.24:5000/sonarqube:5.1.1
  links:
    - sonardb:db
  environment:
    - SONARQUBE_JDBC_USERNAME=sonar
    - SONARQUBE_JDBC_PASSWORD=secret
    - SONARQUBE_JDBC_URL=jdbc:postgresql://db/sonar
  ports:
    - "9000:9000"

```


### Docker - Sonar
- And link the two sonar containers to the jenkins container
- In the `links:` section of the jenkins configuration add links from jenkins to sonar (see below).
- Save the docker-compose.yml file

```
jenkins:
  ...
  links:
    - git:git
    - sonar:sonar
    - sonardb:sonardb
  ...
```


### Docker - Sonar
- Restart compose

```
docker-compose up -d
```


### Jenkins - Sonar (3/4)
- Configure `service-sonar` job
- Edit post build action: Trigger parameterized build on other projects
  - projects to build: `service-deploy`
- Save


### Jenkins - Job Sonar (4/4)
- Configure job `service-stop`
  - Edit post build action: Trigger parameterized build on other projects
    - projects to build: `service-sonar`
- Save and test by starting the `service-main` job.
- Check sonar scores on http://< ip-aws-instance >:9000/
