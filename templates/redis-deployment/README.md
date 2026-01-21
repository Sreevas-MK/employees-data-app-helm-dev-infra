# Redis Deployment

This Helm chart deploys **Redis** as an in-memory data store inside a Kubernetes cluster.
Redis is used by the Flask application for caching and fast key–value access, improving application performance and reducing load on MySQL.

The deployment is designed for a **development environment**, with a strong focus on persistence, security, controlled scheduling, and resource optimization.

## Overview

Redis is deployed as a **single-replica Deployment** backed by persistent storage.
Although Redis is often used as an ephemeral cache, this setup enables **data persistence** using append-only files (AOF), ensuring Redis data survives pod restarts.

All Redis resources are created inside the Kubernetes namespace defined in `values.yaml`.

## Redis Configuration

Redis configuration is provided through a **ConfigMap** that contains a custom `redis.conf` file.

Key configuration settings include:

* Persistent storage directory set to `/data`
* Append-only file (AOF) enabled
* Memory limit enforced
* Least Recently Used (LRU) eviction policy enabled

The configuration file is mounted directly into the Redis container and used during startup.

## Pod Scheduling and Node Affinity

Redis is pinned to a **specific worker node** using `nodeAffinity`.
This ensures:

* Redis always runs on the same node
* HostPath storage remains consistent
* Predictable storage behavior across restarts

The target node is configurable through `values.yaml`.

## Persistent Storage

Redis data is stored on a **PersistentVolume (PV)** backed by a host directory.
A **PersistentVolumeClaim (PVC)** binds the Redis pod to this volume.

Storage characteristics:

* Access mode: ReadWriteOnce
* Reclaim policy: Retain
* Storage type: hostPath

Deleting the Redis pod or PVC does not delete the stored data from the host unless manually removed.

## Init Container and Permissions

Redis runs as a **non-root user (UID 999)** for security reasons.
To ensure Redis can write to the persistent volume, an **initContainer** runs before the main container starts.

The init container:

* Runs briefly as root
* Fixes ownership and permissions on the data directory
* Exits before Redis starts

This pattern allows secure runtime execution without compromising filesystem access.

## Security Context

Redis is hardened using strict security settings:

* Runs as a non-root user and group
* Privilege escalation disabled
* All Linux capabilities dropped
* Uses the RuntimeDefault seccomp profile
* Uses `fsGroup` to ensure volume access

These settings align with Kubernetes security best practices.

## Health Checks

Redis uses **TCP-based liveness and readiness probes** on port 6379.

* Liveness probe restarts the container if Redis becomes unresponsive
* Readiness probe ensures traffic is only sent when Redis is fully ready

Probe delays are increased to account for volume initialization and permission setup.

## Redis Service

A Kubernetes **ClusterIP Service** exposes Redis internally within the cluster.

This service:

* Provides a stable DNS name
* Allows application pods to connect without knowing pod details
* Is not exposed outside the cluster

Flask connects to Redis using this service name.

## Resource Management

Resource requests and limits for Redis are defined in `values.yaml`.
This prevents Redis from consuming excessive CPU or memory and ensures predictable performance.

## Vertical Pod Autoscaler (VPA)

A **Vertical Pod Autoscaler** is enabled for Redis.

The VPA:

* Monitors actual CPU and memory usage
* Automatically adjusts resource requests
* Helps avoid under-provisioning or over-provisioning

This reduces manual tuning effort during development.

---

