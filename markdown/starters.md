# Starters
*Getting prepared*


![logo](images/aws-logo.png)


### AWS
We are using AWS as infrastructure for running our Continuous Delivery services.


### AWS
Below some background information about Amazon Web Services:
- We have automated the creation of our AWS instances with the following tools:
  - Ansible in combination with ec2 module to use AWS EC2 api.
  - EC2 external inventory scripts to use dynamic inventory.
  - we have created ansible-playbook scripts that create the instances and provision the created instances with the necessary software for this handson lab.

- To improve performance for downloading artifacts we also provisioned an AWS server with Artifactory.
With Artifactory remote artifacts e.g. docker containers are cached locally so that you don't have to download them over and over again.


### Linux/Mac Users: How to login to an AWS instance
- Windows users: skip this slide
- We are using a key file (.pem) to access the AWS instances.  
'Save link as' [here](key/JFALL_HANDSON.pem) to download the key and save it locally.
- Open an terminal
- Your key must not be publicly viewable for SSH to work.

```
chmod 400 <path>/JFALL_HANDSON.pem
```

- To prevent ssh timeout add ServerAliveInterval=120 as option  
to ssh command
- Connect via ssh

```
 ssh -o ServerAliveInterval=120 -i <path>/JFALL_HANDSON.pem ubuntu@<ip-aws-instance>
```


### Windows Users: How to login to an AWS instance

- We are using a pem file (.ppk) to access the AWS instances. Click [here](key/JFALL_HANDSON.ppk) to download the key and save it locally.
- Download [putty](http://www.chiark.greenend.org.uk/~sgtatham/putty/download.html)
- Start putty and enter the host ip
- Select `data` on the left hand side under `auto-login username` enter the user `ubuntu`
- Select `SSH`->`Auth` on the left hand side then under `private key for authentication` select the .ppk file we just downloaded.
- To prevent ssh timeout in putty select `connection`. Under "Sending of null packets to keep session active - Seconds between keepalives (0 to turn off)", enter `120` in the text box.
- Go back to the top and connect to your instance.


![logo](images/docker-logo-med.png)


### Docker
We are using Docker for deploying service to our infrastructure. The next slides explain briefly the docker concepts you need to understand today.


### Docker command examples
*Below some examples of docker image commands*

```
Commands:
    images    List images
    inspect   Return low-level information on a container or image
    ps        List containers
    pull      Pull an image or a repository from a Docker registry
    run       Run a command in a new container

Run 'docker help' for all commands.
Run 'docker COMMAND --help' for more information on a command.
```


### Running a web server using docker
- In the next slide you will start a docker container that runs the nginx static content server.
- The server is running on port 80 in the container, you can find the exposed port using `docker inspect`. This port is mapped to port 8080 on the docker engine host, your AWS instance.


### Running a web server using docker
`DIY`: In the terminal session on your dedicated AWS instance
```
# First pull the nginx:
docker pull nginx

# Next inspect your local images
docker images nginx

# Inspect the nginx image
docker inspect nginx

# Create and start a container
docker run -d -p 8080:80 --name mywebserver nginx

# Inspect the running docker processes
docker ps
```
- Try to access your webserver using your browser.


### Running a web server using docker
*Below some examples of docker container commands*
```
Commands:
    logs      Fetch the logs of a container
    rm        Remove one or more containers
    start     Start a stopped container
    stop      Stop a running container

Run 'docker help' for all commands.
Run 'docker COMMAND --help' for more information on a command.
```


### Running a web server using docker
`DIY`: In the terminal session on your dedicated AWS instance
```
# Stop the container
docker stop mywebserver

# Inspect the processes
docker ps
docker ps -a

# Start the container
docker start mywebserver

# Inspect the logging
docker logs mywebserver

# Stop and remove the container
docker stop mywebserver && docker rm -v mywebserver
```


### Building a web server with custom content

Next we build a Docker image ourselves. It is based on the nginx image and a very simple index.html as static content for our server and a Dockerfile, the build file for docker.  

```
# Navigate to atos-workshop/starter-docker/nginx directory
# inspect the files
cat in-container/index.html
cat Dockerfile

# Build your own docker image
docker build -t starter/docker .

# Check that your new image is in the list
docker images
```


### Building a web server with custom content

```
# Start your new container in the foreground.
docker run -p 8080:80 --rm starter/docker

# Verify the image is working using curl (Linux)
curl -X GET <ip-aws-instance>:8080/index.html

# or a browser on your local host (Linux/Windows)
http://<ip-aws-instance>:8080/index.html

# Once done hit ctrl-c to stop
# in the terminal
```


### Running a web server with custom content
In the previous example we added our static content to the image with a docker build. Alternatively we can add the content via a mount. In the next example we mount the local file system to a mount point in the container.

```
# Also run in the starter-docker/nginx directory.  
# The variable ${PWD} refers to the current working dir.
# Hint, try refreshing your browser after this.  

docker run --rm -p 8080:80 -v ${PWD}/via-mount:/usr/share/nginx/html nginx

# Once done hit ctrl-c to stop
# in the terminal
```


### Container linking
- Docker has a linking system that allows you to link multiple containers together and send connection information from one to another.
- When containers are linked, information about a source container can be sent to a recipient container.
- To establish links, Docker relies on the names of your containers.

An example(!), first we create a container for our database.
```
docker run -d --name postgres <image> <command>
```
Secondly we link our database to our web container
```
docker run -d --link postgres:db --name web <image> <command>
```


### Docker Compose
Compose is a tool for defining and running complex applications with Docker. With Compose, you define a multi-container application in a single file, then spin up your application using a single command. Everything needed is started automatically.


### Docker Compose
- With Compose we create two containers that are linked.
- The first container contains a python application that serves a simple web page with a counter. This container will be build by Compose.
- The second container contains a Redis key value store, to store the counter.

```
# Navigate to the starter-docker/compose directory.  
cat docker-compose.yml

# start and build the containers.
docker-compose up

# Inspect by curl or a browser the web app is working.
# Hint port is not 8080
# Use ctrl-c to stop the container
```


### Clean up
You can remove docker containers by executing the command `docker rm`  
The switch `-v` will remove all implicit mounted volumes and the switch `-f` will remove running containers as well.  
Using the command for the `-f` switch, `$(docker ps -q -a)` will list ids of all docker containers.

```
# List all docker containers
docker ps -a

# List ids only of the docker containers
docker ps -q -a

# Remove docker containers using their ids
docker rm -v -f $(docker ps -q -a)
```


![logo](images/gradle-logo.png)


### Gradle
*is a build automation tool that builds upon the concepts of Apache Ant and Apache Maven and introduces a Groovy-based domain-specific language (DSL) instead of the more traditional XML form of declaring the project configuration. (Wikipedia)*


### Gradle basic commands

- We have made a simple gradle project available on your AWS instance in `/home/ubuntu/atos-workshop/starter-gradle`.
- Since we do not have Oracle Java 8 installed on our VM and it is more fun to use docker. We use use Docker to run the build. The docker command is wrapped in the command `run-in-java8-container`.
- First test our wrapper by executing.
```
# Test the wrapper command
run-in-java8-container java -version
```
- The `run-in-java8-container` command mounts the current directory to the container. So all data (service sources) are available in the container.


### Gradle basic command
- Instead of using the gradle command itself we use the wrapper. The wrapper will manage the gradle version by downloading the right version of gradle. Run the command below and observe the output. You can inspect the `build.gradle` to find out more.
```
# Run a gradle clean and build
cd ~/atos-workshop/starter-gradle/hello-world
run-in-java8-container ./gradlew clean build
```
- Next we execute the program
```
run-in-java8-container ./gradlew run
```
- Show all available tasks
```
run-in-java8-container ./gradlew tasks
```
