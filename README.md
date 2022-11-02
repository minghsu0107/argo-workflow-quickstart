# argo-workflow-quickstart
Scripts are referenced from https://github.com/argoproj/argo-workflows/tree/master/examples
## Installation
- https://github.com/argoproj/argo-workflows/releases/tag/v3.3.8
- https://argoproj.github.io/argo-workflows/installation/
    - [Configuration](https://argoproj.github.io/argo-workflows/environment-variables/)
        - https://argoproj.github.io/argo-workflows/workflow-controller-configmap.yaml
            - configure `resourceRateLimit` to mitigate flooding of the Kubernetes API server by workflows with a large amount of parallel nodes
    - [Prometheus Metrics](https://argoproj.github.io/argo-workflows/metrics/)
    - [Argo Server Auth](https://argoproj.github.io/argo-workflows/argo-server-auth-mode/)
        - [Token Creation for Client Authentication](https://argoproj.github.io/argo-workflows/access-token/#token-creation)

Create an admin serviceaccount:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: argoadmin
  namespace: argo
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: argoadmin
subjects:
- kind: ServiceAccount
  name: argoadmin
  namespace: argo
roleRef:
  kind: ClusterRole
  name: admin # default k8s cluster admin role
  apiGroup: rbac.authorization.k8s.io
```
```bash
SECRET=$(kubectl get sa argoadmin -n argo -o=jsonpath='{.secrets[0].name}')
ARGO_TOKEN="Bearer $(kubectl get secret $SECRET -n argo -o=jsonpath='{.data.token}' | base64 --decode)"
echo $ARGO_TOKEN
```

Remember to override `workflow-controller-configmap` with https://argoproj.github.io/argo-workflows/workflow-controller-configmap.yaml in order to configure Postgres (for job results) and S3 (for job logs). Then you can apply the manifests:
```bash
kubectl create namespace argo
kubectl apply -n argo -f https://github.com/argoproj/argo-workflows/releases/download/v3.3.8/install.yaml
```
Web UI: https://localhost:2746

Open port-forward to access the UI:
```bash
kubectl -n argo port-forward deployment/argo-server 2746:2746
```
### Default Workflow Configuration
For the following workflow setting:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: gc-ttl-
  annotations:
    argo: workflows
  labels:
    foo: bar
spec:
  ttlStrategy:
    secondsAfterSuccess: 5  # Time to live after workflow is successful
  parallelism: 3
```
You can set it as the default value in `workflow-controller-configmap`:
```yaml
# This file describes the config settings available in the workflow controller configmap
apiVersion: v1
kind: ConfigMap
metadata:
  name: workflow-controller-configmap
data:
  # Default values that will apply to all Workflows from this controller, unless overridden on the Workflow-level
  workflowDefaults: |
    metadata:
      annotations:
        argo: workflows
      labels:
        foo: bar
    spec:
      ttlStrategy:
        secondsAfterSuccess: 5
      parallelism: 3
```
### Other Configuration
Security context:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: security-context-
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 8737 #; any non-root user
```
## Argo CLI
```bash
argo submit -n argo hello-world.yaml    # submit a workflow spec to Kubernetes
argo list -n argo                       # list current workflows
argo get hello-world-xxx -n argo        # get info about a specific workflow
argo logs hello-world-xxx -n argo       # print the logs from a workflow
argo delete hello-world-xxx -n argo     # delete workflow
```
You can also run workflow specs directly using `kubectl`:
```bash
kubectl create -f hello-world.yaml -n argo
kubectl get wf -n argo
kubectl get wf hello-world-xxx -n argo
kubectl get po --selector=workflows.argoproj.io/workflow=hello-world-xxx --show-all -n argo  # similar to argo
kubectl logs hello-world-xxx-yyy -c main -n argo
kubectl delete wf hello-world-xxx -n argo
```
Note that we should use `kubectl create` instead of `kubectl apply`. Otherwise the workflow will run only once even if you modify its manifest later.
## Usage
Example job:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: hello-world-
  labels:
    workflows.argoproj.io/archive-strategy: "false"
  annotations:
    workflows.argoproj.io/description: |
      This is a simple hello world example.
spec:
  # # Defines "whalesay" as the "main" template
  entrypoint: whalesay
  templates:
  - name: whalesay
    container:
      image: docker/whalesay:latest
      command: [cowsay]
      args: ["hello world"]
```
Submit a job with serviceaccount `argoadmin`:
```bash
argo submit --serviceaccount argoadmin -n argo --watch hello-world.yaml
```
List workflows:
```bash
argo list -n argo --kubeconfig /path/to/kubeconfig --context k3d-k3s-default
```
Get latest workflow details:
```bash
argo get -n argo @latest
```
Get latest workflow logs:
```bash
argo logs -n argo @latest
```
## Workflow Example
### If-Else
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: coinflip-
spec:
  entrypoint: coinflip
  templates:
  - name: coinflip
    steps:
    - - name: flip-coin
        template: flip-coin
    - - name: heads
        template: heads
        when: "{{steps.flip-coin.outputs.result}} == heads"
      - name: tails
        template: tails
        when: "{{steps.flip-coin.outputs.result}} == tails"

  - name: flip-coin
    script:
      image: python:alpine3.6
      command: [python]
      source: |
        import random
        result = "heads" if random.randint(0,1) == 0 else "tails"
        print(result)

  - name: heads
    container:
      image: alpine:3.6
      command: [sh, -c]
      args: ["echo \"it was heads\""]

  - name: tails
    container:
      image: alpine:3.6
      command: [sh, -c]
      args: ["echo \"it was tails\""]
```
### DAG
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: dag-diamond-
spec:
  entrypoint: diamond
  templates:
  - name: diamond
    dag:
      tasks:
      - name: A
        template: echo
        arguments:
          parameters: [{name: message, value: A}]
      - name: B
        dependencies: [A]
        template: echo
        arguments:
          parameters: [{name: message, value: B}]
      - name: C
        dependencies: [A]
        template: echo
        arguments:
          parameters: [{name: message, value: C}]
      - name: D
        dependencies: [B, C]
        template: echo
        arguments:
          parameters: [{name: message, value: D}]
  #- name: diamond
  #    steps:
  #    - - name: A
  #        template: echo
  #        arguments:
  #          parameters: [{name: message, value: A}]
  #    - - name: B
  #        template: echo
  #        arguments:
  #          parameters: [{name: message, value: B}]
  #      - name: C
  #        template: echo
  #        arguments:
  #          parameters: [{name: message, value: C}]
  #    - - name: D
  #        template: echo
  #        arguments:
  #          parameters: [{name: message, value: D}]

  - name: echo
    inputs:
      parameters:
      - name: message
    container:
      image: alpine:3.7
      command: [echo, "{{inputs.parameters.message}}"]
```
### Artifacts and Parameters
Different types of inputs + passing artifacts and parameters:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: example-
spec:
  entrypoint: main
  # in global
  arguments:
    parameters:
    - name: workflow-param-1
  templates:
  - name: main
    dag:
      tasks:
      - name: step-a
        template: step-template-a
        # in dag
        arguments:
          parameters:
          - name: template-param-1
            value: "{{workflow.parameters.workflow-param-1}}"
      - name: step-b
        dependencies: [step-a]
        template: step-template-b
        arguments:
          parameters:
          - name: template-param-2
            value: "{{tasks.step-a.outputs.parameters.output-param-1}}"
          artifacts:
          - name: input-artifact-2
            from: "{{tasks.step-a.outputs.artifacts.output-artifact-1}}"

  - name: step-template-a
    # in template
    inputs:
      parameters:
        - name: template-param-1
    script:
      image: alpine:3.14
      command: [/bin/sh]
      source: |
          echo "{{inputs.parameters.template-param-1}}" | tee /p1.txt
          echo "my message" > /tmp/message
    outputs:
      parameters:
        - name: output-param-1
          valueFrom:
            path: /p1.txt
      artifacts:
        - name: output-artifact-1
          path: /tmp/message
  - name: step-template-b
    inputs:
      parameters:
      - name: template-param-2
      artifacts:
      - name: input-artifact-2
        path: /tmp/message
    script:
      image: alpine:3.14
      command: [/bin/sh]
      source: |
          echo "{{inputs.parameters.template-param-2}}"
          cat /tmp/message
```
Create workflow:
```bash
argo submit --serviceaccount argoadmin -n argo example.yaml -p 'workflow-param-1="abcd"'
```
Argument from command line:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: conditional-
spec:
  entrypoint: conditional-example
  # global arg
  arguments:
    parameters:
    - name: should-print
      value: "false"

  templates:
  # in template
  - name: conditional-example
    inputs:
      parameters:
      - name: should-print
    steps:
    - - name: print-hello
        template: whalesay
        # when: "{{workflow.parameters.should-print}} == true"
        when: "{{inputs.parameters.should-print}} == true"

  - name: whalesay
    container:
      image: docker/whalesay:latest
      command: [sh, -c]
      args: ["cowsay hello"]
```
Pass argument:
```bash
argo submit examples/conditionals.yaml -n argo -p should-print=true
```
Get result from last step:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: retry-script-
spec:
  entrypoint: main
  templates:
    - name: main
      steps:
        - - name: safe-to-retry
            template: safe-to-retry
        - - name: retry
            template: retry-script
            # in steps
            arguments:
              parameters:
                - name: safe-to-retry
                  value: "{{steps.safe-to-retry.outputs.result}}"

    - name: safe-to-retry
      script:
        image: python:alpine3.6
        command: ["python"]
        source: |
          print("true")
    - name: retry-script
      inputs:
        parameters:
            - name: safe-to-retry
      retryStrategy:
        limit: "3"
        # Only continue retrying if the last exit code is greater than 1 and the input parameter is true
        expression: "asInt(lastRetry.exitCode) > 1 && {{inputs.parameters.safe-to-retry}} == true"
      script:
        image: python:alpine3.6
        command: ["python"]
        # Exit 1 with 50% probability and 2 with 50%
        source: |
          import random;
          import sys;
          exit_code = random.choice([1, 2]);
          sys.exit(exit_code)
```
### Conditional Artifacts and Parameters
- https://argoproj.github.io/argo-workflows/conditional-artifacts-parameters/

Conditional artifacts:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: conditional-artifacts-
  labels:
    workflows.argoproj.io/test: "true"
  annotations:
    workflows.argoproj.io/description: |
      Conditional aartifacts provide a way to choose the output artifacts based on expression.

      In this example the main template has two steps which will run conditionall using `when` .

      Based on the `when` condition one of step will not execute. The main template's output artifact named "result"
      will be set to the executed step's output.
    workflows.argoproj.io/version: '>= 3.1.0'
spec:
  entrypoint: main
  templates:
    - name: main
      steps:
        - - name: flip-coin
            template: flip-coin
        - - name: heads
            template: heads
            when: "{{steps.flip-coin.outputs.result}} == heads"
          - name: tails
            template: tails
            when: "{{steps.flip-coin.outputs.result}} == tails"
      outputs:
        artifacts:
          - name: result
            fromExpression: "steps['flip-coin'].outputs.result == 'heads' ? steps.heads.outputs.artifacts.result : steps.tails.outputs.artifacts.result"

    - name: flip-coin
      script:
        image: python:alpine3.6
        command: [ python ]
        source: |
          import random
          print("heads" if random.randint(0,1) == 0 else "tails")

    - name: heads
      script:
        image: python:alpine3.6
        command: [ python ]
        source: |
          with open("result.txt", "w") as f:
            f.write("it was heads")
      outputs:
        artifacts:
          - name: result
            path: /result.txt

    - name: tails
      script:
        image: python:alpine3.6
        command: [ python ]
        source: |
          with open("result.txt", "w") as f:
            f.write("it was tails")
      outputs:
        artifacts:
          - name: result
            path: /result.txt
```
### Retry
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: retry-backoff-
spec:
  entrypoint: retry-backoff
  templates:
  - name: retry-backoff
    retryStrategy:
      limit: "10"
      backoff:
        duration: "1"       # Must be a string. Default unit is seconds. Could also be a Duration, e.g.: "2m", "6h", "1d"
        factor: "2"
        maxDuration: "1m" # Must be a string. Default unit is seconds. Could also be a Duration, e.g.: "2m", "6h", "1d"
    container:
      image: python:alpine3.6
      command: ["python", -c]
      # fail with a 66% probability
      args: ["import random; import sys; exit_code = random.choice([0, 1, 1]); sys.exit(exit_code)"]
```
```yaml


# This example demonstrates the use of retries for a single container.
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: retry-on-error-
spec:
  entrypoint: error-container
  templates:
  - name: error-container
    retryStrategy:
      limit: "2"
      retryPolicy: "Always"   # Retry on errors AND failures. Also available: "OnFailure" (default), "OnError", and "OnTransientError" (retry only on transient errors such as i/o or TLS handshake timeout. Available after v3.0.0-rc2)
    container:
      image: python
      command: ["python", "-c"]
      # fail with a 80% probability
      args: ["import random; import sys; print('retries: {{retries}}'); exit_code = random.choice(range(0, 5)); sys.exit(exit_code)"]
```
### Synchronization
- https://argoproj.github.io/argo-workflows/synchronization/
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
 name: my-config
data:
  workflow: "1"  # Only one workflow can run at given time in particular namespace
  template: "2"  # Two instances of template can run at a given time in particular namespace
```
Workflow level:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: synchronization-wf-level-
spec:
  entrypoint: whalesay
  synchronization:
    semaphore:
      configMapKeyRef:
        name: my-config
        key: workflow
  templates:
  - name: whalesay
    container:
      image: docker/whalesay:latest
      command: [cowsay]
      args: ["hello world"]
```
Template-level synchronization limits parallel execution of the template across workflows, if templates have the same synchronization reference.
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: synchronization-tmpl-level-
spec:
  entrypoint: synchronization-tmpl-level-example
  templates:
  - name: synchronization-tmpl-level-example
    steps:
    - - name: synchronization-acquire-lock
        template: acquire-lock
        arguments:
          parameters:
          - name: seconds
            value: "{{item}}"
        withParam: '["1","2","3","4","5"]'

  - name: acquire-lock
    synchronization:
      semaphore:
        configMapKeyRef:
          name: my-config
          key: template
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["sleep 10; echo acquired lock"]
```
Two instances of templates will be executed at a given time: even multiple steps/tasks within workflow or different workflows referring to the same template.
### Git Clone With SSH + Vault + Volume
Remember to upload your ssh public key to Github in advance.

Vault configuration:
```bash
vault kv put internal/test/config prikey=@my/path/to/cert

vault policy write myconfig - <<EOF
path "internal/data/test/config" {
  capabilities = ["read"]
}
EOF

# grant argoadmin read permission
vault write auth/kubernetes/role/argoadmin \
    bound_service_account_names=argoadmin \
    bound_service_account_namespaces=argo \
    policies=myconfig \
    ttl=24h
```
```yaml
kind: Workflow
metadata:
  generateName: input-artifact-git-
  # namespace: argo    # uncomment this if you use kubectl apply
spec:
  # serviceAccountName: argoadmin    # uncomment this if you use kubectl apply
  entrypoint: demo
  volumeClaimTemplates:
  - metadata:
      name: workdir
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 1Gi
  templates:
  - name: demo
    steps:
    - - name: clone
        template: git-clone
    - - name: print
        template: print-message
  - name: git-clone
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: 'true'
        vault.hashicorp.com/role: 'argoadmin'
        vault.hashicorp.com/agent-inject-secret-myconfig: 'internal/data/test/config'
        vault.hashicorp.com/agent-inject-template-myconfig: |
          {{- with secret "internal/data/test/config" -}}
          {{ .Data.data.prikey }}
          {{- end -}}
    container:
      image: ubuntu:18.04
      command: ["bash", "-c"]
      args:
        - apt-get update && apt-get install -y git;
          mkdir /root/.ssh/ &&touch /root/.ssh/id_rsa && touch /root/.ssh/known_hosts;
          cat /vault/secrets/myconfig > /root/.ssh/id_rsa && chmod 700 /root/.ssh/id_rsa;
          ssh-keyscan github.com >> /root/.ssh/known_hosts;
          git clone git@github.com:minghsu0107/saga-example.git /mnt/vol;
      volumeMounts:
      - name: workdir
        mountPath: /mnt/vol
  - name: print-message
    container:
      image: alpine:3.14
      command: [sh, -c]
      args: ["ls directory from volume; find /mnt/vol; ls /mnt/vol/"]
      volumeMounts:
      - name: workdir
        mountPath: /mnt/vol
```
```bash
argo submit --serviceaccount argoadmin -n argo git.yaml

# or use kubectl (remember to uncomment serviceaccount and namespace)

kubectl create -f git.yaml
```
### Loops
Loop list:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: loops-param-arg-
spec:
  entrypoint: loop-param-arg-example
  arguments:
    parameters:
    - name: os-list
      value: |
        [
          { "image": "debian", "tag": "9.1" },
          { "image": "debian", "tag": "8.9" },
          { "image": "alpine", "tag": "3.6" },
          { "image": "ubuntu", "tag": "17.10" }
        ]
  templates:
  - name: loop-param-arg-example
    inputs:
      parameters:
      - name: os-list
    steps:
    - - name: test-linux
        template: cat-os-release
        arguments:
          parameters:
          - name: image
            value: "{{item.image}}"
          - name: tag
            value: "{{item.tag}}"
        withParam: "{{inputs.parameters.os-list}}"

  - name: cat-os-release
    inputs:
      parameters:
      - name: image
      - name: tag
    container:
      image: "{{inputs.parameters.image}}:{{inputs.parameters.tag}}"
      command: [cat]
      args: [/etc/os-release]
```
With item maps:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: loops-maps-
spec:
  entrypoint: loop-map-example
  templates:
  - name: loop-map-example
    steps:
    - - name: test-linux
        template: cat-os-release
        arguments:
          parameters:
          - name: image
            value: "{{item.image}}"
          - name: tag
            value: "{{item.tag}}"
        withItems:
        - { image: 'debian', tag: '9.1' }
        - { image: 'debian', tag: '8.9' }
        - { image: 'alpine', tag: '3.6' }
        - { image: 'ubuntu', tag: '17.10' }

  - name: cat-os-release
    inputs:
      parameters:
      - name: image
      - name: tag
    container:
      image: "{{inputs.parameters.image}}:{{inputs.parameters.tag}}"
      command: [cat]
      args: [/etc/os-release]
```
Expanding into multiple parallel steps:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: loops-param-result-
spec:
  entrypoint: loop-param-result-example
  templates:
  - name: loop-param-result-example
    steps:
    - - name: generate
        template: gen-number-list
    - - name: sleep
        template: sleep-n-sec
        arguments:
          parameters:
          - name: seconds
            value: "{{item}}"
        withParam: "{{steps.generate.outputs.result}}"

  - name: gen-number-list
    script:
      image: python:alpine3.6
      command: [python]
      source: |
        import json
        import sys
        json.dump([i for i in range(20, 31)], sys.stdout)
  - name: sleep-n-sec
    inputs:
      parameters:
      - name: seconds
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["echo sleeping for {{inputs.parameters.seconds}} seconds; sleep {{inputs.parameters.seconds}}; echo done"]
```
### Function and Variables
Cast to string:
```yaml=
"{{=string(1)}}"
```
Extract data from JSON:
```yaml=
"{{=jsonpath(inputs.parameters.json, '$.some.path')}}"
```
Filter a list and turn into Json string:
```yaml=
"{{=toJson(filter([1, 3], {# > 1}))}}"
```
Map a list:
```yaml=
"{{=map([1, 2], { # * 2 })}}"
```
Integer calculation for increasing memory limit on retry:
```yaml=
retryStrategy:
  limit: "{{inputs.parameters.retry_limit}}"
  retryPolicy: Always
  backoff:
    duration: 1m
    factor: '2'
podSpecPatch: '{"containers":[{"name":"main","resources":{"limits":{"memory":"{{=(sprig.int(retries) + 1) * 64}}Mi"},"requests":{"cpu":"{{inputs.parameters.required_cpu}}","memory":"{{=(sprig.int(retries) + 1) * 64}}Mi"}}}]}'
```
If you are using helm chart, use the following syntax to escape from helm template:
```yaml=
retryStrategy:
  limit: {{ `{{inputs.parameters.retry_limit}}` }}
  retryPolicy: Always
  backoff:
    duration: 1m
    factor: '2'
podSpecPatch: '{"containers":[{"name":"main","resources":{"limits":{"memory":"{{ `{{=(sprig.int(retries) + 1) * 64}}Mi` }}"},"requests":{"cpu":"{{ `{{inputs.parameters.required_cpu}}` }}","memory":"{{ `{{=(sprig.int(retries) + 1) * 64}}Mi` }}"}}}]}'
```
### Parallelism

Parallelism limits the max total parallel pods that can execute at the same time in a workflow.

Only the outer workflow `seq-step` is limited to parallelism=1:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: parallelism-nested-workflow-
spec:
  arguments:
    parameters:
    - name: seq-list
      value: |
        ["a","b","c","d"]
  entrypoint: A
  templates:
  - name: A
    parallelism: 1
    inputs:
      parameters:
      - name: seq-list
    steps:
    - - name: seq-step
        template: B
        arguments:
          parameters:
          - name: seq-id
            value: "{{item}}"
        withParam: "{{inputs.parameters.seq-list}}"

  - name: B
    inputs:
      parameters:
      - name: seq-id
    steps:
    - - name: jobs
        template: one-job
        arguments:
          parameters:
          - name: seq-id
            value: "{{inputs.parameters.seq-id}}"
        withParam: "[1, 2]"

  - name: one-job
    inputs:
      parameters:
      - name: seq-id
    container:
      image: alpine
      command: ['/bin/sh', '-c']
      args: ["echo {{inputs.parameters.seq-id}}; sleep 30"]
```
Example with vertical and horizontal scalability:
```yaml
# Example with vertical and horizontal scalability                                                                                                                                                                                      
#                                                                                                                                                                                                                                                            
# Imagine we have 'M' workers which work in parallel,                                                                                                                                                                                                        
# each worker should performs 'N' loops sequentially                                                                                                                                                                                                         
#                                                                                                                                                                                                                                                            
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: parallelism-nested-
spec:
  arguments:
    parameters:
    - name: seq-list
      value: |
        ["a","b","c","d"]
    - name: parallel-list
      value: |
        [1,2,3,4]

  entrypoint: parallel-worker
  templates:
  - name: parallel-worker
    inputs:
      parameters:
      - name: seq-list
      - name: parallel-list
    steps:
    - - name: parallel-worker
        template: seq-worker
        arguments:
          parameters:
          - name: seq-list
            value: "{{inputs.parameters.seq-list}}"
          - name: parallel-id
            value: "{{item}}"
        withParam: "{{inputs.parameters.parallel-list}}"

  - name: seq-worker
    parallelism: 1
    inputs:
      parameters:
      - name: seq-list
      - name: parallel-id
    steps:
    - - name: seq-step
        template: one-job
        arguments:
          parameters:
          - name: parallel-id
            value: "{{inputs.parameters.parallel-id}}"
          - name: seq-id
            value: "{{item}}"
        withParam: "{{inputs.parameters.seq-list}}"

  - name: one-job
    inputs:
      parameters:
      - name: seq-id
      - name: parallel-id
    container:
      image: alpine
      command: ['/bin/sh', '-c']
      args: ["echo {{inputs.parameters.parallel-id}} {{inputs.parameters.seq-id}}; sleep 10"]
```
### Event and Webhook
- https://argoproj.github.io/argo-workflows/webhooks/
- https://argoproj.github.io/argo-workflows/events/?fbclid=IwAR2ioKfeRYdVXSpKMRVzH4dr9ZwHh1m45PO16r1xEuMCzbNXpaBJ_KamPx4

Needed role:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: submit-workflow-template
  namespace: argo
rules:
  - apiGroups:
      - argoproj.io
    resources:
      - workfloweventbindings
    verbs:
      - list
  - apiGroups:
      - argoproj.io
    resources:
      - workflowtemplates
    verbs:
      - get
  - apiGroups:
      - argoproj.io
    resources:
      - workflows
    verbs:
      - create
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: submit-workflow
  namespace: argo
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: submit-workflow-template
subjects:
  - kind: ServiceAccount
    name: argoadmin
    namespace: argo
```
```yaml
apiVersion: argoproj.io/v1alpha1
kind: WorkflowEventBinding
metadata:
  name: event-consumer
  namespace: argo
spec:
  event:
    # metadata header name must be lowercase to match in selector
    selector: payload.message != "" && metadata["x-argo-e2e"] == ["true"] && discriminator == "my-discriminator"
  submit:
    workflowTemplateRef:
      name: my-wf-tmple
    arguments:
      parameters:
      - name: message
        valueFrom:
          event: payload.message
---
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: my-wf-tmple
  namespace: argo
spec:
  serviceAccountName: argoadmin
  entrypoint: main
  templates:
    - name: main
      inputs:
        parameters:
          - name: message
            value: "{{workflow.parameters.message}}"
      container:
        image: docker/whalesay:latest
        command: [cowsay]
        args: ["{{inputs.parameters.message}}"]
```
```bash
kubectl apply -f event-template.yml
```
Trigger event:
```bash
curl $ARGO_SERVER/api/v1/events/argo/my-discriminator \
    -H "Authorization: $ARGO_TOKEN" \
    -H "X-Argo-E2E: true" \
    -d '{"message": "hello events"}'
```
For example:
```bash
curl https://localhost:2746/api/v1/events/argo/my-discriminator \
    -H "Authorization: $ARGO_TOKEN" \
    -H "X-Argo-E2E: true" \
    -d '{"message": "hello events"}' \
    --insecure
```
### Cronjob
- Use case: define cronjob workflow in a manifest repository, and sync with Kubernetes by ArgoCD.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: CronWorkflow
metadata:
  name: hello-world
spec:
  schedule: "* * * * *"
  timezone: "America/Los_Angeles"   # Default to local machine timezone
  startingDeadlineSeconds: 0
  concurrencyPolicy: "Replace"      # Default to "Allow"
  successfulJobsHistoryLimit: 4     # Default 3
  failedJobsHistoryLimit: 4         # Default 1
  suspend: false                    # Set to "true" to suspend scheduling
  workflowSpec:                     # Same as workflow
    entrypoint: whalesay
    ttlStrategy:
      secondsAfterCompletion: 300
    templates:
      - name: whalesay
        container:
          image: docker/whalesay:latest
          command: [cowsay]
          args: ["🕓 hello world. Scheduled on: {{workflow.scheduledTime}}"]
```
### TTL
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: gc-ttl-
spec:
  ttlStrategy:
    secondsAfterCompletion: 10 # Time to live after workflow is completed, replaces ttlSecondsAfterFinished
    secondsAfterSuccess: 5     # Time to live after workflow is successful
    secondsAfterFailure: 5     # Time to live after workflow fails
  entrypoint: whalesay
  templates:
  - name: whalesay
    container:
      image: docker/whalesay:latest
      command: [cowsay]
      args: ["hello world"]
```
### Status
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: status-reference-
spec:
  entrypoint: status-reference
  templates:
    - name: status-reference
      steps:
        - - name: flakey-container
            template: flakey-container
            continueOn:
              failed: true
        - - name: failed
            template: failed
            when: "{{steps.flakey-container.status}} == Failed"
          - name: succeeded
            template: succeeded
            when: "{{steps.flakey-container.status}} == Succeeded"

    - name: flakey-container
      container:
        image: alpine:3.6
        command: [sh, -c]
        args: ["exit 1"]

    - name: failed
      container:
        image: alpine:3.6
        command: [sh, -c]
        args: ["echo \"the flakey container failed\""]

    - name: succeeded
      container:
        image: alpine:3.6
        command: [sh, -c]
        args: ["echo \"the flakey container passed\""]
```
### Workflow Template
When working with global parameters, you can instantiate your global variables in your `Workflow` and then directly reference them in your `WorkflowTemplate`. Below is a working example:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: hello-world-template-global-arg
spec:
  serviceAccountName: argoadmin
  templates:
    - name: hello-world
      container:
        image: docker/whalesay
        command: [cowsay]
        args: ["{{workflow.parameters.global-parameter}}"]
---
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: hello-world-wf-global-arg-
spec:
  serviceAccountName: argoadmin
  entrypoint: whalesay
  arguments:
    parameters:
      - name: global-parameter
        value: hello
  templates:
    - name: whalesay
      steps:
        - - name: hello-world
            templateRef:
              name: hello-world-template-global-arg
              template: hello-world
```
Local parameters:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: hello-world-template-local-arg
spec:
  templates:
    - name: hello-world
      inputs:
        parameters:
          - name: msg
            value: "hello world"
      container:
        image: docker/whalesay
        command: [cowsay]
        args: ["{{inputs.parameters.msg}}"]
---
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: hello-world-local-arg-
spec:
  entrypoint: whalesay
  templates:
    - name: whalesay
      steps:
        - - name: hello-world
            templateRef:
              name: hello-world-template-local-arg
              template: hello-world
            arguments:                    # You can pass in arguments as normal
              parameters:
              - name: msg
                value: "hello"
```
### Create Workflow from WorkflowTemplate Spec
```yaml
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: workflow-template-submittable
spec:
  entrypoint: whalesay-template
  arguments:
    parameters:
      - name: message
        value: hello world
  templates:
    - name: whalesay-template
      inputs:
        parameters:
          - name: message
      container:
        image: docker/whalesay
        command: [cowsay]
        args: ["{{inputs.parameters.message}}"]
```
Here is an example of a referring WorkflowTemplate as Workflow and using WorkflowTemplates's entrypoint and Workflow Arguments:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: workflow-template-hello-world-
spec:
  workflowTemplateRef:
    name: workflow-template-submittable
```
Here is an example for referring WorkflowTemplate as Workflow with passing entrypoint and Workflow Arguments to WorkflowTemplate:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: workflow-template-hello-world-
spec:
  entrypoint: whalesay-template
  arguments:
    parameters:
      - name: message
        value: "from workflow"
  workflowTemplateRef:
    name: workflow-template-submittable
```
### Slack Notification
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: exit-handler-slack-
spec:
  entrypoint: say-hello
  onExit: exit-handler
  templates:
  - name: say-hello
    container:
      image: alpine:latest
      command: [sh, -c]
      args: ["echo hello"]

  - name: exit-handler
    container:
      image: curlimages/curl
      command: [sh, -c]
      args: [
        "curl -X POST --data-urlencode 'payload={
          \"text\": \"{{workflow.name}} finished\",
          \"blocks\": [
            {
              \"type\": \"section\",
              \"text\": {
                \"type\": \"mrkdwn\",
                \"text\": \"Workflow {{workflow.name}} {{workflow.status}}\",
              }
            }
          ]
        }'
        YOUR_WEBHOOK_URL_HERE"
      ]
```
