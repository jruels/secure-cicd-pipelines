# Microservices setup

This document describes the configuration setup for the VMs used in the microservices course.

This is being provided in case anyone wants to create a similar setup environment for themselves. This is not AWS specific but should work on any clean Ubuntu box.

## AWS VM description

AWS EC2 Ubuntu instance using t3.medium type. Two cores are needed for Kubernetes.  80 gig disk storage.  Running k8s on the default resulted in disk full errors.

The instance should have an external IP address since it is used in several of the labs.

Security group should have inbound rules to open standard `ssh`, `http` and `https` ports as well as port `8080`

## Base setup

`sudo apt-get update`
`sudo apt-get upgrade -y`

`sudo apt-get -y install nodejs npm`
`sudo apt-get -y install fontconfig openjdk-11-jre`

## Docker


`sudo apt-get install -y ca-certificates curl gnupg  lsb-release`

`sudo mkdir -p /etc/apt/keyrings`

`curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg`


```
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

`sudo apt-get update`


`sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin docker-compose`

`sudo service docker start`

`sudo usermod -aG docker $USER`

**Requires logout to take effect. Confirm by running the following after a new login.**

`docker run hello-world`

## Kubernetes

`Set up to run kubernetes and minikube`

`sudo apt-get install socat`

**You will have to accept the oracle license terms during installation**

`curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64`


`sudo cp minikube-linux-amd64 /usr/local/bin/minikube`

`sudo chmod 777 /usr/local/bin/minikube`

`minikube version`

__Should see something like:__
```
minikube version: v1.27.0
commit: 4243041b7a72319b9be7842a7d34b6767bbdac2b
```

`curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"`

`chmod +x ./kubectl`

`sudo mv ./kubectl /usr/local/bin/kubectl`

`kubectl version -o json`

__You should see something like:__

```
ubuntu@ip-172-31-18-218:~$ sudo kubectl version -o json
{
  "clientVersion": {
    "major": "1",
    "minor": "25",
    "gitVersion": "v1.25.2",
    "gitCommit": "5835544ca568b757a8ecae5c153f317e5736700e",
    "gitTreeState": "clean",
    "buildDate": "2022-09-21T14:33:49Z",
    "goVersion": "go1.19.1",
    "compiler": "gc",
    "platform": "linux/amd64"
  },
  "kustomizeVersion": "v4.5.7"
}
```

## The following starts Kubernetes

`minikube start `

__The last line should be something like:__

```
* Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default``
```

`kubectl cluster-info`

__You should see something like:__

```
Kubernetes control plane is running at https://192.168.49.2:8443
CoreDNS is running at https://192.168.49.2:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

## To shut down kubernetes

`minikube stop`