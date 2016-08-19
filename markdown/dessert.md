# Dessert
*Inception*


## Docker in Docker
Now we have everything running in docker we go one step further. Now we will run the build itself in a docker container.


## Docker in Docker
- We run a build in a docker container from Jenkins, remember Jenkins is already running in docker.
- We mount the jenkins data dir to the image, therefore we add an environment variable.
- We use the script `run-containerized` to build an image and run the build.
- The Dockerfile for the build container is part of the service sources. See `ci/docker/Dockerfile` in GitLab (remember?)


## Jenkins docker image
- The first step is to create a new jenkins image that contains:
  - The Docker CLI to run docker commands.
  - Add a shell script to run processes in a container

```
cd $HOME/atos-workshop/continuous
cat jenkins-dessert/Dockerfile
<editor> docker-compose.yml
```


## Jenkins docker image
- Change the jenkins build dir to jenkins-dessert
- Add a section `environment:` to the jenkins container definition.
- Add a DOCKER_HOST variable to the `environment:` section.
- Add a VOLUME_MOUNT_POINT variable to the `environment:` section.

```
jenkins:
  build: ./jenkins-dessert
  volumes:
    - ./.data/jenkins:/var/jenkins_home
  ports:
    - "8080:8080"
  extra_hosts:
    dockerhost: 172.17.0.1
    mavenrepository: 172.31.21.24
  links:
    - git:git
    - sonar:sonar
    - sonardb:sonardb
  environment:
    - DOCKER_HOST=tcp://dockerhost:2375
    - VOLUME_MOUNT_POINT=/home/ubuntu/atos-workshop/continuous/.data/jenkins
```


## Jenkins docker image
- Now we will build the updated jenkins image
- And re create the jenkins container

```
docker-compose build
docker-compose up -d
```


## Create a build Job in Jenkins
- Create new item
  - name: `service-docker-in-docker`
  - copy from existing: `service-main`
- Configure build
  - uncheck "Delivery Pipeline configuration"
  - remove any post build actions
  - add build step "Execute shell" with below statements
  - Save and start this build (service-docker-in-docker) afterwards.

```
  export DOCKER_OPTS="${DOCKER_OPTS} --add-host dockerhost:172.17.0.1"
  export DOCKER_OPTS="${DOCKER_OPTS} --add-host mavenrepository:172.31.21.24"
  export DOCKER_OPTS="${DOCKER_OPTS} -e GRADLE_USER_HOME=/workspace"  
  run-containerized -c "chmod +x gradlew; ./gradlew build buildImage"
```
