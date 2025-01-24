# TESTING INGRESS NO RULE MATCH BEHAVIOUR IN MINIKUBE

## START MINIKUBE AND ENABLE INGRESS ADD-ON:
```bash
minikube start
minikube addons enable ingress
```

## DEPLOY THE DEFAULT BACKEND 
```bash
kubectl apply -f deployment.yaml
kubetcl apply -f ingress.yaml
kubectl apply -f service.yaml
```

## TEST INGRESS BEHAVIOUR
```bash
minikube ip
```
### Update `/etc/hosts`:
Map `example.com` to Minikube IP
```plaintext
<Minikube IP> example.com
```

## Request Not Matching Any Rule (e.g., `/unknown`):
```bash
curl http://example.com/unkown
```
Response:
```plaintext
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>
```
