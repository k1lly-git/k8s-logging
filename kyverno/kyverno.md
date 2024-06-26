## Ошибка с validation
Любой ресурс обязан иметь label team
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-labels
spec:
  background: false
  validationFailureAction: Enforce
  rules:
  - name: check-team
    match:
      any:
      - resources:
          kinds:
          - "*"
          namespaces:
          - default
    validate:
      message: "label 'team' is required"
      pattern:
        metadata:
          labels:
            #1abe1: "?*"
            team: "?*"
  - name: require-image-tag
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "An image tag is required."
      pattern:
        spec:
          containers:
          - image: "*:*"
  - name: validate-image-tag
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "Using a mutable image tag e.g. 'latest' is not allowed."
      pattern:
        spec:
          containers:
          #- image: "!*:*"
          - image: "!*:latest"
  - name: validate-nodeport
    match:
      any:
      - resources:
          kinds:
          - Service
    validate:
      message: "Services of type NodePort are not allowed."
      pattern:
        spec:
          =(type): "!NodePort"
  - name: baseline
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      podSecurity:
        level: baseline
        #level: privileged
        version: latest
  - name: privileged-containers
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: >-
        Privileged mode is disallowed. The fields spec.containers[*].securityContext.privileged,
        spec.initContainers[*].securityContext.privileged, and spec.ephemeralContainers[*].securityContext.privileged must be unset or set to `false`.          
      pattern:
        spec:
          =(ephemeralContainers):
            - =(securityContext):
                =(privileged): "false"
          =(initContainers):
            - =(securityContext):
                =(privileged): "false"
          containers:
            - =(securityContext):
                =(privileged): "false"
  - name: validate-readOnlyRootFilesystem
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "Root filesystem must be read-only."
      pattern:
        spec:
          containers:
          - securityContext:
              readOnlyRootFilesystem: true
```
## Ошибка с mutation
При создании пода мутация меняет образ контейнера на несуществующий
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: mutation-error
spec:
  rules:
  - name: haproxy-image
    match:
      any:
      - resources:
          kinds:
          - Pod
    mutate:
      patchStrategicMerge:
        spec:
          containers:
          - name: image
            image: haproxy:latest
            command:
            - ls
  - name: add-ns
    match:
      any:
      - resources:
          kinds:
          - Pod
          - Service
          - ConfigMap
          - Secret
    mutate:
      patchStrategicMerge:
        metadata:
          namespace: develop
```
## Исключение с capabilities
Контейнерам запрещено использовать capabilities, но можно включать SETUID, SETGID с лейблом dev \
ClusterPolicy:
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: drop-all-capabilities
  annotations:
    policies.kyverno.io/title: Drop All Capabilities
    policies.kyverno.io/category: Best Practices
    policies.kyverno.io/severity: medium
    policies.kyverno.io/minversion: 1.6.0
    policies.kyverno.io/subject: Pod
spec:
  validationFailureAction: Enforce
  background: true
  rules:
    - name: require-drop-all
      match:
        any:
        - resources:
            kinds:
            - Pod
      preconditions:
        all:
        - key: "{{ request.operation || 'BACKGROUND' }}"
          operator: NotEquals
          value: DELETE
      validate:
        message: >-
          Containers must drop `ALL` capabilities.          
        foreach:
          - list: request.object.spec.[ephemeralContainers, initContainers, containers][]
            deny:
              conditions:
                all:
                - key: ALL
                  operator: AnyNotIn
                  value: "{{ element.securityContext.capabilities.drop[].to_upper(@) || `[]` }}"
                  #- key: "{{ request.object.metadata.namespace }}"
                  #operator: NotEquals
                  #value: "dev"
                  #- key: "{{ element.securityContext.capabilities.add[] || '' }}"
                  #operator: AnyNotIn
                  #value:
                  #- SETUID
                  #- SETGID
                  #- ''
    - name: setuid
      match:
        any:
        - resources:
            kinds:
            - Pod
      preconditions:
        all:
        - key: "{{ request.operation || 'BACKGROUND' }}"
          operator: NotEquals
          value: DELETE
      validate:
        message: >-
          Any capabilities added other than SETUID or SETGID are disallowed.          
        foreach:
          - list: request.object.spec.[ephemeralContainers, initContainers, containers][]
            deny:
              conditions:
                all:
                - key: "{{ element.securityContext.capabilities.add[] || '' }}"
                  operator: AnyNotIn
                  value:
                  - SETUID
                  - SETGID
                  - ''
```
PolicyException:
```yaml
apiVersion: kyverno.io/v2beta1
kind: PolicyException
metadata:
  name: exception-cap
spec:
  exceptions:
  - policyName: drop-all-capabilities
    ruleNames:
    - require-drop-all
  match:
    any:
    - resources:
        kinds:
        - Pod
  conditions:
    any:
    #- key: SETUID
    #  operator: AnyNotIn
    #  value: "{{ request.object.spec.[ephemeralContainers, initContainers, containers][].securityContext.capabilities.drop[].to_upper(@) || [] }}"
    - key: "{{ request.object.metadata.labels.app || '' }}"
      operator: Equals
      value: dev
```

alpine образ (для тестов)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: alp
  labels:
    app: dev
spec:
  containers:
  - image: alpine
    name: alp
    imagePullPolicy: Always
    command:
      - /bin/sh
      - "-c"
      - "sleep 10m"
    securityContext:
      capabilities:
        #drop:
        #- ALL
        add: ["SETUID"]
```
