
```bash
kubectl get nodes
kubectl label node ip-192-168-16-15.ap-south-1.compute.internal workload=coolapp
kubectl get nodes --show-labels
kubectl describe node <node-name>
```


Command to do taint:
```bash
kubectl get nodes
kubectl taint nodes <node-name> dedicated=coolapp:NoSchedule
```