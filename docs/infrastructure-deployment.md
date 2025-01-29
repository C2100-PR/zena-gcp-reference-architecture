# Infrastructure Deployment Guide

## Overview
This guide details the deployment process for Zena's core infrastructure components including GKE clusters, Ray deployments, networking, and load balancing configurations.

## Prerequisites
- GCP Project: api-for-warp-drive
- Required IAM permissions
- gcloud CLI installed and configured
- kubectl configured

## Deployment Sequence

### 1. Network Infrastructure
```bash
# Create VPC Network
gcloud compute networks create zena-vpc \
    --project=api-for-warp-drive \
    --subnet-mode=custom \
    --mtu=1460 \
    --bgp-routing-mode=global

# Create Subnets
gcloud compute networks subnets create gke-subnet-west4 \
    --project=api-for-warp-drive \
    --network=zena-vpc \
    --region=us-west4 \
    --range=172.16.0.0/20 \
    --secondary-range=pods=10.0.0.0/14,services=10.4.0.0/19

gcloud compute networks subnets create gke-subnet-west1 \
    --project=api-for-warp-drive \
    --network=zena-vpc \
    --region=us-west1 \
    --range=172.16.16.0/20 \
    --secondary-range=pods=10.8.0.0/14,services=10.12.0.0/19
```

### 2. Cloud NAT Configuration
```bash
# Create Cloud Router
gcloud compute routers create zena-router-west4 \
    --project=api-for-warp-drive \
    --network=zena-vpc \
    --region=us-west4

# Configure NAT
gcloud compute routers nats create zena-nat-west4 \
    --router=zena-router-west4 \
    --auto-allocate-nat-external-ips \
    --nat-all-subnet-ip-ranges \
    --enable-logging
```

### 3. GKE Cluster Deployment
```bash
# Create Primary Cluster
gcloud container clusters create zena-primary \
    --project=api-for-warp-drive \
    --region=us-west4 \
    --network=zena-vpc \
    --subnetwork=gke-subnet-west4 \
    --enable-private-nodes \
    --master-ipv4-cidr=172.16.0.0/28 \
    --enable-ip-alias \
    --cluster-secondary-range-name=pods \
    --services-secondary-range-name=services

# Create Node Pools
gcloud container node-pools create ray-workers \
    --cluster=zena-primary \
    --machine-type=n1-standard-16 \
    --num-nodes=2 \
    --enable-autoscaling \
    --min-nodes=2 \
    --max-nodes=8 \
    --node-labels=ray.io/node-type=worker
```

### 4. Ray Cluster Deployment
```bash
# Install Ray Operator
kubectl create namespace ray-system
helm install ray-operator ray/ray-operator \
    --namespace ray-system

# Deploy Ray Cluster
kubectl apply -f config/infrastructure/ray-cluster.yaml
```

### 5. Load Balancer Configuration
```bash
# Create SSL Certificate
gcloud compute ssl-certificates create zena-ssl-cert \
    --project=api-for-warp-drive \
    --global

# Create Health Check
gcloud compute health-checks create http zena-health-check \
    --project=api-for-warp-drive \
    --port=80 \
    --request-path=/health

# Create Backend Service
gcloud compute backend-services create zena-backend \
    --project=api-for-warp-drive \
    --global \
    --protocol=HTTPS \
    --port-name=https \
    --health-checks=zena-health-check
```

## Validation Steps

### 1. Network Connectivity
```bash
# Verify VPC and Subnet Creation
gcloud compute networks describe zena-vpc
gcloud compute networks subnets list --network=zena-vpc

# Verify NAT Configuration
gcloud compute routers nats describe zena-nat-west4 \
    --router=zena-router-west4 \
    --region=us-west4
```

### 2. GKE Cluster Health
```bash
# Verify Cluster Status
gcloud container clusters describe zena-primary \
    --region=us-west4

# Check Node Pool Status
kubectl get nodes -l ray.io/node-type=worker
```

### 3. Ray Cluster Verification
```bash
# Check Ray Operator Status
kubectl get pods -n ray-system

# Verify Ray Cluster Deployment
kubectl get rayclusters -n ray-system
```

### 4. Load Balancer Validation
```bash
# Check Load Balancer Status
gcloud compute forwarding-rules list

# Verify SSL Certificate
gcloud compute ssl-certificates describe zena-ssl-cert
```

## Troubleshooting

### Common Issues and Resolutions

1. GKE Cluster Creation Failures
   - Verify IAM permissions
   - Check subnet CIDR ranges
   - Validate service account configuration

2. Ray Cluster Issues
   - Check Ray operator logs
   - Verify node pool labels
   - Validate resource quotas

3. Network Connectivity Problems
   - Verify firewall rules
   - Check NAT configuration
   - Validate subnet routing

### Monitoring and Logging

1. GKE Monitoring
```bash
# View Cluster Logs
gcloud logging read "resource.type=container"

# Check Cluster Events
kubectl get events --sort-by=.metadata.creationTimestamp
```

2. Network Monitoring
```bash
# View NAT Logs
gcloud logging read "resource.type=nat_gateway"

# Check VPC Flow Logs
gcloud logging read "resource.type=vpc_flow"
```

## Security Considerations

1. Network Security
   - All private clusters
   - Restricted CIDR ranges
   - Authorized networks configuration

2. Access Control
   - IAM role assignments
   - Service account limitations
   - Network policy enforcement

3. Monitoring and Auditing
   - Stackdriver logging
   - VPC flow logs
   - Audit logging enabled