# minikube-helm-jenkins

This repository contains sample scripts to set up a persistent Jenkins master with dynamically spinned up Jenkins slaves on minikube.

Start minikube and use VirtualBox as VM driver
> This will create a local one-node Kubernetes cluster in VirtualBox.

```bash
minikube start — vm-driver=virtualbox
```

Verify minikube is running

```bash
minikube status
```

```output
    minikube: Running
    cluster: Running
    kubectl: Correctly Configured: pointing to minikube-vm at 192.168.99.100
```

Open the minikube dashboard by typing

```bash
minikube dashboard
```

Create namespace

```bash
kubectl create -f minikube/jenkins-namespace.yaml
```

Create persistent volume (folder /data is persistent on minikube)

```bash
kubectl create -f minikube/jenkins-volume.yaml
```

Execute helm

```bash
helm install --name jenkins -f helm/jenkins-values.yaml stable/jenkins --namespace jenkins-project
```

Check admin password for jenkins

```bash
printf $(kubectl get secret --namespace jenkins-project jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo
```

This implementation is following a full tutorial which can be found [here](https://medium.com/@lvthillo/deploy-jenkins-with-dynamic-slaves-in-minikube-8aef5404e9c1), thank to [@lvthillo](https://itnext.io/@lvthillo).
