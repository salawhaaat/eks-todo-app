# AWS EKS Deployment Guide

## Prerequisites
- AWS CLI, [eksctl](https://eksctl.io/), [kubectl](https://kubernetes.io/docs/reference/kubectl/) installed
- Docker image pushed: `salawhaaat/todo-app:latest` (linux/amd64)

## Architecture
| Component | Type | Details |
|-----------|------|---------|
| Node | EC2 t3.medium | 4GB RAM, 2vCPU, 58 pod limit |
| Storage | EBS gp3 | 5Gi, dynamic provisioning via [CSI Driver](https://kubernetes.io/docs/concepts/storage/volumes/#csi) |
| Database | MongoDB [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/) | Persistent via [volumeClaimTemplates](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#stable-storage) |
| App | Flask [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) | Connects to mongo:27017 |
| Network | LoadBalancer | AWS ALB on port 80 |
| Security | [OIDC](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html) + [IRSA](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) | IAM role for EBS CSI |

## Step 1: Create Cluster
```bash
eksctl create cluster -f k8s/cluster.yaml
```

This single command:
- Creates the EKS cluster with a `t3.medium` node
- Enables the IAM OIDC provider (required for IRSA)
- Installs the EBS CSI driver addon

**Verify:** `kubectl get pods -n kube-system | grep ebs`

## Step 2: Install Prometheus + Alertmanager (with Slack)
```bash
# Add Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install kube-prometheus-stack with Slack configured via values.yaml
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  -f k8s/components/prometheus/prometheus-values.yaml
```

**Verify:**
```bash
kubectl get pods -n monitoring          # All pods Running
kubectl get prometheusrule -n monitoring  # Rules applied
```

## Step 3: Deploy Application
```bash
kubectl apply -k k8s/
```

**Verify:**
```bash
kubectl get pods -n todo-app           # Flask + MongoDB Running
kubectl get pvc -n todo-app            # PVC Bound (EBS volume auto-created)
kubectl get svc -n todo-app            # Services exist
```

## Step 4: Test Application
```bash
EXTERNAL_IP=$(kubectl get svc flask-service -n todo-app \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

curl http://$EXTERNAL_IP
```

## Step 5: Test Data Persistence
```bash
# Delete MongoDB pod (simulates crash)
kubectl delete pod mongo-0 -n todo-app

# Watch it restart on same volume
kubectl get pods -n todo-app -w

# Verify PVC still Bound (same EBS volume)
kubectl get pvc -n todo-app
```

**Result:** ✅ Pod restarted, EBS volume persisted → data survives

## Step 6: Test Slack Alerting
```bash
# Delete Flask pod to trigger FlaskPodDown alert (fires after 1 minute)
kubectl delete pod -l app=flask-app -n todo-app

# Watch Prometheus pick up the alert (takes ~1-2 min)
kubectl get prometheusrule -n monitoring
kubectl port-forward svc/prometheus-kube-prometheus-prometheus -n monitoring 9090:9090
# Open http://localhost:9090/alerts — FlaskPodDown should show as "firing"
# Check your Slack channel for the notification
```

**Result:** ✅ Alert fires → Alertmanager routes to Slack → message appears in #alerts channel

## Step 7: Cleanup
```bash
eksctl delete cluster --name todo-app-cluster --region us-east-1
```

EBS volumes auto-deleted ([reclaimPolicy: Delete](https://kubernetes.io/docs/concepts/storage/storage-classes/#reclaim-policy))

---

## Key Concepts

**Dynamic Provisioning**: [StorageClass](https://kubernetes.io/docs/concepts/storage/storage-classes/) + [PVC](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) → [EBS CSI Driver](https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html) → Auto-create AWS EBS volumes

**IAM Security**: [OIDC provider](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html) authenticates cluster → CSI controller assumes IAM role → only EBS permissions ([AmazonEBSCSIDriverPolicy](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_managed-vs-inline.html))

**StatefulSet + volumeClaimTemplates**: Each pod gets its own PVC → EBS volume follows pod across restarts

