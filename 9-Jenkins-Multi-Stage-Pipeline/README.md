# Exercise – Jenkins Multi-Stage Pipeline: Deploying a Python Application

## Objective

The objective of this exercise was to create and run a Jenkins Pipeline with multiple stages (Build, Test, Deploy, Run and Test Application) for a sample Python Flask application, in order to understand how multi-stage CI/CD pipelines are defined and managed in Jenkins.

## 1. Set Up Jenkins

Made sure Jenkins was installed and running (see the *Introduction to CI and Jenkins Installation* exercise) and that the **Pipeline** plugin was installed.

## 2. Created the Sample Python Application

Created a directory for the application:

```bash
mkdir python-flask-app
cd python-flask-app
```

Created `app.py`, a Flask application with a single route:

```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def home():
    return "Hello, Jenkins Multi-Stage Pipeline!"

if __name__ == "__main__":
    app.run(debug=True)
```

Created `requirements.txt`:

```text
flask==2.1.2
```

Created `test_app.py`, a unit test that calls the root route and checks the status code and response body:

```python
import unittest
from app import app

class TestApp(unittest.TestCase):
    def test_home(self):
        tester = app.test_client()
        response = tester.get("/")
        print(response.data.decode("utf-8"))
        self.assertEqual(response.status_code, 200)
        self.assertEqual(response.data.decode("utf-8"), "Hello, Jenkins Multi-Stage Pipeline!")

if __name__ == "__main__":
    unittest.main()
```

## 3. Pushed the Application to GitHub

Initialized the repository, committed the files and pushed them to GitHub:

```bash
git init
git add .
git commit -m "Initial commit"
git remote add origin https://github.com/<your-GitHub-username>/devops-sample-code.git
git branch -M main
git push -u origin main
```

## 4. Created the Pipeline Job in Jenkins

1. On the Jenkins dashboard, clicked **New Item**.
2. Entered the name `Python-MultiStage-Pipeline` and selected **Pipeline**.
3. Clicked **OK**.

## 5. Defined the Jenkinsfile

Created a `Jenkinsfile` in the project directory with five stages and a `post` section that reports success or failure:

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                echo 'Creating virtual environment and installing dependencies...'
            }
        }
        stage('Test') {
            steps {
                echo 'Running tests...'
                sh 'python3 -m unittest discover -s .'
            }
        }
        stage('Deploy') {
            steps {
                echo 'Deploying application...'
                sh '''
                mkdir -p ${WORKSPACE}/python-app-deploy
                cp ${WORKSPACE}/app.py ${WORKSPACE}/python-app-deploy/
                '''
            }
        }
        stage('Run Application') {
            steps {
                echo 'Running application...'
                sh '''
                nohup python3 ${WORKSPACE}/python-app-deploy/app.py > ${WORKSPACE}/python-app-deploy/app.log 2>&1 &
                echo $! > ${WORKSPACE}/python-app-deploy/app.pid
                '''
            }
        }
        stage('Test Application') {
            steps {
                echo 'Testing application...'
                sh '''
                python3 ${WORKSPACE}/test_app.py
                '''
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check the logs for more details.'
        }
    }
}
```

| Stage | What it does |
|-------|--------------|
| **Build** | Prints a message for the build step (dependencies are installed in the Jenkins container, see section 7) |
| **Test** | Runs the unit tests with `python3 -m unittest discover -s .` |
| **Deploy** | Copies `app.py` into a mock deployment directory, `python-app-deploy`, inside the workspace |
| **Run Application** | Starts the deployed app in the background with `nohup` and saves its process ID to `app.pid` |
| **Test Application** | Runs `test_app.py` against the application |

Pushed the `Jenkinsfile` to the repository:

```bash
git add Jenkinsfile
git commit -m "Added Jenkinsfile"
git push
```

## 6. Configured the Jenkins Job

In the pipeline configuration:

1. Under **Pipeline**, selected **Pipeline script from SCM**.
2. Chose **Git** as the SCM and entered the repository URL.
3. Added credentials where necessary (not needed for a public repository).
4. Clicked **Save**.

## 7. Fixed Build Errors in the Jenkins Container

The first run of the pipeline failed because the Jenkins Docker image does not include Python 3, pip or Flask. Found the Jenkins container ID:

```bash
docker ps -a
```

```text
CONTAINER ID   IMAGE                 COMMAND                  CREATED       STATUS       PORTS                                                                                      NAMES
78a4587c55da   jenkins/jenkins:lts   "/usr/bin/tini -- /u…"   8 hours ago   Up 8 hours   0.0.0.0:8080->8080/tcp, :::8080->8080/tcp, 0.0.0.0:50000->50000/tcp, :::50000->50000/tcp   jenkins
```

Opened a root shell inside the container:

```bash
docker exec -it -u root 78a4587c55da bash
```

Installed the missing packages and ran the tests manually to confirm they work:

```bash
apt-get update
apt install python3
apt install python3-pip
apt install python3.11-venv
apt install python3-flask
python3 -m unittest discover -s .
```

## 8. Ran the Pipeline

Clicked **Build Now** on the job page, watched the stages execute, and opened **Console Output** for the logs.

Output recorded in the exercise:

```text
Started by user admin
Obtained Jenkinsfile from git https://github.com/<your-GitHub-username>/devops-sample-code.git
[Pipeline] Start of Pipeline
[Pipeline] node
Running on Jenkins in /var/jenkins_home/workspace/Python-MultiStage-Pipeline
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Declarative: Checkout SCM)
[Pipeline] checkout
Selected Git installation does not exist. Using Default
The recommended git tool is: NONE
No credentials specified
 > git rev-parse --resolve-git-dir /var/jenkins_home/workspace/Python-MultiStage-Pipeline/.git # timeout=10
Fetching changes from the remote Git repository
 > git config remote.origin.url https://github.com/<your-GitHub-username>/devops-sample-code.git # timeout=10
Fetching upstream changes from https://github.com/<your-GitHub-username>/devops-sample-code.git
 > git --version # timeout=10
 > git --version # 'git version 2.39.5'
 > git fetch --tags --force --progress -- https://github.com/<your-GitHub-username>/devops-sample-code.git +refs/heads/*:refs/remotes/origin/* # timeout=10
 > git rev-parse refs/remotes/origin/main^{commit} # timeout=10
Checking out Revision f82e59e16ad6e72100e54d776df07dd26edf80de (refs/remotes/origin/main)
 > git config core.sparsecheckout # timeout=10
 > git checkout -f f82e59e16ad6e72100e54d776df07dd26edf80de # timeout=10
Commit message: "Updated Jenkinsfile"
 > git rev-list --no-walk 7733cb39ce899622faf875a95ee6a5b39effa66c # timeout=10
[Pipeline] }
[Pipeline] // stage
[Pipeline] withEnv
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Build)
[Pipeline] echo
Creating virtual environment and installing dependencies...
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Test)
[Pipeline] echo
Running tests...
[Pipeline] sh
+ python3 -m unittest discover -s .
.
----------------------------------------------------------------------
Ran 1 test in 0.004s

OK
Hello, Jenkins Multi-Stage Pipeline!
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Deploy)
[Pipeline] echo
Deploying application...
[Pipeline] sh
+ mkdir -p /var/jenkins_home/workspace/Python-MultiStage-Pipeline/python-app-deploy
+ cp /var/jenkins_home/workspace/Python-MultiStage-Pipeline/app.py /var/jenkins_home/workspace/Python-MultiStage-Pipeline/python-app-deploy/
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Run Application)
[Pipeline] echo
Running application...
[Pipeline] sh
+ echo 5064
+ nohup python3 /var/jenkins_home/workspace/Python-MultiStage-Pipeline/python-app-deploy/app.py
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Test Application)
[Pipeline] echo
Testing application...
[Pipeline] sh
+ python3 /var/jenkins_home/workspace/Python-MultiStage-Pipeline/test_app.py
.
----------------------------------------------------------------------
Ran 1 test in 0.004s

OK
Hello, Jenkins Multi-Stage Pipeline!
[Pipeline] }
[Pipeline] // stage
[Pipeline] stage
[Pipeline] { (Declarative: Post Actions)
[Pipeline] echo
Pipeline completed successfully!
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // withEnv
[Pipeline] }
[Pipeline] // node
[Pipeline] End of Pipeline
Finished: SUCCESS
```

The console output shows the repository being checked out, every stage running in order, the unit test passing (`Ran 1 test ... OK`) both in the Test stage and in the Test Application stage, and the build ending with `Finished: SUCCESS`.

## 9. Additional Enhancements

Possible improvements to the pipeline described in the exercise:

- Add a code quality stage using a tool such as `flake8` or `pylint`:

```groovy
stage('Code Quality') {
    steps {
        echo 'Running code quality checks...'
        sh 'flake8 .'
    }
}
```

- Deploy the application to a Docker container as part of the pipeline.
- Add notifications (e.g. email or Slack) for pipeline success or failure.

## 10. Result

The exercise covered creating a Flask application with a unit test, pushing it to GitHub, and defining a declarative Jenkins pipeline in a `Jenkinsfile` with Build, Test, Deploy, Run Application and Test Application stages. After installing Python 3, pip and Flask inside the Jenkins container, the pipeline checked out the code from GitHub, passed its tests, copied `app.py` to the deployment directory, started the application and finished with `Finished: SUCCESS`.