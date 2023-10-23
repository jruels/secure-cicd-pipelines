# Tasks

At the end of this chapter, you will be able to :

- Understand what a [Task](https://github.com/tektoncd/pipeline/blob/main/docs/tasks.md) is?
- Understand how to clone your build resources using a Task
- Understand where cloned sources reside i.e., Workspace
- Create a Task that can build the sources
- How to run a Task
- Use the pipeline resource with TaskRun

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
- **steps** - One or more sub-tasks that will be executed in the defined order. The step has all the attributes like a [Pod spec](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.14/#pod-v1-core)

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

|      | The [TaskRun](http://localhost:3000/tekton-tutorial/tasks.html#tekton-task-run) could override the parameter values; if no parameter value is passed then the **default** value will be used. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

The parameters that were part of the **spec** **params** can be used in the steps using the notation `$(<variable-name>)`.

### Clone Source Code

Review the Task that clones the sources using the `git` command and lists the sources:

```bash
vim tasks/source-lister.yaml
```

The sources are usually cloned to a standard path called `/workspace/source/<params.contextDir>`, in this example the `contextDir` having `quarkus` as default value.

You can create the Task using the command:

```bash
kubectl apply -n tektonlabs -f tasks/source-lister.yaml
```

Verify that your task was created:

Verify task

```bash
tkn task ls
NAME            AGE
source-lister   37 seconds ago
```

Lets describe the `task` resource:

```bash
tkn task describe source-lister
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
tkn task start source-lister --showlog
```

|      | The command will ask you to confirm any default values, in this task we have many parameters with a default value. The Tekton CLI will ask you confirm them. Be sure press Enter to start the TaskRun after providing inputs to the prompts. When prompted for the name for the workspace, type `output`:`Please give specifications for the workspace: output ? Name for the workspace : output` |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

The command should show output like:

```bash
? Value for param `url` of type `string`? (Default is `https://github.com/redhat-scholars/tekton-tutorial-greeter`) https://github.com/redhat-scholars/tekton-tutorial-greeter
? Value for param `revision` of type `string`? (Default is `master`) master
? Value for param `refspec` of type `string`? (Default is ``)
? Value for param `contextDir` of type `string`? (Default is `quarkus`) quarkus
? Value for param `submodules` of type `string`? (Default is `true`) true
? Value for param `depth` of type `string`? (Default is `1`) 1
? Value for param `sslVerify` of type `string`? (Default is `false`) false
? Value for param `crtFileName` of type `string`? (Default is `ca-bundle.crt`) ca-bundle.crt
? Value for param `subdirectory` of type `string`? (Default is ``)
? Value for param `sparseCheckoutDirectories` of type `string`? (Default is ``)
? Value for param `deleteExisting` of type `string`? (Default is `true`) true
? Value for param `httpProxy` of type `string`? (Default is ``)
? Value for param `httpsProxy` of type `string`? (Default is ``)
? Value for param `noProxy` of type `string`? (Default is ``)
? Value for param `verbose` of type `string`? (Default is `true`) true
? Value for param `gitInitImage` of type `string`? (Default is `gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/git-init:v0.40.2`) gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/git-init:v0.40.2
? Value for param `userHome` of type `string`? (Default is `/home/git`) /home/git
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
kubectl get pods -w
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

e.g.

Logs starting with **[ls-build-sources]** are from the container that is responsible for running the Task step i.e. `ls-build-sources`.

## Know the workspace directory

In the example above, there is a log which shows the `git clone` command that cloned the application sources to the `/workspace/output` directory. The **workspace** directory is where your Task/Pipeline sources/build artifacts will be cloned and generated. The `source` is a sub-path, under which Tekton cloned the application sources. It is usually the name of the resources → inputs → Resource of type **Git**.

## Cluster Task

Tasks are by default tied to a namespace i.e their visibility is restricted to the namespace where they were created. E.g. the `source-lister` that we created and ran earlier is tied to tektonlabs.

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

Let us now create a very simple ClusterTask called echoer as shown in the below listing:

List echoer ClusterTask

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
echoer                 18 minutes ago
```

Let us decribe ClusterTask `echoer`:

```bash
tkn clustertask describe echoer
```

The comand should return:

```bash
Name:   echoer

📨 Input Resources

 No input resources

📡 Output Resources

 No output resources

⚓ Params

 No params

📝 Results

 No results

📂 Workspaces

 No workspaces

🦶 Steps

 ∙ unnamed-0

🗂  Taskruns

 No taskruns
```

|      | If you compare the output above with [source-lister Task](http://localhost:3000/tekton-tutorial/tasks.html#localtask-source-lister), the one big difference with respect to definition is the missing `Namespace` for the `echoer` Task. |
| ---- | ------------------------------------------------------------ |
|      |                                                              |

### Run ClusterTask `echoer`

Lets run the task in the current namespace `clustertask-demo`:

```bash
tkn clustertask start echoer --showlog
TaskRun started: echoer-run-75n6g
Waiting for logs to be available...
[unnamed-0] Meeow!! from Tekton 😺🚀
```

### Run ClusterTask `echoer` in other namespace(s)

Let us now shift back to `tektonlabs` and run the `echoer` task again:

```bash
kubectl config set-context --current --namespace=tektonlabs
Context "tektonlabs" modified.
tkn clustertask start echoer --showlog
```

The command should produce identical output as shown above.

## Points to Ponder

- Tasks are namespace bound i.e. available only in namespace(s) where they were created
- Tasks resources are interacted with using `tkn task` command and its options
- ClusterTasks are available across the cluster i.e. in any namespace(s) of the cluster
- ClusterTasks resources are interacted with using `tkn clustertask` command and its options

## Congrats

You have successfully run tasks using Tekton. We will use pipelines to chain tasks together in an upcoming lab.