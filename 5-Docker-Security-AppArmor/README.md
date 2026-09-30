# Exercise – Docker Security with AppArmor and Python

## Objective

The objective of this exercise was to understand how AppArmor profiles can be used to secure Docker containers, apply a profile to a Flask application, and test restricted actions using Python and the Docker SDK.

## 1. Prepared the Flask Application

Created `app.py` with a Flask route that returns a message when the root URL is accessed.

```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def home():
    return "Hello, this is a secure Flask application running inside a Docker container!"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

## 2. Created the Dockerfile and Built the Image

Created a `Dockerfile` to package the Flask application using the Python 3.8 slim image, install Flask, and expose port `5000`.

```dockerfile
# Use an official Python runtime as a parent image
FROM python:3.8-slim

# Set the working directory in the container
WORKDIR /app

# Copy the current directory contents into the container at /app
COPY . /app

# Install Flask
RUN pip install flask

# Expose port 5000
EXPOSE 5000

# Run app.py when the container launches
CMD ["python", "app.py"]
```

Built the image with the name `flask-apparmor`:

```bash
docker build -t flask-apparmor .
```

Output recorded in the exercise:

```text
Sending build context to Docker daemon  3.072kB
Step 1/5 : FROM python:3.8-slim
 ---> e83d9d28b2f6
Step 2/5 : WORKDIR /app
 ---> Using cache
 ---> 2c4979d5c6f3
Step 3/5 : COPY . /app
 ---> 91d94a789d7a
Step 4/5 : RUN pip install flask
 ---> Running in 7fd028e9370d
Collecting flask
  Downloading Flask-2.0.2-py3-none-any.whl (93 kB)
  ...
Successfully built flask-apparmor
```

The image build output indicates that Docker processed the Dockerfile and installed Flask.

## 3. Created and Loaded the AppArmor Profile

Created the profile file `my-apparmor-profile` at `/etc/apparmor.d/my-apparmor-profile` with rules to restrict access to sensitive directories, limit binary execution, allow network stream access, and deny the `sys_admin` capability.

```text
#include <tunables/global>

/usr/bin/python3 {
    # Deny access to sensitive system files
    deny /etc/** r,
    deny /var/** rw,

    # Allow Flask app to bind to port 5000
    network inet stream,

    # Permissions to the application directory
    /app/** rwk,

    # Deny execution of any binaries in /bin or /usr/bin
    deny /bin/** rmix,
    deny /usr/bin/** rmix,

    # Capability restrictions
    capability net_bind_service,
    deny capability sys_admin,
}
```

Loaded the profile using the command provided in the exercise:

```bash
sudo apparmor_parser -r /etc/apparmor.d/my-apparmor-profile
```

The exercise then applied the profile when starting the Flask container:

```bash
docker run --security-opt="apparmor=my-apparmor-profile" -p 5000:5000 flask-apparmor
```

## 4. Applied and Verified the Profile Using the Docker SDK

Created `apply_apparmor.py` to build the image, start a container with the AppArmor profile, inspect its security options, and stop the container.

```python
import docker

# Create a Docker client
client = docker.from_env()

# Build the Docker image
client.images.build(path=".", tag="flask-apparmor")

# Run the container with the AppArmor profile
container = client.containers.run(
    "flask-apparmor",
    ports={'5000/tcp': 5000},
    security_opt=["apparmor=my-apparmor-profile"],
    detach=True
)

print(f"Container started: {container.short_id}")

# Verify AppArmor profile applied
container_info = client.api.inspect_container(container.id)
apparmor_profile = container_info['HostConfig']['SecurityOpt']

print(f"AppArmor profile applied: {apparmor_profile}")

# Stop the container
container.stop()
```

Installed the Docker SDK for Python and ran the script using the commands from the exercise:

```bash
pip install docker
python apply_apparmor.py
```

Expected output recorded in the exercise:

```text
Building image from Dockerfile...
[INFO] Sending build context to Docker daemon  3.072kB
[INFO] Step 1/5 : FROM python:3.8-slim
 ---> e83d9d28b2f6
[INFO] Step 2/5 : WORKDIR /app
 ---> Using cache
 ---> 2c4979d5c6f3
[INFO] Step 3/5 : COPY . /app
 ---> Using cache
 ---> 91d94a789d7a
[INFO] Step 4/5 : RUN pip install flask
 ---> Using cache
 ---> 4f7e4a98128f
[INFO] Step 5/5 : EXPOSE 5000
 ---> Using cache
 ---> 230dece26d37
[INFO] Successfully built flask-apparmor

Running container with AppArmor profile...

Container started: f8c2a7f9b9b8

Inspecting container to verify AppArmor profile...

AppArmor profile applied: ['apparmor=my-apparmor-profile']

Stopping the container...
```

The reported security options contain `apparmor=my-apparmor-profile`, which is the profile specified when the container was started.

## 5. Tested Restricted Actions

Created `test_restricted_actions.py` to attempt to read `/etc/passwd` and execute `/bin/bash` from inside a container started with the AppArmor profile.

```python
import docker

# Create a Docker client
client = docker.from_env()

# Run the container with the AppArmor profile
container = client.containers.run(
    "flask-apparmor",
    ports={'5000/tcp': 5000},
    security_opt=["apparmor=my-apparmor-profile"],
    detach=True
)

# Test restricted actions
exit_code, output = container.exec_run("cat /etc/passwd")
print(f"Attempt to read /etc/passwd: Exit Code {exit_code}, Output: {output.decode()}")

exit_code, output = container.exec_run("/bin/bash")
print(f"Attempt to execute /bin/bash: Exit Code {exit_code}, Output: {output.decode()}")

# Stop the container
container.stop()
```

Ran the test script:

```bash
python test_restricted_actions.py
```

Expected output recorded in the exercise:

```text
Container started: f8c2a7f9b9b8

Attempt to read /etc/passwd: Exit Code 1, Output:
Attempt to execute /bin/bash: Exit Code 126, Output:
Container stopped
```

The recorded output shows non-zero exit codes for both attempted actions, as expected by the exercise.

## 6. Questions and Answers

### 1. What is the purpose of using AppArmor with Docker containers?

AppArmor is used to enforce security policies and confine applications to a limited set of resources. With Docker containers, it helps to limit access to system resources, files, and networks, thus providing an additional layer of security.

### 2. How do AppArmor profiles help secure a Docker container?

AppArmor profiles define what a containerized application can or cannot do. They restrict access to sensitive directories, network capabilities, file execution, and system calls, ensuring the container behaves securely without affecting the host system.

### 3. Why is it important to restrict access to sensitive directories such as `/etc/` and `/var/`?

Sensitive directories like `/etc/` contain configuration files and sensitive information such as user data and system settings. Restricting access prevents the container from reading or modifying important system files, reducing the risk of security breaches.

### 4. What other capabilities can you restrict using AppArmor profiles?

AppArmor can restrict a container's ability to access the network, bind to specific ports, execute binaries, write to specific directories, and use system administration capabilities (`cap_sys_admin`).

### 5. How can you verify if an AppArmor profile is successfully applied to a Docker container?

You can verify if an AppArmor profile is applied by inspecting the container using the Docker SDK or the Docker CLI. The `HostConfig.SecurityOpt` field will show the applied security options, including the AppArmor profile name.

## 7. Result

The exercise covered building a Docker image for the Flask application, creating and loading an AppArmor profile, applying the profile through Docker CLI and the Docker SDK for Python, and testing attempts to read `/etc/passwd` and execute `/bin/bash`. The expected outputs provided in the exercise show the profile name in the container security options and non-zero exit codes for the restricted-action tests.
