### Setting up GitLab - docker-compose
- The containers that we need for our continuous delivery environment will be managed by docker compose. Later we will add more.
- You will find a docker-compose file in <root>/continuous. For now this file only contains one container definition for GitLab, our Git server.

```
git:
  image: gitlab/gitlab-ce:8.0.4-ce.1
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
cd <root>/continuous
docker-compose up -d

# to see the log
docker-compose logs -f
```


### Setting up GitLab - Configure
- Start a webbrowser and browse to: `http://<ip-aws-instance/`
- Login with the default user and password.
  - username: root
  - password: 5iveL!fe
- Next you have to set a new password. After setting the password login again.
- Go to create new project, fill in the form as follows:
  - Project path: jfall
  - Import project from git any repo url: `https://bitbucket.org/jcz-jfall/stickynote-service.git`
  - Visibility Level: Public
- Create project, now you have your own Git server running.
