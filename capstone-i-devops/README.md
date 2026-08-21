# Capstone I — Jenkins, Ansible, Docker, and Apache

This directory is self-contained so it can live safely alongside other projects in this repository. It deploys a static Apache web application in a Docker container and documents an Ubuntu 24.04 EC2 CI/CD setup.

## Repository and branch model

The repository originally had `main` and `develop`, but the assignment calls for `master` and `develop`. Create `master` from the current baseline, retain `main` if it is useful to you, and use this flow:

| Branch | Build | Test | Production deploy |
| --- | --- | --- | --- |
| `develop` | Yes | Yes | No |
| `master` | Yes | Yes | Yes |

```bash
git checkout main
git pull origin main
git checkout -b master
git push -u origin master
git checkout develop
git merge master
git push origin develop
```

After committing this directory, merge it to `develop` for test builds; merge `develop` into `master` only for production releases.

## Included files

- `Dockerfile` builds an Ubuntu 24.04 image with Apache and the supplied `index.html`.
- `.dockerignore` keeps repository and CI files out of the Docker build context.
- `Jenkinsfile` is the recommended multibranch pipeline: `Job1-Build`, `Job2-Test`, and a `master`-only `Job3-Prod` stage.
- `jenkins/jobs/` holds three separate Pipeline scripts if an assessment requires three jobs with those exact names.
- `ansible/` configures the test and production Jenkins agents with Java 21, Git, Python, Docker Engine, and Docker access for `ubuntu`.

## Local Docker check

Run this from `capstone-i-devops`:

```bash
docker build -t abode-web:local .
docker run --rm -d --name abode-web-local -p 8080:80 abode-web:local
curl -f http://localhost:8080/
docker rm -f abode-web-local
```

## EC2 architecture and security groups

Create three Ubuntu 24.04 EC2 instances:

- **Controller**: Jenkins, Ansible, Git, Java 21, and Docker.
- **Test agent**: Jenkins SSH agent, Java 21, Git, and Docker.
- **Production agent**: Jenkins SSH agent, Java 21, Git, and Docker.

Allow SSH (22) and Jenkins (8080) only from your own IP. Allow HTTP (80) to the production server from the audience that must reach the website. Do not open SSH to the internet. Do not put a PEM file, an SSH private key, an EC2 address, Jenkins credentials, or webhook secrets in this repository.

## Configure the Controller

On the controller, install the baseline tools and Jenkins LTS. The commands below intentionally use the current official Docker repository and Java 21.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl wget unzip python3 software-properties-common openjdk-21-jdk
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install -y ansible

sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
sudo tee /etc/apt/sources.list.d/docker.sources >/dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker ubuntu

sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
echo 'deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/' | sudo tee /etc/apt/sources.list.d/jenkins.list >/dev/null
sudo apt update && sudo apt install -y jenkins
sudo usermod -aG docker jenkins
sudo systemctl enable --now docker jenkins
```

Log out and back in after changing group membership. Open `http://CONTROLLER_PUBLIC_IP:8080`, retrieve the initial password with `sudo cat /var/lib/jenkins/secrets/initialAdminPassword`, and complete Jenkins setup. Install or confirm these plugins: Git, GitHub, Pipeline, Pipeline Stage View, SSH Build Agents, and Credentials Binding.

## Configure the agents using Ansible

1. On the controller, create an SSH key for controller-to-agent access with `ssh-keygen -t ed25519`.
2. Copy **only** the public-key contents from `~/.ssh/id_ed25519.pub` to each agent's `~/.ssh/authorized_keys`. Keep the private key off GitHub.
3. Edit `ansible/inventory` locally and replace `TEST_PRIVATE_IP` and `PROD_PRIVATE_IP`. Do not commit that edited inventory; use a private inventory or an ignored copy in a real environment.
4. From this directory, run:

```bash
cd ansible
ansible agents -m ping
ansible-playbook setup.yml
```

The playbook only targets `[agents]`, and it verifies Ubuntu before installing software. Each agent must reconnect after the playbook before the new Docker group membership is active.

## Jenkins agents and pipeline

Create two SSH-launched Jenkins nodes:

- `TEST-AGENT`, label `test`, connected as `ubuntu`.
- `PROD-AGENT`, label `prod`, connected as `ubuntu`.

Store the controller's SSH private key in Jenkins Credentials, not in source control. Ensure the corresponding public key is on each agent. The Jenkinsfile uses the `test` label for build/test and the `prod` label only for master deployment.

### Recommended: one multibranch pipeline

Create a **Multibranch Pipeline** named `Capstone-I`. Set the Git repository URL to `https://github.com/maniyaraman744-hue/projects.git` and the script path to `capstone-i-devops/Jenkinsfile`. Enable branch discovery for `develop` and `master`. This lets Jenkins receive the actual branch name and guarantees that only `master` reaches the production stage.

### Assessment format: three named jobs

If you must demonstrate `Job1-Build → Job2-Test → Job3-Prod` as separate Pipeline jobs, create the jobs with exactly those names and paste the matching file from `jenkins/jobs/` into each job's Pipeline script box. Configure `Job1-Build` as the webhook entry job. The chain sends the branch parameter through the jobs; `Job2-Test` triggers `Job3-Prod` only when that parameter is `master`.

The multibranch pipeline and the three-job alternative are two ways to run the same workflow; do not enable both against the same push unless duplicate builds are acceptable.

## GitHub webhook

In GitHub, open **Settings → Webhooks → Add webhook** and use:

```text
http://CONTROLLER_PUBLIC_IP:8080/github-webhook/
```

Use `application/json` and select push events. The controller must be reachable from GitHub for this to work. For the multibranch pipeline, configure periodic branch indexing as a fallback and verify webhook deliveries in GitHub. For the three-job alternative, enable **GitHub hook trigger for GITScm polling** on `Job1-Build`.

## Release test

```bash
git checkout develop
# edit capstone-i-devops/index.html
git add capstone-i-devops
git commit -m "Update Capstone application"
git push origin develop

git checkout master
git merge develop
git push origin master
```

A `develop` push should build and test the Apache container on the test agent only. A `master` push should build, test, then replace `abode-production` on the production agent; browse to the production server on port 80 to verify it.
