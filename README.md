# kaniko-example

This example was suitable for beginners who want to start using kaniko with local environment. It requires only general kubernetes and docker usage, and NO google cloud or aws or any third part cloud service support. It aims to help user establish a quick start successfully and deep dive after that.

## Table of Content
1. [Prerequisities](#Prerequisities)
2. [Install minikube to run kubernetes locally](#Install-minikube-to-run-kubernetes-locally)
3. [Prepare config files for kaniko](#Prepare-config-files-for-kaniko)
4. [Prepare the mounted directory in minikube](#Prepare-the-mounted-directory-in-minikube)
5. [Create a Secret that holds your authorization token](#Create-a-Secret-that-holds-your-authorization-token)
6. [Create resources in kubernetes](#Create-resources-in-kubernetes)
7. [Pull the image and test](#Pull-the-image-and-test)



## Prerequisities

- Kubernetes basic knowledge about [container](https://kubernetes.io/docs/concepts/overview/what-is-kubernetes/), [pod](https://kubernetes.io/docs/concepts/workloads/pods/pod/), [volume](https://kubernetes.io/docs/concepts/storage/volumes/) and [secret](https://kubernetes.io/docs/concepts/configuration/secret/).

- Docker basic usage and a [dockerhub](https://hub.docker.com/) account to push built image public.

## Install minikube to run kubernetes locally

[Minikube](https://kubernetes.io/docs/setup/minikube/) is a simplest tool to run kubernetes locally. The installation differs based on the os and hypervisor you are using locally, for details please refer to [minikube installation guide](https://kubernetes.io/docs/tasks/tools/install-minikube/). Here in this example, I use ubuntu and virtual box as dependencies.
```
# install virtualbox
$ sudo apt-get update
$ sudo apt-get install virtualbox

# install kubectl
$ sudo apt-get update && sudo apt-get install -y apt-transport-https
$ curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
$ echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
$ sudo apt-get update
$ sudo apt-get install -y kubectl

# install minikube
$ curl -Lo minikube https://storage.googleapis.com/minikube/releases/v0.31.0/minikube-linux-amd64 && chmod +x minikube && sudo cp minikube /usr/local/bin/ && rm minikube

# start minikube
$ minikube start
```

## Prepare config files for kaniko

After installing minikube, we should prepare several config files to create resources in kubernetes, which are:
- [pod.yml] is for starting a kaniko container to build the example image. 
- [volume.yml] is for create a persistent volume used as kaniko build context.
- [volume-claim.yml] is for create a persistent volume claim which will mounted in the kaniko container.

## Prepare the mounted directory in minikube

SSH into the minikube cluster, and create a directory which will be mounted in kaniko container as build context. And put the docker file here. 

```
$ minikube ssh
$ mkdir kaniko && cd kaniko
$ echo 'FROM ubuntu' >> dockerfile
$ echo 'ENTRYPOINT ["/bin/bash", "-c", "echo hello"]' >> dockerfile
$ cat dockerfile
FROM ubuntu
ENTRYPOINT ["/bin/bash", "-c", "echo hello"]
$ pwd
/home/docker/kaniko
```

> Note: It is important to notice that the "hostPath" set in the volume.yml denotes to the path in the virtaul minikube cluster, NOT in the local. So you should run ```minikube ssh``` command first. Or you could use other ways to mount a local directory into minikube vm, including ```minikube mount```, default host folders, etc.

## Create a Secret that holds your authorization token
A Kubernetes cluster uses the Secret of docker-registry type to authenticate with a docker registry to push an image.

Create this Secret, naming it regcred:

```
kubectl create secret docker-registry regcred --docker-server=<your-registry-server> --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email>
```

It will be used in pod.yml config.

## Create resources in kubernetes

```
# create persistent volume
$ kubectl create -f volume.yml
persistentvolume/dockerfile created

# create persistent volume claim
$ kubectl create -f volume-claim.yml
persistentvolumeclaim/dockerfile-claim created

# check whether the volume mounted correctly
$ kubectl get pv dockerfile
NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                      STORAGECLASS    REASON   AGE
dockerfile   10Gi       RWO            Retain           Bound    default/dockerfile-claim   local-storage            1m

# create pod
$ kubectl create -f pod.yml
pod/kaniko created
$ kubectl get pods
NAME     READY   STATUS              RESTARTS   AGE
kaniko   0/1     ContainerCreating   0          7s

# check whether the build complete and show the build logs
$ kubectl get pods
NAME     READY   STATUS      RESTARTS   AGE
kaniko   0/1     Completed   0          34s
$ kubectl logs kaniko
INFO[0000] Downloading base image ubuntu
INFO[0006] Taking snapshot of full filesystem...
INFO[0007] Skipping paths under /dev, as it is a whitelisted directory
INFO[0007] Skipping paths under /kaniko, as it is a whitelisted directory
INFO[0007] Skipping paths under /proc, as it is a whitelisted directory
INFO[0007] Skipping paths under /root, as it is a whitelisted directory
INFO[0007] Skipping paths under /sys, as it is a whitelisted directory
INFO[0007] Skipping paths under /var/run, as it is a whitelisted directory
INFO[0007] Skipping paths under /workspace, as it is a whitelisted directory
INFO[0007] ENTRYPOINT ["/bin/bash", "-c", "echo hello"]
2018/12/16 11:33:39 existing blob: sha256:da1315cffa03c17988ae5c66f56d5f50517652a622afc1611a8bdd6c00b1fde3
2018/12/16 11:33:39 existing blob: sha256:32802c0cfa4defde2981bec336096350d0bb490469c494e21f678b1dcf6d831f
2018/12/16 11:33:39 existing blob: sha256:f85999a86bef2603a9e9a4fa488a7c1f82e471cbb76c3b5068e54e1a9320964a
2018/12/16 11:33:39 existing blob: sha256:fa83472a3562898caaf8d77542181a473a84039376f2ba56254619d9317ba00d
2018/12/16 11:33:40 pushed blob sha256:84e6f8e8c18e55599c4f7c8c3ccb52867fdc398aaeccef85750a40e66fc634c1
2018/12/16 11:33:41 index.docker.io/<user-name>/<repo-name>:latest: digest: sha256:5706190951cbf69cfd880f010da015703a1fdca0f2f25b992c3b9193116bbd88 size: 909
```

## Pull the image and test

If as expected, the kaniko will build image and push to dockerhub successfully. Pull the image to local and run it to test:

```
$ sudo docker run -it <user-name>/<repo-name>
Unable to find image '<user-name>/<repo-name>:latest' locally
latest: Pulling from <user-name>/<repo-name>
32802c0cfa4d: Already exists
da1315cffa03: Already exists
fa83472a3562: Already exists
f85999a86bef: Already exists
Digest: sha256:5706190951cbf69cfd880f010da015703a1fdca0f2f25b992c3b9193116bbd88
Status: Downloaded newer image for <user-name>/<repo-name>:latest
hello
```
