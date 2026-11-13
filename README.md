# Lab 01 – Kubernetes Cluster Architecture & AWS kOps Setup

## Overview

**Scenario:** You have joined a company as a Junior DevOps Engineer. The company currently runs applications inside Docker containers on individual EC2 instances. As the number of applications increases, managing containers, scaling workloads, networking, and availability manually becomes difficult. Your responsibility is to prepare an AWS environment for deploying a self-managed Kubernetes cluster using **kOps**.

**Goal:** Build the AWS foundation required to provision and manage a Kubernetes cluster using kOps.

---

## Learning Objectives

By completing this lab, you should understand:

- Why Kubernetes is required for managing multiple containers
- The difference between monolithic and microservices architecture
- Kubernetes control plane and worker node architecture
- What kOps does and how it communicates with AWS
- How to prepare an EC2 management machine
- How IAM permissions allow kOps to provision AWS resources
- Why an S3 bucket is required as a kOps state store

---

## Hands-On Tasks

### Task 1 – Understand the Application Architecture

**Scenario:** A company has an application consisting of authentication, payment, user management, and product services.

| Concept | Description |
|---|---|
| **Monolithic Architecture** | Application components are packaged and deployed together |
| **Microservices Architecture** | Application functionality is divided into independently deployable services |
| **Docker** | Packages each application service into containers |
| **Kubernetes** | Automates deployment, scaling, networking, and management of containers |

**Challenge:** Explain why managing 50 containers across five EC2 instances manually becomes difficult and how Kubernetes helps solve these problems.

---

### Task 2 – Understand Kubernetes Cluster Architecture

Study the architecture of a Kubernetes cluster and identify the responsibilities of its **control plane** and **worker nodes**.

| Component | Responsibility |
|---|---|
| **API Server** | Entry point for Kubernetes API requests |
| **etcd** | Stores Kubernetes cluster data |
| **Scheduler** | Assigns Pods to suitable worker nodes |
| **Controller Manager** | Maintains the desired cluster state |
| **Kubelet** | Manages Pods on worker nodes |
| **Kube-proxy** | Implements service networking rules |
| **Container Runtime** | Runs containers |

**Challenge:** Explain what happens inside Kubernetes from the moment a developer submits a deployment request (with three replicas) until the containers start running. Cover the interaction between the API Server, etcd, Scheduler, Controller Manager, and worker nodes.

---

### Task 3 – Prepare the kOps Management Machine

Create an AWS EC2 instance that will act as your Kubernetes cluster management machine.

| Configuration | Requirement |
|---|---|
| **Instance Name** | `kops-client` |
| **Operating System** | Amazon Linux 2023 |
| **Instance Type** | Small burstable instance suitable for administration |
| **Storage** | 30 GB gp3 |
| **Key Pair** | Your existing regional key pair |
| **Network** | Appropriate VPC and subnet |
| **Security** | SSH access restricted to your own IP |

**Important Notes:**
- AWS ended support for Amazon Linux 2 on June 30, 2026. **Use Amazon Linux 2023 instead.**

**Key Concepts to Understand:**
- Why do you need a separate management machine?
- Is this machine part of the Kubernetes worker nodes?
- What is the purpose of an SSH key pair?
- Why should SSH access not be open to the entire internet?
- What happens if the management machine is terminated?

---

### Task 4 – Configure AWS IAM Permissions

The kOps management machine needs permission to communicate with AWS and provision infrastructure resources.

**Key Concepts to Understand:**
- IAM users versus IAM roles
- Access keys and secret access keys
- Temporary security credentials
- IAM policies and permissions
- EC2 instance profiles
- Least privilege principle

**Assignment:** Create an **IAM role** that the EC2 instance can assume and associate it with your `kops-client` instance. Ensure it has the AWS permissions required by kOps.

> ⚠️ **Important Security Challenge:** The tutorial suggests creating an IAM user with administrative access and configuring permanent credentials using AWS CLI. For this lab, **prefer an EC2 instance role with temporary credentials**. Avoid storing long-lived access keys on your instance or in GitHub.

---

### Task 5 – Understand and Create the S3 State Store

kOps needs a centralized location to maintain the desired configuration and state of the Kubernetes cluster.

**Key Concepts to Understand:**
- What is an S3 bucket?
- What is a kOps state store?
- Why does kOps need persistent state?
- What happens if you lose access to the state store?
- Why is versioning useful?
- Why should the bucket remain private?

**Assignment:** Create a dedicated S3 bucket for this Kubernetes cluster with the following configuration:

- Globally unique bucket name
- Appropriate AWS region
- Block all public access
- Enable bucket versioning
- Enable default encryption
- Restrict access to the appropriate IAM identity

> The S3 state store maintains kOps cluster configuration and state. It is **not** the same as the Kubernetes etcd database or application storage.

---

## Final Lab Challenge

After completing all five tasks, be prepared to answer your team lead's question:

> *"You have prepared the AWS environment. Are we ready to provision a Kubernetes cluster using kOps?"*

Prepare an explanation covering:

1. **What infrastructure have you prepared?**
2. **How will kOps authenticate with AWS?**
3. **Where will kOps store the cluster configuration?**
4. **How will kOps provision EC2 instances?**
5. **What is the difference between the kOps management machine and the Kubernetes control plane?**
6. **What resources could generate AWS charges?**

---

## Lab Completion Checklist

- [ ] Task 1: Understand the application architecture
- [ ] Task 2: Understand Kubernetes cluster architecture
- [ ] Task 3: Prepare the kOps management machine
- [ ] Task 4: Configure AWS IAM permissions
- [ ] Task 5: Understand and create the S3 state store
- [ ] Final Lab Challenge: Prepare explanation for team lead
- [ ] Explain architecture without referring to the tutorial
- [ ] Show the AWS resources created
- [ ] Justify IAM and S3 security configurations

---

## Next Steps

The actual kOps installation, cluster creation, configuration, and validation will be covered in the next hands-on lab after you complete this foundation.

> **Reminder:** Make sure you understand **why** every resource exists, not just **how** to create it. Don't rush to the next lab.
