# Day 45: Resolve Dockerfile Issues

The Nautilus DevOps team is working to create new images per requirements shared by the development team. One of the team members is working to create a `Dockerfile` on `App Server 3` in `Stratos DC`. While working on it she ran into issues in which the docker build is failing and displaying errors. Look into the issue and fix it to build an image as per details mentioned below:

a. The `Dockerfile` is placed on `App Server 3` under `/opt/docker` directory.

b. Fix the issues with this file and make sure it is able to build the image.

c. Do not change base image, any other valid configuration within Dockerfile, or any of the data been used — for example, index.html.

`Note:` Please note that once you click on `FINISH` button all the existing containers will be destroyed and new image will be built from your `Dockerfile`.

## Specific Requirements:

1. The `Dockerfile` is placed on `App Server 3` under `/opt/docker` directory.
2. Fix the issues with this file and make sure it is able to build the image.
3. Do not change base image, any other valid configuration within Dockerfile, or any of the data been used — for example, index.html.

## Solution

The build failed because the Dockerfile used `IMAGE`, which is not a Dockerfile instruction, and used `ADD` for commands that must execute during the image build. The correction changed only those instruction keywords: `FROM` retains the existing base image, and `RUN` executes the existing `sed` commands. The certificates and static `index.html` copy instructions remained unchanged.

### 🔐 Step 1: Connect to App Server 3 and open the Dockerfile directory

```bash
ssh banner@stapp03
sudo su -
cd /opt/docker
```

> **Why:** `ssh` opens a remote shell on App Server 3 as `banner`. `sudo su -` opens a root shell, which has permission to edit the Dockerfile and communicate with the Docker daemon without repeating `sudo`. `cd /opt/docker` changes to the directory containing the Dockerfile specified by the challenge.

### 🔎 Step 2: Identify the invalid Dockerfile instructions

```bash
cat Dockerfile
```

The first instruction was `IMAGE httpd:2.4.43`, which Docker rejected as an unknown instruction. The next configuration lines began with `ADD`, which copies sources into an image and does not execute `sed` commands.

> **Why:** `cat` displays the existing Dockerfile without changing it. Reading the failed instructions first isolates the root cause and prevents unnecessary changes to the base image, certificate files, or static website content.

### 🛠️ Step 3: Correct only the instruction keywords

```bash
vi Dockerfile
```

Keep the existing values and replace the file content with:

```dockerfile
FROM httpd:2.4.43

RUN sed -i "s/Listen 80/Listen 8080/g" conf/httpd.conf

RUN sed -i '/LoadModule\ ssl_module modules\/mod_ssl.so/s/^#//g' conf/httpd.conf

RUN sed -i '/LoadModule\ socache_shmcb_module modules\/mod_socache_shmcb.so/s/^#//g' conf/httpd.conf

RUN sed -i '/Include\ conf\/extra\/httpd-ssl.conf/s/^#//g' conf/httpd.conf

COPY certs/server.crt /usr/local/apache2/conf/server.crt

COPY certs/server.key /usr/local/apache2/conf/server.key

COPY html/index.html /usr/local/apache2/htdocs/
```

Save and exit with `Esc`, `:wq`, and `Enter`.

> **Why:** `vi` edits the Dockerfile in place. `FROM` is the required first instruction and preserves the required `httpd:2.4.43` base image. `RUN` executes each `sed -i` command during the image build; the existing expressions change Apache to port `8080` and enable the existing SSL-related configuration. `conf/httpd.conf` is relative to the Apache image's working directory, `/usr/local/apache2`, so it resolves to `/usr/local/apache2/conf/httpd.conf`. The `COPY` instructions retain the supplied certificate and `index.html` data without modification.

### ✅ Step 4: Build the image without cached layers

```bash
docker build --no-cache -t dockerfile-check .
```

The build completed successfully and produced image `dockerfile-check`:

```text
=> exporting to image
=> writing image sha256:2b86aeb5f1c792f0f35db1d65e503de7143029cfabaf9083efb53a9f98ddb1c7
=> naming to docker.io/library/dockerfile-check
```

> **Why:** `docker build` creates an image using the Dockerfile in the current directory, represented by `.`. `--no-cache` forces every layer to run again, proving that no cached layer hides an error. `-t dockerfile-check` assigns a temporary local name to the image. A completed build verifies that Docker can rebuild the image when the lab's `FINISH` action runs.

## Best Practices

- **Use Dockerfile instructions for their intended purpose.** Use `FROM` to select a base image, `RUN` to execute build-time commands, and `COPY` or `ADD` to place files into an image.
- **Make the smallest valid correction.** Preserve the requested base image, existing valid settings, and supplied application data when resolving build failures.
- **Rebuild without cache after a fix.** `--no-cache` ensures every Dockerfile instruction is tested rather than reusing a successful layer from an earlier build.
- **Prefer `COPY` for local files.** `COPY` clearly expresses the intent to copy local build-context files, such as certificates and static website assets.

### 📚 Official Documentation

- [Dockerfile reference](https://docs.docker.com/reference/dockerfile/)
- [Dockerfile `FROM` instruction](https://docs.docker.com/reference/dockerfile/#from)
- [Dockerfile `RUN` instruction](https://docs.docker.com/reference/dockerfile/#run)
- [Dockerfile `COPY` instruction](https://docs.docker.com/reference/dockerfile/#copy)
- [Docker image build reference](https://docs.docker.com/reference/cli/docker/image/build/)
