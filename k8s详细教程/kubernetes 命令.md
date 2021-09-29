### 命令

```powershell
# 1、查看pod 状态的的命令
kubectl get pod -n kube-system -o wide
# 2、删除pod
kubectl delete pod/kub-flannel-ds-xxxx -n kube-system --grace-period=0 --force
```

