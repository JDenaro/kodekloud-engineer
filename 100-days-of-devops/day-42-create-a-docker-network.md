# Day 42: Create a Docker Network

The Nautilus DevOps team needs to set up several docker environments for different applications. One of the team members has been assigned a ticket where he has been asked to create some docker networks to be used later. Complete the task based on the following ticket description:

a. Create a docker network named as `beta` on App Server `2` in `Stratos DC`.

b. Configure it to use `macvlan` drivers.

c. Set it to use subnet `192.168.0.0/24` and iprange `192.168.0.0/24`.

## Specific Requirements:

1. Create a docker network named as `beta` on App Server `2` in `Stratos DC`.
2. Configure it to use `macvlan` drivers.
3. Set it to use subnet `192.168.0.0/24` and iprange `192.168.0.0/24`.

## Solution

The `macvlan` driver requires a parent network interface on the Docker host. Normally, `ip route show default` is the preferred command to discover that interface. This lab host does not include the `ip` command, so `/proc/net/route` was used instead. It identified `eth0` as the default-route interface, which was supplied as the parent when creating `beta`.

### 🔐 Step 1: Connect to App Server 2

```bash
ssh steve@stapp02
```

> **Why:** `ssh` opens a remote shell on App Server 2 as `steve`, the user authorized to manage Docker on this host.

### 🔎 Step 2: Identify the macvlan parent interface

Normally, use the `iproute2` utility to find the interface used by the default route:

```bash
ip route show default
```

On this lab host, the command was unavailable:

```text
-bash: ip: command not found
```

Use the kernel routing table as the fallback:

```bash
awk '$2 == "00000000" { print $1 }' /proc/net/route
```

The command returned:

```text
eth0
```

> **Why:** A macvlan network needs a parent interface that carries its traffic to the physical network. On a typical Linux host, `ip route show default` reveals the interface after `dev`; it is the usual and more readable approach. The lab's minimal environment did not provide the `ip` command, so `awk` read `/proc/net/route` directly. In that file, `00000000` represents the default destination; `$1` is the interface name and `$2` is the destination field. The result established `eth0` as the required parent without guessing.

### 🌐 Step 3: Create the macvlan network

```bash
docker network create \
  --driver macvlan \
  --subnet 192.168.0.0/24 \
  --ip-range 192.168.0.0/24 \
  --opt parent=eth0 \
  beta
```

Docker returned the network ID:

```text
61115de4e4fa9da0b085684f64bc23690cef216454b0105a29d79c3a7109c249
```

> **Why:** `docker network create` adds a Docker network. `--driver macvlan` gives containers their own MAC addresses and connects their traffic through a host interface. `--subnet` defines the network's address block, while `--ip-range` defines the same requested block from which Docker allocates container IP addresses. `--opt parent=eth0` supplies the discovered parent interface required by the macvlan driver. `beta` is the exact network name requested by the ticket.

### ✅ Step 4: Verify the network configuration

```bash
docker network inspect beta
```

The relevant configuration was:

```text
"Name": "beta"
"Driver": "macvlan"
"Subnet": "192.168.0.0/24"
"IPRange": "192.168.0.0/24"
"parent": "eth0"
```

> **Why:** `docker network inspect` displays Docker's stored configuration for a named network. The output verifies the network name, driver, subnet, IP range, and parent interface required by the challenge.

## Best Practices

- **Discover the parent interface.** Do not assume an interface is named `eth0`; determine it from the host routing configuration before creating a macvlan network.
- **Use `ip` when available.** `ip route show default` is the standard, readable method. In minimal environments without `iproute2`, `/proc/net/route` provides the same kernel routing information.
- **Plan macvlan address ranges carefully.** Avoid overlapping container address pools with addresses already used by the physical network.
- **Understand macvlan host isolation.** Linux normally prevents direct communication between the host and containers on the same macvlan network; account for this when designing connectivity.
- **Inspect after creation.** Confirm the driver and IPAM configuration before attaching application containers.

### 📚 Official Documentation

- [Docker macvlan network driver](https://docs.docker.com/engine/network/drivers/macvlan/)
- [Docker network create reference](https://docs.docker.com/reference/cli/docker/network/create/)
- [Docker network inspect reference](https://docs.docker.com/reference/cli/docker/network/inspect/)
- [proc(5) manual page](https://man7.org/linux/man-pages/man5/proc.5.html)
