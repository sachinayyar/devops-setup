# Docker Hub Registry Integration with Jenkins

## Overview

This document describes how Jenkins is integrated with **Docker Hub** to push and pull container images during the CI/CD pipeline.

Authentication with Docker Hub is performed using a **Personal Access Token (PAT)** stored securely in **Jenkins Global Credentials**. The pipeline uses **Buildah** to authenticate with Docker Hub and push the container image.

---

## Prerequisites

- Docker Hub account
- Docker Hub Personal Access Token (PAT)
- Jenkins server
- Buildah installed on the Jenkins agent
- Jenkins Credentials configured

---

# 1. Generate a Docker Hub Personal Access Token

1. Log in to your Docker Hub account.
2. Navigate to **Account Settings**.
3. Select **Personal Access Tokens**.
4. Click **Generate New Token**.
5. Provide:
   - Token Description
   - Required Permissions (Read, Write, Delete)
6. Copy the generated token.

> **Note:** The token is displayed only once. Save it securely before leaving the page.

---

# 2. Configure Docker Hub Credentials in Jenkins

Navigate to:

```
Jenkins Dashboard
    └── Manage Jenkins
          └── Credentials
                └── System
                      └── Global credentials (unrestricted)
                            └── Add Credentials
```

Configure the credentials as follows:

| Field | Value |
|-------|-------|
| Kind | Username with password |
| Scope | Global |
| Username | Docker Hub Username |
| Password | Docker Hub Personal Access Token |
| ID | `DOCKERHUB_CREDENTIALS` |
| Description | Docker Hub Registry Credentials |

Click **Create** to save the credentials.

---

# 3. Use Docker Hub Credentials in the Pipeline

Add the Docker Hub credentials to the pipeline environment.

```groovy
environment {
    DOCKERHUB_CREDENTIALS = credentials('DOCKERHUB_CREDENTIALS')
}
```

Jenkins automatically creates the following environment variables:

| Variable | Description |
|-----------|-------------|
| `DOCKERHUB_CREDENTIALS_USR` | Docker Hub username |
| `DOCKERHUB_CREDENTIALS_PSW` | Docker Hub Personal Access Token |

---

# 4. Push Image to Docker Hub

The following stage authenticates with Docker Hub and pushes the image using Buildah.

```groovy
stage('Push Image to Dockerhub Registry') {
    steps {
        sh '''
        echo $DOCKERHUB_CREDENTIALS_PSW | buildah login \
            -u $DOCKERHUB_CREDENTIALS_USR \
            --password-stdin docker.io

        buildah push --storage-driver=vfs \
            docker.io/${IMAGE_NAME}-ms-stage:${BUILD_NUMBER} \
            docker://docker.io/${IMAGE_NAME}-ms-stage:${BUILD_NUMBER}
        '''
    }
}
```

---

# How It Works

1. Jenkins retrieves the Docker Hub credentials from the Global Credentials store.
2. The Docker Hub Personal Access Token is securely passed to the `buildah login` command using `--password-stdin`.
3. Buildah authenticates with the Docker Hub registry (`docker.io`).
4. The container image is pushed to the configured Docker Hub repository.
5. The image is tagged using the Jenkins build number.

Example image tag:

```
docker.io/<dockerhub-username>/<image-name>-ms-stage:<BUILD_NUMBER>
```

Example:

```
docker.io/ayyarsachin/springboot-ms-stage:15
```

---

# Verify the Image

After the pipeline completes successfully, verify the image in your Docker Hub repository.

Or pull the image manually:

```bash
docker pull docker.io/<dockerhub-username>/<image-name>-ms-stage:<BUILD_NUMBER>
```

Example:

```bash
docker pull docker.io/ayyarsachin/springboot-ms-stage:15
```

---

# Best Practices

- Use a Docker Hub **Personal Access Token** instead of your account password.
- Store credentials in **Jenkins Global Credentials**.
- Never hardcode registry credentials in the Jenkinsfile.
- Tag images using the Jenkins build number or Git commit SHA.
- Rotate Docker Hub tokens periodically.

---

# Expected Outcome

After a successful pipeline execution:

- Jenkins authenticates with Docker Hub.
- Buildah pushes the container image to the Docker Hub registry.
- The image is available in the Docker Hub repository and can be pulled for deployment.

Example:

```text
docker.io/ayyarsachin/springboot-ms-stage:15
```
