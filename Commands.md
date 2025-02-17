Delete All Resources in the Current Namespace
```bash kubectl delete all --all```

```bash kubectl get events --field-selector involvedObject.name=oom-pod```

Get all the pods across all namespaces:
```bash kubectl get pods --all-namespaces```

Get Name of Cluster:
```bash kubectl config view --minify -o jsonpath='{.clusters[0].name}'```
