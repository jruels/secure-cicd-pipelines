# Building pipelines with Tekton Chains

This tutorial will cover how to set up Chains to sign OCI images built in Tekton, and how to automatically generate and sign in-toto attestations for each image. This tutorial will also cover how to store these attestations in a transparency log and query the log for the attestation.

This tutorial will guide you through:

- Generating your own keypair and storing it as a Kubernetes Secret

- Setting up authentication for your OCI registry to store images, image signatures and signed image attestations

- Configuring Tekton Chains to generate and sign provenance

- Building an image with kaniko in a Tekton TaskRun

- Verifying the signed image and the signed provenance

  

## Prerequisites

## Start minikube with a local registry enabled

1. Delete any previous clusters:

   ```bash
   minikube delete
   ```

2. Start up minikube with insecure registry enabled:

   ```bash
   minikube start --insecure-registry "10.0.0.0/24"
   ```

   The process takes a few seconds, you see an output similar to the following, depending on the [minikube driver](https://minikube.sigs.k8s.io/docs/drivers/) that you are using:

   ```
   😄  minikube v1.29.0
   ✨  Automatically selected the docker driver. Other choices: none, ssh
   📌  Using Docker driver with root privileges
   👍  Starting control plane node minikube in cluster minikube
   🚜  Pulling base image ...
   🔥  Creating docker container (CPUs=2, Memory=7900MB) ...
   🐳  Preparing Kubernetes v1.26.1 on Docker 20.10.23 ...
       ▪ Generating certificates and keys ...
       ▪ Booting up control plane ...
       ▪ Configuring RBAC rules ...
   🔗  Configuring bridge CNI (Container Networking Interface) ...
       ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
   🌟  Enabled addons: storage-provisioner, default-storageclass
   🔎  Verifying Kubernetes components...
   🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
   ```

3. Enable the local registry plugin:

   ```bash
   minikube addons enable registry 
   ```

   The output confirms that the registry plugin is enabled:

   ```
   💡  registry is an addon maintained by Google. For any concerns contact minikube on GitHub.
   You can view the list of minikube maintainers at: https://github.com/kubernetes/minikube/blob/master/OWNERS
       ▪ Using image docker.io/registry:2.8.1
       ▪ Using image gcr.io/google_containers/kube-registry-proxy:0.4
   🔎  Verifying registry addon...
   🌟  The 'registry' addon is enabled
   ```

Now you can push images to a registry within your minikube cluster.



## Install Cosign

Cosign is used for signing OCI containers (and other artifacts) 

```bash
wget  https://github.com/sigstore/cosign/releases/download/v2.2.0/cosign_2.2.0_amd64.deb
sudo dpkg -i cosign_2.2.0_amd64.deb
```

Confirm `cosign` was installed correctly

```bash
cosign version
```



## Install and configure the necessary Tekton components

1. Install Tekton Pipelines:

   ```bash
   kubectl apply --filename \
   https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
   ```

2. Monitor the installation:

   ```bash
   kubectl get po -n tekton-pipelines -w
   ```

   When both `tekton-pipelines-controller` and `tekton-pipelines-webhook` show `1/1` under the `READY` column, you are ready to continue. For example:

   ```
   NAME                                          READY   STATUS    RESTARTS   AGE
   tekton-pipelines-controller-9675574d7-sxtm4   1/1     Running   0          2m28s
   tekton-pipelines-webhook-58b5cbb7dd-s6lfs     1/1     Running   0          2m28s
   ```

   Hit *Ctrl + C* to stop monitoring.

3. Install Tekton Chains:

   ```bash
   kubectl apply --filename \
   https://storage.googleapis.com/tekton-releases/chains/latest/release.yaml
   ```

4. Monitor the installation

   ```bash
   kubectl get po -n tekton-chains -w
   ```

   When `tekton-chains-controller` shows `1/1` under the `READY` column, you are ready to continue. For example:

   ```
   NAME                                        READY   STATUS    RESTARTS   AGE
   tekton-chains-controller-57dcc994b9-vs2f2   1/1     Running   0          2m23s
   ```

   Hit *Ctrl + C* to stop monitoring.

5. Configure Tekton Chains to store the provenance metadata locally:

   ```bash
   kubectl patch configmap chains-config -n tekton-chains \
   -p='{"data":{"artifacts.oci.storage": "", "artifacts.taskrun.format":"in-toto", "artifacts.taskrun.storage": "tekton"}}'
   ```

   The output confirms that the configuration was updated successfully:

   ```
   configmap/chains-config patched
   ```

6. Generate a key pair to sign the artifact provenance:

   ```bash
   cosign generate-key-pair k8s://tekton-chains/signing-secrets
   ```

   You are prompted to enter a password for the private key. For this guide, leave the password empty and press *Enter* twice. A public key, `cosign.pub`, is created in your current directory.

## Build and push a container image

1. Create a file called `pipeline.yaml` and add the following:

   ```yaml
   apiVersion: tekton.dev/v1
   kind: Pipeline
   metadata:
     name: build-push
   spec:
     params:
     - name: image-reference
       type: string
     results:
     - name: image-ARTIFACT_OUTPUTS
       description: Built artifact.
       value:
         uri: $(tasks.kaniko-build.results.IMAGE_URL)
         digest: sha1:$(tasks.kaniko-build.results.IMAGE_DIGEST)
     workspaces:
     - name: shared-data
     tasks: 
     - name: dockerfile
       taskRef:
         name: create-dockerfile
       workspaces:
       - name: source
         workspace: shared-data
     - name: kaniko-build
       runAfter: ["dockerfile"]
       taskRef:
         name: kaniko
       workspaces:
       - name: source
         workspace: shared-data
       params:
       - name: IMAGE
         value: $(params.image-reference)
   ---
   apiVersion: tekton.dev/v1
   kind: Task
   metadata:
     name: create-dockerfile
   spec:
     workspaces:
     - name: source
     steps:
     - name: add-dockerfile
       workingDir: $(workspaces.source.path)
       image: alpine
       script: |
         #!/usr/bin/env ash
         cat <<EOF > Dockerfile
         FROM alpine:3.16
         RUN echo "hello world" > hello.log
         EOF
   ---
   apiVersion: tekton.dev/v1
   kind: Task
   metadata:
     name: kaniko
     labels:
       app.kubernetes.io/version: "0.6"
     annotations:
       tekton.dev/pipelines.minVersion: "0.17.0"
       tekton.dev/categories: Image Build
       tekton.dev/tags: image-build
       tekton.dev/displayName: "Build and upload container image using Kaniko"
       tekton.dev/platforms: "linux/amd64,linux/arm64,linux/ppc64le"
   spec:
     description: >-
       This Task builds a simple Dockerfile with kaniko and pushes to a registry.
       This Task stores the image name and digest as results, allowing Tekton Chains to pick up
       that an image was built & sign it.    
     params:
       - name: IMAGE
         description: Name (reference) of the image to build.
       - name: DOCKERFILE
         description: Path to the Dockerfile to build.
         default: ./Dockerfile
       - name: CONTEXT
         description: The build context used by Kaniko.
         default: ./
       - name: EXTRA_ARGS
         type: array
         default: []
       - name: BUILDER_IMAGE
         description: The image on which builds will run (default is v1.5.1)
         default: gcr.io/kaniko-project/executor:v1.5.1@sha256:c6166717f7fe0b7da44908c986137ecfeab21f31ec3992f6e128fff8a94be8a5
     workspaces:
       - name: source
         description: Holds the context and Dockerfile
       - name: dockerconfig
         description: Includes a docker `config.json`
         optional: true
         mountPath: /kaniko/.docker
     results:
       - name: IMAGE_DIGEST
         description: Digest of the image just built.
       - name: IMAGE_URL
         description: URL of the image just built.
     steps:
       - name: build-and-push
         workingDir: $(workspaces.source.path)
         image: $(params.BUILDER_IMAGE)
         args:
           - $(params.EXTRA_ARGS)
           - --dockerfile=$(params.DOCKERFILE)
           - --context=$(workspaces.source.path)/$(params.CONTEXT) # The user does not need to care the workspace and the source.
           - --destination=$(params.IMAGE)
           - --digest-file=$(results.IMAGE_DIGEST.path)
         # kaniko assumes it is running as root, which means this example fails on platforms
         # that default to run containers as random uid (like OpenShift). Adding this securityContext
         # makes it explicit that it needs to run as root.
         securityContext:
           runAsUser: 0
       - name: write-url
         image: docker.io/library/bash:5.1.4@sha256:c523c636b722339f41b6a431b44588ab2f762c5de5ec3bd7964420ff982fb1d9
         script: |
           set -e
           image="$(params.IMAGE)"
           echo -n "${image}" | tee "$(results.IMAGE_URL.path)"        
   ```

2. Get your cluster IPs:

   ```bash
   kubectl get service --namespace kube-system
   ```

   This shows the IPs of the services on your cluster:

   ```
   NAME       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                  AGE
   kube-dns   ClusterIP   10.96.0.10      <none>        53/UDP,53/TCP,9153/TCP   48m
   registry   ClusterIP   10.101.134.48   <none>        80/TCP,443/TCP           47m
   ```

   Save your registry IP, in this case `10.101.134.48`, for the next step.

3. Create a file called `pipelinerun.yaml` and add the following:

   ```yaml
   apiVersion: tekton.dev/v1
   kind: PipelineRun
   metadata:
     generateName: build-push-run-
   spec: 
     pipelineRef:
       name: build-push
     params:
     - name: image-reference
       value: <registry-ip>/tekton-test
     workspaces:
     - name: shared-data
       volumeClaimTemplate:
         spec:
           accessModes:
           - ReadWriteOnce
           resources:
             requests:
               storage: 1Gi
   ```

   Replace `<registry-ip>` with the value from the previous step.

4. Apply the Pipeline to your cluster:

   ```bash
   kubectl apply -f pipeline.yaml
   ```

   You see the following output:

   ```
   pipeline.tekton.dev/build-push created
   task.tekton.dev/create-dockerfile created
   task.tekton.dev/kaniko created
   ```

5. Run the Pipeline:

   ```bash
   kubectl create -f pipelinerun.yaml
   ```

   A new PipelineRun with a unique name is created:

   ```
   pipelinerun.tekton.dev/build-push-run-q22b5 created 
   ```

6. Use the PipelineRun name, `build-push-run-q22b5` , to monitor the execution:

   ```bash
   tkn pr logs build-push-run-q22b5 -f
   ```

   The output shows the Pipeline completed successfully:

   ```
   [kaniko-build : build-and-push] INFO[0000] Retrieving image manifest alpine:3.16
   [kaniko-build : build-and-push] INFO[0000] Retrieving image alpine:3.16 from registry index.docker.io
   [kaniko-build : build-and-push] INFO[0000] Built cross stage deps: map[]
   [kaniko-build : build-and-push] INFO[0000] Retrieving image manifest alpine:3.16
   [kaniko-build : build-and-push] INFO[0000] Returning cached image manifest
   [kaniko-build : build-and-push] INFO[0000] Executing 0 build triggers
   [kaniko-build : build-and-push] INFO[0000] Unpacking rootfs as cmd RUN echo "hello world" > hello.log requires it.
   [kaniko-build : build-and-push] INFO[0000] RUN echo "hello world" > hello.log
   [kaniko-build : build-and-push] INFO[0000] Taking snapshot of full filesystem...
   [kaniko-build : build-and-push] INFO[0000] cmd: /bin/sh
   [kaniko-build : build-and-push] INFO[0000] args: [-c echo "hello world" > hello.log]
   [kaniko-build : build-and-push] INFO[0000] Running: [/bin/sh -c echo "hello world" > hello.log]
   [kaniko-build : build-and-push] INFO[0000] Taking snapshot of full filesystem...
   [kaniko-build : build-and-push] INFO[0000] Pushing image to 10.101.134.48/tekton-test
   [kaniko-build : build-and-push] INFO[0001] Pushed image to 1 destinations
   
   [kaniko-build : write-url] 10.101.134.48/tekton-test
   ```

## Retrieve and verify the artifact provenance

Tekton Chains silently monitored the execution of the PipelineRun. It recorded and signed the provenance metadata, information about the container that the PipelineRun built and pushed.

1. Get the PipelineRun UID:

   ```bash
   export PR_UID=$(tkn pr describe --last -o  jsonpath='{.metadata.uid}')
   ```

2. Fetch the metadata and store it in a JSON file:

   ```bash
   tkn pr describe --last \
   -o jsonpath="{.metadata.annotations.chains\.tekton\.dev/signature-pipelinerun-$PR_UID}" \
   | base64 -d > metadata.json
   ```

3. View the provenance:

   ```bash
   cat metadata.json | jq -r '.payload' | base64 -d | jq .
   ```

   The output contains a detailed description of the build:

   ```json
   {
     "_type": "https://in-toto.io/Statement/v0.1",
     "predicateType": "https://slsa.dev/provenance/v0.2",
     "subject": null,
     "predicate": {
       "builder": {
         "id": "https://tekton.dev/chains/v2"
       },
       "buildType": "tekton.dev/v1beta1/PipelineRun",
       "invocation": {
         "configSource": {},
         "parameters": {
           "image-reference": "10.101.124.48/tekton-test"
         },
         "environment": {
           "labels": {
             "tekton.dev/pipeline": "build-push"
           }
         }
       },
       "buildConfig": {
         "tasks": [
           {
             "name": "dockerfile",
             "ref": {
               "name": "create-dockerfile",
               "kind": "Task"
             },
             "startedOn": "2023-04-18T04:23:16Z",
             "finishedOn": "2023-04-18T04:23:24Z",
             "status": "Succeeded",
             "steps": [
               {
                 "entryPoint": "cat <<EOF > Dockerfile\nFROM alpine:3.16\nRUN echo \"hello world\" > hello.log\nEOF\n",
                 "arguments": null,
                 "environment": {
                   "container": "add-dockerfile",
                   "image": "docker-pullable://bash@sha256:e0acf0b8fb59c01b6a2b66de360c86bcad5c3cd114db325155970e6bab9663a0"
                 },
                 "annotations": null
               }
             ],
             "invocation": {
               "configSource": {},
               "parameters": {},
               "environment": {
                 "annotations": {
                   "pipeline.tekton.dev/affinity-assistant": "affinity-assistant-fd643aaebc",
                   "pipeline.tekton.dev/release": "086e76a"
                 },
                 "labels": {
                   "app.kubernetes.io/managed-by": "tekton-pipelines",
                   "tekton.dev/memberOf": "tasks",
                   "tekton.dev/pipeline": "build-push",
                   "tekton.dev/pipelineRun": "build-push-run-7gn4d",
                   "tekton.dev/pipelineTask": "dockerfile",
                   "tekton.dev/task": "create-dockerfile"
                 }
               }
             }
           },
           {
             "name": "kaniko-build",
             "after": [
               "dockerfile"
             ],
             "ref": {
               "name": "kaniko",
               "kind": "Task"
             },
             "startedOn": "2023-04-18T04:23:24Z",
             "finishedOn": "2023-04-18T04:23:32Z",
             "status": "Succeeded",
             "steps": [
               {
                 "entryPoint": "",
                 "arguments": [
                   "--dockerfile=./Dockerfile",
                   "--context=/workspace/source/./",
                   "--destination=10.101.124.48/tekton-test",
                   "--digest-file=/tekton/results/IMAGE_DIGEST"
                 ],
                 "environment": {
                   "container": "build-and-push",
                   "image": "docker-pullable://gcr.io/kaniko-project/executor@sha256:c6166717f7fe0b7da44908c986137ecfeab21f31ec3992f6e128fff8a94be8a5"
                 },
                 "annotations": null
               },
               {
                 "entryPoint": "set -e\nimage=\"10.101.124.48/tekton-test\"\necho -n \"${image}\" | tee \"/tekton/results/IMAGE_URL\"\n",
                 "arguments": null,
                 "environment": {
                   "container": "write-url",
                   "image": "docker-pullable://bash@sha256:c523c636b722339f41b6a431b44588ab2f762c5de5ec3bd7964420ff982fb1d9"
                 },
                 "annotations": null
               }
             ],
             "invocation": {
               "configSource": {},
               "parameters": {
                 "BUILDER_IMAGE": "gcr.io/kaniko-project/executor:v1.5.1@sha256:c6166717f7fe0b7da44908c986137ecfeab21f31ec3992f6e128fff8a94be8a5",
                 "CONTEXT": "./",
                 "DOCKERFILE": "./Dockerfile",
                 "EXTRA_ARGS": [],
                 "IMAGE": "10.101.124.48/tekton-test"
               },
               "environment": {
                 "annotations": {
                   "pipeline.tekton.dev/affinity-assistant": "affinity-assistant-fd643aaebc",
                   "pipeline.tekton.dev/release": "086e76a",
                   "tekton.dev/categories": "Image Build",
                   "tekton.dev/displayName": "Build and upload container image using Kaniko",
                   "tekton.dev/pipelines.minVersion": "0.17.0",
                   "tekton.dev/platforms": "linux/amd64,linux/arm64,linux/ppc64le",
                   "tekton.dev/tags": "image-build"
                 },
                 "labels": {
                   "app.kubernetes.io/managed-by": "tekton-pipelines",
                   "app.kubernetes.io/version": "0.6",
                   "tekton.dev/memberOf": "tasks",
                   "tekton.dev/pipeline": "build-push",
                   "tekton.dev/pipelineRun": "build-push-run-7gn4d",
                   "tekton.dev/pipelineTask": "kaniko-build",
                   "tekton.dev/task": "kaniko"
                 }
               }
             },
             "results": [
               {
                 "name": "IMAGE_DIGEST",
                 "type": "string",
                 "value": "sha256:423d2382809c377fcf3a890316769852a6d298e760b34d784dc0222ec7630de3"
               },
               {
                 "name": "IMAGE_URL",
                 "type": "string",
                 "value": "10.101.124.48/tekton-test"
               }
             ]
           }
         ]
       },
       "metadata": {
         "buildStartedOn": "2023-04-18T04:23:16Z",
         "buildFinishedOn": "2023-04-18T04:23:32Z",
         "completeness": {
           "parameters": false,
           "environment": false,
           "materials": false
         },
         "reproducible": false
       },
       "materials": [
         {
           "uri": "docker-pullable://bash",
           "digest": {
             "sha256": "c523c636b722339f41b6a431b44588ab2f762c5de5ec3bd7964420ff982fb1d9"
           }
         },
         {
           "uri": "docker-pullable://bash",
           "digest": {
             "sha256": "e0acf0b8fb59c01b6a2b66de360c86bcad5c3cd114db325155970e6bab9663a0"
           }
         },
         {
           "uri": "docker-pullable://gcr.io/kaniko-project/executor",
           "digest": {
             "sha256": "c6166717f7fe0b7da44908c986137ecfeab21f31ec3992f6e128fff8a94be8a5"
           }
         }
       ]
     }
   }
   ```

4. To verify that the metadata hasn’t been tampered with, check the signature with `cosing`:

   ```bash
   cosign verify-blob-attestation --insecure-ignore-tlog \
   --key k8s://tekton-chains/signing-secrets --signature metadata.json \
   --type slsaprovenance --check-claims=false /dev/null
   ```

   The output confirms that the signature is valid:

   ```
   Verified OK
   ```

## What you just created

This diagram shows what you just deployed:

![getting-started-setup](https://raw.github.com/tektoncd/chains/main/docs/tutorials/images/getting_started.png)