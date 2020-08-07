

Prerequisites:
- Create a Kubernetes cluster.
- Install and configure the latest release of Tekton
- Install Tekton CLI


#### Create a task which contains the steps to complete a set amount of build work.

eg- git-task.yaml

```
# kubectl apply -f git-task.yaml 
task.tekton.dev/build-docker-image-from-git-source created
```

#### Create Pipeline Resources 

PipelineResources are used to define the artifacts you want to pass in and out of your Task.

We have created 2 pipeline resources in this example using [my_pipeline-resource.yaml](https://github.com/RagniGarg97/Tektoncd-example/blob/master/docker-git-example/my_pipeline-resource.yaml) and [git-pipelineResource.yaml](https://github.com/RagniGarg97/Tektoncd-example/blob/master/docker-git-example/git-pipelineResource.yaml)

The resource Created using my_pipeline-resource.yaml specifies a git repository with a specific revision from which the Task will pull the source code and the resource created using git-pipelineResourec.yaml pecifies the repository to which the image built by the Task will be pushed.

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
 
 #### Create a service Account 
 
 Service Account uses the secret to execute your TaskRun. [serviceAccount-taskRun.yaml](https://github.com/RagniGarg97/Tektoncd-example/blob/master/docker-git-example/serviceAccount-taskRun.yaml)
 
 
 #### Create TaskRun
 
 TaskRun is used to run a specific Task. It binds the inputs and outputs to already defined PipelineResources, sets values for variable substitution parameters, and executes the Steps in the Task. [git-taskrun.yaml](https://github.com/RagniGarg97/Tektoncd-example/blob/master/docker-git-example/git-taskrun.yaml)
 
 
 ##### Examine the resources created
 
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


#### Create a Pipeline

A Pipeline defines an ordered series of Tasks that you want to execute along with the corresponding inputs and outputs for each Task.

> In the above example we have created only one task (build-docker-image-from-git-source) and in order to run that task, we created a taskrun (build-docker-image-from-git-source-task-run). Therefore we will create one more task as pipeline is used to run more than one tasks.

we have created another task using [git-deploytask.yaml](https://github.com/RagniGarg97/Tektoncd-example/blob/master/docker-git-example/git-deploytask.yaml)

```
# kubectl get tasks
NAME                                 AGE
build-docker-image-from-git-source   84m
deploy-using-kubectl                 60m
```

Creating a pipeline that uses the tasks created. [git-pipeline.yaml](https://github.com/RagniGarg97/Tektoncd-example/blob/master/docker-git-example/git-pipeline.yaml)

The 'run-kubectl' step in the above example( in the step of 'deploy-using-kubectl' task) requires additional permissions. You must grant those permissions to your ServiceAccount.

First, create a new role called tutorial-role:

```
kubectl create clusterrole tutorial-role \
               --verb=* \
               --resource=deployments,deployments.apps
 ```
               
Next, assign this new role to your ServiceAccount:

```
kubectl create clusterrolebinding tutorial-binding \
             --clusterrole=tutorial-role \
             --serviceaccount=default:tutorial-service
```

#### Create PipelineRun

The PipelineRun automatically defines a corresponding TaskRun for each Task you have defined in your Pipeline collects the results of executing each TaskRun.
[new_pipelinerun.yaml](https://github.com/RagniGarg97/Tektoncd-example/blob/master/docker-git-example/new_pipelinerun.yaml)

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

