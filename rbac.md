rbac.yaml:
```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: svc-test

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: role-test
rules:
  - apiGroups: ["rbac.authorization.k8s.io"]
    resources: ["rolebindings"]
    verbs: ["create"]
  - apiGroups: ["rbac.authorization.k8s.io"]
    resources: ["roles"]
    verbs: ["bind"]
    resourceNames: ["admin","edit","view"]
  - apiGroups: [""]
    resources: ["serviceaccounts"]
    verbs: ["create"]
  - apiGroups: [""]
    resources: ["secrets", "configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch", "create"]
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["create", "get"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: bind-test
roleRef:
  name: role-test
  kind: ClusterRole
subjects:
- kind: ServiceAccount
  name: svc-test
  namespace: default
```

```bash
kubectl auth can-i --as='system:serviceaccount:default:svc-test' delete rolebindings
no
kubectl auth can-i --as='system:serviceaccount:default:svc-test' create pod
yes
kubectl auth can-i --as='system:serviceaccount:default:svc-test' create configmap
no
kubectl auth can-i --as='system:serviceaccount:default:svc-test' get secret
yes
kubectl auth can-i --as='system:serviceaccount:default:svc-test' delete secret
no
kubectl auth can-i --as='system:serviceaccount:default:svc-test' create serviceaccount
yes
kubectl auth can-i --as='system:serviceaccount:default:svc-test' create rolebinding
yes
kubectl auth can-i --as='system:serviceaccount:default:svc-test' create pods/exec
yes
```
