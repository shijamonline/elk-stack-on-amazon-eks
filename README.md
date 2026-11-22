# Deploying the ELK Stack on Amazon EKS using Helm and ECK

## Project Overview

This project demonstrates the deployment of a centralized logging stack on Amazon Elastic Kubernetes Service (EKS), using Elasticsearch, Kibana, and Filebeat.

The deployment uses **Helm** for package management and **Elastic Cloud on Kubernetes (ECK)** to manage the lifecycle of Elasticsearch and Kibana within Kubernetes.

The objective was to understand how containerized applications generate logs, how log collectors gather and enrich those logs, and how Elasticsearch stores them for centralized search and analysis.

## Architecture

```text
                 Amazon EKS Cluster
    ┌──────────────────────────────────────┐
    │                                      │
    │  Worker Node 1                       │
    │  ┌────────────────┐                  │
    │  │ Application    │                  │
    │  │ Container      │                  │
    │  └───────┬────────┘                  │
    │          │                           │
    │  ┌───────▼────────┐                  │
    │  │   Filebeat     │                  │
    │  └───────┬────────┘                  │
    │          │                           │
    │  Worker Node 2                       │
    │  ┌───────▼────────┐                  │
    │  │   Filebeat     │                  │
    │  └───────┬────────┘                  │
    │          │                           │
    │  Worker Node 3                       │
    │  ┌───────▼────────┐                  │
    │  │   Filebeat     │                  │
    │  └───────┬────────┘                  │
    │          │                           │
    │          ▼                           │
    │  ┌──────────────────────┐            │
    │  │   Elasticsearch      │            │
    │  │   3-node cluster     │            │
    │  └──────────┬───────────┘            │
    │             │                        │
    │       ┌─────▼──────┐                 │
    │       │   Kibana   │                 │
    │       └────────────┘                 │
    │                                      │
    └──────────────────────────────────────┘
```

*Conceptual representation of the deployed logging pipeline.*

## Technology Stack

The project uses the following technologies and services:

**Cloud Infrastructure**
- **Amazon EKS:** Managed Kubernetes control plane.
- **Amazon EC2:** Kubernetes worker nodes.
- **Amazon EBS:** Persistent storage for Elasticsearch.
- **EBS CSI Driver:** Dynamic provisioning of EBS volumes.
- **IAM Roles for Service Accounts (IRSA):** IAM permissions for the EBS CSI controller.

**Container Orchestration and Deployment**
- **Kubernetes:** Container orchestration.
- **Helm:** Kubernetes package management.
- **Elastic ECK Operator:** Manages Elastic Stack resources and their lifecycle.

**Logging and Observability**
- **Elasticsearch:** Log storage, indexing and search.
- **Kibana:** Visualization and exploration of Elasticsearch data.
- **Filebeat:** Collection and forwarding of container logs.

## Implementation

### 1. Provisioned the EKS Cluster

- Created an Amazon EKS cluster using `eksctl`.
- Deployed three managed worker nodes.
- Configured the AWS EBS CSI driver for persistent storage.
- Configured IAM Roles for Service Accounts (IRSA) for the EBS CSI controller.

### 2. Configured Persistent Storage

- Created a Kubernetes StorageClass using AWS EBS `gp3`.
- Enabled volume encryption and volume expansion.
- Configured `WaitForFirstConsumer` volume binding to align volume provisioning with pod scheduling.

### 3. Installed the ECK Operator

- Added the Elastic Helm repository.
- Installed the ECK operator using Helm.
- Verified the Elastic Kubernetes custom resource definitions (CRDs).
- Used ECK to manage Elasticsearch and Kibana deployments.

### 4. Deployed Elasticsearch and Kibana

- Deployed the Elastic Stack using the `eck-stack` Helm chart.
- Configured a three-node Elasticsearch cluster.
- Configured 20 GiB gp3 persistent volumes for Elasticsearch nodes.
- Deployed a single Kibana instance connected to Elasticsearch.

### 5. Deployed Filebeat

- Deployed Filebeat as a Kubernetes DaemonSet using an ECK Beat resource.
- Configured one Filebeat pod per worker node.
- Collected container logs from `/var/log/containers/*.log`.
- Configured CRI container log parsing.
- Used the `add_kubernetes_metadata` processor to enrich events with pod, namespace, container and node metadata.
- Forwarded collected logs to Elasticsearch.

### 6. Tested End-to-End Log Ingestion

Created a sample BusyBox application deployment that continuously generated JSON-formatted log messages.

Example log:

```json
{
  "level": "info",
  "msg": "order placed",
  "order_id": 117
}
```

Verified that:

- Filebeat collected the application logs.
- Events were successfully acknowledged by Elasticsearch.
- Documents were indexed in the Filebeat data stream.
- Elasticsearch searches returned the demo application events.
- Kubernetes metadata identified the originating pod, namespace, container and worker node.

## Validation and Results

The deployment was validated using Kubernetes commands and Elasticsearch REST APIs.

**EKS Infrastructure**
- Worker nodes: 3 Ready.
- EBS CSI driver: Active.
- Persistent storage: gp3 volumes provisioned.

**Elasticsearch**
- Cluster health: Green.
- Elasticsearch nodes: 3.
- Unassigned shards: 0.
- Log data successfully indexed and searchable.

**Kibana**
- Deployment: Healthy.
- Instance count: 1.
- Connected to the Elasticsearch cluster.

**Filebeat**
- DaemonSet pods: 3 Running.
- Log collection: Successful.
- Events acknowledged by Elasticsearch.
- Kubernetes metadata enrichment: Verified.

**End-to-End Verification**

Successfully searched for `order placed` events generated by the BusyBox demo application in Elasticsearch.

The returned documents included:
- Application log messages.
- Filebeat agent information.
- Kubernetes pod and namespace details.
- Container and worker node information.
- Container log file paths and timestamps.

Evidence files are available in the [`docs/`](docs/) directory.

**Note:** Kibana was deployed and its health was verified through Kubernetes. Browser-based dashboard access was not tested in this lab.

## Repository Structure

```text
elk-stack-on-amazon-eks/
│
├── README.md
├── cluster.yaml
├── storageclass.yaml
├── elk-values.yaml
├── filebeat.yaml
│
└── docs/
    ├── README.md
    ├── elasticsearch-cluster-health.json
    ├── filebeat-pods-evidence.txt
    └── elasticsearch-search-evidence.json
```

## Key Learnings

Through this project, I gained hands-on experience with:

- Provisioning and managing Kubernetes infrastructure on AWS.
- Deploying stateful workloads with persistent storage.
- Understanding Helm charts, releases and values files.
- Understanding Kubernetes operators and custom resource definitions.
- Configuring Filebeat for Kubernetes container log collection.
- Enriching logs with Kubernetes metadata.
- Validating centralized log ingestion using Elasticsearch REST APIs.
- Troubleshooting Kubernetes workloads and log collection configurations.

## Cleanup

AWS resources were created for a temporary hands-on lab. After collecting project evidence, the EKS cluster and associated resources should be deleted to avoid ongoing infrastructure charges.

## Disclaimer

This is a personal hands-on learning project built in an AWS lab environment. It is intended for educational and portfolio purposes and is not a production deployment blueprint.
