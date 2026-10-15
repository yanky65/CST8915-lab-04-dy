# CST8915 Lab 4 - Introduction to Docker

**Student:** David Yin

## Demo Video

[Watch the Part 6 demo on YouTube](https://youtu.be/Pt3duQFr1xM)

## Reflection Questions

### 1. What are the main differences between a Docker image and a Docker container?

A Docker image is a read-only template containing an application and its dependencies. A container is an instance of that image, with its own processes and writable layer. Multiple containers can use the same image while keeping their changes separate. Containers can be running or stopped.

### 2. Explain how Docker's layered architecture improves efficiency.

Docker reuses unchanged image layers, which reduces storage use and download time. It also caches build steps to speed up rebuilds. In this lab, copying requirements before the application code allows Docker to reuse the dependency installation step when only `app.py` changes.

### 3. Why does each container get its own writable layer?

The writable layer lets a container change files without changing the image or other containers. This keeps each container's data separate while allowing containers to share the same image layers. Deleting a container removes its writable layer, so data that needs to survive should use a volume.

### 4. What are the benefits of using Docker Compose over running containers individually?

Docker Compose defines services, networks, ports, and volumes in one file. It makes the application easier to start, stop, inspect, and scale with fewer commands. In this lab, Compose connects Flask to Redis using the service name and waits for Redis's health check before starting the web service.
