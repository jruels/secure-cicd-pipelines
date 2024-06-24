# Tasks

At the end of this chapter, you will be able to :

- Understand what a [Task](https://github.com/tektoncd/pipeline/blob/main/docs/tasks.md) is?
- Understand how to clone your build resources using a Task
- Understand where cloned sources reside i.e., Workspace
- Create a Task that can build the sources
- How to run a Task
- Write custom tasks

If you are not in the tutorial chapter directory, navigate to it:

```bash
cd ~/secure-cicd-pipelines/labs/tekton_tasks
```

## Tekton Task

Each Task must have the following:

- **apiVersion** - The API version used by the Tekton resource; for example, `tekton.dev/v1beta1`.
- **kind** - Identifies this resource object as a Task object.
- **metadata** - Metadata that uniquely identifies the Task resource object, like name.
  - **name** - the unique name by which the task can be referred
- **spec** - The configuration for a Task resource object.
- **steps** - One or more sub-tasks that will be executed in the defined order. The step has all the attributes like a Kubernetes pod.

There are also a series of optional fields for better control over the resource:

- **inputs** - the resources ingested by the task
- **outputs** - the resources produced by the task

- **params** - the parameters that will be used in the task steps. Each parameter has
  - **name** - the name of the parameter
  - **description** - the description of the parameter
  - **default** - the default value of the parameter
- **results** - The names under which Tasks write execution results.
- **workspaces** - The paths to volumes needed by the Task.
- **volumes** - the task can also mount external volumes using the **volumes** attribute.

NOTE: The TaskRun could override the parameter values; if no parameter value is passed then the **default** value will be used. |

The parameters that were part of the **spec** **params** can be used in the steps using the notation `$(<variable-name>)`.

### Clone Source Code

Review the Task that clones the sources using the `git` command and lists the sources:

```bash
vim tasks/source-lister.yaml
```

The sources are usually cloned to a standard path called `/workspace/source/<params.contextDir>`, in this example the `contextDir` having `quarkus` as default value.

Create a new namespace for the tasks

```bash
kubectl create ns tektonlabs
```



You can create the Task using the command:

```bash
kubectl apply -n tektonlabs -f tasks/source-lister.yaml
```

Verify that your task was created:

Verify task

```bash
tkn -n tektonlabs task ls
NAME            AGE
source-lister   37 seconds ago
```

Lets describe the `task` resource:

```bash
tkn -n tektonlabs task describe source-lister
```

The command shows output similar to:

source-lister Task

```bash
Name:        source-lister
Namespace:   tektonlabs

📨 Input Resources

 No input resources

📡 Output Resources

 No output resources

⚓ Params

 NAME             TYPE     DESCRIPTION              DEFAULT VALUE
 ∙ url            string                            <a href="https://github.com/redhat-scholars/tekton-tutorial-greeter" class="bare">https://github.com/redhat-scholars/tekton-tutorial-greeter</a>
 ∙ revision       string                            master
 ∙ subdirectory   string
 ∙ contextDir     string   The context directo...   quarkus
 ∙ sslVerify      string   defines if http.ssl...   false

📝 Results

 NAME       DESCRIPTION
 ∙ commit   The precise commit ...
 ∙ url      The precise URL tha...

📂 Workspaces

 NAME       DESCRIPTION
 ∙ source   The git repo will b...

🦶 Steps

 ∙ clone
 ∙ ls-build-sources

🗂  Taskruns

 No taskruns
```

Since we have not run the Task yet, the command shows **No taskruns**. You can use the `tkn` CLI tool to run the Task, pass parameters and input sources.

|      | Use the `help` command to see the possible options for the start command:`tkn task start --help` |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

Since we just have to pass the GitHub repo for the source-lister, run the following command to start the `TaskRun`:

```bash
tkn -n tektonlabs task start source-lister --showlog --use-param-defaults
```

|      | The command will automatically accept parameters with default values. You will be asked to provide parameters without default values.When prompted for the name for the workspace, type `output`. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |



Use the following output as an example of the values you should pass.

```bash
Please give specifications for the workspace: output
? Name for the workspace : output
? Value of the Sub Path :
? Type of the Workspace : emptyDir
? Type of EmptyDir :
? Do you want to give specifications for the optional workspace `ssh-directory`: (y/N) n
? Do you want to give specifications for the optional workspace `basic-auth`: (y/N) n
? Do you want to give specifications for the optional workspace `ssl-ca-directory`: (y/N) n
TaskRun started: source-lister-run-22dh4
Waiting for logs to be available...
[clone] + '[' false '=' true ]
[clone] + '[' false '=' true ]
[clone] + '[' false '=' true ]
[clone] + CHECKOUT_DIR=/workspace/output/
[clone] + '[' true '=' true ]
[clone] + cleandir
[clone] + '[' -d /workspace/output/ ]
[clone] + rm -rf /workspace/output//quarkus
[clone] + rm -rf '/workspace/output//.[!.]*'
[clone] + rm -rf '/workspace/output//..?*'
[clone] + test -z
[clone] + test -z
[clone] + test -z
[clone] + git config --global --add safe.directory /workspace/output
[clone] + /ko-app/git-init '-url=https://github.com/redhat-scholars/tekton-tutorial-greeter' '-revision=master' '-refspec=' '-path=/workspace/output/' '-sslVerify=false' '-submodules=true' '-depth=1' '-sparseCheckoutDirectories='
[clone] {"level":"info","ts":1698066996.1377401,"caller":"git/git.go:176","msg":"Successfully cloned https://github.com/redhat-scholars/tekton-tutorial-greeter @ d9291c456db1ce29177b77ffeaa9b71ad80a50e6 (grafted, HEAD, origin/master) in path /workspace/output/"}
[clone] {"level":"info","ts":1698066996.162434,"caller":"git/git.go:215","msg":"Successfully initialized and updated submodules in path /workspace/output/"}
[clone] + cd /workspace/output/
[clone] + git rev-parse HEAD
[clone] + RESULT_SHA=d9291c456db1ce29177b77ffeaa9b71ad80a50e6
[clone] + EXIT_CODE=0
[clone] + '[' 0 '!=' 0 ]
[clone] + git log -1 '--pretty=%ct'
[clone] + RESULT_COMMITTER_DATE=1622532621
[clone] + printf '%s' 1622532621
[clone] + printf '%s' d9291c456db1ce29177b77ffeaa9b71ad80a50e6
[clone] + printf '%s' https://github.com/redhat-scholars/tekton-tutorial-greeter

[ls-build-sources] total 20
[ls-build-sources] drwxr-xr-x    4 65532    65532         4096 Oct 23 13:16 src
[ls-build-sources] -rw-r--r--    1 65532    65532         4322 Oct 23 13:16 pom.xml
[ls-build-sources] -rw-r--r--    1 65532    65532          380 Oct 23 13:16 README.md
[ls-build-sources] -rw-r--r--    1 65532    65532           87 Oct 23 13:16 Dockerfile
```

|      | The container images used by the steps need to be downloaded therefore the first execution of the task will take some time before it starts logging the output to the terminal. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

Remember a Task is running as a pod therefore you can watch your pod to see task lifecycle: `Init`,` PodInitializing`,` Running`, and finally `Completed` as shown below:

While the task is running open a new terminal and run:

Watch the pods

```bash
kubectl -n tektonlabs get pods -w
```

The command should show an output like:

```bash
NAME                                 READY   STATUS       AGE
source-lister-run-6xpgx-pod-c7dc67   0/2     Init:0/2     9s
...
NAME                                 READY   STATUS       AGE
source-lister-run-6kt8d-pod-67b326   0/2     Completed    41s
```

To exit type `ctrl-c`



## Watching logs

If the Task ran successfully you will notice the following logs in your terminal (lines truncated for brevity):

```bash
TaskRun started: source-lister-run-pqx9r
Waiting for logs to be available...
...
[clone] + git config --global --add safe.directory /workspace/output
[clone] + /ko-app/git-init '-url=https://github.com/redhat-scholars/tekton-tutorial-greeter' '-revision=master' '-refspec=' '-path=/workspace/output/' '-sslVerify=false' '-submodules=true' '-depth=1' '-sparseCheckoutDirectories='
[clone] {"level":"info","ts":1698066996.1377401,"caller":"git/git.go:176","msg":"Successfully cloned https://github.com/redhat-scholars/tekton-tutorial-greeter @ d9291c456db1ce29177b77ffeaa9b71ad80a50e6 (grafted, HEAD, origin/master) in path /workspace/output/"}
[clone] {"level":"info","ts":1698066996.162434,"caller":"git/git.go:215","msg":"Successfully initialized and updated submodules in path /workspace/output/"}
[clone] + cd /workspace/output/
[clone] + git rev-parse HEAD
[clone] + RESULT_SHA=d9291c456db1ce29177b77ffeaa9b71ad80a50e6
[clone] + EXIT_CODE=0
[clone] + '[' 0 '!=' 0 ]
[clone] + git log -1 '--pretty=%ct'
[clone] + RESULT_COMMITTER_DATE=1622532621
[clone] + printf '%s' 1622532621
[clone] + printf '%s' d9291c456db1ce29177b77ffeaa9b71ad80a50e6
[clone] + printf '%s' https://github.com/redhat-scholars/tekton-tutorial-greeter

[ls-build-sources] total 20
[ls-build-sources] drwxr-xr-x    4 65532    65532         4096 Oct 23 13:16 src
[ls-build-sources] -rw-r--r--    1 65532    65532         4322 Oct 23 13:16 pom.xml
[ls-build-sources] -rw-r--r--    1 65532    65532          380 Oct 23 13:16 README.md
[ls-build-sources] -rw-r--r--    1 65532    65532           87 Oct 23 13:16 Dockerfile
```

The logs are the consolidated logs from all the Task step containers. You can identify the source of the log i.e the step that has generated the logs using the text within the square brackets `[]` of each log line.

Logs starting with **[ls-build-sources]** are from the container that is responsible for running the Task step i.e. `ls-build-sources`.

## Know the workspace directory

In the example above, there is a log which shows the `git clone` command that cloned the application sources to the `/workspace/output` directory. The **workspace** directory is where your Task/Pipeline sources/build artifacts will be cloned and generated. The `source` is a sub-path, under which Tekton cloned the application sources. It is usually the name of the resources → inputs → Resource of type **Git**.

## Cluster Task

Tasks are by default tied to a namespace i.e their visibility is restricted to the namespace where they were created. E.g. the `source-lister` that we created and ran earlier is tied to tektonlabs.



Cluster Tasks have been deprecated, but for the purposes of this class we are going to play around with them. Later we will discuss Resolvers, which replace Cluster Tasks. Due to this, you will receive deprecation warnings for the following exercises. It is safe to ignore them.

To check lets create a new namespace called `clustertask-demo`:

```bash
kubectl create namespace clustertask-demo
kubectl config set-context --current --namespace clustertask-demo
```

Listing the tasks in the `clustertask-demo` will not show any Tasks as shown in the command output below.

```bash
tkn task ls
No Tasks found
```

The reason that there are no Tasks found is that, we have not created any `ClusterTask` yet. If the `Task` is not of a `ClusterTask` type then we will not be able to run the Task in namespaces other than where it was deployed. Try running our `source-lister` task from within `clustertask-demo`:

```bash
tkn task start source-lister --showlog
```

The command should fail with following output:

```bash
Error: Task name source-lister does not exist in namespace clustertask-demo
```

Review a very simple ClusterTask named `echoer.yaml` as shown in the below listing:

ClusterTask connects.

```yaml
apiVersion: tekton.dev/v1beta1
kind: ClusterTask 
metadata:
  name: echoer
spec:
  steps:
    - image: alpine
      script: | 
        #!/bin/sh
        echo 'Meeow!! from Tekton 😺🚀'
```

|      | The kind ClusterTask makes the task available in all namespaces. |
| ---- | ------------------------------------------------------------ |
|      | The step can also be a shell script 😄                        |

```bash
kubectl apply -n clustertask-demo -f tasks/echoer.yaml
```

### List and Describe ClusterTasks

```bash
tkn clustertask ls
```

|      | You have to use `clustertask` command to list all cluster tasks |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

The comand should show one ClusterTask `echoer`:

```bash
NAME     DESCRIPTION   AGE
echoer                 17 seconds ago
```

Let us decribe ClusterTask `echoer`:

```bash
tkn clustertask describe echoer
```

The comand should return:

```bash
Name:   echoer

🦶 Steps

 ∙ unnamed-0
```

|      | If you compare the output above with the `source-lister` Task, the one big difference with respect to definition is the missing `Namespace` for the `echoer` Task. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

### Run ClusterTask `echoer`

Lets run the task in the current namespace `clustertask-demo`:

```bash
tkn clustertask start echoer --showlog
TaskRun started: echoer-run-5zcqb
Waiting for logs to be available...
[unnamed-0] Meeow!! from Tekton 😺🚀
```

### Run ClusterTask `echoer` in other namespace(s)

Let us now shift back to `tektonlabs` and run the `echoer` task again:

```bash
kubectl config set-context --current --namespace=tektonlabs
tkn clustertask start echoer --showlog
```

The command should produce identical output as shown above.

## Points to Ponder

- Tasks are namespace bound i.e. available only in namespace(s) where they were created
- Tasks resources are interacted with using `tkn task` command and its options
- ClusterTasks are available across the cluster i.e. in any namespace(s) of the cluster
- ClusterTasks resources are interacted with using `tkn clustertask` command and its options



## Creating your own tasks

Now use what you've learned to create some custom Tekton Tasks that meet the following requirements.

Task 1: Create a task that echoes the environment variables.

* Name: `env-printer`
* Echoes "Printing environment details" followed by the system environment variables.

Run the task



Task 2: Create a task that accepts parameters and uses them in a script.

* Name: `greet-task`
* Parameter:
  * Name: `name`
  * Description: `The name of the person to greet`
  * Default: `World`
* Echoes "Hello, [name defined in parameter]!"

Run the task with the default parameter. Once that has run successfully, run it with a custom parameter. 



## Congrats

You have successfully run tasks using Tekton. We will use pipelines to chain tasks together in an upcoming lab.
