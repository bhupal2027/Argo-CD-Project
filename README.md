**✅ MASTER CHECKLIST / FULL STEPS (High-Level Workflow)**

Perfect for documentation or a LinkedIn project write-up.
**Phase 0 — Prerequisites**

1. AWS Account
https://aws.amazon.com/resources/create-account/

2. IAM user with admin or EKS full permissions

3.AWS CLI installed & configured

https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

aws configure

4.kubectl installed

https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/

5.eksctl installed

https://docs.aws.amazon.com/eks/latest/eksctl/installation.html

6.argocd CLI installed

https://argo-cd.readthedocs.io/en/stable/cli_installation/

7.SSH key pair for nodes

(Example: 2nd.pub)

8.Security group inbound rules ready

9. Allow NodePort (default ArgoCD port range 30000–32767)

10. Allow ArgoCD UI access

**Phase 1 — Create EKS Clusters (HUB + SPOKEs)**

**1. Create HUB EKS cluster**

eksctl create cluster \
  --name HUB \
  --region ap-south-1 \
  --zones ap-south-1a,ap-south-1b \
  --nodegroup-name eksdemo1-ng-public1 \
  --node-type m7i-flex.large \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 4 \
  --node-volume-size 20 \
  --ssh-access \
  --ssh-public-key 2nd \
  --managed \
  --asg-access \
  --external-dns-access \
  --full-ecr-access \
  --appmesh-access \
  --alb-ingress-access

**2. Create SPOKE-01 cluster**

(Same command with different name)

**3. Create SPOKE-02 cluster**

(Same command with different name)

**Phase 2 — Configure kubectl Contexts**

1. **List available contexts**

   kubectl config get-contexts
   
2.**Switch to HUB context**

kubectl config use-context Badhri@HUB.ap-south-1.eksctl.io

**Phase 3 — Install ArgoCD on HUB Cluster**

1. **Create namespace**

   kubectl create namespace argocd
   
2. **Install ArgoCD**
   
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   
3. **Wait for pods**
   
   kubectl get pods -n argocd
   
**Phase 4 — Expose ArgoCD UI**

1. **Edit ArgoCD configmap to enable insecure mode**
   
   kubectl edit cm argocd-cmd-params-cm -n argocd
           ADD: data:
                 server.insecure: "true"
   
2. **Convert ArgoCD server service → NodePort**
   kubectl edit svc argocd-server -n argocd
   Change:
   type: ClusterIP → NodePort
   
3. **Allow NodePort in SG**

4. **Retrieve initial admin password**
   
   kubectl get secret argocd-initial-admin-secret -n argocd -o yaml
   echo <base64-value> | base64 --decode
   
5. **Login to ArgoCD using browser**

   https://<NodeIP>:<NodePort>
   
**Phase 5 — Setup ArgoCD CLI**

**Login via CLI**

argocd login <ARGOCD-IP:PORT>

**Phase 6 — Add SPOKE Clusters to HUB ArgoCD**

**Add SPOKE-01**

argocd cluster add Badhri@SPOKE-01.ap-south-1.eksctl.io --server <ARGOCD-IP:PORT>

**Add SPOKE-02**

argocd cluster add Badhri@SPOKE-02.ap-south-1.eksctl.io --server <ARGOCD-IP:PORT>

ArgoCD automatically:
Creates argocd-manager service account
Creates cluster-admin role + binding
Creates bearer token secret
Registers cluster in HUB

**Phase 7 — Deploy Application using GitOps**

Create a Git repo (manifests folder)
Add Application from ArgoCD UI or YAML
Point Destination to SPOKE-01 or SPOKE-02
Sync application
Verify pods & ELB working
Test auto-sync by updating Git

**Phase 8 — Validate Multi-Cluster GitOps**

ArgoCD UI → Applications
Check status: Healthy/Synced
Verify Cluster list → Connected
Open both ELB URLs → Application works in both SPOKE clusters

**Phase 9 — Clean-up (Avoid AWS Charges)**
eksctl delete cluster --name HUB --region ap-south-1
eksctl delete cluster --name SPOKE-01 --region ap-south-1
eksctl delete cluster --name SPOKE-02 --region ap-south-1





