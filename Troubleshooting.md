**Issue:**
siddharth312@siddharth312-GF65-Thin-9SD:~/Kubernetes/Pods$ kubectl exec -it etcd-my-cluster -n kube-system -- sh
sh-5.2# etcdctl member list
{"level":"warn","ts":"2025-02-04T03:57:44.884581Z","logger":"etcd-client","caller":"v3@v3.5.15/retry_interceptor.go:63","msg":"retrying of unary invoker failed","target":"etcd-endpoints://0xc00002a000/127.0.0.1:2379","attempt":0,"error":"rpc error: code = DeadlineExceeded desc = latest balancer error: last connection error: connection error: desc = \"error reading server preface: read tcp 127.0.0.1:40794->127.0.0.1:2379: read: connection reset by peer\""}
Error: context deadline exceeded
```
**Solution:**
etcdctl can't connect to the etcd service, resulting in a connection reset by peer error
``` bash
sh-5.2# export ETCDCTL_API=3
sh-5.2# export ETCDCTL_ENDPOINTS=https://192.168.49.2:2379
sh-5.2# etcdctl member list                               
{"level":"warn","ts":"2025-02-04T04:07:29.786905Z","logger":"etcd-client","caller":"v3@v3.5.15/retry_interceptor.go:63","msg":"retrying of unary invoker failed","target":"etcd-endpoints://0xc00045e000/192.168.49.2:2379","attempt":0,"error":"rpc error: code = DeadlineExceeded desc = latest balancer error: last connection error: connection error: desc = \"transport: authentication handshake failed: tls: failed to verify certificate: x509: certificate signed by unknown authority\""}
Error: context deadline exceeded
```
Error indicates a problem with the TLS (Transport Layer Security) handshake when trying to connect to etcd. Specifically, the certificate verification is failing because the certificate is signed by an unknown authority. This often happens when the etcd server is using self-signed certificates, which aren't trusted by default.
To resolve this, you can bypass the certificate verification or configure etcdctl to trust the etcd certificate
Use the CA Certificate
```bash
export ETCDCTL_CACERT=/path/to/etcd-ca.crt
```
