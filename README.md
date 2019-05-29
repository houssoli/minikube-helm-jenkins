# minikube-helm-jenkins

This repository contains sample scripts to set up a persistent Jenkins master with dynamically spinned up Jenkins slaves on minikube.

Start minikube and use VirtualBox as VM driver
> This will create a local one-node Kubernetes cluster in VirtualBox.

```bash
minikube start — vm-driver=virtualbox
```

> Alternatively, as I'm using a linux workstation, running the Kubernetes cluster directly on my local docker host by starting Minikube will require the --vm-driver=none option.

```bash
minikube start --vm-driver=none --apiserver-ips 127.0.0.1 --apiserver-name localhost
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

Verify the namespace using the command

```bash
kubectl get ns
```

Create persistent volume (folder /data is persistent on minikube)
> [Here the official minikube persistent_volumes documentation](https://github.com/kubernetes/minikube/blob/master/docs/persistent_volumes.md)
>In a multi-node Kubernetes cluster you’ll need some solution like NFS to make the mount directory available in the whole cluster.

```bash
kubectl create -f minikube/jenkins-volume.yaml
```

Execute helm to deploy a Jenkins master-slave cluster

> Here we will be using the [Jenkins Helm Chart](https://github.com/helm/charts/tree/master/stable/jenkins) which aims to deploy Jenkins master and slave cluster utilizing the [Jenkins Kubernetes plugin](https://wiki.jenkins-ci.org/display/JENKINS/Kubernetes+Plugin).  
We use the `jenkins-values.yaml` to provide the template our own values which are necessary for our setup.  
We will claim our volume and mount the Docker socket so we can execute Docker commands inside our slave pods.  
Because we are using `minikube` we need to use `NodePort` as service type. Only cloud providers offer load balancers. We define port `32000` as port.  
We can also define which plugins we want to install in our Jenkins. We use some default plugins like [git](https://wiki.jenkins.io/display/JENKINS/Git+Plugin) and the [pipeline plugin](https://wiki.jenkins.io/display/JENKINS/Pipeline+Plugin), but we also add the [greenballs plugin](https://wiki.jenkins.io/display/JENKINS/Green+Balls) which will show a green ball instead of a blue ball after a successful build.

```bash
helm install --namespace jenkins-project --name jenkins -f helm/jenkins-values.yaml stable/jenkins
```

<details><summary markdown='span'>Expand to see what the output looks like</summary>
<p>

```output
NAME:   jenkins
LAST DEPLOYED: Wed May 29 13:23:15 2019
NAMESPACE: jenkins-project
STATUS: DEPLOYED

RESOURCES:
==> v1/ConfigMap
NAME           DATA  AGE
jenkins        6     0s
jenkins-tests  1     0s

==> v1/Deployment
NAME     READY  UP-TO-DATE  AVAILABLE  AGE
jenkins  0/1    1           0          0s

==> v1/PersistentVolumeClaim
NAME     STATUS  VOLUME      CAPACITY  ACCESS MODES  STORAGECLASS  AGE
jenkins  Bound   jenkins-pv  20Gi      RWO           jenkins-pv    0s

==> v1/Pod(related)
NAME                      READY  STATUS    RESTARTS  AGE
jenkins-6bcf9d87cc-sqd49  0/1    Init:0/1  0         0s

==> v1/Secret
NAME     TYPE    DATA  AGE
jenkins  Opaque  2     0s

==> v1/Service
NAME           TYPE       CLUSTER-IP      EXTERNAL-IP  PORT(S)         AGE
jenkins        NodePort   10.107.131.175  <none>       8080:32000/TCP  0s
jenkins-agent  ClusterIP  10.105.104.242  <none>       50000/TCP       0s

==> v1/ServiceAccount
NAME     SECRETS  AGE
jenkins  1        0s

==> v1beta1/ClusterRoleBinding
NAME                  AGE
jenkins-role-binding  0s


NOTES:
1. Get your 'admin' user password by running:
  printf $(kubectl get secret --namespace jenkins-project jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo
2. Get the Jenkins URL to visit by running these commands in the same shell:
  export NODE_PORT=$(kubectl get --namespace jenkins-project -o jsonpath="{.spec.ports[0].nodePort}" services jenkins)
  export NODE_IP=$(kubectl get nodes --namespace jenkins-project -o jsonpath="{.items[0].status.addresses[0].address}")
  echo http://$NODE_IP:$NODE_PORT/login

3. Login with the password from step 1 and the username: admin

For more information on running Jenkins on Kubernetes, visit:
https://cloud.google.com/solutions/jenkins-on-container-engine
Configure the Kubernetes plugin in Jenkins to use the following Service Account name jenkins using the following steps:
  Create a Jenkins credential of type Kubernetes service account with service account name jenkins
  Under configure Jenkins -- Update the credentials config in the cloud section to use the service account credential you created in the step above.
```

</p>
</details>

```output
kubectl -n jenkins-project get configmap,deployment,pvc,pod,secret,svc,serviceaccount,clusterrolebinding
```

<details><summary markdown='span'>Expand to see output</summary>
<p>

```output
NAME                      DATA   AGE
configmap/jenkins         6      4m44s
configmap/jenkins-tests   1      4m44s
NAME                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/jenkins   1/1     1            1           4m44s
NAME                            STATUS   VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/jenkins   Bound    jenkins-pv   20Gi       RWO            jenkins-pv     4m44s
NAME                           READY   STATUS    RESTARTS   AGE
pod/jenkins-6bcf9d87cc-sqd49   1/1     Running   0          4m44s
NAME                         TYPE                                  DATA   AGE
secret/default-token-mxhmh   kubernetes.io/service-account-token   3      45m
secret/jenkins               Opaque                                2      4m44s
secret/jenkins-token-qxlf8   kubernetes.io/service-account-token   3      4m44s
NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
service/jenkins         NodePort    10.107.131.175   <none>        8080:32000/TCP   4m44s
service/jenkins-agent   ClusterIP   10.105.104.242   <none>        50000/TCP        4m44s
NAME                     SECRETS   AGE
serviceaccount/default   1         45m
serviceaccount/jenkins   1         4m44s
NAME                                                                                                AGE
clusterrolebinding.rbac.authorization.k8s.io/cluster-admin                                          16h
clusterrolebinding.rbac.authorization.k8s.io/jenkins-role-binding                                   4m44s
clusterrolebinding.rbac.authorization.k8s.io/kubeadm:kubelet-bootstrap                              16h
clusterrolebinding.rbac.authorization.k8s.io/kubeadm:node-autoapprove-bootstrap                     16h
clusterrolebinding.rbac.authorization.k8s.io/kubeadm:node-autoapprove-certificate-rotation          16h
clusterrolebinding.rbac.authorization.k8s.io/kubeadm:node-proxier                                   16h
clusterrolebinding.rbac.authorization.k8s.io/minikube-rbac                                          16h
clusterrolebinding.rbac.authorization.k8s.io/storage-provisioner                                    16h
clusterrolebinding.rbac.authorization.k8s.io/system:aws-cloud-provider                              16h
clusterrolebinding.rbac.authorization.k8s.io/system:basic-user                                      16h
clusterrolebinding.rbac.authorization.k8s.io/system:controller:attachdetach-controller              16h
clusterrolebinding.rbac.authorization.k8s.io/system:controller:certificate-controller               16h
clusterrolebinding.rbac.authorization.k8s.io/system:controller:clusterrole-aggregation-controller   16h
clusterrolebinding.rbac.authorization.k8s.io/system:controller:cronjob-controller                   16h
clusterrolebinding.rbac.authorization.k8s.io/system:controller:daemon-set-controller                16h
clusterrolebinding.rbac.authorization.k8s.io/system:controller:deployment-controller                16h
clusterrolebinding.rbac.authorization.k8s.io/system:controller:disruption-controller                16h
clusterrolebinding.rbac.authorization.k8s.io/system:controller:endpoint-controller                  16h
clusterrolebinding.rbac.authorization.k8s.io/system:controller:expand-controller                    16h
clusterrolebinding.rbac.authorization.k8s.io/system:controller:generic-garbage-collector            16h
clusterrolebinding.rbac.authorization.k8s.io/system:controller:horizontal-pod-autoscaler            16h
clusterrolebinding.rbac.authorization.k8s.io/system:controller:job-controller                       16h
clusterrolebinding.rbac.authorization.k8s.io/system:controller:namespace-controller                 16h
clusterrolebinding.rbac.authorization.k8s.io/system:controller:node-controller                      16h
clusterrolebinding.rbac.authorization.k8s.io/system:controller:persistent-volume-binder             16h
clusterrolebinding.rbac.authorization.k8s.io/system:controller:pod-garbage-collector                16h
clusterrolebinding.rbac.authorization.k8s.io/system:controller:pv-protection-controller             16h
clusterrolebinding.rbac.authorization.k8s.io/system:controller:pvc-protection-controller            16h
clusterrolebinding.rbac.authorization.k8s.io/system:controller:replicaset-controller                16h
clusterrolebinding.rbac.authorization.k8s.io/system:controller:replication-controller               16h
clusterrolebinding.rbac.authorization.k8s.io/system:controller:resourcequota-controller             16h
clusterrolebinding.rbac.authorization.k8s.io/system:controller:route-controller                     16h
clusterrolebinding.rbac.authorization.k8s.io/system:controller:service-account-controller           16h
clusterrolebinding.rbac.authorization.k8s.io/system:controller:service-controller                   16h
clusterrolebinding.rbac.authorization.k8s.io/system:controller:statefulset-controller               16h
clusterrolebinding.rbac.authorization.k8s.io/system:controller:ttl-controller                       16h
clusterrolebinding.rbac.authorization.k8s.io/system:coredns                                         16h
clusterrolebinding.rbac.authorization.k8s.io/system:discovery                                       16h
clusterrolebinding.rbac.authorization.k8s.io/system:kube-controller-manager                         16h
clusterrolebinding.rbac.authorization.k8s.io/system:kube-dns                                        16h
clusterrolebinding.rbac.authorization.k8s.io/system:kube-scheduler                                  16h
clusterrolebinding.rbac.authorization.k8s.io/system:node                                            16h
clusterrolebinding.rbac.authorization.k8s.io/system:node-proxier                                    16h
clusterrolebinding.rbac.authorization.k8s.io/system:public-info-viewer                              16h
clusterrolebinding.rbac.authorization.k8s.io/system:volume-scheduler                                16h
```

</p>
</details>

Verify the helm install

```bash
helm ls
```

```output
NAME    REVISION        UPDATED                         STATUS          CHART           APP VERSION     NAMESPACE      
jenkins 1               Wed May 29 20:23:34 2019        DEPLOYED        jenkins-0.28.9  lts             jenkins-project
```

Get the admin user password for jenkins

```bash
printf $(kubectl get secret --namespace jenkins-project jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo

O0wF7MjV2O
```

Get the Jenkins URL to visit within Minikube

In case of MiniKube cluster we will be using the minikube externalIP adress that can be found by issuing the command :

```bash
export NODE_PORT=$(kubectl get --namespace jenkins-project -o jsonpath="{.spec.ports[0].nodePort}" services jenkins)
export NODE_IP=$(minikube ip)
echo http://$NODE_IP:$NODE_PORT/login
```

See also

```bash
minikube service list
```

<details><summary markdown='span'>Expand to see output</summary>
<p>

```output
|-----------------|----------------------|-----------------------------|
|    NAMESPACE    |         NAME         |             URL             |
|-----------------|----------------------|-----------------------------|
| default         | kubernetes           | No node port                |
| jenkins-project | jenkins              | http://192.168.99.100:32000 |
| jenkins-project | jenkins-agent        | No node port                |
| kube-system     | kube-dns             | No node port                |
| kube-system     | kubernetes-dashboard | No node port                |
| kube-system     | tiller-deploy        | No node port                |
|-----------------|----------------------|-----------------------------|
```

</p>
</details>

Get the Jenkins URL to visit (when not in minikube, like with the --vm-driver=none option)

```bash
export NODE_PORT=$(kubectl get --namespace jenkins-project -o jsonpath="{.spec.ports[0].nodePort}" services jenkins)
export NODE_IP=$(kubectl get nodes --namespace jenkins-project -o jsonpath="{.items[0].status.addresses[0].address}")
echo http://$NODE_IP:$NODE_PORT/login
```

Get more information

```bash
kubectl get nodes --namespace jenkins-project -o yaml
```

This implementation is following a full tutorial which can be found [here](https://medium.com/@lvthillo/deploy-jenkins-with-dynamic-slaves-in-minikube-8aef5404e9c1), thank to [@lvthillo](https://itnext.io/@lvthillo).
