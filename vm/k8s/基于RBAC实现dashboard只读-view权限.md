# 基于RBAC实现dashboard只读——view权限

只是简单利用默认的clusterrole - view实现了只读所有namespace下的对象(除去secret、role、rolebinding)，不支持读取集群信息，后期深入了解resource后再重新梳理role和rule

```yaml
kind: ServiceAccount
apiVersion: v1
metadata: 
    name: view 
    namespace: kube-system 

---

kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
    name: dashboard-dev-rolebinding
subjects:
  - kind: ServiceAccount 
    name: view
    namespace: kube-system
roleRef:
    kind: ClusterRole
    name: view 
    apiGroup: rbac.authorization.k8s.io 
```

使用默认的clusterrole：view`(Allows read-only access to see most objects in a namespace. It does not allow viewing roles or rolebindings. It does not allow viewing secrets, since those are escalating. )`

https://kubernetes.io/docs/reference/access-authn-authz/rbac/

拿token

```shell
 kubectl describe secret -n kube-system `kubectl describe sa view  -n kube-system |  grep "Mountable secrets" | awk '{print $3}'`  | grep -E ^token | awk '{print $2}' 
```

登录dashboard即可