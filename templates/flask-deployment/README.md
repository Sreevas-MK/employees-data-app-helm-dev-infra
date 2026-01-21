# Flask Application Deployment

This Helm chart deploys a Flask-based web application that connects to MySQL and Redis services running inside the same Kubernetes cluster.
The deployment is designed for a **development and learning environment**, with emphasis on clarity, stability, and Kubernetes best practices rather than production hardening.

## Overview

The Flask application is deployed as a Kubernetes **Deployment** and exposed internally using a **ClusterIP Service**.
Autoscaling is enabled using a **Horizontal Pod Autoscaler (HPA)** based on CPU and memory utilization.

The application depends on:

* **MySQL** for persistent data storage
* **Redis** for caching and fast in-memory access

Startup ordering is handled using an **initContainer** that waits for MySQL to become available before the Flask application starts.

## Flask Deployment

The Flask application runs as a Kubernetes **Deployment**, allowing multiple replicas of the application to run simultaneously.
This ensures high availability and enables scaling based on load.

Each Flask pod:

* Runs the application container
* Exposes port `3000`
* Uses non-root security settings
* Mounts a temporary writable filesystem using `emptyDir`

The container image is configurable through `values.yaml`, making it easy to update application versions without changing templates.

## Application Startup Dependency Handling

Before the Flask container starts, an **initContainer** named `wait-for-mysql` is executed.
This container continuously checks whether the MySQL service (`mysql-access:3306`) is reachable.

Only after MySQL is confirmed to be running does Kubernetes start the main Flask application container.
This prevents application crashes due to database unavailability during startup.

## Health Checks

The Flask application exposes a `/status` endpoint that is used for both **liveness** and **readiness probes**.

* **Liveness probe** ensures Kubernetes restarts the container if the application becomes unhealthy.
* **Readiness probe** ensures traffic is sent to the pod only after it is fully ready to serve requests.

These probes improve reliability and smooth rolling updates.


## Resource Management

Resource requests and limits are defined to ensure fair scheduling and prevent resource starvation.

Each Flask pod:

* Requests a minimum amount of CPU and memory
* Has upper limits to prevent excessive usage

These values are configurable via `values.yaml`.

## Horizontal Pod Autoscaling (HPA)

The Flask deployment is automatically scaled using a **Horizontal Pod Autoscaler**.

Scaling is based on:

* Average CPU utilization
* Average memory utilization

The number of replicas dynamically adjusts between the configured minimum and maximum values, ensuring better performance under load while conserving resources during low usage.

## Environment Configuration

The Flask application is configured using Kubernetes **ConfigMaps** and **Secrets**.

Environment variables include:

* MySQL connection details (host, port, database, user, password)
* Redis host and port

Sensitive data such as database passwords are securely injected from Kubernetes Secrets.


## Flask Service

A Kubernetes **ClusterIP Service** is created for the Flask application.
This service provides a stable internal endpoint that load-balances traffic across all running Flask pods.

The service is intended to be accessed through:

* An Ingress controller (such as Traefik), or
* Other internal Kubernetes services

It is not exposed directly to the external network.


## Configuration via values.yaml

All important parameters are configurable through `values.yaml`, including:

* Application name
* Container image and tag
* Resource limits
* Autoscaling thresholds
* Namespace

This separation allows the same templates to be reused across environments with minimal changes.

## Intended Usage

This Flask deployment is intended for:

* Development environments
* Kubernetes learning and experimentation
* Demonstrating best practices such as probes, autoscaling, and secure configuration

It is not designed as a production-ready deployment without further hardening.

---

