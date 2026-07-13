# Complete End-to-End CI/CD Workflow

We implemented an end-to-end CI/CD pipeline using **Jenkins** for Continuous Integration and **ArgoCD** for Continuous Deployment following the **GitOps** approach. The pipeline automates application build, testing, security scanning, artifact management, containerization, and Kubernetes deployments across UAT and Production environments.

---

##
Architectural Diagram


## Tools Used

| Component | Tool |
|-----------|------|
| Source Code Management | GitHub / GitLab |
| CI Tool | Jenkins |
| Build Tool | Maven |
| Static Code Analysis | SonarQube |
| Artifact Repository | Nexus Repository |
| Containerization | Docker |
| Vulnerability Scanner | Trivy, Clair |
| Container Registry | DockerHub / Quay |
| GitOps | ArgoCD |
| Container Orchestration | Kubernetes |

---

## CI/CD Workflow

## Step 1: Source Code Checkout

The CI/CD pipeline begins when a developer pushes code to the Git repository or merges a pull request into the target branch. A webhook automatically triggers the Jenkins pipeline, which checks out the latest version of the source code. This ensures that every pipeline execution builds and deploys the most recent application code.

---

## Step 2: Maven Build

After the source code is checked out, Jenkins uses Maven to compile the application, download all required dependencies, execute unit tests, and package the application into a deployable JAR or WAR file. If the build or unit tests fail, the pipeline stops immediately, preventing faulty code from progressing further.

**Command Used**

```bash
mvn clean package
```

---

## Step 3: SonarQube Analysis

Once the application is successfully built, Jenkins triggers SonarQube to perform static code analysis. SonarQube scans the source code for bugs, vulnerabilities, code smells, duplicated code, and code coverage. The Quality Gate validates whether the application meets the organization's coding standards. If the Quality Gate fails, the pipeline is terminated until the identified issues are resolved.

---

## Step 4: Upload Artifact to Nexus

After the application passes the quality checks, the generated JAR or WAR file is uploaded to the Nexus Repository. Nexus acts as the centralized artifact repository, storing versioned application packages that can be reused for future deployments, rollbacks, or auditing purposes.

---

## Step 5: Docker Image Build (UAT)

Using the generated application artifact and a Dockerfile, Jenkins builds a Docker image for the application. This image packages the application together with its runtime dependencies, ensuring consistent behavior across development, testing, and production environments.

---

## Step 6: Container Image Vulnerability Scan

Before publishing the Docker image, security scanning tools such as Trivy and Clair analyze the image for known vulnerabilities, insecure packages, exposed secrets, and operating system CVEs. The pipeline proceeds only if the image satisfies the organization's security policies, ensuring that vulnerable images are not deployed to Kubernetes.

---

## Step 7: Push Image to Container Registry

Once the Docker image successfully passes the security scan, it is pushed to a container registry such as DockerHub or Quay. This registry serves as the centralized storage location from which Kubernetes clusters can securely pull the application images during deployment.

---

## Step 8: Deployment Approval (UAT)

Before deploying to the User Acceptance Testing (UAT) environment, Jenkins pauses the pipeline and waits for manual approval from the authorized team, such as QA engineers or release managers. This approval acts as a checkpoint to verify that the application is ready for functional and integration testing.

---

## Step 9: Update GitOps Repository

After deployment approval, Jenkins updates the Kubernetes deployment manifest stored in the GitOps repository by replacing the existing Docker image tag with the newly generated image tag. The updated manifest is committed and pushed to Git. This follows GitOps principles, where Git becomes the single source of truth for all Kubernetes deployments.

---

## Step 10: UAT Deployment using ArgoCD

ArgoCD continuously monitors the GitOps repository for changes. Once it detects the updated deployment manifest, it automatically synchronizes the Kubernetes cluster with the desired state defined in Git. ArgoCD performs a rolling deployment, replacing the existing application pods with new ones while minimizing downtime.

---

## Step 11: Production Image Promotion

After successful validation in the UAT environment, the tested Docker image is promoted to production by assigning it a production-specific image tag. Instead of rebuilding the application, the same validated image is reused, ensuring consistency and maintaining the principle of immutable artifacts across environments.

---

## Step 12: Push Production Image

The production-tagged Docker image is pushed to the container registry, making it available for deployment to the production Kubernetes cluster. Since the image has already been validated in UAT, no additional build process is required.

---

## Step 13: Production Deployment Approval

Before releasing the application to production, Jenkins requires manual approval from the release manager, product owner, or Change Advisory Board (CAB). This approval ensures that all required testing, business validations, and release procedures have been completed before deployment.

---

## Step 14: Update Production Deployment Manifest

Following production approval, Jenkins updates the production Kubernetes deployment manifest in the GitOps repository with the approved production image tag. The updated configuration is committed and pushed to Git, creating a complete audit trail of the production release.

---

## Step 15: Production Deployment using ArgoCD

ArgoCD detects the updated production deployment manifest and synchronizes the production Kubernetes cluster with the desired state stored in Git. It performs a rolling update to deploy the new version while continuously monitoring application health, ensuring a reliable and automated production deployment.

---

## Rollback Procedure

- Identify the last stable Docker image tag.
- Update the Kubernetes deployment manifest with the stable image tag.
- Commit and push the changes to the GitOps repository.
- ArgoCD automatically rolls back the deployment by synchronizing the cluster.
- Verify the application health after rollback.
