# Setup environment

## Start the Kubernetes cluster 

The lab VMs already have Minikube installed on them. This provides an easy-to-use Kubernetes cluster. 

Start up the cluster 

```bash
minikube start \
   --memory="2200" --cpus="2" \
   --disk-size=15g
```



## Clone lab files

Before setting up the environment, let’s clone the files we will use during class. 

```bash
git clone https://github.com/jruels/secure-cicd-pipelines
```



## Installation of required tools

We need to install some tools used for interacting with Kubernetes and Tekton.

Start by installing the Tekton CLI `tkn`

```bash 
curl -LO https://github.com/tektoncd/cli/releases/download/v0.32.2/tkn_0.32.2_Linux_x86_64.tar.gz 
tar -zxvf tkn_0.32.2_Linux_x86_64.tar.gz -C /tmp/
sudo mv /tmp/tkn /usr/local/bin
```



Confirm it is installed correctly. 

```bash 
tkn version
```



Install `kubectl`

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

Install Tekton into the Kubernetes cluster 

```bash 
kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
```

Create a new namespace for the Tekton lab resources

```bash
kubectl create namespace tektonlabs &&\
kubectl config set-context --current --namespace=tektonlabs
```

