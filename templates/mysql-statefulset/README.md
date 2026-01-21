# MySQL StatefulSet Deployment

This Helm chart deploys a **stateful MySQL database** inside a Kubernetes cluster using a **StatefulSet**.
It is designed for a **development environment**, focusing on data persistence, controlled scheduling, security best practices, and backup automation.

The MySQL setup supports:

* Persistent storage using hostPath volumes
* Controlled pod placement using node affinity
* Secure credential management
* Automated backups using a CronJob
* Resource tuning with Vertical Pod Autoscaler (VPA)

## Overview

MySQL is deployed as a **single-replica StatefulSet**, ensuring stable network identity, predictable storage attachment, and safe restarts.
The database stores its data on a **PersistentVolume (PV)** backed by the host filesystem, ensuring data survives pod restarts.

All MySQL-related resources run in the same Kubernetes namespace defined in `values.yaml`.

## Node Affinity and Scheduling

The MySQL pod is **pinned to a specific worker node** using `nodeAffinity`.
This ensures that:

* The pod always runs on the same node
* The hostPath storage location remains consistent
* Backups and data volumes are aligned with the same node

The node name is configurable via `values.yaml`.


## Persistent Storage (MySQL Data)

MySQL data is stored on a **PersistentVolume (PV)** backed by a host directory.
A **PersistentVolumeClaim (PVC)** binds the MySQL pod to this volume.

Key characteristics:

* Storage type: hostPath
* Access mode: ReadWriteOnce
* Reclaim policy: Retain

Even if the MySQL pod or PVC is deleted, the data remains on the host filesystem unless manually removed.

## Database Initialization and Hardening

A **ConfigMap** is used to initialize database privileges and enforce security settings during first startup.

Initialization actions include:

* Creating the application database
* Creating a dedicated database user
* Enforcing SSL for database connections
* Removing default and test users
* Dropping unnecessary system databases

This logic runs automatically when MySQL initializes for the first time.

## Security Context and Permissions

MySQL runs as a **non-root user (UID 999)** to follow container security best practices.
Filesystem permissions are handled using initContainers that run briefly as root to prepare the data directory.

Security features include:

* Dropped Linux capabilities
* Disabled privilege escalation
* RuntimeDefault seccomp profile
* Group-based filesystem access using `fsGroup`

## Secrets and Configuration

Sensitive credentials are stored in a Kubernetes **Secret**, while non-sensitive configuration is stored in **ConfigMaps**.

Secrets include:

* MySQL root password
* Application user password

ConfigMaps include:

* Database name
* Application database user

These values are injected into the MySQL container as environment variables.

## MySQL Services

Two Kubernetes services are created for MySQL:

1. **Headless Service**

   * Used by the StatefulSet
   * Provides stable DNS identity
   * Enables direct pod access if required

2. **ClusterIP Service**

   * Used by application pods and backup jobs
   * Simplifies connectivity
   * Abstracts pod identity from clients

Applications always connect through the ClusterIP service.

## Health Probes

MySQL uses **TCP-based liveness and readiness probes** on port 3306.

* Liveness probe restarts the container if MySQL becomes unresponsive
* Readiness probe ensures traffic is only sent when MySQL is fully ready

These probes improve reliability and safe restarts.

## Automated MySQL Backups

A Kubernetes **CronJob** runs scheduled MySQL backups daily.
The backup job connects to the MySQL service and stores SQL dumps on a persistent backup volume.

Backup behavior:

* Runs on the same node as MySQL
* Stores backups in a hostPath-backed volume
* Automatically deletes backups older than two days
* Uses non-root execution with controlled permissions

Backup storage is isolated from MySQL data storage.

## Backup Storage

Backup files are stored in a separate **PersistentVolume and PersistentVolumeClaim**.
This separation ensures:

* Clear distinction between live data and backups
* Easier cleanup and retention control
* Reduced risk of data corruption

The backup path is configurable through `values.yaml`.

## Vertical Pod Autoscaler (VPA)

A **Vertical Pod Autoscaler** is enabled for the MySQL StatefulSet.
The VPA automatically adjusts CPU and memory requests based on observed usage.

This helps:

* Prevent over-provisioning
* Improve performance stability
* Reduce manual tuning effort
---

