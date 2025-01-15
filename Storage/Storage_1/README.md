### Write Data to the PVC
``` bash
kubectl exec -it pvc-demo -- /bin/bash
echo "Hello from Persistent Storage!" > /usr/share/nginx/html/index.html

### Restart the Pod
``` bash
