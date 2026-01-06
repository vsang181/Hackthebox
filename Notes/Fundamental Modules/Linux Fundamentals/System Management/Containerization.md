# Containerization

Containerization is the practice of packaging an application together with everything it needs to run—libraries, dependencies, configuration, and runtime—inside an isolated environment called a **container**. The goal is consistency: the application should behave the same way regardless of where you run it. On Linux systems, this is achieved using technologies such as Docker, Docker Compose, and Linux Containers (LXC) 

Unlike virtual machines, containers **share the host system’s kernel**. This makes them significantly lighter, faster to start, and more resource-efficient. Instead of virtualising an entire operating system, containers virtualise the application environment itself.

You should think of containers as controlled, repeatable execution environments rather than full machines.

---

## Why Containers Exist

Containers solve several practical problems:

* Inconsistent environments between development and production
* Complex dependency chains
* Slow deployment and scaling
* Resource-heavy virtual machines

By isolating applications while still sharing the host kernel, containers allow you to:

* Run many applications on the same host
* Scale services quickly
* Reproduce environments reliably
* Reduce configuration drift

This is why containerization is heavily used in modern DevOps, cloud platforms, and security testing.

---

## Isolation and Security Considerations

Containers provide **process, filesystem, and network isolation**, which reduces the blast radius of failures and limits unintended interactions between applications. However, this isolation is **not equivalent to a virtual machine**.

Key points you must remember:

* Containers share the host kernel
* A kernel-level vulnerability can affect all containers
* Misconfigurations can allow privilege escalation or container escape

From a security perspective, containers add a layer of defence, not an impenetrable barrier. As a penetration tester, you should always treat containers as **potential pivot points**, not safe zones.

---

## Docker Overview

**Docker** is the most widely used container platform. It focuses on **application-centric containerization**, meaning each container typically runs a single service or application.

Docker introduces several key concepts:

* **Images** – read-only templates used to create containers
* **Containers** – running instances of images
* **Dockerfiles** – build instructions for images
* **Registries** – storage locations for images (for example, Docker Hub)

A useful mental model is to think of Docker images as blueprints and containers as buildings created from those blueprints.

---

## Installing Docker Engine (Ubuntu)

On Ubuntu-based systems, Docker can be installed using the official repository.

```bash
#!/bin/bash

# Preparation
sudo apt update -y
sudo apt install ca-certificates curl gnupg lsb-release -y
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt update -y
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

# Add current user to Docker group
sudo usermod -aG docker htb-student
echo '[!] Log out and log back in for group changes to apply.'

# Verify installation
docker run hello-world
```

---

## Docker Images and Docker Hub

Docker images can be:

* Pulled from **Docker Hub**
* Built locally using a **Dockerfile**

Docker Hub is a public registry that hosts:

* Official images (maintained by vendors or projects)
* Community images
* Private images (restricted access)

Images are versioned and tagged, which makes them easy to track and reproduce.

---

## Creating a Docker Image with a Dockerfile

A **Dockerfile** defines how an image is built. Each instruction creates a new immutable layer.

Example Dockerfile (Ubuntu + Apache + SSH):

```dockerfile
FROM ubuntu:22.04

RUN apt-get update && \
    apt-get install -y apache2 openssh-server && \
    rm -rf /var/lib/apt/lists/*

RUN useradd -m docker-user && \
    echo "docker-user:password" | chpasswd

RUN usermod -aG sudo docker-user && \
    echo "docker-user ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers

EXPOSE 22 80

CMD service ssh start && /usr/sbin/apache2ctl -D FOREGROUND
```

This image is useful for controlled testing, file hosting, or simulating services during assessments.

---

## Building and Running a Docker Image

Build the image:

```text
docker build -t fs_docker .
```

Run the container:

```text
docker run -p 8022:22 -p 8080:80 -d fs_docker
```

This maps:

* Host port `8022` → container SSH (22)
* Host port `8080` → container HTTP (80)

The container runs in the background.

---

## Managing Docker Containers

Docker provides a rich set of management commands:

| Command          | Purpose                   |
| ---------------- | ------------------------- |
| `docker ps`      | List running containers   |
| `docker start`   | Start a stopped container |
| `docker stop`    | Stop a running container  |
| `docker restart` | Restart a container       |
| `docker rm`      | Remove a container        |
| `docker rmi`     | Remove an image           |
| `docker logs`    | View container logs       |

Remember: **containers are ephemeral by default**. Changes made inside a running container are lost unless you rebuild the image or use volumes.

---

## Stateless Design and Volumes

Docker containers are designed to be **stateless**. This means:

* Files written inside the container are lost when it stops
* Configuration changes do not persist

To retain data, you should use **volumes** or bind mounts, which store data outside the container lifecycle.

---

## Linux Containers (LXC)

**LXC** is a lower-level container technology that provides full Linux system containers. Unlike Docker, which focuses on applications, LXC focuses on **system-level isolation**.

LXC uses:

* **Namespaces** – isolate processes, networks, mounts, users
* **cgroups** – limit resource usage (CPU, memory, I/O)

LXC containers behave more like lightweight virtual machines, but still share the host kernel.

---

## Docker vs LXC (High-Level Comparison)

| Aspect            | Docker            | LXC                             |
| ----------------- | ----------------- | ------------------------------- |
| Focus             | Application-level | System-level                    |
| Image format      | Standardised      | Manual root filesystems         |
| Portability       | High              | Lower                           |
| Ease of use       | Beginner-friendly | Requires deeper Linux knowledge |
| Security defaults | Strong by default | Requires tuning                 |

Both can be abused if misconfigured and can act as vectors for **local privilege escalation**.

---

## Installing and Using LXC

Install LXC on Ubuntu:

```text
sudo apt install lxc -y
```

Create a container:

```text
sudo lxc-create -n linuxcontainer -t ubuntu
```

Start the container:

```text
sudo lxc-start -n linuxcontainer
```

Attach to it:

```text
sudo lxc-attach -n linuxcontainer
```

---

## Managing LXC Containers

Common LXC commands:

| Command       | Description                    |
| ------------- | ------------------------------ |
| `lxc-ls`      | List containers                |
| `lxc-start`   | Start a container              |
| `lxc-stop`    | Stop a container               |
| `lxc-restart` | Restart a container            |
| `lxc-attach`  | Attach to a container          |
| `lxc-config`  | Manage container configuration |

---

## Resource Limiting with cgroups

You can restrict container resources using cgroups.

Example configuration:

```text
lxc.cgroup.cpu.shares = 512
lxc.cgroup.memory.limit_in_bytes = 512M
```

This limits:

* CPU share to half of the default
* Memory usage to 512 MB

After updating configuration files, restart the LXC service:

```text
sudo systemctl restart lxc.service
```

---

## Namespaces and Isolation

LXC relies heavily on **namespaces** to isolate:

* Process IDs
* Network interfaces
* Mount points
* Filesystems

Each container has its own view of the system, but all containers still rely on the same kernel. This is why additional hardening is always recommended.

---

## Why Containers Matter in Security Testing

As a penetration tester, containers are extremely valuable:

* You can reproduce complex environments quickly
* You can test exploits safely
* You can isolate malware and vulnerable services
* You can simulate real-world infrastructure

At the same time, containers themselves are often **misconfigured in production**, making them excellent targets during assessments.
