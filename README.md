# K8s learning - Gratitude App

## Argo CD documentation

See `ARGOCD.md` for:
- Argo CD install and login on EKS
- App creation/sync commands
- Manifest file order for this repository

## Helm documentation

See `HELM.md` for:
- Recommended Helm workflow for this repository
- Dependency install via Helm (ingress-nginx)
- Optional migration path from raw manifests to a Helm chart

creating the cluster:

```bash
eksctl create cluster --name b15-pk-eks --region ap-south-1 --version 1.34 --zones ap-south-1a,ap-south-1b --managed --with-oidc --nodegroup-name ng-1 --node-type t3.medium --nodes 2 --nodes-min 2 --nodes-max 2 --node-ami-family AmazonLinux2023
```

You can also create cluster through yml file with the content given below by naming the file as `eks-cluster.yml`:

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: b15-pk-eks
  region: ap-south-1
  version: "1.34"

availabilityZones:
  - ap-south-1a
  - ap-south-1b

vpc:
  cidr: 10.1.0.0/16
  nat:
    gateway: Single  

iam:
  withOIDC: true

managedNodeGroups:
  - name: ng-1
    instanceType: t3.medium
    desiredCapacity: 2
    minSize: 2
    maxSize: 2
    amiFamily: AmazonLinux2023
```

```bash
eksctl create cluster -f eks-cluster.yml
```

Finally switch the context of the kubectl to the eks cluster created:

```bash
aws eks update-kubeconfig --region ap-south-1 --name b15-pk-eks
```


Installing ingress:

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace --set controller.service.type=LoadBalancer --set controller.ingressClassResource.name=nginx --set controller.ingressClassByName=true
```


Install ebs driver:
```bash
eksctl create addon --cluster b15-pk-eks --region ap-south-1 --name aws-ebs-csi-driver --force
```

Check if that got created:

```bash
kubectl get csidrivers
```

Note: the postgres-init-cluster needs to be applied and post that the rollout the deployment of the datbaase:

```bash
kubectl rollout restart deployment postgres-deployment
kubectl get pods -w
```


## HPA

Pre-requisite:

1. Metrics Server (needed for HPA CPU/Memory)

```bash
kubectl get deployment -n kube-system metrics-server
kubectl top nodes
kubectl top pods -A
```

If kubectl top fails, install metrics-server (standard manifest):
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/component
```
On EKS you may need to allow insecure kubelet TLS (common in some setups). Patch metrics-server args if required:
```bash
kubectl -n kube-system patch deployment metrics-server --type='json' -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

## VPA

By default VPA is not present in EKS, so we need to install it.

```bash
helm repo add fairwinds-stable https://charts.fairwinds.com/stable
helm install vpa fairwinds-stable/vpa --namespace vpa --create-namespace
```

## RBAC over EKS

The idea is to create a role for a user to access the EKS cluster. Before we create, we need few things:
- IAM User with programmatic access (CLI).
- This IAM user should have `AmazonEKSClusterPolicy` attached to it.
- Roles and Rolebinding that needs to be created.

Let's follow the steps
- Step 1: Create IAM user and attach the policy specified above. Let's say the user that you created is `demo-eks-user`
- Step 2: EKS authentication works via `aws-auth ConfigMap`. So, we need to configure this to add the user to it.
    Run the command 
    ```bash
    kubectl edit configmap aws-auth -n kube-system
    ```
    Add this below block:
    ```bash
    mapUsers: |
        - userarn: arn:aws:iam::975050024946:user/demo-eks-user
          username: dev-user
          groups:
            - dev-group
    ```

    Do modify the arn of the user.

- Step 3: Create Role and apply it.
    ```bash
    apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
    namespace: default
    name: developer-role
    rules:
    - apiGroups: ["", "apps"]
    resources:
        - pods
        - services
        - deployments
        - replicasets
    verbs:
        - get
        - list
        - watch
        - create
        - update
        - delete
    ```
    ```bash
    kubectl apply -f developer-role.yml
    ```

4. Step 4: Now bind this role to the group.
    ```bash
    apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
    name: developer-binding
    namespace: default
    subjects:
    - kind: Group
    name: dev-group
    apiGroup: rbac.authorization.k8s.io

    roleRef:
    kind: Role
    name: developer-role
    apiGroup: rbac.authorization.k8s.io
    ```
    ```bash
    kubectl apply -f developer-binding.yaml
    ```

5. Step 5: Now configure this user to the cluster and run the kubectl command.
    ```bash
    aws configure
    aws eks update-kubeconfig  --region ap-south-1 --name <your-cluster-name>
    ```
    User can:
    ```bash
    kubectl get pods
    kubectl create deployment
    kubectl update services
    ```

    User cannot:
    ```bash
    delete nodes
    modify RBAC
    access other namespaces
    ```
