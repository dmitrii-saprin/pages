
* [Docker Internals Basics](#docker-internals-basics)
* [Why a Pause Container Is Needed](#why-a-pause-container-is-needed)
* [ENTRYPOINT vs CMD in a Dockerfile](#entrypoint-vs-cmd-in-a-dockerfile)
* [Docker Architecture and Interaction with CRI](#docker-architecture-and-interaction-with-cri)
* [What Is Docker Cache](#what-is-docker-cache)
* [Docker Multi-Stage Build](#docker-multi-stage-build)
* [Docker Basics](#docker-basics)

---
[↑ Top ](#top)
### Docker Internals Basics

Docker relies on several core Linux technologies: cgroups to limit and account for resources like CPU, memory, disk, and network; namespaces to isolate the runtime environment (process tree, networking, mount points, PID space, and more); and union filesystems (for example, OverlayFS) to implement layered, reusable image storage.

The Docker Engine is typically described as a stack of components where dockerd is the main daemon that receives API/CLI requests and orchestrates everything, containerd is the lower-level runtime responsible for managing container lifecycle, and runc is the OCI-compliant runtime that actually starts containers as isolated Linux processes.

In Docker terms, an image is an immutable, layered artifact that contains everything needed to run an application, while a container is a running instance of that image in an isolated environment.

For networking, Docker can attach containers to different network modes such as bridge (default), host (sharing the host’s network stack), overlay (for multi-node scenarios), or none (no network).

For persistent data, Docker uses volumes to keep data across container restarts and bind mounts to mount directories from the host into a container.

From a security perspective, Docker’s isolation can be strengthened with separate user namespaces, Linux security modules like AppArmor/SELinux, and capabilities to reduce the privileges available to processes running inside containers.

---
[↑ Top ](#top)
### Why a Pause Container Is Needed
A pause container is a small utility container that Kubernetes starts first inside every Pod. It doesn’t run your application code, but it provides the technical “base” the Pod is built on.

Its main job is to hold the Pod’s network namespace and act as an anchor process for the rest of the containers. Because all other containers in the Pod join the pause container’s namespace, they effectively share the same networking environment — meaning the Pod gets one IP address, and containers can share ports and other namespaces (like IPC and UTS) in a predictable way.

In practice, Kubernetes creates the Pod by starting the pause container, then launches the workload containers so they attach to it (conceptually like --net=container:<pause-container>). If a workload container crashes or restarts, the pause container stays alive, so the Pod’s namespaces don’t get recreated and the Pod doesn’t “disappear” just because an app process restarted.

The pause image itself is intentionally tiny and does almost nothing (often it just sleeps indefinitely). This simplicity improves stability: Kubernetes can manage the Pod lifecycle cleanly, keep isolation between Pods, and implement shared namespaces inside a Pod without coupling that responsibility to any particular application container.

So even though the pause container has no business logic, it’s a core part of Kubernetes Pod architecture that makes the “shared Pod environment” work reliably.

---
[↑ Top ](#top)
### ENTRYPOINT vs CMD in a Dockerfile

ENTRYPOINT and CMD are Dockerfile instructions that control what a container runs when it starts. They look similar, but they serve different purposes: ENTRYPOINT defines the “main executable” of the container, while CMD defines the default command or default arguments that can be replaced at runtime.

With ENTRYPOINT, you’re essentially saying: “this container is meant to run this exact program.” Anything you add after the image name in docker run is treated as arguments passed to the ENTRYPOINT command. If you want to change what actually runs, you usually rebuild the image (or explicitly override it using --entrypoint). For example:


```bash
FROM ubuntu
CMD ["echo", "Hello"]
```

```bash
docker run image-name World
```

The output will be Hello World, because World becomes an argument to the ENTRYPOINT.

With CMD, you’re defining a default command (or arguments) that Docker will use only if you don’t provide something else at runtime. If you do provide a command in docker run, it replaces the CMD entirely. Also, a Dockerfile can only have one effective CMD — the last one wins. For example:


This also prints Hello World, but here the World replaces the default ["echo","Hello"].

Here’s the comparison table in Markdown:

| Feature                      | ENTRYPOINT                       | CMD                   |
|-----------------------------|----------------------------------|-----------------------|
| Primary purpose             | Fixed command                    | Default command       |
| Override at runtime         | No (unless using `--entrypoint`) | Yes                   |
| Effect of runtime arguments | Appended as args to ENTRYPOINT   | Replaces CMD entirely |
| Can be used together        | Yes                              | Yes                   |

In practice: use ENTRYPOINT when the container should always run a specific executable, and use CMD when you want a default that users can override. You can also combine them: ENTRYPOINT defines the main command, and CMD supplies default arguments.

A Dockerfile can have multiple CMD (or ENTRYPOINT) instructions, but only the last one takes effect; earlier ones are overridden.

---
[↑ Top ](#top)
### Docker Architecture and Interaction with CRI

The docker command-line tool is essentially a thin client: it provides a convenient CLI, but all real container management is done by the Docker daemon, dockerd, through its API. When you start a container, the usual flow is that the Docker CLI sends a request to dockerd, dockerd delegates lifecycle operations to containerd, and then containerd invokes runc—the OCI-compliant runtime that actually creates and starts the container as an isolated process.

If you want, you can bypass the Docker CLI entirely and work at lower levels: containers can be managed directly via containerd, or even started using runc itself, which is basically a way to launch a containerized workload as an isolated Linux process without the higher-level Docker tooling.

Conceptually, a container is not a “special thing” from the kernel’s point of view—it’s just a regular Linux process that is isolated using namespaces, controlled with cgroups, and constrained by other kernel mechanisms. Because of that, container processes are visible on the host with normal tools like ps, top, or htop, and they still use standard system calls and require the same kinds of permissions as any other process.

In Kubernetes, the runtime is abstracted behind the Container Runtime Interface (CRI), which lets Kubernetes work with different container runtimes without being tied to Docker. Common CRI implementations include containerd, which integrates natively and is lightweight, and CRI-O, which is often used in OpenShift-oriented ecosystems and fits well with Podman-style workflows.

While Docker is the most familiar tooling, it’s not the only option: for example, podman provides a Docker-compatible CLI but does not require a long-running daemon like dockerd, which can be attractive in security-sensitive environments and is widely used in Red Hat and Fedora ecosystems.

The key point is that the Docker CLI is only one interface on top of a stack of lower-level components (dockerd, containerd, runc) and Linux primitives (namespaces and cgroups). Kubernetes, meanwhile, doesn’t depend on Docker specifically—it relies on CRI and can run containers via containerd, CRI-O, and historically via the deprecated Docker shim.

---
[↑ Top ](#top)
### What Is Docker Cache
Docker Cache (also called the Docker Build Cache) is the mechanism Docker uses to speed up image builds by reusing the output of Dockerfile steps that were already executed before. Instead of re-running every instruction on every build, Docker can reuse previously built layers as long as the instruction and everything it depends on hasn’t changed.

In practice, this makes builds much faster, especially when you make small changes to application code but the earlier steps (like installing OS packages or dependencies) remain the same. Cache reuse means Docker avoids repeating expensive operations such as apt install or npm install when it can prove nothing relevant has changed.

The cache is automatically invalidated (“busted”) when Docker detects a change that affects a layer. Typical reasons are changes in source code, changes to files copied with COPY, or changes in the Dockerfile instructions themselves. That’s important because it ensures the image still ends up containing fresh, correct content rather than reusing stale build artifacts.

A big practical tool here is .dockerignore: it lets you exclude irrelevant files and folders from the build context, which reduces how much data Docker has to send to the builder and also prevents accidental cache busting due to changes in files that shouldn’t matter for the build.

You can control cache behavior explicitly: for example, building with --no-cache forces Docker to rebuild every step from scratch (docker build --no-cache -t my-image .). During a normal build, Docker also tells you when a step is reused, typically showing messages like “Using cache,” so you can see which layers are being cached.

Overall, Docker Cache is one of the main reasons container builds can be efficient in CI/CD, reducing both time and compute cost—but it works best when the Dockerfile is structured thoughtfully (for example, copying frequently changing files too early in the Dockerfile can cause unnecessary cache invalidation and slow builds).

---
[↑ Top ](#top)
### Docker Multi-Stage Build
A Docker Multi-Stage Build is a Dockerfile pattern that lets you produce smaller, cleaner images by splitting the build into several stages. Instead of installing build tools and dependencies directly into the final runtime image, you do the heavy work in an earlier stage (for example, compiling code or producing static assets) and then copy only the resulting artifacts into a final stage that contains just what’s needed to run.

The core idea is simple: one Dockerfile can contain multiple FROM sections, and each of those sections is its own stage. Different stages can use different base images and have their own setup steps. When the build is finished, Docker only keeps the final stage as the resulting image, and everything from previous stages is left behind unless you explicitly copy it forward.

This approach has several practical advantages. The final image becomes much smaller because it contains only the necessary runtime artifacts, not compilers, package managers, or temporary build files. It also improves security, because fewer components in the final image means a smaller attack surface and fewer potential vulnerabilities. Dependency management becomes more straightforward: build-time dependencies stay in the build stage, runtime dependencies stay in the runtime stage, and changes are easier to reason about. Builds can also become faster over time, because Docker can cache layers within each stage separately, and the clearer separation of tasks often makes caching more effective. Finally, multi-stage Dockerfiles tend to be easier to maintain because each stage has a single responsibility—build, test, package—rather than one large, messy sequence of steps.

In practice, a common example is building a frontend app in a Node.js stage and then serving only the compiled static files from a minimal Nginx image. The Node stage exists only to produce the build output, while the Nginx stage becomes the actual production image. Overall, multi-stage builds are widely considered a best practice for creating images that are smaller, safer, more readable, and better aligned with efficient caching.

---

[↑ Наверх](#top)
### Docker Basics

Docker is implemented in Go and builds on core Linux kernel features to isolate processes and manage files efficiently. The foundation of containerization in Docker comes from three key building blocks: namespaces, which isolate what a process can “see” (like processes, networking, and filesystem); cgroups (control groups), which enforce resource limits and accounting; and union filesystems, which provide Docker’s layered image model.

Namespaces are what make a container feel like its own machine. When Docker starts a container, it creates separate namespaces so that the container has an isolated view of system resources. For example, the pid namespace isolates process IDs, the net namespace isolates networking, ipc isolates inter-process communication, mnt isolates mounts and filesystem view, and uts isolates hostname and related identifiers. Because of these namespaces, a container can’t directly see or interfere with processes and resources outside its own isolated environment.

Cgroups provide the resource governance layer. They allow Docker to limit and distribute CPU, memory, network, and other resources across containers, and they prevent one container from consuming so much that it destabilizes the host. This is what makes containers behave like “good neighbors” on the same machine, even under load.

Union filesystems are what enable Docker images to be layered. Each Dockerfile instruction typically creates a new layer, and those layers can be cached and reused across builds, which is one reason Docker builds can be fast. Docker supports multiple union filesystem backends such as AUFS, OverlayFS, and btrfs. You can inspect an image’s layers with docker history <image-name>, and you can flatten and move container filesystems using commands like docker export <container-id> | docker import - sample:flat.

At the container startup level, Dockerfile instructions ENTRYPOINT and CMD determine what runs. ENTRYPOINT defines the command that should always execute when the container starts, while CMD provides a default command or arguments that can be overridden at runtime. Together they let you define both the intended executable and flexible defaults.

Finally, it’s important to distinguish Docker images from containers: an image is an immutable, read-only template that contains everything needed to create a container, while a container is a running instance of that image—ultimately just an isolated Linux process controlled by namespaces and cgroups. For practical usage, common best practices include combining commands with && to reduce layers, using .dockerignore to keep the build context clean, leveraging multi-stage builds to produce smaller runtime images, and setting runtime resource limits like --memory and --cpus to prevent noisy-neighbor issues.
