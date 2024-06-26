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
    validate:
      message: "label 'team' is required"
      pattern:
        metadata:
          labels:
            team: "?*"
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
  - name: mutation-error
    match:
      any:
      - resources:
          kinds:
          - Pod
    mutate:
      patchStrategicMerge:
        spec:
          containers:
          - name: invalid-container
            image: invalid-image
```
## Исключение с capabilities
Контейнерам запрещено использовать capabilities, но можно включать SETUID, SETGID с лейблом dev
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
    policies.kyverno.io/description: >-
      Capabilities permit privileged actions without giving full root access. All
      capabilities should be dropped from a Pod, with only those required added back.
      This policy ensures that all containers explicitly specify the `drop: ["ALL"]`
      ability. Note that this policy also illustrates how to cover drop entries in any
      case although this may not strictly conform to the Pod Security Standards.      
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
                - key: "{{ request.object.metadata.namespace }}"
                  operator: NotEquals
                  value: "dev"
                - key: "{{ element.securityContext.capabilities.drop[] | contains('SETUID') || element.securityContext.capabilities.drop[] | contains('SETGID') }}"
                  operator: NotEquals
                  value: true
```
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
