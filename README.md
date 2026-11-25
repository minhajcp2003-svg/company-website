# Company Website — Company Landing Page

A responsive company landing page made with plain HTML5 and CSS3. It is served by Nginx in Docker and includes a Jenkins Declarative Pipeline for building, publishing and remotely deploying the container image.

## Features

- Responsive navigation, hero, About Us, three Services cards, Why Choose Us, Contact and footer sections.
- CSS hover effects, responsive breakpoints and reduced-motion support.
- No JavaScript, frontend framework, backend, database, API or build dependency.
- Docker image based on the official `nginx:alpine` image.
- Jenkins automation for Git checkout, Docker Hub push and remote deployment.

## Project structure

```text
company-website/
├── index.html
├── style.css
├── Dockerfile
├── Jenkinsfile
└── README.md
```

## Run locally

Open `index.html` in a browser. It loads `style.css` from the same directory.

## Build and run with Docker

Run from the project root:

```bash
docker build -t company-website:local .
docker run -d --name company-website-local -p 9090:80 company-website:local
```

Visit `http://localhost:9090`. To stop and remove the container:

```bash
docker rm -f company-website-local
```

The Dockerfile copies `index.html` and `style.css` into Nginx's default web root at `/usr/share/nginx/html/` and exposes container port `80`.

## Jenkins deployment architecture

The pipeline declares `agent none`. Each stage obtains an agent with label `build-server`. The Docker image is built and pushed on that agent only. During deployment, the build agent runs the SSH client, but the Docker pull, old-container removal, new-container start and HTTP verification commands execute on the separate deployment server over SSH. The Jenkins controller does not build the image, and the application is not deployed as a container on the build agent.

### Prerequisites

1. Jenkins has an agent/node labelled `build-server`.
2. Docker Engine and Docker CLI are installed on that agent; the Jenkins agent user can run Docker commands.
3. Git and SSH client are installed on the agent.
4. A separate Linux deployment server has Docker Engine installed and is reachable by SSH from the build agent.
5. The deployment SSH user can run Docker commands and `curl` on the deployment server.
6. Jenkins plugins: Pipeline, Git, Credentials Binding and SSH Agent.
7. TCP port `9090` is allowed through the deployment server firewall and any cloud firewall/security group.

### Placeholder values to replace

Edit the `environment` block in `Jenkinsfile`:

| Placeholder | Replace with |
|---|---|
| `DOCKERHUB_USERNAME` | Your Docker Hub account username. |
| `DOCKERHUB_REPOSITORY` | The Docker Hub repository name only, e.g. `company-website`. Do not include username or tag. |
| `DOCKERHUB_CREDENTIAL_ID` | Jenkins credential ID for a **Username with password** credential. Store your Docker Hub username and access token there. |
| `DEPLOYMENT_SERVER_IP` | SSH destination for the remote server, preferably `ssh-user@server-ip` or a configured SSH host alias. |
| `DEPLOYMENT_SSH_CREDENTIAL_ID` | Jenkins credential ID for an **SSH Username with private key** credential for the deployment server. |
| `GITHUB_REPOSITORY_URL` | Git clone URL, e.g. `https://github.com/your-org/company-website.git`. |

The uppercase strings are placeholders, not working values. Replace them before running the pipeline. Never place passwords, access tokens or private keys in source code or the Jenkinsfile.

### Configure Jenkins credentials

**Docker Hub:** In Jenkins, open **Manage Jenkins → Credentials** and add a **Username with password** credential. Use your Docker Hub username as the username and a Docker Hub access token as the password. Set its ID to the value configured for `DOCKERHUB_CREDENTIAL_ID`.

**Deployment SSH:** Add an **SSH Username with private key** credential. Set its username and private key, and set its ID to `DEPLOYMENT_SSH_CREDENTIAL_ID`. Add the matching public key to the deployment account's `~/.ssh/authorized_keys`. Ensure that account can execute Docker commands non-interactively.

For a private GitHub repository, configure a Jenkins Git credential and add its credentials ID to the `git` checkout step. The supplied checkout assumes a public repository or an agent already able to access the repository.

### Pipeline stages

1. **Checkout:** the `build-server` agent clones the configured GitHub repository's `main` branch.
2. **Build:** the `build-server` agent builds the image from the Dockerfile and tags it with the Jenkins build number and `latest`.
3. **Push:** Jenkins injects Docker Hub credentials from its credential store, logs in using `--password-stdin`, and pushes both tags.
4. **Deploy:** the build agent connects to the deployment server with SSH. The remote server pulls the build-number image, removes the previous `company-website` container if present, then starts the new container with host port `9090` mapped to container port `80`.
5. **Verify:** the remote server retries `curl http://localhost:9090/`. The pipeline fails if the site does not respond within the retry period.

### Open the deployed website

After a successful deployment, visit:

```text
http://DEPLOYMENT_SERVER_IP:9090
```

Replace the placeholder with the deployment server's reachable IP address or DNS name. Confirm TCP port `9090` is permitted by its firewall/security group.

## Operational notes

- The Jenkinsfile assumes the Git branch is `main`. Change `branch: 'main'` if your repository uses another branch.
- Create the Docker Hub repository before the first push.
- `StrictHostKeyChecking=accept-new` accepts a host key on first connection. For production, pre-verify and provision the deployment host key in the build agent's `known_hosts`.
- The deployment account needs Docker permissions. Apply your organization's least-privilege and host-hardening practices.
- Build-number tags make releases traceable; an earlier tag can be deployed for rollback by selecting that image tag in the deployment step.
- The contact link opens the visitor's email application. A working contact form would require a backend, which is intentionally outside this static project.
