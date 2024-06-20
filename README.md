# k8s-logging with sending to SIEM/Syslog server

## Grafana + Loki + Promtail
Устанавливаем helm https://helm.sh/docs/intro/install/
```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```
values.yaml:
```yaml
loki:
  enabled: true
  isDefault: true
  url: http://{{(include "loki.serviceName" .)}}:{{ .Values.loki.service.port }}
  readinessProbe:
    httpGet:
      path: /ready
      port: http-metrics
    initialDelaySeconds: 45
  livenessProbe:
    httpGet:
      path: /ready
      port: http-metrics
    initialDelaySeconds: 45
  datasource:
    jsonData: "{}"
    uid: ""
  image:
    tag: latest

promtail:
  enabled: true
  config:
    logLevel: info
    serverPort: 3101
    clients:
      - url: http://{{ .Release.Name }}:3100/loki/api/v1/push
  image:
    tag: latest

grafana:
  enabled: true 
  sidecar:
    datasources:
      label: ""
      labelValue: ""
      enabled: true
      maxLines: 1000
  image:
    tag: latest
```
```bash
kubectl create ns log
helm install -n log --values values.yaml loki-stack grafana/loki-stack
```

```bash
kubectl -n log port-forward svc/loki-stack-grafana 8000:80
```

```bash
kubectl -n log get secret/loki-stack-grafana -o custom-columns="VALUE":.data.admin-user --no-headers | base64 --decode
kubectl -n log get secret/loki-stack-grafana -o custom-columns="VALUE":.data.admin-password --no-headers | base64 --decode
````

Логинимся в grafana через проброс портов [localhost:8000](http://127.0.0.1:8000) \
Заходим в explore, выбираем data source Loki и проверяем




## Fluentd

### Поднимаем syslog сервер
```bash
sudo apt install rsyslog
systemctl enable rsyslog
systemctl start rsyslog
```
В зависимости от выбранного протокола нужно раскомметировать строки и выбрать порт в /etc/rsyslog.conf:
```bash
UDP
#module(load="imudp")
#input(type="imudp" port="1514")

TCP
#module(load="imtcp")
#input(type="imtcp" port="1514")
```

Билдим образ контейнера и пуллим
```bash
git clone https://github.com/k1lly-git/k8s-logging
cd k8s-logging/debian-syslog
docker build -t fluentd-syslog .
docker pull fluentd-syslog
```

На ноде где работает kube-apiserver настраиваем работу Audit Logs
```bash
mkdir /var/lib/k8s_audit
mkdir /var/log/audit/
touch /var/log/audit/audit.log
cat > /var/lib/k8s_audit/audit-policy.yaml <<EOF
apiVersion: audit.k8s.io/v1 # This is required.
kind: Policy
omitStages:
  - "RequestReceived"
rules:
  - level: RequestResponse
    resources:
    - group: ""
      resources: ["pods"]

  - level: None
    resources:
    - group: ""
      resources: ["configmaps"]
      resourceNames: ["controller-leader"]

  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources:
    - group: "" # core API group
      resources: ["endpoints", "services"]

  - level: None
    userGroups: ["system:authenticated"]
    nonResourceURLs:
    - "/api*" # Wildcard matching.
    - "/version"

  - level: Metadata
    resources:
    - group: "" # core API group
      resources: ["secrets", "configmaps"]

  - level: Metadata
    omitStages:
      - "RequestReceived"

  - level: None
    verbs:
    - get
    - list
    - watch
    resources:
    - group: ""
      resources:
      - pods
      - configmaps
      - secrets
    namespaces:
    - calico-apiserver
    - calico-system
    - log
    - metallb-system
    - tigera-operator
    - olm
EOF
```

В файле /etc/kubernetes/manifests/kube-apiserver.yaml добавляем строки в аргументы запуска (containers.command)
```bash
    - --audit-policy-file=/var/lib/k8s_audit/audit-policy.yaml
    - --audit-log-path=/var/log/audit/audit.log
    - --audit-log-maxsize=500
    - --audit-log-maxbackup=5
```
Ниже монтируем тома в volumeMounts:
```bash
    - mountPath: /var/lib/k8s_audit/audit-policy.yaml
      name: audit
      readOnly: true
    - mountPath: /var/log/audit/audit.log
      name: audit-log
      readOnly: false
```
Листаем вниз до раздела volumes и добавляем наши тома:
```bash
  - name: audit
    hostPath:
      path: /var/lib/k8s_audit/audit-policy.yaml
      type: File
  - name: audit-log
    hostPath:
      path: /var/log/audit/audit.log
      type: FileOrCreate
```

После изменения конфигурации запуска kube-apiserver кластер должен автоматически перезагрузиться \
Передаем через переменные окружения IP syslog сервера, порт, протокол в daemonset \
fluentd.yaml:
```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluentd
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
    version: v1
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluentd
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - nodes
  - namespaces
  verbs:
  - get
  - list
  - watch

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: fluentd
roleRef:
  kind: ClusterRole
  name: fluentd
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: fluentd
  namespace: kube-system

---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
    version: v1
spec:
  selector:
    matchLabels:
      k8s-app: fluentd-logging
  template:
    metadata:
      labels:
        k8s-app: fluentd-logging
        version: v1
    spec:
      serviceAccount: fluentd
      serviceAccountName: fluentd
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd
        # image: fluent/fluentd-kubernetes-daemonset:v1-debian-syslog
        image: k1llydocker/fluentd:1 # наш запуленный ранее образ
        imagePullPolicy: Always
        env:
          - name: K8S_NODE_NAME
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
          - name:  SYSLOG_HOST
            value: "<IP>"
          - name:  SYSLOG_PORT
            value: "<PORT>"
          - name:  SYSLOG_PROTOCOL
            value: "<tcp/udp>"
          - name: FLUENT_CONTAINER_TAIL_PARSER_TYPE
            value: "cri"
          - name: FLUENT_CONTAINER_TAIL_PARSER_TIME_FORMAT
            value: "%Y-%m-%dT%H:%M:%S.%N%:z"
          - name: KUBERNETES_SERVICE_HOST
            value: "<IP>"
          - name: KUBERNETES_SERVICE_PORT
            value: "443"
          - name: FLUENT_CONTAINER_TAIL_EXCLUDE_PATH
            value: /var/log/containers/fluent*
        resources:
          limits:
            memory: 1024Mi
          requests:
            cpu: 100m
            memory: 512Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlogaudit
          mountPath: /var/log/audit
        # volumeMounts:
        # - name: fluentd-config
        #   mountPath: /fluentd/etc
        # When actual pod logs in /var/lib/docker/containers, the following lines should be used.
        # - name: dockercontainerlogdirectory
        #   mountPath: /var/lib/docker/containers
        #   readOnly: true

        # - name: dockercontainerlogdirectory
        #   mountPath: /var/log/containers
          # readOnly: true

        # When actual pod logs in /var/log/pods, the following lines should be used.
        - name: dockercontainerlogdirectory
          mountPath: /var/log/pods
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      # - name: fluentd-config
        # configMap:
        #   name: fluentd-config
      - name: varlog
        hostPath:
          path: /var/log
      # When actual pod logs in /var/lib/docker/containers, the following lines should be used.
      # - name: dockercontainerlogdirectory
      #   hostPath:
      #     path: /var/lib/docker/containers
      # - name: dockercontainerlogdirectory
      #   hostPath:
      #     path: /var/log/containers
      # # When actual pod logs in /var/log/pods, the following lines should be used.
      - name: dockercontainerlogdirectory
        hostPath:
          path: /var/log/pods
      - name: varlogaudit
        hostPath:
          path: /var/log/audit
```

#### refs:
[fluentd-kubernetes-daemonset](https://github.com/fluent/fluentd-kubernetes-daemonset/tree/master) \
[fluent-plugin-remote_syslog](https://github.com/fluent-plugins-nursery/fluent-plugin-remote_syslog)
