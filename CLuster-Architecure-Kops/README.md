# Lab 01 – Kubernetes Cluster Architecture & AWS Setup using kOps

\<p align="center">   \<img src="[https://img.shields.io/badge/Kubernetes-Hands--On-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white](https://img.shields.io/badge/Kubernetes-Hands--On-326CE5?style=for-the-badge\&logo=kubernetes\&logoColor=white)" />   \<img src="[https://img.shields.io/badge/AWS-Infrastructure-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white](https://img.shields.io/badge/AWS-Infrastructure-FF9900?style=for-the-badge\&logo=amazonaws\&logoColor=white)" />   \<img src="[https://img.shields.io/badge/kOps-Cluster%20Management-blue?style=for-the-badge](https://img.shields.io/badge/kOps-Cluster%20Management-blue?style=for-the-badge)" /> \</p>

## Overview

This lab focuses on understanding Kubernetes architecture and preparing an AWS environment for provisioning a self-managed Kubernetes cluster using kOps.

The objective is to understand container orchestration, prepare an EC2 management machine, configure AWS IAM authentication, install Kubernetes management tools, and establish an S3 bucket as the kOps state store.

**Platform:** AWS
**Operating System:** Amazon Linux 2023
**Cluster Management Tool:** kOps
**Kubernetes CLI:** kubectl

---

# Part 1 – Kubernetes Architecture & Fundamentals

## 1. Monolithic vs Microservices Architecture

### Monolithic Architecture

A monolithic application is developed as a single application where multiple functionalities are packaged and deployed together.

### Microservices Architecture

Microservices architecture divides an application into smaller, independently deployable services.

Example of an e-commerce application:

- Authentication Service
- User Management Service
- Product Service
- Payment Service
- Order Management Service

### Comparison

| Monolithic                 | Microservices                 |
| -------------------------- | ----------------------------- |
| Single application         | Multiple independent services |
| Tightly coupled components | Loosely coupled services      |
| Single deployment unit     | Independent deployments       |
| Application-wide scaling   | Individual service scaling    |

## 2. Containerization and Orchestration

Docker provides containerization by packaging applications and their dependencies into portable containers.

However, managing multiple containers across multiple servers introduces challenges:

- Manual deployment.
- Container failure recovery.
- Application scaling.
- Networking between containers.
- Workload management.

Kubernetes addresses these challenges through container orchestration.

## 3. Kubernetes Architecture

A Kubernetes cluster consists of two major components:

### Control Plane

The control plane manages the overall cluster.

| Component                | Responsibility                                       |
| ------------------------ | ---------------------------------------------------- |
| API Server               | Handles Kubernetes API requests                      |
| etcd                     | Stores cluster data and state                        |
| Scheduler                | Assigns Pods to worker nodes                         |
| Controller Manager       | Maintains the desired state                          |
| Cloud Controller Manager | Integrates with cloud infrastructure when configured |

### Worker Nodes

Worker nodes execute application workloads.

| Component         | Responsibility                                 |
| ----------------- | ---------------------------------------------- |
| kubelet           | Ensures assigned Pods are running              |
| kube-proxy        | Implements Service networking rules where used |
| Container Runtime | Executes containers                            |

## 4. Application Deployment Workflow

**Scenario:** A developer requests deployment of a web application with three replicas.

1. Developer submits a Deployment manifest.
2. API Server validates the request.
3. Desired configuration is stored in etcd.
4. Deployment controller observes the requested state.
5. ReplicaSet controller ensures three Pods are created.
6. Scheduler selects suitable worker nodes.
7. Kubelet receives the Pod specifications.
8. Container runtime starts the containers.
9. Kubernetes monitors the application state.

**Expected result:** Kubernetes works toward maintaining three application replicas.

---

# Part 2 – AWS Management Environment & IAM Configuration

## 1. EC2 Management Machine

Created an AWS EC2 instance to act as the Kubernetes management machine.

### Instance Configuration

| Configuration | Value                             |
| ------------- | --------------------------------- |
| Instance Name | kops-client                       |
| AMI           | Amazon Linux 2023                 |
| Architecture  | 64-bit x86                        |
| Instance Type | t3.micro                          |
| Storage       | 30 GB gp3                         |
| Access        | SSH                               |
| Purpose       | Kubernetes cluster administration |

The management instance is used to execute AWS CLI, kOps and kubectl operations.

## 2. IAM Role Configuration

Created an IAM role to allow the EC2 management machine to communicate with AWS services.

| Configuration   | Value                    |
| --------------- | ------------------------ |
| Role Name       | kops-admin-role          |
| Trusted Entity  | EC2                      |
| Attached Policy | AdministratorAccess      |
| Association     | kops-client EC2 instance |

### Why IAM Roles?

IAM roles allow EC2 instances to obtain temporary security credentials without storing permanent AWS access keys.

For this learning environment, AdministratorAccess was used to simplify infrastructure provisioning. Production environments should use least-privilege permissions.

## 3. AWS Management Tools

Prepared the management environment with the following tools:

| Tool    | Purpose                                    |
| ------- | ------------------------------------------ |
| AWS CLI | Communicates with AWS services             |
| kOps    | Provisions and manages Kubernetes clusters |
| kubectl | Communicates with Kubernetes clusters      |

### AWS Authentication Verification

Verified that the EC2 instance successfully authenticated using its attached IAM role.

**Expected identity format:**

`arn:aws:sts::ACCOUNT-ID:assumed-role/kops-admin-role/SESSION-NAME`

Temporary IAM credentials were used instead of manually configured permanent access keys.

---

# Part 3 – S3 State Store Configuration & Final Validation

## 1. Create S3 State Store

Created a dedicated Amazon S3 bucket to maintain kOps cluster configuration and state.

### S3 Bucket Configuration

| Configuration   | Value                    |
| --------------- | ------------------------ |
| Bucket Name     | kubernetes-lab1-15-11-26 |
| Region          | us-east-1 (N. Virginia)  |
| Bucket Type     | General Purpose          |
| Versioning      | Enabled                  |
| Encryption      | SSE-S3 (AES256)          |
| Public Access   | Blocked                  |
| Bucket Policy   | No custom policy         |
| Lifecycle Rules | Not configured           |
| Tags            | Not configured           |

## 2. S3 Security Configuration

### Public Access Protection

All four Block Public Access settings were enabled:

- BlockPublicAcls
- IgnorePublicAcls
- BlockPublicPolicy
- RestrictPublicBuckets

### Encryption

Enabled default server-side encryption using SSE-S3 (AES256).

### Versioning

Enabled bucket versioning to retain previous versions of objects and support recovery from accidental overwrites.

## 3. S3 State Store – Purpose

The kOps state store maintains cluster configuration and state information.

It provides persistent storage independently of the EC2 management machine.

**Important distinction:**

| S3 State Store                              | Kubernetes etcd                      |
| ------------------------------------------- | ------------------------------------ |
| Used by kOps                                | Used by Kubernetes                   |
| Stores kOps cluster configuration and state | Stores live Kubernetes cluster data  |
| Independent of management EC2               | Part of the Kubernetes control plane |

## 4. AWS Infrastructure Validation

Completed final validation of the AWS environment.

| Verification                | Result |
| --------------------------- | ------ |
| AWS assumed-role identity   | PASSED |
| EC2-to-S3 access            | PASSED |
| S3 bucket location          | PASSED |
| S3 versioning               | PASSED |
| S3 encryption               | PASSED |
| S3 public access protection | PASSED |
| kOps installation           | PASSED |
| kubectl installation        | PASSED |
| AWS CLI installation        | PASSED |
| Temporary IAM credentials   | PASSED |
| kOps state store            | PASSED |

### Final Verification Summary

**11 out of 11 verification checks passed.**

The AWS management environment and S3 state store were successfully validated.

---

# Key Learnings

Through this lab, I learned:

- Kubernetes cluster architecture and component responsibilities.
- Differences between monolithic and microservices architectures.
- Containerization and container orchestration concepts.
- AWS EC2 infrastructure preparation.
- IAM roles and temporary AWS credentials.
- AWS CLI, kOps and kubectl setup.
- S3 state-store configuration.
- S3 security, versioning and encryption.
- AWS infrastructure validation.

---

# Lab Completion Status

| Part   | Description                       | Status    |
| ------ | --------------------------------- | --------- |
| Part 1 | Kubernetes Architecture           | Completed |
| Part 2 | AWS Management Environment & IAM  | Completed |
| Part 3 | S3 State Store & Final Validation | Completed |

**Overall Result:** Lab 01 successfully completed.

---

## Next Lab

**Lab 02 – Kubernetes Cluster Provisioning using kOps**

Planned activities:

- Configure kOps cluster specifications.
- Configure the Kubernetes cluster name.
- Connect kOps with the S3 state store.
- Configure AWS networking and DNS requirements.
- Provision the Kubernetes control plane and worker nodes.
- Validate the deployed cluster.

---

**Project:** Kubernetes Hands-on Practice
**Lab:** 01 – Cluster Architecture & AWS Setup using kOps
**Platform:** AWS
**Status:** Completed
