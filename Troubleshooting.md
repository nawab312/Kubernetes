**Issue:**
siddharth312@siddharth312-GF65-Thin-9SD:~/Kubernetes/Pods$ kubectl exec -it etcd-my-cluster -n kube-system -- sh
sh-5.2# etcdctl member list
{"level":"warn","ts":"2025-02-04T03:57:44.884581Z","logger":"etcd-client","caller":"v3@v3.5.15/retry_interceptor.go:63","msg":"retrying of unary invoker failed","target":"etcd-endpoints://0xc00002a000/127.0.0.1:2379","attempt":0,"error":"rpc error: code = DeadlineExceeded desc = latest balancer error: last connection error: connection error: desc = \"error reading server preface: read tcp 127.0.0.1:40794->127.0.0.1:2379: read: connection reset by peer\""}
Error: context deadline exceeded

**Solution:**
etcdctl can't connect to the etcd service, resulting in a connection reset by peer error
