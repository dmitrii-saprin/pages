# Running GitLab Runner and Docker in RAM

## Introduction

To speed up CI/CD, you can run **GitLab Runner** and **Docker** on a RAM-backed filesystem using `tmpfs`. This reduces disk I/O and can noticeably improve performance, but it requires significant RAM (often **10–20 GB**, depending on workload). This guide shows how to configure `tmpfs` for these services.

## What is `tmpfs`?

`tmpfs` is a **temporary filesystem** that stores data in RAM instead of on disk.

Benefits:

* High speed due to RAM-based storage
* Reduced wear on physical disks (especially SSDs)
* Dynamic sizing based on available memory (up to a configured limit)

Main limitation:

* **All data stored in `tmpfs` is lost on reboot**

## Running GitLab Runner in RAM

### 1) Create the `/builds` directory

```bash
sudo mkdir -p /builds
```

### 2) Mount `/builds` as `tmpfs` (temporary)

GitLab Runner uses `/builds` for temporary build files. Mount it as `tmpfs`:

```bash
sudo mount -t tmpfs -o size=15G,mode=1777 tmpfs /builds
```

Verify the mount:

```bash
mount | grep /builds
```

### 3) Make the mount persistent via `/etc/fstab`

Add the following line to `/etc/fstab`:

```fstab
tmpfs /builds tmpfs defaults,size=15G,mode=1777 0 0
```

Apply the change:

```bash
sudo mount -a
```

### 4) Configure GitLab Runner

Open the Runner configuration:

```bash
sudo vi /etc/gitlab-runner/config.toml
```

Modify or add the parameters below:

```toml
[[runners]]
  name = "your_runner_name"
  url = "https://your-gitlab-instance.com/"
  clean_runner_cache = true
  builds_dir = "/builds"
```

Restart GitLab Runner and check status:

```bash
sudo systemctl restart gitlab-runner
sudo systemctl status gitlab-runner
```

## Running Docker fully in RAM

Docker stores its data under `/var/lib/docker`. Moving this directory to RAM can improve container performance.

### 1) Add mount points in `/etc/fstab`

Add these lines:

```fstab
tmpfs /var/lib/docker               tmpfs defaults,size=5G,mode=1777 0 0
tmpfs /var/lib/docker/volumes       tmpfs defaults,size=5G,mode=1777 0 0
tmpfs /var/lib/docker/containerd    tmpfs defaults,size=5G,mode=1777 0 0
```

Apply mounts:

```bash
sudo mount -a
```

Verify:

```bash
mount | grep /var/lib/docker
```

### 2) Restart Docker

```bash
sudo systemctl stop docker docker.socket
sudo systemctl start docker.socket
sudo systemctl start docker
docker info
```

## Possible issues and solutions

### 1) Insufficient memory

Check available RAM:

```bash
free -m
```

### 2) Docker fails to start

Confirm Docker is using the expected root directory:

```bash
docker info | grep "Docker Root Dir"
```

### 3) Data loss after reboot

Because `tmpfs` is RAM-backed, all data will be wiped on reboot. Use this approach **only for temporary files and cache**, not for persistent container storage.

## Conclusion

Running **GitLab Runner** and **Docker** in RAM can significantly accelerate CI/CD by reducing disk I/O, but it requires sufficient memory. It is best for **temporary data** and **faster builds**, and it is not suitable for **persistent container storage**.
