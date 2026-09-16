# Day 38: Pull Docker Image

Nautilus project developers are planning to start testing on a new project. As per their meeting with the DevOps team, they want to test containerized environment application features. As per details shared with DevOps team, we need to accomplish the following task:

a. Pull `busybox:musl` image on `App Server 3` in Stratos DC and re-tag (create new tag) this image as `busybox:news`.

## Specific Requirements:

a. Pull `busybox:musl` image on `App Server 3` in Stratos DC and re-tag (create new tag) this image as `busybox:news`.

## Solution

The required `busybox:musl` image was downloaded on App Server 3, then a second local tag named `busybox:news` was created. Both tags point to the same image ID, so the retagging operation does not download or duplicate the image layers.

### 🔐 Step 1: Connect to App Server 3

```bash
ssh banner@stapp03
```

> **Why:** `ssh` opens a remote shell on App Server 3 as `banner`, the account used to run Docker commands in this lab.

### 📥 Step 2: Pull the BusyBox image with the `musl` tag

```bash
docker pull busybox:musl
```

Docker downloaded the requested image:

```text
Digest: sha256:32b5cdad7cce41dfd53d0ae06baebcf8357a147ee7694dc706911c373bc30c37
Status: Downloaded newer image for busybox:musl
docker.io/library/busybox:musl
```

> **Why:** `docker pull` downloads an image from its registry to the local Docker host. `busybox` is the image repository and `musl` is its tag, which selects the required variant rather than Docker's default tag.

### 🏷️ Step 3: Create the `news` tag

```bash
docker tag busybox:musl busybox:news
```

> **Why:** `docker tag` adds another local name and tag to an existing image. The first argument, `busybox:musl`, is the source image; `busybox:news` is the new repository-and-tag reference. This operation does not create a second image or download data because both tags reference the same immutable image ID.

### ✅ Step 4: Verify both image tags

```bash
docker images busybox
```

The output showed both required tags with the same image ID:

```text
REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
busybox      musl      654fc8fd836e   4 months ago   1.53MB
busybox      news      654fc8fd836e   4 months ago   1.53MB
```

> **Why:** `docker images` lists locally available images, and the `busybox` argument limits the list to that repository. The identical `IMAGE ID` confirms that `news` is a new tag for the pulled `musl` image, not a different image.

## Best Practices

- **Use explicit tags.** Pulling `busybox:musl` identifies the intended image variant precisely and avoids relying on a mutable default tag.
- **Retag locally when no content changes are needed.** `docker tag` creates a lightweight second reference without duplicating image layers.
- **Verify image identity.** Confirm that source and destination tags have the same image ID after retagging.
- **Keep local tags meaningful.** A tag such as `news` can identify the image's intended application context without changing its contents.

### 📚 Official Documentation

- [Docker pull reference](https://docs.docker.com/reference/cli/docker/image/pull/)
- [Docker tag reference](https://docs.docker.com/reference/cli/docker/image/tag/)
- [Docker images reference](https://docs.docker.com/reference/cli/docker/image/ls/)
