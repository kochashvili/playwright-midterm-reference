# Docker on AWS exam guide

This guide covers Quiz 1 on Ubuntu and Quiz 2 on RHEL, from launching EC2 through collecting screenshots. Commands labeled **Mac** run in your local terminal. Commands labeled **EC2** run inside the SSH session. Each short command is in its own block. Paste each Dockerfile heredoc from `cat` through `EOF` in one go. Full examples repeat the individual steps, so choose one format for each exercise.

## Contents

1. [Exam checklist and app map](#exam-checklist-and-app-map)
2. [Launch and connect to EC2](#launch-and-connect-to-ec2)
3. [Install Docker, Git, and AWS CLI](#install-docker-git-and-aws-cli)
4. [Sign in to AWS CLI and Docker on EC2](#sign-in-to-aws-cli-and-docker-on-ec2)
5. [Quiz 1 and Quiz 2 application walkthroughs](#application-walkthroughs)
6. [Reuse the GitHub and ECR repositories](#reuse-the-github-and-ecr-repositories)
7. [Troubleshooting and command reference](#troubleshooting-and-command-reference)
8. [Submission and instance state](#submission-and-instance-state)
9. [Theory questions](#theory-questions)

## Exam checklist and app map

1. Read the prompt for the OS, instance name, host port, allowed access, and repository/tag.
2. Connect by SSH. Install and test Docker. For ECR, install AWS CLI, set up lab credentials, and authenticate Docker to ECR.
3. Clone the source, create the Dockerfile, build, run, and check the app in a browser.
4. For Quiz 2, create or locate the ECR repository, tag the image, push, and verify the tag in the AWS Console.
5. Save the Dockerfile and take the screenshots specified by the exam.

| App | GitHub source under `Ttsartsidze` | Local image | Host → container | Browser path | Quiz 2 ECR repository and tag |
| --- | --- | --- | --- | --- | --- |
| 1 · Flask | `Python-Hello-World` | `python-hello-world:latest` | 8081 → 8080 | `/` | `python-hello-world:latest` |
| 2 · Calculator | `Python-Calculator` | `python-calculator:latest` | 8082 → 5000 | `/` | `angular:python-calculator` |
| 3 · Node.js | `NodeJS-web-app` | `nodejs-web-app:latest` | 8080 → 8080 | `/` | `angular:nodejs-web-app` |
| 4 · Java Demo | `Java-Hello-World2` | `java-demo:latest` | 8088 → 8080 | `/` | `java-demo:latest` |
| 5 · Java Hello World | `Java-Hello-World` | `java-hello-world:latest` | 8081 → 8080 | `/demo/hello` | `java-hello-world:latest` |

Quiz 1 uses App 1 and ends after the browser test and submission evidence. Quiz 2 uses all five apps. The Calculator and Node.js images share the same ECR repository with different tags. Flask and Java Hello World share host port `8081` on the RHEL instance, so we stopped Flask before launching App 5.

## Launch and connect to EC2

| Setting | Quiz 1 | Quiz 2 |
| --- | --- | --- |
| Instance name | `Lesson1` | `Docker-Rhel` |
| OS in worksheet | Ubuntu 22.04 LTS | RHEL 9 x86_64 |
| Instance type | `t2.small` | `t2.small` |
| SSH user | `ubuntu` | `ec2-user` |
| Practice notes | Ubuntu 26.04 was used after the visible SQL AMI hit a lab Marketplace permission error. Choose the plain 22.04 AMI when the exam requires it. | We used 20 GiB and a RHEL 9.8 AMI. The package upgrade advanced it to 9.9. |

In Learner Lab, choose **Start Lab**, open the **AWS Console**, and use the region assigned for the exam. We used **US East (N. Virginia)**, `us-east-1`. Go to **EC2 → Instances → Launch instances**. Select an x86_64 AMI, the required instance type, and the key pair matching the private key you downloaded from the lab. Our key pair was `vockey` and our local key file was `labsuser.pem`. Use a public subnet with a public IPv4 address for the SSH setup shown here. Allow inbound **SSH / TCP 22** from **My IP**. After launch, copy **Public IPv4 DNS** from the instance details.

**Mac.** Set the key permission once.

```bash
chmod 400 /Users/Jaba/Desktop/Docker/labsuser.pem
```

**Mac.** Save the hostnames in your current terminal tab. Replace the example values with the current Public IPv4 DNS from EC2. Repeat the assignment in each new tab where you use the variable.

```bash
UBUNTU_HOST='replace-with-lesson1-public-dns'
```

```bash
RHEL_HOST='replace-with-docker-rhel-public-dns'
```

**Mac.** Connect to the instance for the quiz you are doing.

```bash
ssh -i /Users/Jaba/Desktop/Docker/labsuser.pem "ubuntu@$UBUNTU_HOST"
```

```bash
ssh -i /Users/Jaba/Desktop/Docker/labsuser.pem "ec2-user@$RHEL_HOST"
```

For the first connection, check the host fingerprint through a trusted source before accepting it. The public IP and DNS can change after an EC2 stop/start. Refresh them in the console if an old command times out. [AWS EC2 launch and connect](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html) · [AWS stop/start behavior](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/Stop_Start.html)

### Browser access and port mapping

- **Public browser test.** Run with `-p HOST_PORT:CONTAINER_PORT`. In EC2, select the instance, open its **Security** tab, click its security group, and choose **Edit inbound rules → Add rule → Custom TCP**. Enter the **host** port from the app map. The worksheets specify `0.0.0.0/0` for the app ports. Use **My IP** when the task permits it. Then open `http://PUBLIC_DNS:HOST_PORT/PATH` in the browser. Use the current instance DNS or IPv4 address.
- **SSH tunnel.** We used this for Apps 2–5. The EC2 command publishes the port on `127.0.0.1`. The Mac SSH command forwards a local port over TCP 22. Leave the tunnel tab open and browse to the corresponding local URL. Use the public browser method if the exam requires public DNS evidence.
- `EXPOSE` describes the container's port. `docker run -p` publishes it. The source app must listen on the container's network interface, usually `0.0.0.0`. An inbound security group rule alone cannot publish a container port. [AWS security group rules](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/changing-security-group.html) · [Docker port publishing](https://docs.docker.com/engine/network/port-publishing/)

## Install Docker, Git, and AWS CLI

### Quiz 1 · Ubuntu

**EC2 (Ubuntu).** The worksheet's legacy installation commands download and run the Rancher Docker 20.10 script. Use this route only if the exam requires that exact installer and the selected Ubuntu image supports it.

```bash
wget https://releases.rancher.com/install-docker/20.10.sh
```

```bash
sh 20.10.sh
```

For a practice setup using Ubuntu's packages, use this sequence instead of the legacy installer.

```bash
sudo apt update
```

```bash
sudo apt install -y docker.io git curl
```

Use `sudo docker` in the commands below. The worksheets show `chmod 777 /var/run/docker.sock`, which gives every local user access to the Docker daemon. Our RHEL practice used `sudo` to retain the socket permissions. [Docker daemon access](https://docs.docker.com/engine/install/linux-postinstall/)

### Quiz 2 · RHEL 9 x86_64

**EC2 (RHEL).** This is the Docker CE setup we used.

```bash
cat /etc/redhat-release
```

```bash
uname -m
```

```bash
sudo dnf upgrade -y
```

```bash
sudo dnf install -y dnf-plugins-core
```

```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
```

```bash
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

The worksheet also shows a manual repo file. If the exam requires that method on RHEL 9, use `sudo vi /etc/yum.repos.d/docker.repo` and enter the block below. Use one repo setup method.

```ini
[docker-ce-stable]
name=Docker CE Stable - $basearch
baseurl=https://download.docker.com/linux/rhel/9/$basearch/stable
enabled=1
gpgcheck=1
gpgkey=https://download.docker.com/linux/rhel/gpg
```

The worksheet's install variation adds `--nobest`. In practice the official repo command worked. RHEL printed a subscription warning while its RHUI repositories still resolved and installed the packages. Check the final transaction status. [Docker Engine on RHEL](https://docs.docker.com/engine/install/rhel/)

### Start and verify Docker on either instance

**EC2.**

```bash
sudo systemctl enable --now docker
```

```bash
sudo systemctl status docker --no-pager
```

```bash
sudo docker run --rm hello-world
```

A successful test prints `Hello from Docker!`. `enable --now` starts Docker immediately and enables it at boot.

### AWS CLI v2 and Git on RHEL

**EC2.** Install `unzip` first. Our first extraction attempt failed because the instance lacked that command.

```bash
sudo dnf install -y unzip git
```

```bash
cd ~
```

```bash
curl -fL https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip -o awscliv2.zip
```

```bash
unzip -q awscliv2.zip
```

```bash
sudo ./aws/install
```

```bash
aws --version
```

The download above matches our `x86_64` instance. Select the ARM installer only if `uname -m` reports `aarch64`. [AWS CLI v2 installation and installer verification](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)

### Full fresh RHEL setup example

**EC2.** Paste into a fresh RHEL 9 x86_64 instance. The subshell stops when a command fails and leaves your interactive shell open.

```bash
(
  set -e
  sudo dnf upgrade -y
  sudo dnf install -y dnf-plugins-core git unzip
  sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
  sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
  sudo systemctl enable --now docker
  sudo docker run --rm hello-world
  cd ~
  curl -fL https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip -o awscliv2.zip
  unzip -q awscliv2.zip
  sudo ./aws/install
  aws --version
)
```

## Sign in to AWS CLI and Docker on EC2

We used the **temporary Learner Lab credentials** on the RHEL instance. The SSH `.pem` works for logging into EC2. The AWS CLI uses the lab access key, secret key, and session token. Docker then obtains its own ECR login token through the AWS CLI.

1. In AWS Academy Learner Lab, start or resume the lab. Open **AWS Details → AWS CLI → Show**.
2. On EC2, create the credentials directory and open the credentials file as `ec2-user`.

```bash
mkdir -p -m 700 ~/.aws
```

```bash
vi ~/.aws/credentials
```

3. In `vi`, press `i` and paste the full block from the lab, including `[default]` and all three fields. When refreshing an existing default profile, replace the three values together. Press **Esc**, type `:wq`, and press **Enter**. The format below uses dummy values for reference.

```ini
[default]
aws_access_key_id=REPLACE_WITH_CURRENT_LAB_ACCESS_KEY
aws_secret_access_key=REPLACE_WITH_CURRENT_LAB_SECRET_KEY
aws_session_token=REPLACE_WITH_CURRENT_LAB_SESSION_TOKEN
```

4. Restrict the credentials file and verify the active identity. Keep credentials out of screenshots, shell history, Docker build contexts, and submitted files.

```bash
chmod 600 ~/.aws/credentials
```

```bash
aws sts get-caller-identity --region us-east-1 --no-cli-pager
```

Check `Account` and the role in `Arn`. We passed `--region us-east-1` explicitly. Keep these variable assignments in the same EC2 shell for the examples. Use the region shown in the exam.

```bash
AWS_REGION=us-east-1
```

```bash
AWS_ACCOUNT_ID="$(aws sts get-caller-identity --region "$AWS_REGION" --query Account --output text --no-cli-pager)"
```

```bash
ECR_REGISTRY="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"
```

```bash
printf '%s\n' "$ECR_REGISTRY"
```

5. Log Docker into ECR. In our setup `aws` ran as `ec2-user` and Docker ran through `sudo`. Keep `sudo docker` consistent for login, push, and pull so Docker reads the same login configuration.

```bash
aws ecr get-login-password --region "$AWS_REGION" | sudo docker login --username AWS --password-stdin "$ECR_REGISTRY"
```

Look for `Login Succeeded`. If STS returns `ExpiredToken`, refresh the three lab fields, verify STS again, and log Docker in again. If STS works but Docker reports an expired authorization token, repeat the Docker login. ECR authorization tokens are valid for up to 12 hours. [AWS CLI credentials files](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html) · [ECR registry authentication](https://docs.aws.amazon.com/AmazonECR/latest/userguide/registry_auth.html)

## Application walkthroughs

For individual steps, run commands from top to bottom. Each **full example** is for a fresh checkout with the corresponding container name free. The examples assume Docker is running and `ECR_REGISTRY` and `AWS_REGION` are set in the current EC2 shell for Quiz 2. Save or edit the existing Dockerfile intentionally if repeating an exercise.

### App 1 · Flask

The page should show the Flask greeting. We disabled Flask debug mode before publishing the app on EC2.

#### Individual commands

**EC2.** Clone and enter the source checkout. If it already exists, go directly to the second command.

```bash
git clone https://github.com/Ttsartsidze/Python-Hello-World.git ~/Python-Hello-World
```

```bash
cd ~/Python-Hello-World
```

**EC2.** Disable Flask debug mode as we did for the RHEL build.

```bash
sed -i 's/debug=True/debug=False/' app.py
```

**EC2.** Write the Dockerfile. Paste this entire block at once.

```bash
cat > Dockerfile <<'EOF'
FROM python:3-alpine
WORKDIR /service
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
EXPOSE 8080
CMD ["python3", "app.py"]
EOF
```

**EC2.** Build and start the container.

```bash
sudo docker build -t python-hello-world:latest .
```

```bash
sudo docker run -d --name python-hello-world -p 8081:8080 python-hello-world:latest
```

**EC2.** Verify the running app, its logs, and its port mapping.

```bash
curl -fsS --retry 12 --retry-connrefused --retry-delay 2 http://127.0.0.1:8081/
```

```bash
sudo docker logs --tail 30 python-hello-world
```

```bash
sudo docker port python-hello-world
```

**Browser.** Open `http://PUBLIC_DNS:8081/` using the DNS for the instance you just built on and the TCP 8081 inbound rule. **Take the browser screenshot** when the greeting is visible. For Quiz 1, the app walkthrough ends here. For Quiz 2, continue to ECR.

**EC2 (Quiz 2).** Create the private repository if it does not already exist. The worksheet describes using the AWS Console, where you can choose **ECR → Private repositories → Create repository** and enter `python-hello-world`. We used the equivalent CLI command below. If you see `RepositoryAlreadyExistsException`, continue with tagging.

```bash
aws ecr create-repository --repository-name python-hello-world --region "$AWS_REGION" --no-cli-pager
```

```bash
sudo docker tag python-hello-world:latest "$ECR_REGISTRY/python-hello-world:latest"
```

```bash
sudo docker push "$ECR_REGISTRY/python-hello-world:latest"
```

```bash
aws ecr describe-images --repository-name python-hello-world --image-ids imageTag=latest --region "$AWS_REGION" --query 'imageDetails[0].imageDigest' --output text --no-cli-pager
```

**AWS Console.** Open **ECR → Private repositories → python-hello-world → Images**. **Take the ECR screenshot** with `latest` visible. A successful push prints a digest. The console digest should match for the tagged row.

#### Full Quiz 2 example · EC2

Authenticate as shown above first. This example builds, starts, checks the HTTP response on EC2, and pushes the image. Use the browser and ECR instructions above for screenshots.

```bash
(
  set -euo pipefail
  git clone https://github.com/Ttsartsidze/Python-Hello-World.git ~/Python-Hello-World
  cd ~/Python-Hello-World
  sed -i 's/debug=True/debug=False/' app.py
cat > Dockerfile <<'EOF'
FROM python:3-alpine
WORKDIR /service
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
EXPOSE 8080
CMD ["python3", "app.py"]
EOF
  sudo docker build -t python-hello-world:latest .
  sudo docker run -d --name python-hello-world -p 8081:8080 python-hello-world:latest
  curl -fsS --retry 12 --retry-connrefused --retry-delay 2 http://127.0.0.1:8081/
  if ! aws ecr describe-repositories --repository-names python-hello-world --region "$AWS_REGION" --no-cli-pager >/dev/null 2>&1; then
    aws ecr create-repository --repository-name python-hello-world --region "$AWS_REGION" --no-cli-pager
  fi
  sudo docker tag python-hello-world:latest "$ECR_REGISTRY/python-hello-world:latest"
  sudo docker push "$ECR_REGISTRY/python-hello-world:latest"
  aws ecr describe-images --repository-name python-hello-world --image-ids imageTag=latest --region "$AWS_REGION" --query 'imageDetails[0].imageDigest' --output text --no-cli-pager
)
```

#### Full Quiz 1 example · EC2 Ubuntu

Docker and Git should already be installed. The debug-setting edit matches our Quiz 2 Flask build and keeps the public test out of debug mode.

```bash
(
  set -e
  git clone https://github.com/Ttsartsidze/Python-Hello-World.git ~/Python-Hello-World
  cd ~/Python-Hello-World
  sed -i 's/debug=True/debug=False/' app.py
cat > Dockerfile <<'EOF'
FROM python:3-alpine
WORKDIR /service
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
EXPOSE 8080
CMD ["python3", "app.py"]
EOF
  sudo docker build -t python-hello-world:latest .
  sudo docker run -d --name python-hello-world -p 8081:8080 python-hello-world:latest
  curl -fsS --retry 12 --retry-connrefused --retry-delay 2 http://127.0.0.1:8081/
  sudo docker ps -a
  sudo docker logs --tail 30 python-hello-world
  sudo docker port python-hello-world
)
```

Open the Ubuntu instance's public DNS on port 8081, capture the result, and follow the Quiz 1 submission steps below.

### App 2 · Python Calculator

Enter `(3+(4-1))*5` and verify `30`. The worksheet also lists `2 * x + 0.5 = 1` and `2*x + 1 = 2*(1-x)`, each with expected result `x = 0.25`. We verified the first expression in practice.

#### Individual commands

**EC2.** Clone and enter the source checkout. If it already exists, go directly to the second command.

```bash
git clone https://github.com/Ttsartsidze/Python-Calculator.git ~/Python-Calculator
```

```bash
cd ~/Python-Calculator
```

**EC2.** Write the Dockerfile. Paste this entire block at once.

```bash
cat > Dockerfile <<'EOF'
FROM python:2.7
WORKDIR /app
COPY app/requirements.txt .
RUN pip install -r requirements.txt
COPY app/ .
EXPOSE 5000
CMD ["python", "app.py"]
EOF
```

**EC2.** Build and start the container.

```bash
sudo docker build -t python-calculator:latest .
```

```bash
sudo docker run -d --name python-calculator -p 127.0.0.1:8082:5000 python-calculator:latest
```

**EC2.** Verify the running app, its logs, and its port mapping.

```bash
curl -fsS --retry 12 --retry-connrefused --retry-delay 2 http://127.0.0.1:8082/
```

```bash
sudo docker logs --tail 30 python-calculator
```

```bash
sudo docker port python-calculator
```

**Mac.** Open a new terminal tab with `RHEL_HOST` set and start the tunnel.

```bash
ssh -i /Users/Jaba/Desktop/Docker/labsuser.pem -o ExitOnForwardFailure=yes -N -L 127.0.0.1:8082:127.0.0.1:8082 "ec2-user@$RHEL_HOST"
```

**Browser.** Open [http://127.0.0.1:8082/](http://127.0.0.1:8082/). **Take the browser screenshot** when the expected result and URL are visible.

**EC2 (Quiz 2).** Create the private repository if it does not already exist. The worksheet describes using the AWS Console, where you can choose **ECR → Private repositories → Create repository** and enter `angular`. We used the equivalent CLI command below. If you see `RepositoryAlreadyExistsException`, continue with tagging.

```bash
aws ecr create-repository --repository-name angular --region "$AWS_REGION" --no-cli-pager
```

```bash
sudo docker tag python-calculator:latest "$ECR_REGISTRY/angular:python-calculator"
```

```bash
sudo docker push "$ECR_REGISTRY/angular:python-calculator"
```

```bash
aws ecr describe-images --repository-name angular --image-ids imageTag=python-calculator --region "$AWS_REGION" --query 'imageDetails[0].imageDigest' --output text --no-cli-pager
```

**AWS Console.** Open **ECR → Private repositories → angular → Images**. **Take the ECR screenshot** with `python-calculator` visible. A successful push prints a digest. The console digest should match for the tagged row.

#### Full Quiz 2 example · EC2

Authenticate as shown above first. This example builds, starts, checks the HTTP response on EC2, and pushes the image. Use the browser and ECR instructions above for screenshots.

```bash
(
  set -euo pipefail
  git clone https://github.com/Ttsartsidze/Python-Calculator.git ~/Python-Calculator
  cd ~/Python-Calculator
cat > Dockerfile <<'EOF'
FROM python:2.7
WORKDIR /app
COPY app/requirements.txt .
RUN pip install -r requirements.txt
COPY app/ .
EXPOSE 5000
CMD ["python", "app.py"]
EOF
  sudo docker build -t python-calculator:latest .
  sudo docker run -d --name python-calculator -p 127.0.0.1:8082:5000 python-calculator:latest
  curl -fsS --retry 12 --retry-connrefused --retry-delay 2 http://127.0.0.1:8082/
  if ! aws ecr describe-repositories --repository-names angular --region "$AWS_REGION" --no-cli-pager >/dev/null 2>&1; then
    aws ecr create-repository --repository-name angular --region "$AWS_REGION" --no-cli-pager
  fi
  sudo docker tag python-calculator:latest "$ECR_REGISTRY/angular:python-calculator"
  sudo docker push "$ECR_REGISTRY/angular:python-calculator"
  aws ecr describe-images --repository-name angular --image-ids imageTag=python-calculator --region "$AWS_REGION" --query 'imageDetails[0].imageDigest' --output text --no-cli-pager
)
```

### App 3 · Node.js web app

Expected browser response is `Hello world`. Reuse the `angular` repository from App 2 and use the Node.js tag.

#### Individual commands

**EC2.** Clone and enter the source checkout. If it already exists, go directly to the second command.

```bash
git clone https://github.com/Ttsartsidze/NodeJS-web-app.git ~/NodeJS-web-app
```

```bash
cd ~/NodeJS-web-app
```

**EC2.** Write the Dockerfile. Paste this entire block at once.

```bash
cat > Dockerfile <<'EOF'
FROM node:10
WORKDIR /usr/app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 8080
CMD ["node", "server.js"]
EOF
```

**EC2.** Build and start the container.

```bash
sudo docker build -t nodejs-web-app:latest .
```

```bash
sudo docker run -d --name nodejs-web-app -p 127.0.0.1:8080:8080 nodejs-web-app:latest
```

**EC2.** Verify the running app, its logs, and its port mapping.

```bash
curl -fsS --retry 12 --retry-connrefused --retry-delay 2 http://127.0.0.1:8080/
```

```bash
sudo docker logs --tail 30 nodejs-web-app
```

```bash
sudo docker port nodejs-web-app
```

**Mac.** Open a new terminal tab with `RHEL_HOST` set and start the tunnel.

```bash
ssh -i /Users/Jaba/Desktop/Docker/labsuser.pem -o ExitOnForwardFailure=yes -N -L 127.0.0.1:8080:127.0.0.1:8080 "ec2-user@$RHEL_HOST"
```

**Browser.** Open [http://127.0.0.1:8080/](http://127.0.0.1:8080/). **Take the browser screenshot** when the expected result and URL are visible.

**EC2 (Quiz 2).** Create the private repository if it does not already exist. The worksheet describes using the AWS Console, where you can choose **ECR → Private repositories → Create repository** and enter `angular`. We used the equivalent CLI command below. If you see `RepositoryAlreadyExistsException`, continue with tagging.

```bash
aws ecr create-repository --repository-name angular --region "$AWS_REGION" --no-cli-pager
```

```bash
sudo docker tag nodejs-web-app:latest "$ECR_REGISTRY/angular:nodejs-web-app"
```

```bash
sudo docker push "$ECR_REGISTRY/angular:nodejs-web-app"
```

```bash
aws ecr describe-images --repository-name angular --image-ids imageTag=nodejs-web-app --region "$AWS_REGION" --query 'imageDetails[0].imageDigest' --output text --no-cli-pager
```

**AWS Console.** Open **ECR → Private repositories → angular → Images**. **Take the ECR screenshot** with `nodejs-web-app` visible. A successful push prints a digest. The console digest should match for the tagged row.

#### Full Quiz 2 example · EC2

Authenticate as shown above first. This example builds, starts, checks the HTTP response on EC2, and pushes the image. Use the browser and ECR instructions above for screenshots.

```bash
(
  set -euo pipefail
  git clone https://github.com/Ttsartsidze/NodeJS-web-app.git ~/NodeJS-web-app
  cd ~/NodeJS-web-app
cat > Dockerfile <<'EOF'
FROM node:10
WORKDIR /usr/app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 8080
CMD ["node", "server.js"]
EOF
  sudo docker build -t nodejs-web-app:latest .
  sudo docker run -d --name nodejs-web-app -p 127.0.0.1:8080:8080 nodejs-web-app:latest
  curl -fsS --retry 12 --retry-connrefused --retry-delay 2 http://127.0.0.1:8080/
  if ! aws ecr describe-repositories --repository-names angular --region "$AWS_REGION" --no-cli-pager >/dev/null 2>&1; then
    aws ecr create-repository --repository-name angular --region "$AWS_REGION" --no-cli-pager
  fi
  sudo docker tag nodejs-web-app:latest "$ECR_REGISTRY/angular:nodejs-web-app"
  sudo docker push "$ECR_REGISTRY/angular:nodejs-web-app"
  aws ecr describe-images --repository-name angular --image-ids imageTag=nodejs-web-app --region "$AWS_REGION" --query 'imageDetails[0].imageDigest' --output text --no-cli-pager
)
```

### App 4 · Java Demo

Expected browser response is `Hello World`. The worksheet asks for `MAINTAINER`, so the build completed with a deprecation warning. For a new Dockerfile without that rubric requirement use `LABEL maintainer="Jaba"`.

#### Individual commands

**EC2.** Clone and enter the source checkout. If it already exists, go directly to the second command.

```bash
git clone https://github.com/Ttsartsidze/Java-Hello-World2.git ~/Java-Hello-World2
```

```bash
cd ~/Java-Hello-World2
```

**EC2.** Write the Dockerfile. Paste this entire block at once.

```bash
cat > Dockerfile <<'EOF'
FROM maven:3.8.5-openjdk-11 AS maven_build
WORKDIR /tmp/
COPY . .
RUN mvn clean package

FROM eclipse-temurin:11
MAINTAINER Jaba
COPY --from=maven_build /tmp/target/hello-world-0.1.0.jar /data/hello-world-0.1.0.jar
EXPOSE 8080
USER 1002:1002
CMD ["java", "-jar", "/data/hello-world-0.1.0.jar"]
EOF
```

**EC2.** Build and start the container.

```bash
sudo docker build -t java-demo:latest .
```

```bash
sudo docker run -d --name java-demo -p 127.0.0.1:8088:8080 java-demo:latest
```

**EC2.** Verify the running app, its logs, and its port mapping.

```bash
curl -fsS --retry 12 --retry-connrefused --retry-delay 2 http://127.0.0.1:8088/
```

```bash
sudo docker logs --tail 30 java-demo
```

```bash
sudo docker port java-demo
```

**Mac.** Open a new terminal tab with `RHEL_HOST` set and start the tunnel.

```bash
ssh -i /Users/Jaba/Desktop/Docker/labsuser.pem -o ExitOnForwardFailure=yes -N -L 127.0.0.1:8088:127.0.0.1:8088 "ec2-user@$RHEL_HOST"
```

**Browser.** Open [http://127.0.0.1:8088/](http://127.0.0.1:8088/). **Take the browser screenshot** when the expected result and URL are visible.

**EC2 (Quiz 2).** Create the private repository if it does not already exist. The worksheet describes using the AWS Console, where you can choose **ECR → Private repositories → Create repository** and enter `java-demo`. We used the equivalent CLI command below. If you see `RepositoryAlreadyExistsException`, continue with tagging.

```bash
aws ecr create-repository --repository-name java-demo --region "$AWS_REGION" --no-cli-pager
```

```bash
sudo docker tag java-demo:latest "$ECR_REGISTRY/java-demo:latest"
```

```bash
sudo docker push "$ECR_REGISTRY/java-demo:latest"
```

```bash
aws ecr describe-images --repository-name java-demo --image-ids imageTag=latest --region "$AWS_REGION" --query 'imageDetails[0].imageDigest' --output text --no-cli-pager
```

**AWS Console.** Open **ECR → Private repositories → java-demo → Images**. **Take the ECR screenshot** with `latest` visible. A successful push prints a digest. The console digest should match for the tagged row.

#### Full Quiz 2 example · EC2

Authenticate as shown above first. This example builds, starts, checks the HTTP response on EC2, and pushes the image. Use the browser and ECR instructions above for screenshots.

```bash
(
  set -euo pipefail
  git clone https://github.com/Ttsartsidze/Java-Hello-World2.git ~/Java-Hello-World2
  cd ~/Java-Hello-World2
cat > Dockerfile <<'EOF'
FROM maven:3.8.5-openjdk-11 AS maven_build
WORKDIR /tmp/
COPY . .
RUN mvn clean package

FROM eclipse-temurin:11
MAINTAINER Jaba
COPY --from=maven_build /tmp/target/hello-world-0.1.0.jar /data/hello-world-0.1.0.jar
EXPOSE 8080
USER 1002:1002
CMD ["java", "-jar", "/data/hello-world-0.1.0.jar"]
EOF
  sudo docker build -t java-demo:latest .
  sudo docker run -d --name java-demo -p 127.0.0.1:8088:8080 java-demo:latest
  curl -fsS --retry 12 --retry-connrefused --retry-delay 2 http://127.0.0.1:8088/
  if ! aws ecr describe-repositories --repository-names java-demo --region "$AWS_REGION" --no-cli-pager >/dev/null 2>&1; then
    aws ecr create-repository --repository-name java-demo --region "$AWS_REGION" --no-cli-pager
  fi
  sudo docker tag java-demo:latest "$ECR_REGISTRY/java-demo:latest"
  sudo docker push "$ECR_REGISTRY/java-demo:latest"
  aws ecr describe-images --repository-name java-demo --image-ids imageTag=latest --region "$AWS_REGION" --query 'imageDetails[0].imageDigest' --output text --no-cli-pager
)
```

### App 5 · Java Hello World

Expected browser response is `Hello World!`. The root `/` returns Spring Boot’s Whitelabel 404. Use `/demo/hello`. The endpoint also accepts `?name=Jaba`. [Controller source](https://github.com/Ttsartsidze/Java-Hello-World/blob/main/src/main/java/com/example/demo/controller/DemoController.java)

#### Individual commands

**EC2.** Clone and enter the source checkout. If it already exists, go directly to the second command.

```bash
git clone https://github.com/Ttsartsidze/Java-Hello-World.git ~/Java-Hello-World
```

```bash
cd ~/Java-Hello-World
```

**EC2.** Write the Dockerfile. Paste this entire block at once.

```bash
cat > Dockerfile <<'EOF'
FROM maven:3.8.3-openjdk-17 AS maven_build
WORKDIR /
COPY . .
RUN mvn clean package -Dmaven.test.skip=true

FROM maven:3.8.3-openjdk-17
COPY --from=maven_build /target/demo-0.0.1-SNAPSHOT.jar /app/demo-0.0.1-SNAPSHOT.jar
EXPOSE 8080
USER 1002:1002
ENTRYPOINT ["java", "-jar", "/app/demo-0.0.1-SNAPSHOT.jar"]
EOF
```

**EC2.** Build and start the container.

```bash
sudo docker build -t java-hello-world:latest .
```

If the Flask container is running on this RHEL instance, stop it first to free port 8081. We already had the Flask browser evidence when we did this.

```bash
sudo docker stop python-hello-world
```

```bash
sudo docker run -d --name java-hello-world -p 127.0.0.1:8081:8080 java-hello-world:latest
```

**EC2.** Verify the running app, its logs, and its port mapping.

```bash
curl -fsS --retry 12 --retry-connrefused --retry-delay 2 http://127.0.0.1:8081/demo/hello
```

```bash
sudo docker logs --tail 30 java-hello-world
```

```bash
sudo docker port java-hello-world
```

**Mac.** Open a new terminal tab with `RHEL_HOST` set and start the tunnel.

```bash
ssh -i /Users/Jaba/Desktop/Docker/labsuser.pem -o ExitOnForwardFailure=yes -N -L 127.0.0.1:8081:127.0.0.1:8081 "ec2-user@$RHEL_HOST"
```

**Browser.** Open [http://127.0.0.1:8081/demo/hello](http://127.0.0.1:8081/demo/hello). **Take the browser screenshot** when the expected result and URL are visible.

**EC2 (Quiz 2).** Create the private repository if it does not already exist. The worksheet describes using the AWS Console, where you can choose **ECR → Private repositories → Create repository** and enter `java-hello-world`. We used the equivalent CLI command below. If you see `RepositoryAlreadyExistsException`, continue with tagging.

```bash
aws ecr create-repository --repository-name java-hello-world --region "$AWS_REGION" --no-cli-pager
```

```bash
sudo docker tag java-hello-world:latest "$ECR_REGISTRY/java-hello-world:latest"
```

```bash
sudo docker push "$ECR_REGISTRY/java-hello-world:latest"
```

```bash
aws ecr describe-images --repository-name java-hello-world --image-ids imageTag=latest --region "$AWS_REGION" --query 'imageDetails[0].imageDigest' --output text --no-cli-pager
```

**AWS Console.** Open **ECR → Private repositories → java-hello-world → Images**. **Take the ECR screenshot** with `latest` visible. A successful push prints a digest. The console digest should match for the tagged row.

#### Full Quiz 2 example · EC2

Authenticate as shown above first. This example builds, starts, checks the HTTP response on EC2, and pushes the image. Use the browser and ECR instructions above for screenshots.

```bash
(
  set -euo pipefail
  git clone https://github.com/Ttsartsidze/Java-Hello-World.git ~/Java-Hello-World
  cd ~/Java-Hello-World
cat > Dockerfile <<'EOF'
FROM maven:3.8.3-openjdk-17 AS maven_build
WORKDIR /
COPY . .
RUN mvn clean package -Dmaven.test.skip=true

FROM maven:3.8.3-openjdk-17
COPY --from=maven_build /target/demo-0.0.1-SNAPSHOT.jar /app/demo-0.0.1-SNAPSHOT.jar
EXPOSE 8080
USER 1002:1002
ENTRYPOINT ["java", "-jar", "/app/demo-0.0.1-SNAPSHOT.jar"]
EOF
  sudo docker build -t java-hello-world:latest .
  if sudo docker ps --format '{{.Names}}' | grep -Fxq python-hello-world; then
    sudo docker stop python-hello-world
  fi
  sudo docker run -d --name java-hello-world -p 127.0.0.1:8081:8080 java-hello-world:latest
  curl -fsS --retry 12 --retry-connrefused --retry-delay 2 http://127.0.0.1:8081/demo/hello
  if ! aws ecr describe-repositories --repository-names java-hello-world --region "$AWS_REGION" --no-cli-pager >/dev/null 2>&1; then
    aws ecr create-repository --repository-name java-hello-world --region "$AWS_REGION" --no-cli-pager
  fi
  sudo docker tag java-hello-world:latest "$ECR_REGISTRY/java-hello-world:latest"
  sudo docker push "$ECR_REGISTRY/java-hello-world:latest"
  aws ecr describe-images --repository-name java-hello-world --image-ids imageTag=latest --region "$AWS_REGION" --query 'imageDetails[0].imageDigest' --output text --no-cli-pager
)
```

## Reuse the GitHub and ECR repositories

### When the exam asks you to build or change the app

Clone the GitHub repository to get the source. If the repo is already on EC2, `cd` to its folder and inspect it. Rebuild after editing the Dockerfile or application. A registry pull gives you the packaged image. The Git clone gives you the source and build context.

**EC2.** Example using App 5.

```bash
git clone https://github.com/Ttsartsidze/Java-Hello-World.git ~/Java-Hello-World
```

```bash
cd ~/Java-Hello-World
```

```bash
git status --short
```

```bash
ls -la
```

If the checkout lacks a Dockerfile, use the matching heredoc in the app walkthrough. The copy in this workspace can also be transferred to EC2.

**Mac.** Example transfer, after the source checkout exists on EC2 and `RHEL_HOST` is set.

```bash
scp -i /Users/Jaba/Desktop/Docker/labsuser.pem '/Users/Jaba/Desktop/Docker/Quiz 2/Dockerfiles/App 5 - Java Hello World/Dockerfile' "ec2-user@$RHEL_HOST:Java-Hello-World/Dockerfile"
```

If asked to demonstrate an updated image, build it and push a new tag when the rubric permits it. Use the existing repository if it belongs to the account and region for the exam. This example uses an exam tag to preserve the existing `latest` tag.

**EC2.** In the App 5 checkout after signing in to ECR.

```bash
sudo docker build -t java-hello-world:exam-v2 .
```

```bash
sudo docker tag java-hello-world:exam-v2 "$ECR_REGISTRY/java-hello-world:exam-v2"
```

```bash
sudo docker push "$ECR_REGISTRY/java-hello-world:exam-v2"
```

### When the exam asks you to pull and run a previously built image

ECR is the private **image registry**. You can pull a built image onto a fresh Docker host without cloning the source or rebuilding it. Make sure the lab account and region are correct. If the exam starts a new account or the prior lab resources have been reset, list the current repositories and rebuild and push from GitHub if the images are absent. Accessing a different account's private ECR requires authorization to that registry. Our images were built on Linux x86_64. Use a matching EC2 host for the easiest reuse. [AWS private ECR pull guide](https://docs.aws.amazon.com/AmazonECR/latest/userguide/docker-pull-ecr-image.html)

**EC2.** After the AWS CLI sign-in and Docker ECR login, list the current repositories.

```bash
aws ecr describe-repositories --region "$AWS_REGION" --query 'repositories[].repositoryName' --output table --no-cli-pager
```

List named tags in the shared `angular` repo.

```bash
aws ecr list-images --repository-name angular --filter tagStatus=TAGGED --region "$AWS_REGION" --no-cli-pager
```

List the digest for a specific tag.

```bash
aws ecr describe-images --repository-name angular --image-ids imageTag=python-calculator --region "$AWS_REGION" --query 'imageDetails[0].imageDigest' --output text --no-cli-pager
```

**EC2.** Individual pull commands. Run the ones you need.

```bash
sudo docker pull "$ECR_REGISTRY/python-hello-world:latest"
```

```bash
sudo docker pull "$ECR_REGISTRY/angular:python-calculator"
```

```bash
sudo docker pull "$ECR_REGISTRY/angular:nodejs-web-app"
```

```bash
sudo docker pull "$ECR_REGISTRY/java-demo:latest"
```

```bash
sudo docker pull "$ECR_REGISTRY/java-hello-world:latest"
```

Run a pulled image by replacing the local image at the end of the matching `docker run` command with its full ECR image URI. Choose a free host port and a free container name. The port on the right remains the port the app listens on inside the container.

### Full example · pull Calculator onto a fresh EC2 host

This assumes AWS CLI and Docker are installed and the current Learner Lab profile is configured. The block finds the current account, logs Docker in to ECR, checks the tag, pulls, runs, and verifies HTTP on EC2.

```bash
(
  set -euo pipefail
  AWS_REGION=us-east-1
  AWS_ACCOUNT_ID="$(aws sts get-caller-identity --region "$AWS_REGION" --query Account --output text --no-cli-pager)"
  ECR_REGISTRY="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"
  IMAGE_URI="$ECR_REGISTRY/angular:python-calculator"

  aws ecr get-login-password --region "$AWS_REGION" | sudo docker login --username AWS --password-stdin "$ECR_REGISTRY"
  aws ecr describe-images --repository-name angular --image-ids imageTag=python-calculator --region "$AWS_REGION" --no-cli-pager
  sudo docker pull "$IMAGE_URI"
  sudo docker run -d --name exam-calculator -p 127.0.0.1:8082:5000 "$IMAGE_URI"
  curl -fsS --retry 12 --retry-connrefused --retry-delay 2 -o /dev/null http://127.0.0.1:8082/
  sudo docker port exam-calculator
)
```

**Mac.** Set `RHEL_HOST` to the host you pulled the image onto.

```bash
ssh -i /Users/Jaba/Desktop/Docker/labsuser.pem -o ExitOnForwardFailure=yes -N -L 127.0.0.1:8082:127.0.0.1:8082 "ec2-user@$RHEL_HOST"
```

Open [http://127.0.0.1:8082/](http://127.0.0.1:8082/) and check the calculator. If asked for a specific immutable image, pull using `registry/repository@sha256:…` with the full digest obtained from ECR. A tag such as `latest` can point to a different digest after another push. [AWS ECR push and tag guide](https://docs.aws.amazon.com/AmazonECR/latest/userguide/docker-push-ecr-image.html)

## Troubleshooting and command reference

**EC2.** List running containers, all containers, and local images.

```bash
sudo docker ps
```

```bash
sudo docker ps -a
```

```bash
sudo docker image ls
```

Inspect one container by name. Substitute the app's container name as needed.

```bash
sudo docker logs --tail 50 java-hello-world
```

```bash
sudo docker port java-hello-world
```

```bash
sudo docker inspect --format '{{.State.Status}} exit={{.State.ExitCode}} oom={{.State.OOMKilled}}' java-hello-world
```

Test the app directly on EC2 to distinguish a Docker issue from SSH or security-group routing.

```bash
curl -i http://127.0.0.1:8081/demo/hello
```

Check listening ports, free disk space, and memory.

```bash
sudo ss -ltnp
```

```bash
df -h
```

```bash
free -h
```

| Symptom | What to check |
| --- | --- |
| SSH times out | Confirm instance state, current public DNS, public address, and the security group's inbound TCP 22 source. |
| SSH reports `Permission denied (publickey)` | Check the login user, key file permissions, and matching EC2 key pair. |
| `Cannot connect to the Docker daemon` | Check `systemctl status docker` and start the service if needed. Use `sudo docker` with the lab setup. |
| `port is already allocated` | Check `docker ps` and `ss` on EC2. App 1 and App 5 both use host port 8081. For a Mac tunnel bind failure, close the conflicting local tunnel or choose a different Mac-side port. |
| `container name … is already in use` | Use `docker ps -a`. Start the existing container if you want its original image and port settings, or choose a new name. |
| Browser times out but EC2 curl succeeds | For public access, check the host port, Docker bind and the security group. For tunnel access, check the Mac SSH tab and browse to its local forwarded port. |
| Java 404 at `/` | App 5 serves `/demo/hello`. App 4 serves `/`. |
| Container exited | Use `docker logs` and the inspect command. If `oom=true`, check memory and consider stopping other lab containers before retrying. |
| ECR `ExpiredToken` or `no basic auth credentials` | Verify `aws sts get-caller-identity`, refresh the lab credentials if expired, then repeat the Docker ECR login with the same `sudo` usage. |
| ECR `AccessDenied` | Check current account, region, role, and lab permissions. A repository from a different lab account may need a rebuild in the current account. |
| ECR `manifest unknown` | Check both repo and tag. Calculator is `angular:python-calculator` and Node.js is `angular:nodejs-web-app`. |
| ECR shows three rows after one push | Our Buildx pushes produced an image index, image, and attestation. Use the row with your tag for the screenshot. |
| Old base image fails to resolve | Check the exact spelling, Docker Hub access, and architecture. Use the worksheet's base image when the rubric requires it. Confirm a substitute if the tag is unavailable. |
| Build works but old app's changes do not appear | Rebuilding creates an image. Create a container from the new build to use it. Restarting the old container keeps its original image. |

Restart an existing stopped container.

```bash
sudo docker start java-hello-world
```

To replace a container after rebuilding, stop and remove that specific container, then run the matching `docker run` command again. Removing a container discards its writable layer, so first save any data you need.

```bash
sudo docker stop java-hello-world
```

```bash
sudo docker rm java-hello-world
```

After finishing App 5, if you need the original Flask app on the RHEL host port 8081 again, stop App 5 and start the existing Flask container.

```bash
sudo docker stop java-hello-world
```

```bash
sudo docker start python-hello-world
```

Use `exit` to leave a normal SSH shell. Use **Ctrl+C** in a tunnel tab to stop that tunnel. The detached containers run independently on EC2 while the instance is up.

## Submission and instance state

| Worksheet | Evidence and final instance state |
| --- | --- |
| Quiz 1 | Submit the Flask Dockerfile and browser screenshot. Keep the EC2 instance and put it in **Stopped** state after taking the screenshot. |
| Quiz 2 | The worksheet requests all five Dockerfiles, browser screenshots, ECR screenshots, and a PEM key by email. Confirm an instructor-approved way to grant grading access before sharing a private key. Keep the instance. The worksheet gives no instruction to stop it. Leave it running if live grading requires access. |

- [Quiz 1 worksheet](quiz-1.pdf) · [Quiz 1 evidence](Quiz%201/)
- [Quiz 2 worksheet](quiz-2.pdf) · [Quiz 2 evidence](Quiz%202/) · [Five Dockerfiles](Quiz%202/Dockerfiles/)
- [Quiz 1 theory worksheet](Docker%20-%20Quiz1.docx)

Take browser evidence with the address and expected output visible. Take ECR evidence with the correct region, repository, and tag visible. We also saved terminal push screenshots for both Java apps.

If you need an exact copy of an EC2 Dockerfile, pull it with `scp`. This example saves a fresh copy of App 5 in the Mac's current directory under a unique filename.

```bash
scp -i /Users/Jaba/Desktop/Docker/labsuser.pem "ec2-user@$RHEL_HOST:Java-Hello-World/Dockerfile" ./java-hello-world.Dockerfile
```

In the AWS Console, **EC2 → Instances → select the instance → Instance state → Stop instance** is the action for Quiz 1. When you start an EBS-backed instance again, check its current public DNS and `sudo docker ps -a`. The lab containers were created without a Docker restart policy, so start the containers you need and reopen any SSH tunnels. [EC2 stop/start](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/Stop_Start.html)

## Theory questions

1. **What is Docker and why use it?** Docker is an open-source platform and toolset for building, distributing, and running containerized applications. Images let you package the app and its dependencies into a repeatable artifact.
2. **What is a container and what does it include?** A container is an isolated instance of an image. It runs the app with the image's userspace files and dependencies, plus runtime configuration and a writable layer.
3. **Image vs. container?** An image is the packaged template. A container is an instance created from it. Several containers can use one image.
4. **Why are containers lightweight compared with virtual machines?** Containers share the host kernel and isolate processes. Virtual machines boot separate guest kernels and operating systems.
5. **What does a Dockerfile do?** It describes how to build the image, including the base image, working directory, files, dependencies, and default startup command.
6. **Fill-ins in worksheet order.** `open-source`, `image`, `kernel`, `container`, `Dockerfile`.

| Instruction or command | Meaning |
| --- | --- |
| `FROM` | Starts a build stage from a base image. |
| `WORKDIR` | Sets the working directory for subsequent instructions. |
| `COPY` | Copies files from the build context, or from another stage with `--from`. |
| `RUN` | Executes a build-time command, such as installing dependencies or compiling Java. |
| `USER` | Sets the user and optional group for later commands and at runtime when set in the final stage. |
| `CMD` | Sets the default command or default arguments for the entrypoint. |
| `ENTRYPOINT` | Sets the executable run when the container starts. |
| `EXPOSE` | Documents the intended container port. |
| `docker build -t NAME:TAG .` | Builds an image from the current directory as the build context and assigns a local name and tag. |
| `docker run -d --name NAME -p HOST:CONTAINER IMAGE` | Creates and starts a detached container and maps the host port to the container port. |
| `docker tag` | Adds another name and tag to an image locally. |
| `docker push` / `docker pull` | Uploads or downloads an image via a registry. |
| Multi-stage build | Builds in one stage and copies the runtime artifact into the final stage. App 4 copies the JAR into a Java runtime image. App 5 uses the worksheet's Maven base in both stages. |

[Docker overview](https://docs.docker.com/get-started/docker-overview/) · [Dockerfile reference](https://docs.docker.com/reference/dockerfile/) · [Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
