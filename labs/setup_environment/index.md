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
curl -LO https://github.com/tektoncd/cli/releases/download/v0.37.0/tkn_0.37.0_Linux_x86_64.tar.gz
sudo tar xvzf tkn_0.37.0_Linux_x86_64.tar.gz -C /usr/local/bin/ tkn
rm -rf tkn_0.37.0_Linux_x86_64.tar.gz
```



Confirm it is installed correctly. 

```bash 
tkn version
```



## Install the Tekton Operator

Install the Tekton operator in your Kubernetes cluster

```bash
curl -sL https://github.com/operator-framework/operator-lifecycle-manager/releases/download/v0.28.0/install.sh | bash -s v0.28.0
kubectl create -f https://operatorhub.io/install/tektoncd-operator.yaml
```



After install, watch your operator come up using next command.

```bash
kubectl get csv -n operators -w
```



Confirm the output is similar to: 

```
NAME                        DISPLAY             VERSION   REPLACES                    PHASE
tektoncd-operator.v0.70.0   Tektoncd Operator   0.70.0    tektoncd-operator.v0.69.1   Succeeded
```

To exit type `ctrl+c`



## Congrats 

Congratulations! You've set up your lab environment.



