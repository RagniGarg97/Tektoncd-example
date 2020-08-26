#### Create task

Task contains the steps to complete a set amount of build work.

```
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: build-docker-image-from-git-source
spec:
  params:
    - name: pathToDockerFile
      type: string
      description: The path to the dockerfile to build
      default: $(resources.inputs.docker-source.path)/Dockerfile
    - name: pathToContext
      type: string
      description: |
        The build context used by Kaniko
        (https://github.com/GoogleContainerTools/kaniko#kaniko-build-contexts)
      default: $(resources.inputs.docker-source.path)
  resources:
    inputs:
      - name: docker-source
        type: git
    outputs:
      - name: builtImage
        type: image
  steps:
    - name: build-and-push
      image: gcr.io/kaniko-project/executor:v0.16.0
      # specifying DOCKER_CONFIG is required to allow kaniko to detect docker credential
      env:
        - name: "DOCKER_CONFIG"
          value: "/tekton/home/.docker/"
      command:
        - /kaniko/executor
      args:
        - --dockerfile=$(params.pathToDockerFile)
        - --destination=$(resources.outputs.builtImage.url)
        - --context=$(params.pathToContext)
```

```
# kubectl apply -f git-task.yaml 
task.tekton.dev/build-docker-image-from-git-source created

```

#### Create Pipeline Resources

PipelineResources are used to define the artifacts you want to pass in and out of your Task.

The following resource specifies a git repository with a specific revision from which the Task will pull the source code:

```apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: skaffold-git
spec:
  type: git
  params:
    - name: revision
      value: master
    - name: url
      value: https://github.com/GoogleContainerTools/skaffold #configure: change if you want to build something else, perhaps from your own local git repository
```

```
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: skaffold-image-leeroy-web
spec:
  type: image
  params:
    - name: url
      value: docker.io/raginigarg/myrepository  

```

#### Create a secret 

It is used to push your image to your desired image registry:

```
kubectl create secret docker-registry regcred \      
                    --docker-server=<your-registry-server> \
                    --docker-username=<your-name> \
                    --docker-password=<your-pword> \
                    --docker-email=<your-email>
````
> note: regcred is the name of secret 

#### Create a service Account

Service Account uses the secret to execute your TaskRun.

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tutorial-service
secrets:
  - name: regcred
```

#### Create TaskRun

TaskRun is used to run a specific Task. It binds the inputs and outputs to already defined PipelineResources, sets values for variable substitution parameters, and executes the Steps in the Task.

```
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: build-docker-image-from-git-source-task-run
spec:
  serviceAccountName: tutorial-service
  taskRef:
    name: build-docker-image-from-git-source
  params:
    - name: pathToDockerFile
      value: Dockerfile
    - name: pathToContext
      value: $(resources.inputs.docker-source.path)/examples/microservices/leeroy-web #configure: may change according to your source
  resources:
    inputs:
      - name: docker-source
        resourceRef:
          name: skaffold-git
    outputs:
      - name: builtImage
        resourceRef:
          name: skaffold-image-leeroy-web
```

Examine the resources created:
```
# kubectl get tekton-pipelines

NAME                                                 AGE
task.tekton.dev/build-docker-image-from-git-source   72m


NAME                                                                  SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
taskrun.tekton.dev/build-docker-image-from-git-source-task-run        True        Succeeded   72m         71m


NAME                                                    AGE
pipelineresource.tekton.dev/skaffold-git                5d
pipelineresource.tekton.dev/skaffold-image-leeroy-web   150m
```

```
# kubectl get pods
NAME                                                         READY   STATUS      RESTARTS 

build-docker-image-from-git-source-task-run-pod-v7lmd        0/4     Completed   0 
        
```

We can also confirm that the output Docker image has been created in the location specified in the resource definition.


> In the above example we have created only one task (build-docker-image-from-git-source) and in order to run that task, we created a taskrun (build-docker-image-from-git-source-task-run).

#### Create a Pipeline

A Pipeline defines an ordered series of Tasks that you want to execute along with the corresponding inputs and outputs for each Task.

Creating another task:

```
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: deploy-using-kubectl
spec:
  params:
    - name: path
      type: string
      description: Path to the manifest to apply
    - name: yamlPathToImage
      type: string
      description: |
        The path to the image to replace in the yaml manifest (arg to yq)
  resources:
    inputs:
      - name: source
        type: git
      - name: image
        type: image
  steps:
    - name: replace-image
      image: mikefarah/yq
      command: ["yq"]
      args:
        - "w"
        - "-i"
        - "$(params.path)"
        - "$(params.yamlPathToImage)"
        - "$(resources.inputs.image.url)"
    - name: run-kubectl
      image: lachlanevenson/k8s-kubectl
      command: ["kubectl"]
      args:
        - "apply"
        - "-f"
        - "$(params.path)"
 ```
 
 ```
# kubectl get tasks
NAME                                 AGE
build-docker-image-from-git-source   84m
deploy-using-kubectl                 60m
```

Creating a pipeline that uses the tasks created:

```
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: tutorial-pipeline
spec:
  resources:
    - name: source-repo
      type: git
    - name: web-image
      type: image
  tasks:
    - name: build-skaffold-web
      taskRef:
        name: build-docker-image-from-git-source
      params:
        - name: pathToDockerFile
          value: Dockerfile
        - name: pathToContext
          value: /workspace/docker-source/examples/microservices/leeroy-web #configure: may change according to your source
      resources:
        inputs:
          - name: docker-source
            resource: source-repo
        outputs:
          - name: builtImage
            resource: web-image
    - name: deploy-web
      taskRef:
        name: deploy-using-kubectl
      resources:
        inputs:
          - name: source
            resource: source-repo
          - name: image
            resource: web-image
            from:
              - build-skaffold-web
      params:
        - name: path
          value: /workspace/source/examples/microservices/leeroy-web/kubernetes/deployment.yaml #configure: may change according to your source
        - name: yamlPathToImage
          value: "spec.template.spec.containers[0].image"
```
#### Create PipelineRun

The PipelineRun automatically defines a corresponding TaskRun for each Task you have defined in your Pipeline collects the results of executing each TaskRun.

```
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: tutorial-pipeline-run-1
spec:
  serviceAccountName: tutorial-service
  pipelineRef:
    name: tutorial-pipeline
  resources:
    - name: source-repo
      resourceRef:
        name: skaffold-git
    - name: web-image
      resourceRef:
        name: skaffold-image-leeroy-web
```

 In our example, the TaskRun order is as follows:

- tutorial-pipeline-run-1-build-skaffold-web runs build-skaffold-web, since it has no from or runAfter clauses.
- tutorial-pipeline-run-1-deploy-web runs deploy-web because its input web-image comes from build-skaffold-web. Thus, build-skaffold-web must run before deploy-web.

To get the detailed information of pipelineRun:
```
# tkn pipelinerun describe tutorial-pipeline-run-1
Name:              tutorial-pipeline-run-1
Namespace:         default
Pipeline Ref:      tutorial-pipeline
Service Account:   tutorial-service
Timeout:           1h0m0s
Labels:
 tekton.dev/pipeline=tutorial-pipeline

🌡️  Status

STARTED      DURATION   STATUS
2 days ago   1 minute   Succeeded

📦 Resources

 NAME            RESOURCE REF
 ∙ source-repo   skaffold-git
 ∙ web-image     skaffold-image-leeroy-web

⚓ Params

 No params

🗂  Taskruns

 NAME                                                 TASK NAME            STARTED      DURATION     STATUS
 ∙ tutorial-pipeline-run-1-deploy-web-5sjc6           deploy-web           2 days ago   26 seconds   Succeeded
 ∙ tutorial-pipeline-run-1-build-skaffold-web-8kggv   build-skaffold-web   2 days ago   52 seconds   Succeeded
```

```
# kubectl get pods
NAME                                                         READY   STATUS      RESTARTS   
tutorial-pipeline-run-1-build-skaffold-web-8kggv-pod-clvqq   0/4     Completed   0          
tutorial-pipeline-run-1-deploy-web-5sjc6-pod-l8dvn           0/3     Completed   0          
```
