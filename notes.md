# Docker and Container Ecosystem

## 1. What is a Container? (Docker, containerd, CRI-O, runC)

- A **container** is a lightweight, portable unit that packages application code along with dependencies and libraries, sharing the host OS kernel.
- Contrasts with **virtual machines (VMs)** which virtualize hardware and include an entire OS instance for each VM.

### Docker vs containerd vs CRI-O vs runC

- **Docker**: High-level container platform with CLI, API, and daemon managing images, lifecycle, and networking.
- **containerd**: Core container runtime developed by Docker, handles image transfer/storage and container lifecycle.
- **CRI-O**: Lightweight container runtime implementation of Kubernetes Container Runtime Interface (CRI), uses lower-level runtimes like runC.
- **runC**: Low-level OCI-compliant container runtime that actually creates and runs containers.

References:  
- https://phoenixnap.com/kb/docker-vs-containerd-vs-cri-o  
- https://www.tutorialworks.com/difference-docker-containerd-runc-crio-oci/

## 2. Docker vs Virtual Machines

- VMs virtualize an entire OS including kernel and application layers.
- Containers share the host OS kernel and isolate only the application layer, resulting in more lightweight and faster startup times.

## 3. Docker Architecture (Buildah related)

- For building container images, tools like **Buildah** provide advanced build capabilities and integrate with OCI container standards.
- Buildah allows building container images without requiring a daemon, improving scripting and automation workflows.

Tutorials:  
- https://buildah.io/  
- https://github.com/containers/buildah/tree/main/docs/tutorials

## 4. Main Docker Commands

- `docker help ` – Show help for a specific Docker command.
- `docker pull ` – Download an image from a registry.
- `docker images` / `docker image ls` – List local images.
- `docker run  [command]` – Start a container from an image; pulls image if missing.  
  Options:  
  - `-d` runs container detached (background).  
  - `-p :` maps ports.  
  - `-n ` names the container.
- `docker ps` – List running containers; `-a` shows all containers.
- `docker stop/start ` – Stop or start a container.

## 5. Docker Debug Commands

- `docker logs ` – View logs; `-f` to follow live output.
- `docker exec -it  ` – Execute interactive command inside a running container (e.g., `/bin/bash`).

## 6. Developing with Docker (Mongo Example)

- Pull official images:  
  ```bash
  docker pull mongo
  docker pull mongo-express
  ```
- Create a Docker network for container communication:  
  ```bash
  docker network create mongo-network
  ```
- Run containers attached to this network, passing environment variables and port mappings. Example MongoDB and mongo-express containers configured appropriately.

- Containers on the same Docker network can communicate via container names without explicit IP addressing.

- Run the Node.js app locally, connecting it to MongoDB in Docker containers:
  ```bash
  cd demo-projects/developing-with-docker/app
  npm install
  npm start
  ```
- Access the app and the mongo-express UI via mapped ports on `localhost`.

## 7. Docker Compose (Orchestrating Multiple Containers)

- Use a `docker-compose.yaml` file to define multiple services, networks, and volumes declaratively.
- Example YAML for mongo, mongo-express, and custom app containers attached to a shared network with environment variables and port bindings.
- Commands:  
  - Start containers:  
    ```bash
    docker-compose up
    docker-compose up -d  # detached
    ```
  - Stop containers and remove network:  
    ```bash
    docker-compose down
    ```

## 8. Dockerfile – Build Your Own Docker Image

**Common Dockerfile instructions:**

```dockerfile
FROM                     # Base image
ENV =                   # Set environment variable
RUN                  # Run a command during build
COPY     # Copy files
CMD ["cmd", "arg"]                  # Default command on container start
```

**Build an image:**

```bash
docker build -t : .
```

**Cross-container communication notes:**

- Docker containers cannot reach `localhost` of other containers.
- On Mac/Windows Docker Desktop, use `host.docker.internal` to reach host services.
- On Linux, add `--add-host=host.docker.internal:host-gateway` to `docker run` or in docker-compose use:

```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

to enable cross-container networking with host.

## 9. Private Docker Registry (AWS ECR Example)

- Setup AWS CLI and credentials; create IAM user with appropriate permissions.
- Create a private Elastic Container Registry (ECR) repository in AWS Console.
- Authenticate Docker CLI with ECR:

```bash
aws ecr get-login-password --region  | docker login --username AWS --password-stdin .dkr.ecr..amazonaws.com
```

- Build and tag Docker image with the ECR repository URI:

```bash
docker build -t user-profile:1.0.0 .
docker tag user-profile:1.0.0 .dkr.ecr..amazonaws.com/user-profile:1.0.0
```

- Push the image:

```bash
docker push .dkr.ecr..amazonaws.com/user-profile:1.0.0
```

## 10. Deploying Docker Applications on a Server (Using Docker Compose)

- Extend your existing `docker-compose.yaml` with your application service, specifying image, network, ports.
- Ensure all services share the same Docker network so service discovery using container names works internally.
- Run compose on server:

```bash
docker-compose up -d
```

## 11. Docker Volumes – Persisting Data

- Volumes types:
  - **Host volumes:** Bind-mount host directories  
    `-v /host/path:/container/path`
  - **Anonymous volumes:** Docker creates a volume with no explicit name  
    `-v /container/path`
  - **Named volumes:** Docker-created volumes referenced by name  
    `-v volume-name:/container/path`
- Named volumes are preferred for persistent container data, easily shared between containers.
- Define named volumes in docker-compose under `volumes:` and use in services.

## 12. Creating Docker Hosted Repository on Nexus

- In Nexus UI, create a repository of type **docker (hosted)**.
- Create a specific **role** with `nx-repository-view-docker-docker-hosted-*` privilege and assign it to a Nexus user for `docker login` access.
- Enable **HTTP** access on a custom port (e.g., 8083), and add corresponding firewall rule for port 8083.
- Activate **Docker Bearer Token Realm** in Nexus Security realms for authentication tokens.
- Configure Docker clients to allow insecure registries over HTTP by modifying daemon config (Linux/macOS).
- Login with:

```bash
docker login :8083
```

- Tag and push images:

```bash
docker tag user-profile:1.1.0 :8083/user-profile:1.1.0
docker push :8083/user-profile:1.1.0
```

## 13. Deploying Nexus as Docker Container

- Create a DigitalOcean Droplet with sufficient RAM and disk space.
- Install Docker (`apt update` + `snap install docker`).
- Create a Docker volume for Nexus data:

```bash
docker volume create --name nexus-data
```

- Run Nexus container:

```bash
docker run -d -p 8081:8081 --name nexus -v nexus-data:/nexus-data sonatype/nexus3
```

- Access Nexus UI at:

```
http://:8081
```

- Docker volume data path on the host can be found with:

```bash
docker inspect nexus-data
```

## 14. Docker Best Practices

- Use official base images where possible.
- Specify explicit image versions to avoid surprises.
- Prefer small base images (e.g., Alpine variants).
- Optimize Dockerfile layering for efficient cache usage.
- Use `.dockerignore` to exclude unnecessary files from images.
- Use **multi-stage builds** to keep images slim by separating build environment from runtime.
- Run containers with the least privilege by creating and switching to a non-root user.
- Regularly scan images for vulnerabilities:

```bash
docker scan 
```
