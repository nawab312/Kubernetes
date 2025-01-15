### Write Data to the PVC
``` bash
kubectl exec -it pvc-demo -- /bin/bash
echo "Hello from Persistent Storage!" > /usr/share/nginx/html/index.html
```
### Restart the Pod
``` bash
kubectl delete pod pvc-demo , kubectl apply -f pod.yaml
```
### Verify Data Persistence
``` bash
kubectl exec -it pvc-demo -- /bin/bash
cat /usr/share/nginx/html/index.html
```
