# Exercise – Creating a "Hello World" Jenkins Job

## Objective

The objective of this exercise was to create a simple shell script, push it to a GitHub repository, and configure a Jenkins Freestyle job that pulls the repository and runs the script, then verify the result in the build's console output.

## 1. Created a GitHub Repository and Access Token

Created a new public repository on GitHub:

1. Logged in to GitHub, clicked the **+** icon (top-right) and selected **New repository**.
2. Set the repository name to `devops-sample-code`.
3. Set the description to "A demo repository for Jenkins scripting".
4. Set the visibility to **Public**.
5. Clicked **Create repository**.

Created a fine-grained personal access token (PAT) on GitHub to use in place of a password when pushing code from the command line.

## 2. Created the Script Locally

Created `hello-world.sh`, added the content below, and made it executable.

```bash
touch hello-world.sh
```

```bash
#!/bin/bash
echo "Hello, Jenkins!"
```

```bash
chmod +x hello-world.sh
```

## 3. Initialized a Local Git Repository

Configured Git with a name and email and initialized the repository.

```bash
git config --global user.name "Your Name"
git config --global user.email "your-email@example.com"
git init
```

## 4. Added and Committed the Script

Added the script to the repository and checked the status:

```bash
git add hello-world.sh
git status
```

Output recorded in the exercise:

```text
On branch master

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
        new file:   hello-world.sh
```

Committed the change:

```bash
git commit -m "Add hello-world.sh"
```

```text
[master (root-commit) 2e1b718] Add hello-world.sh
 1 file changed, 2 insertions(+)
 create mode 100644 hello-world.sh
```

## 5. Linked the Local Repository to GitHub

Linked the local repository to the GitHub repository using its remote URL:

```bash
git remote add origin https://github.com/<your-GitHub-username>/devops-sample-code.git
```

## 6. Pushed the Script to GitHub

The local branch was created as `master`, so it was renamed to `main` to match the branch used in the push command, then pushed:

```bash
git branch -M main
git push -u origin main
```

When prompted, entered the GitHub username and pasted the personal access token as the password.

Output recorded in the exercise:

```text
Username for 'https://github.com': <your-GitHub-username>
Password for 'https://<your-GitHub-username>@github.com': <personal-access-token>
Enumerating objects: 4, done.
Counting objects: 100% (4/4), done.
Delta compression using up to 4 threads
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 306 bytes | 30.00 KiB/s, done.
Total 3 (delta 0), reused 0 (delta 0), pack-reused 0
To https://github.com/<your-GitHub-username>/devops-sample-code.git
   4be1c56..1b647f7  main -> main
branch 'main' set up to track 'origin/main'.
```

## 7. Verified the Script on GitHub

Opened `https://github.com/<your-GitHub-username>/devops-sample-code` in the browser and confirmed that `hello-world.sh` was present in the repository.

## 8. Accessed Jenkins

Made sure Jenkins was running (see the *Introduction to CI and Jenkins Installation* exercise), then opened `http://localhost:8080` in the browser and logged in with the admin credentials.

## 9. Created the Jenkins Job

1. On the Jenkins dashboard, clicked **New Item** in the left-hand menu.
2. Entered the name `HelloWorld`.
3. Selected **Freestyle project**.
4. Clicked **OK**.

## 10. Configured the Job

**General**

- Added the description: `Hello World! Jenkins job.`

**Source Code Management**

- Selected **Git**.
- Entered the repository URL: `https://github.com/<your-GitHub-username>/devops-sample-code.git`

**Build Steps**

- Clicked **Add build step → Execute shell** and added the script:

```bash
sh hello-world.sh
```

Clicked **Save** to save the job configuration.

## 11. Ran the Job

Triggered the build manually from **Jenkins Dashboard → HelloWorld → Build Now**.

## 12. Viewed the Build Output

1. Opened the **Build History** section on the left-hand side of the job page.
2. Clicked the build number (`#1`).
3. Clicked **Console Output** to see the build logs.

Output recorded in the exercise:

```text
Started by user Admin
Building in workspace /var/jenkins_home/workspace/HelloWorld
[HelloWorld] $ /bin/sh -xe /tmp/jenkins1281930102.sh
+ echo 'Hello, Jenkins!'
Hello, Jenkins!
Finished: SUCCESS
```

The console output shows Jenkins running the script and printing `Hello, Jenkins!`, and the build finishing with `SUCCESS`.

## 13. Result

The exercise covered creating a shell script, committing it to a local Git repository, pushing it to GitHub using a personal access token, and configuring a Jenkins Freestyle job that fetches the repository and runs the script with an **Execute shell** build step. The build completed with `Finished: SUCCESS` and the console output displayed `Hello, Jenkins!`, confirming that Jenkins was able to pull code from GitHub and execute it.