# Exercise – Introduction to Continuous Integration (CI) and Jenkins Installation

## Objective

The objective of this exercise was to understand what Continuous Integration (CI) is and how it works, review common CI tools, learn the core concepts of Jenkins, and install Jenkins using Docker and unlock it with the initial admin password.

## 1. Studied Continuous Integration (CI)

Continuous Integration is a development practice where developers frequently integrate their code changes into a shared repository. Its primary goal is to detect and address errors early in the development process, improving software quality and accelerating delivery.

**Key features of CI**

- **Frequent code integration:** developers commit code to a shared repository multiple times a day.
- **Automated builds:** every commit triggers an automated build to verify the code changes.
- **Automated testing:** automated tests run as part of the CI process to ensure functionality remains intact.
- **Immediate feedback:** developers receive quick feedback on their changes and can address issues promptly.

**Benefits of CI**

- **Early bug detection:** integration issues are found early, which reduces the cost of fixing bugs.
- **Improved collaboration:** developers work on the same codebase and changes are integrated smoothly.
- **Faster development cycles:** automated builds and tests speed up development.
- **High-quality code:** continuous testing ensures code quality and prevents regressions.

## 2. Understood How CI Works

1. **Developer workflow:** developers write code and push changes to a version control system such as Git.
2. **CI server workflow:** the CI server (e.g. Jenkins, GitLab CI) detects the code changes and triggers the build process.
3. **Automated build:** the codebase is compiled, dependencies are resolved, and the application is built.
4. **Automated testing:** unit tests, integration tests and other automated tests are run to verify functionality.
5. **Feedback:** the results (success or failure) are communicated back to the developers for further action.

A simple CI workflow therefore looks like this:

```text
Code change  →  Trigger build  →  Run tests  →  Feedback to developers
```

## 3. Reviewed Common CI Tools

| Tool | Overview | Key features | Ideal for |
|------|----------|--------------|-----------|
| **GitLab CI/CD** | CI/CD built into GitLab's platform and integrated with Git repositories | Built in with GitLab; YAML-based pipelines; Auto DevOps | Teams using GitLab for version control |
| **CircleCI** | Cloud-based CI/CD tool that integrates easily with GitHub and Bitbucket | Container-based environments; parallelism and caching; native Docker support | Teams with heavily containerized workflows |
| **Travis CI** | CI service for open-source and private projects | YAML configuration; support for many languages; easy GitHub integration | Open-source projects and small teams |
| **Bamboo** | Atlassian's CI/CD tool, designed to work with JIRA and Bitbucket | Build and deployment pipelines; Atlassian integrations; prebuilt deployment environments | Teams in the Atlassian ecosystem |
| **TeamCity** | JetBrains CI/CD server with many integrations and advanced features | Detailed build history and reports; prebuilt runners; Docker and Kubernetes plugins | Complex enterprise CI/CD setups |
| **Azure DevOps (Pipelines)** | Microsoft's CI/CD pipelines, part of its DevOps suite | Azure and Visual Studio integration; YAML or classic editor; Windows, macOS and Linux | Teams using Azure cloud services |
| **GitHub Actions** | CI/CD provided by GitHub for automating workflows from repositories | YAML workflows; marketplace of prebuilt actions; deep GitHub integration | Teams using GitHub for version control |
| **Spinnaker** | Open-source continuous delivery tool for cloud-native applications | Multi-cloud deployments; Kubernetes, AWS and GCP integration; canary and blue-green strategies | Complex cloud-based deployments |
| **Buildkite** | Hybrid platform that runs pipelines on your own infrastructure | Cloud management with self-hosted agents; parallel builds; containerized workflows | Teams with security concerns about fully cloud-based CI |
| **Drone** | Lightweight, containerized CI/CD platform | Docker containers for pipeline steps; GitHub, GitLab and Bitbucket integration; open source | Teams preferring container-based workflows |

## 4. Studied Jenkins

Jenkins is an open-source automation server used to build, test and deploy software. It automates the parts of the development process related to building, testing and deploying, which enables Continuous Integration (CI) and Continuous Delivery (CD).

**Why use Jenkins**

- **Automation:** automates repetitive tasks.
- **Continuous Integration:** ensures that code changes integrate smoothly.
- **Extensibility:** supports plugins for a wide range of tools and services.
- **Scalability:** can run distributed builds on multiple machines.

**Key Jenkins concepts**

| Concept | Description |
|---------|-------------|
| **Job / Project** | A task or process that Jenkins runs |
| **Build** | The process of compiling and packaging code |
| **Pipeline** | A series of steps to build, test and deploy an application |
| **Plugins** | Extensions to Jenkins' functionality (e.g. Git plugin, Docker plugin) |
| **Nodes** | Machines where Jenkins runs jobs (master/agent setup) |

## 5. Installed Jenkins Using Docker

Started Jenkins in a container, exposing the web interface on port `8080` and the agent communication port on `50000`:

```bash
docker run -d --name jenkins -p 8080:8080 -p 50000:50000 jenkins/jenkins:lts
```

External access to Jenkins is through port `8080`.

Output recorded in the exercise:

```text
Unable to find image 'jenkins/jenkins:lts' locally
lts: Pulling from jenkins/jenkins
7d98d813d54f: Pull complete
a36a1f8447e8: Pull complete
f7ee35b739bf: Pull complete
bb3229cfea8c: Pull complete
d6f97f67d9a1: Pull complete
866fdfadf828: Pull complete
28ae6a41194b: Pull complete
9393a6815458: Pull complete
80c15ff4ebe9: Pull complete
4b1bc6ce70dc: Pull complete
1b47cf50f1d7: Pull complete
089de16bf95f: Pull complete
Digest: sha256:7ea4989040ce0840129937b339bf8c8f878c14b08991def312bdf51ca05aa358
Status: Downloaded newer image for jenkins/jenkins:lts
d350facb3a26a8282e286a9f63ea585b93372c97222692cb89429a298e39d1d1
```

The output shows Docker pulling the `jenkins/jenkins:lts` image and starting the container.

## 6. Retrieved the Initial Admin Password

Checked that the container was running:

```bash
docker ps -a
```

```text
CONTAINER ID   IMAGE                 COMMAND                  CREATED          STATUS          PORTS                                                                                      NAMES
d350facb3a26   jenkins/jenkins:lts   "/usr/bin/tini -- /u…"   14 minutes ago   Up 14 minutes   0.0.0.0:8080->8080/tcp, :::8080->8080/tcp, 0.0.0.0:50000->50000/tcp, :::50000->50000/tcp   jenkins
```

Opened a shell inside the container and read the initial admin password:

```bash
docker exec -it d350facb3a26 bash
cat /var/jenkins_home/secrets/initialAdminPassword
```

```text
<initial-admin-password>
```

The password is different for every new Jenkins instance, so it is shown here as a placeholder.

## 7. Accessed Jenkins and Completed the Setup

Opened Jenkins in the browser on port `8080`:

```text
http://localhost:8080/
```

Setup steps followed in the browser:

1. On the **Unlock Jenkins** screen, pasted the initial admin password from the previous step.
2. On the **Getting Started** screen, installed the suggested plugins.
3. Created the admin user and reached the Jenkins landing page (dashboard).

## 8. Result

The exercise covered the concept of Continuous Integration, its features, benefits and workflow, a comparison of popular CI tools, and the core concepts of Jenkins (jobs, builds, pipelines, plugins and nodes). Jenkins was installed using the `jenkins/jenkins:lts` Docker image, the initial admin password was retrieved from inside the container, and Jenkins was accessed on `http://localhost:8080/`, ready for creating pipelines.