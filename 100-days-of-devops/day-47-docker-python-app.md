# Day 47: Docker Python App

A python app needed to be Dockerized, and then it needs to be deployed on App Server 1. We have already copied a requirements.txt file (having the app dependencies) under /python_app/src/ directory on App Server 1. Further complete this task as per details mentioned below:

1. Create a Dockerfile under /python_app directory:
   - Use any python image as the base image.
   - Install the dependencies using requirements.txt file.
   - Expose the port 3004.
   - Run the server.py script using CMD.

2. Build an image named nautilus/python-app using this Dockerfile.

3. Once image is built, create a container named pythonapp_nautilus:
   - Map port 3004 of the container to the host port 8097.

4. Once deployed, you can test the app using curl command on App Server 1.

```sh
curl http://localhost:8097/
```

## Specific Requirements:

1. Create a `Dockerfile` under `/python_app` directory using a Python base image, installing the dependencies from `requirements.txt`, exposing port `3004`, and running `server.py` with `CMD`.
2. Build an image named `nautilus/python-app` from this Dockerfile.
3. Create a container named `pythonapp_nautilus` and map its port `3004` to host port `8097`.
4. Test the application on App Server 1 with `curl http://localhost:8097/`.

## Solution

The supplied Flask application listens on all container interfaces at port `3004`. A single-stage Python image is sufficient because its only dependency is Flask; Docker already isolates the application's Python packages from the host. The image copies the dependency manifest first so that the dependency-installation layer can be reused when only application code changes.

### 🔐 Step 1: Connect to App Server 1 and open the application directory

```bash
ssh tony@stapp01
sudo su -
cd /python_app
```

> **Why:** `ssh` opens a remote shell on App Server 1 as `tony`. `sudo su -` opens a root shell, which has the required permission to write the root-owned application directory and manage Docker resources. `cd /python_app` selects the required Docker build context and the location where the Dockerfile must be created.

### 📝 Step 2: Create the Dockerfile

```bash
vi Dockerfile
```

Add the following content, then save and exit with `Esc`, `:wq`, and `Enter`:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY src/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY src/ .

EXPOSE 3004

CMD ["python", "server.py"]
```

> **Why:** `vi Dockerfile` creates the required file in `/python_app`. `FROM python:3.12-slim` selects a compact Python runtime as the base image. `WORKDIR /app` makes `/app` the directory for subsequent instructions and for the application process. The first `COPY` transfers only `requirements.txt`, and `RUN pip install --no-cache-dir -r requirements.txt` installs Flask without retaining pip's downloaded-package cache in the image. `COPY src/ .` adds `server.py` after dependency installation. `EXPOSE 3004` documents the port used by the application inside the container; it does not publish the port by itself. `CMD ["python", "server.py"]` is the default command in exec form, starting the Flask application when the container starts.

### 🏗️ Step 3: Build the Python application image

```bash
docker build --no-cache -t nautilus/python-app .
```

The image was built successfully as `nautilus/python-app`.

> **Why:** `docker build` assembles an image from the Dockerfile. `.` sends the current `/python_app` directory as the build context, allowing the Dockerfile to copy the `src` directory. `-t nautilus/python-app` assigns the exact repository and image name required by the task. `--no-cache` forces Docker to run every build instruction again, ensuring the corrected Dockerfile and current dependency installation are used.

### 🚀 Step 4: Start the container with the required port mapping

```bash
docker run -d --name pythonapp_nautilus -p 8097:3004 nautilus/python-app
```

> **Why:** `docker run` creates and starts a container from `nautilus/python-app`. `-d` runs it in detached mode so it remains running after the command returns. `--name pythonapp_nautilus` assigns the exact required container name. `-p 8097:3004` publishes host port `8097` and forwards its traffic to port `3004` inside the container, where the Flask application listens.

### ✅ Step 5: Verify the deployed application

```bash
curl http://localhost:8097/
```

The application returned:

```text
Welcome to xFusionCorp Industries!
```

> **Why:** `curl` sends an HTTP request to the application through the host port. Receiving the expected response confirms that the container is running, Docker is forwarding `8097` to `3004`, and Flask is serving the supplied route. If the request is sent immediately after starting the container, wait a few seconds for Flask to finish starting and run the same command again.

## Best Practices

- **Use a virtual environment only when it adds value.** Docker already isolates installed packages per image. A multi-stage build and virtual environment are useful when build tools or compiled dependencies must be excluded from the runtime image, but they add unnecessary complexity for this Flask-only lab.
- **Run Flask behind a production WSGI server.** The supplied application runs with `debug=True`, which enables Flask's development debugger. Do not use that server or debugger in a production deployment; use a dedicated WSGI server and disable debug mode instead.
- **Pin production images more strictly.** The `python:3.12-slim` tag is appropriate for this lab, but production images should use a specific patch release or image digest so rebuilds remain predictable.
- **Keep build context small.** Add a `.dockerignore` file in real projects to prevent local virtual environments, test output, credentials, and other unnecessary files from being sent to the Docker daemon during a build.

### 📚 Official Documentation

- [Dockerfile reference](https://docs.docker.com/reference/dockerfile/)
- [docker image build reference](https://docs.docker.com/reference/cli/docker/image/build/)
- [docker container run reference](https://docs.docker.com/reference/cli/docker/container/run/)
- [Docker port publishing](https://docs.docker.com/engine/network/port-publishing/)
- [Flask: Deploying to Production](https://flask.palletsprojects.com/en/stable/deploying/)
- [Flask: Debugging Application Errors](https://flask.palletsprojects.com/en/stable/debugging/)
