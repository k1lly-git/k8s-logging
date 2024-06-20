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
```
no
```bash
kubectl auth can-i --as='system:serviceaccount:default:svc-test' create pod
```
yes
```bash
kubectl auth can-i --as='system:serviceaccount:default:svc-test' create configmap
```
no
```bash
kubectl auth can-i --as='system:serviceaccount:default:svc-test' get secret
```
yes
```bash
kubectl auth can-i --as='system:serviceaccount:default:svc-test' delete secret
```
no
```bash
kubectl auth can-i --as='system:serviceaccount:default:svc-test' create serviceaccount
```
yes
```bash
kubectl auth can-i --as='system:serviceaccount:default:svc-test' create rolebinding
```
yes
```bash
kubectl auth can-i --as='system:serviceaccount:default:svc-test' create pods/exec
```
yes

```bash
kubectl auth can-i --as='system:serviceaccount:default:svc-test' --list
```
Resources                                       Non-Resource URLs                      Resource Names   Verbs
roles.rbac.authorization.k8s.io                 []                                     [admin]          [bind]
roles.rbac.authorization.k8s.io                 []                                     [edit]           [bind]
roles.rbac.authorization.k8s.io                 []                                     [view]           [bind]
pods/exec                                       []                                     []               [create get]
serviceaccounts                                 []                                     []               [create]
selfsubjectreviews.authentication.k8s.io        []                                     []               [create]
selfsubjectaccessreviews.authorization.k8s.io   []                                     []               [create]
selfsubjectrulesreviews.authorization.k8s.io    []                                     []               [create]
rolebindings.rbac.authorization.k8s.io          []                                     []               [create]
pods                                            []                                     []               [get list watch create]
configmaps                                      []                                     []               [get list watch]
secrets                                         []                                     []               [get list watch]
                                                [/.well-known/openid-configuration/]   []               [get]
                                                [/.well-known/openid-configuration]    []               [get]
                                                [/api/*]                               []               [get]
                                                [/api]                                 []               [get]
                                                [/apis/*]                              []               [get]
                                                [/apis]                                []               [get]
                                                [/healthz]                             []               [get]
                                                [/healthz]                             []               [get]
                                                [/livez]                               []               [get]
                                                [/livez]                               []               [get]
                                                [/openapi/*]                           []               [get]
                                                [/openapi]                             []               [get]
                                                [/openid/v1/jwks/]                     []               [get]
                                                [/openid/v1/jwks]                      []               [get]
                                                [/readyz]                              []               [get]
                                                [/readyz]                              []               [get]
                                                [/version/]                            []               [get]
                                                [/version/]                            []               [get]
                                                [/version]                             []               [get]
                                                [/version]                             []               [get]
