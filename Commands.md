Delete All Resources in the Current Namespace
```bash 
kubectl delete all --all`
```

```bash 
kubectl get events --field-selector involvedObject.name=oom-pod
```

Get all the pods across all namespaces:
```bash 
kubectl get pods --all-namespaces
```

Get Name of Cluster:
```bash 
kubectl config view --minify -o jsonpath='{.clusters[0].name}'
```

Run a pod with a Label:
```bash
kubectl run my-pod --image=nginx –labels=app=web
```

Command to show labels of all pods in default namespace
```bash
kubectl get pods --show-labels
```

Command to scale deploayment named sid-deployment to 2 replicas
```bash
kubectl scale deployment sid-deployment --replicas=2
```

Update the Deployment: Update the nginx version to nginx:1.19 This will trigger a rolling update.
```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.19
```

Check the Rollout Status
```bash
kubectl rollout status deployment/nginx-deployment
```

Rollback the Update
```bash
kubectl rollout undo deployment/nginx-deployment
```

Connect to EKS Cluster
```bash
aws eks update-kubeconfig --region ap-south-1 --name cluster-name
```
