# K8s learning - Gratitude App

creating the cluster:

```bash
eksctl create cluster --name b15-pk-eks --region ap-south-1 --version 1.34 --zones ap-south-1a,ap-south-1b --managed --with-oidc --nodegroup-name ng-1 --node-type t3.medium --nodes 2 --nodes-min 2 --nodes-max 2 --node-ami-family AmazonLinux2023
```

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
