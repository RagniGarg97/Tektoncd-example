

##### Prerequisites:
- Create a Kubernetes cluster.
- Install and configure the latest release of Tekton
- Install Tekton CLI


#### Create a task which contains the steps to complete a set amount of build work.

eg- [task.yaml](https://github.com/RagniGarg97/Tektoncd-example/blob/master/ibmcloud/task.yaml)

```
# kubectl apply -f task.yaml 
task.tekton.dev/new-task created
```

#### Create Pipeline Resources 

PipelineResources are used to define the artifacts you want to pass in and out of your Task.

We have created 2 pipeline resources in this example using [git-resource.yaml](https://github.com/RagniGarg97/Tektoncd-example/blob/master/ibmcloud/git-resource.yaml) and [docker-resource.yaml](https://github.com/RagniGarg97/Tektoncd-example/blob/master/ibmcloud/docker-resource.yaml)

The resource Created using git-resource.yaml specifies a git repository with a specific revision from which the Task will pull the source code and the resource created using docker-resource.yaml specifies the repository to which the image built by the Task will be pushed.

#### Create a secret

A secret to push your image to your desired image registry:

``` 
kubectl create secret docker-registry regcred \      
                    --docker-server=<your-registry-server> \
                    --docker-username=<your-name> \
                    --docker-password=<your-pword> \
                    --docker-email=<your-email> 
```
                    
 > note: regcred is the name of the secret
 
 or use can use [docker-secret.yaml](https://github.com/RagniGarg97/Tektoncd-example/blob/master/ibmcloud/docker-secret.yaml)
 
 #### Create a service Account 
 
 Service Account uses the secret to execute your TaskRun. [serviceaccount.yaml](https://github.com/RagniGarg97/Tektoncd-example/blob/master/ibmcloud/serviceaccount.yaml)
 
 
 #### Create TaskRun
 
 TaskRun is used to run a specific Task. It binds the inputs and outputs to already defined PipelineResources, sets values for variable substitution parameters, and executes the Steps in the Task. [taskrun.yaml](https://github.com/RagniGarg97/Tektoncd-example/blob/master/ibmcloud/taskrun.yaml)
 
 
 ##### Examine the resources created
 
 ```
 # kubectl get tekton-pipelines
NAME                                                                  SUCCEEDED   REASON      STARTTIME   COMPLETIONTIME
taskrun.tekton.dev/new-taskrun                                        True        Succeeded   17h         17h

NAME                                                 AGE
task.tekton.dev/new-task                             19h

NAME                                                    AGE
pipelineresource.tekton.dev/ibmcloud-git                23h
pipelineresource.tekton.dev/ibmcloud-image              23h

```

To see the result of executing your TaskRun:

```
# tkn taskrun describe new-taskrun
Name:              new-taskrun
Namespace:         default
Task Ref:          new-task
Service Account:   tutorial-service
Timeout:           1h0m0s
Labels:
 app.kubernetes.io/managed-by=tekton-pipelines
 tekton.dev/task=new-task

🌡️  Status

STARTED        DURATION    STATUS
17 hours ago   1 minute    Succeeded

📨 Input Resources

 NAME              RESOURCE REF
 ∙ docker-source   ibmcloud-git

📡 Output Resources

 NAME           RESOURCE REF
 ∙ builtImage   ibmcloud-image

⚓ Params

 NAME                 VALUE
 ∙ pathToDockerFile   images/ibm-cloud/Dockerfile

🦶 Steps

 NAME                               STATUS
 ∙ create-dir-builtimage-5l58l      Completed
 ∙ git-source-docker-source-h5cj7   Completed
 ∙ build-and-push                   Completed
 ∙ image-digest-exporter-gpzfz      Completed

🚗 Sidecars

No sidecars
```
