# Phase 1 – Kubernetes Architecture & Fundamentals

## Overview

This phase focuses on understanding Kubernetes architecture, container orchestration, and the transition from monolithic applications to microservices.

As applications grow, managing containers manually across multiple EC2 instances becomes difficult. Kubernetes solves this problem by providing a platform to deploy, scale, manage, and maintain containerized applications.

The objective of this phase is to understand how Kubernetes works internally before setting up a cluster using **kOps on AWS**.

---

## Learning Objectives

By completing this phase, I aim to understand:

- The differences between monolithic and microservices architectures.
- The role of Docker in containerization.
- Why container orchestration is necessary.
- Kubernetes cluster architecture.
- The responsibilities of the control plane and worker nodes.
- How Kubernetes manages application deployments.
- How Kubernetes maintains the desired state of applications.

---

## 1. Monolithic vs Microservices Architecture

### Monolithic Architecture

A monolithic application is structured as a single application where multiple functionalities are developed, packaged, and deployed together.

**Characteristics:**
- Application components are tightly coupled.
- Deployment is generally performed as a single unit.
- Scaling individual components independently is difficult.
- Changes to one component can require redeployment of the application.

### Microservices Architecture

Microservices architecture divides an application into smaller, independent services. Each service handles a specific business functionality.

For example, an e-commerce application can contain:

- Authentication Service
- User Management Service
- Product Service
- Payment Service
- Order Management Service

**Characteristics:**
- Services can be developed and deployed independently.
- Individual services can be scaled independently.
- Services communicate through APIs or messaging systems.
- Failures can be isolated more effectively when services are designed properly.

### Comparison

| Monolithic | Microservices |
|---|---|
| Single application unit | Multiple independent services |
| Tightly coupled components | Loosely coupled services |
| Shared deployment lifecycle | Independent deployment |
| Scaling generally affects the application as a whole | Individual services can be scaled |
| Simpler initial architecture | More distributed-system complexity |

---

## 2. Containerization and Container Orchestration

### Containerization

Containerization packages an application and its dependencies into a portable, isolated environment.

Docker is a popular platform used to build, package, and run containers.

**Problems without orchestration:**
- Manual container deployment.
- Difficult container scaling.
- Manual failure recovery.
- Complicated networking between containers.
- Difficult management across multiple servers.

### Container Orchestration

Container orchestration automates the deployment, scaling, networking, and management of containers across infrastructure.

Kubernetes provides these orchestration capabilities.

| Docker | Kubernetes |
|---|---|
| Builds and runs containers | Orchestrates containerized workloads |
| Creates container images | Deploys and manages Pods |
| Runs containers on hosts | Coordinates workloads across nodes |
| Provides container networking features | Manages cluster-level service networking |

---

## 3. Kubernetes Architecture

A Kubernetes cluster consists of two major parts:

1. Control Plane
2. Worker Nodes

### 3.1 Control Plane

The control plane manages the overall Kubernetes cluster.

Its responsibilities include:
- Managing cluster state.
- Accepting API requests.
- Scheduling workloads.
- Monitoring the desired state.
- Coordinating cluster operations.

#### Control Plane Components

| Component | Responsibility |
|---|---|
| kube-apiserver | Exposes the Kubernetes API and handles cluster requests. |
| etcd | Stores Kubernetes cluster data and state. |
| kube-scheduler | Selects appropriate worker nodes for unscheduled Pods. |
| kube-controller-manager | Runs controllers that reconcile actual state with desired state. |
| cloud-controller-manager | Integrates Kubernetes with cloud-provider infrastructure when configured. |

### 3.2 Worker Nodes

Worker nodes are responsible for running application workloads.

Each worker node contains the components required to run and manage Pods.

#### Worker Node Components

| Component | Responsibility |
|---|---|
| kubelet | Communicates with the control plane and ensures assigned Pods are running. |
| kube-proxy | Implements Service networking rules where the selected networking implementation uses it. |
| Container Runtime | Runs containers, for example containerd or CRI-O. |

---

## 4. Control Plane vs Worker Nodes

| Control Plane | Worker Nodes |
|---|---|
| Manages the cluster | Runs application workloads |
| Handles API requests | Runs Pods |
| Stores cluster state through etcd | Hosts containers |
| Schedules workloads | Executes assigned workloads |
| Maintains desired state through controllers | Reports workload and node status |

**Key understanding:**

The control plane makes decisions about cluster management, while worker nodes provide the environment where application workloads execute.

---

## 5. How Kubernetes Handles an Application Deployment

### Practical Scenario

A developer wants to deploy an application with three replicas.

The desired state is:

- Application: Web Application
- Required replicas: 3
- Container image: Application Docker image

### Deployment Workflow

1. The developer submits a Deployment manifest to the Kubernetes API Server.
2. The API Server validates the request.
3. The desired configuration is persisted in etcd.
4. The Deployment controller observes the new desired state.
5. The ReplicaSet controller ensures that three Pods are created.
6. The Scheduler assigns unscheduled Pods to suitable worker nodes.
7. The kubelet on each selected worker node receives the Pod specification.
8. The container runtime downloads the required image if necessary and starts the containers.
9. The kubelet reports the status of the Pods to the control plane.

**Expected result:** Kubernetes works toward maintaining three application replicas.

If a Pod fails, Kubernetes controllers attempt to restore the desired replica count.

---

## 6. Key Concepts Learned

| Concept | Understanding |
|---|---|
| Cluster | A collection of machines working together to run Kubernetes workloads |
| Control Plane | Manages the Kubernetes cluster |
| Worker Node | Executes application workloads |
| Pod | Smallest deployable unit in Kubernetes |
| Deployment | Manages application rollout and replica requirements |
| ReplicaSet | Maintains the required number of Pod replicas |
| Scheduler | Selects worker nodes for Pods |
| etcd | Stores Kubernetes cluster state |
| Container Runtime | Runs containers |

---

## 7. Practical Challenge

### Scenario

A company runs an e-commerce application using five EC2 instances. Each instance hosts several Docker containers.

The company experiences the following problems:

- Manual application deployment.
- Container failures.
- Difficulty scaling during high traffic.
- Complex container management across EC2 instances.

### Challenge Questions

1. Why is manual container management difficult at this scale?
2. How does Kubernetes simplify container management?
3. What is the difference between a Kubernetes control plane and a worker node?
4. What happens when a developer requests three application replicas?
5. Which Kubernetes component selects a worker node for a Pod?
6. What happens when a running Pod fails?

**Completion requirement:** Be able to explain the architecture and deployment workflow without referring to the learning material.

---

## 8. Phase 1 Completion Checklist

- [ ] Understand monolithic architecture.
- [ ] Understand microservices architecture.
- [ ] Understand containerization using Docker.
- [ ] Understand container orchestration.
- [ ] Identify Kubernetes control plane components.
- [ ] Identify Kubernetes worker node components.
- [ ] Understand the difference between control plane and worker nodes.
- [ ] Explain the Kubernetes application deployment workflow.
- [ ] Complete the practical challenge questions.

---

## 9. Key Takeaways

- Docker helps package and run applications in containers.
- Kubernetes automates the management of containerized workloads.
- The control plane manages the cluster.
- Worker nodes run application workloads.
- Kubernetes continuously works to maintain the desired state.
- Understanding cluster architecture is essential before provisioning a Kubernetes cluster.

---

## Next Phase

**Phase 2 – AWS EC2 Management Machine Setup**

The next phase focuses on preparing an AWS EC2 instance that will act as the kOps management machine.

Planned activities:

- Launch an EC2 instance.
- Configure SSH access.
- Prepare the operating system.
- Verify system resources and networking.
- Prepare the environment for AWS and Kubernetes management tools.

---

**Project:** Kubernetes Hands-on Practice  
**Lab:** Lab 01 – Cluster Architecture & AWS Setup using kOps  
**Phase:** 01 – Kubernetes Architecture & Fundamentals  
**Focus:** Kubernetes concepts and cluster architecture
