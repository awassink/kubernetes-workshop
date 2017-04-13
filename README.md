# kubernetes-workshop

## Setup
Once all the services on the worker node are up and running we can check that the node is added to the cluster on the master node. That can be done with kubectl, a tool for managing kubernetes. Kubectl for ARM can be downloaded from googleapis storage.  The default cluster server (master node) is configured, so that we don't have to specify it every time we run a command with kubectl. You can specify multiple clusters and use contexts to switch between them.

```bash
$ kubectl config view
apiVersion: v1
clusters:
- cluster:
    server: http://rpi-master-1:8080
  name: rpi-cluster
contexts:
- context:
    cluster: rpi-cluster
    namespace: rpi-node-61
    user: ""
  name: rpi-cluster
current-context: rpi-cluster
kind: Config
preferences: {}
users: []
kubectl get nodes shows which cluster nodes are registered along with its status.
$ kubectl get nodes
NAME            STATUS    AGE
10.150.42.100   Ready     2h
10.150.42.101   Ready     2h
```
 
## Running a container on kubernetes
An easy way to test the cluster is by running a simple docker image for ARM like the rpi-nginx one. kubectl run can be used to run the image as a container in a pod. kubectl get pods shows the pods that are registered along with its status. kubectl describe gives more detailed information on a specific resource. Each pod (container) get a unique ip-address assigned wihtin the cluster and is accessible on that througout the cluster, thanks to flannel overlay network. A pod can be deleted using kubectl delete pod/deployment/service <name>. Note that the run command creates a deployment (http://kubernetes.io/docs/user-guide/deployments/) which will ensure a crashed or deleted pod is restored. To remove your deployment, use kubectl delete deployment nginx. (kubectl help is your friend!) The --port flag exposes the pods to the internal network. The --labels flag ensures your pods are visible. Please use both flags. 
Note that your nginx pod may seem to be stuck at "ContainerCreating" because it has to download the image first.

```bash
$ kubectl run nginx --image=nginx --port=80 --labels="run=nginx,visualize=true"
deployment "nginx" created
 
$ kubectl get pods -o wide
NAME                      READY     STATUS    RESTARTS   AGE       IP           NODE
nginx-1665122148-4amzl    1/1       Running   0          47s       10.1.87.2    10.150.42.100
$ kubectl describe pod nginx-1665122148-4amzl
Name:       nginx-1665122148-4amzl
Namespace:  rpi-node-61
Node:       10.150.42.100/10.150.42.100
Start Time: Wed, 19 Oct 2016 02:18:38 +0200
Labels:     pod-template-hash=1665122148
        run=nginx
        visualize=true
Status:     Running
IP:     10.1.87.2
Controllers:    ReplicaSet/nginx-1665122148
Containers:
$ kubectl get rs -o=wide
NAME                DESIRED   CURRENT   AGE       CONTAINER(S)   IMAGE(S)  SELECTOR
nginx-1665122148    1         1         3m        nginx          nginx     pod-template-hash=1665122148,run=nginx,visualize=true
  
$ kubectl get deployment -o=wide
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx     1         1         1            1           4m
```
Now the container is running with kubernetes, the NGINX application is directly accessible via its IP address within the kubernetes cluster. Note that this is an IP address within the flannel overlaying network and is not accessible from outside the cluster. Also note that we do not have to specify any port mappings from the container to the host. 
```bash
$ curl http://10.1.87.2/
<html>
<head>
<title>Welcome to nginx!</title>
</head>
<body bgcolor="white" text="black">
<center><h1>Welcome to nginx!</h1></center>
</body>
</html>
```
##Exposing containers on kubernetes
Now the pod is running, but the application is not generally accessible. That can be achieved by creating a service in kubernetes. The service will have a cluster IP-address assigned, which is the IP-address the service is avalailable at within the cluster (10.0.0.*). Use the IP-address of your a worker node as external IP and the service becomes available outside of the cluster (e.g.10.150.42.103 in my case). Check in your browser that http://<ip-address-of-your-node>:90/ is available.
```bash
$ kubectl expose deployment nginx --port=90 --target-port=80 --external-ip=<my node's ip>
service "nginx" exposed
  
$ kubectl get svc
NAME      CLUSTER-IP   EXTERNAL-IP     PORT(S)   AGE
nginx     10.0.0.102   10.150.42.103   90/TCP    57s
$ curl http://10.150.42.103:90
AND/OR (but not accessible to the outside world)
$ curl http://10.0.0.102:90
<html>
<head>
<title>Welcome to nginx!</title>
</head>
<body bgcolor="white" text="black">
<center><h1>Welcome to nginx!</h1></center>
</body>
</html>
```
 
##Scaling
The number of pod serving a service can easily be scaled via kubectl. Use kubectl scale to do so. Check out the visualizer the moment when you execute the scale command.
```bash
$ kubectl scale --replicas=3 deployment nginx
deployment "nginx" scaled
 
$ kubectl get pods -o=wide
NAME                      READY     STATUS              RESTARTS   AGE       IP           NODE
nginx-1665122148-4amzl    1/1       Running             0          13m       10.1.87.2    10.150.42.100
nginx-1665122148-peep7    0/1       ContainerCreating   0          1m        <none>       10.150.42.176
nginx-1665122148-shm28    0/1       ContainerCreating   0          1m        <none>       10.150.42.183
```
##Doing a rolling update with Kubernetes (no service downtime)
In order to demonstrate a rolling update, we will use some prepared nginx containers which serve different static html depending on the version.  Please remove your current deployment and deploy version 1 of this image, with 4 replicas, exposing port 80 on the pods (tip: if you don't remove the service exposing your raspi's port 90 to the world you can reuse it for this deployment).
```bash
$ kubectl delete deployment nginx
deployment "nginx" deleted
 
$ kubectl run nginx --image=awassink/nginx-withcontent:3 --port=80 --replicas=4 --labels="run=nginx,visualize=true"
deployment "nginx" created
```
Now we can see Kubernetes' full magic at work. We will edit the deployment (http://kubernetes.io/docs/user-guide/deployments/#updating-a-deployment) to start using the second version of the image, which will be rolled out by the system, replacing one pod at a time. The service will never go down, during the update a user simply gets served either the old or the new version. To kick the update off, you must edit the deployment. Take note of the different parts of this deployment file. You can write such a file yourself to deploy your applications, which is often more practical than having a bloke or gall hammer commands into a cluster with kubectl. For now, change the container image to version 4. For those unfamiliar with this editor, start editing with insert, stop editing with esc, save the result with :w and quit with :q. Alternatively, you can set the image directly.
```bash
$ kubectl edit deployment nginx
deployment nginx edited
 
OR
 
$ kubectl set image deployment/nginx nginx=awassink/nginx-withcontent:4
deployment "nginx" image updated
```
Now let's assume that sometimes we inadvertently mess up and deploy a version of our application that is utterly broken. We get that dreaded midnight phonecall that a memory leak is destroying everything we care about, like uptime and service availability and professional pride.
Thanks to Kubernetes, we can run a single command from our laptop and get back to bed. First we will checkout the rollout history, pick a version to restore and then deploy it before snoring of happily.
```bash
$ kubectl rollout history deployment/nginx
deployments "nginx":
REVISION    CHANGE-CAUSE
1        <none>
2        <none>
 
# see details of a specific revision
$ kubectl rollout history deployment/nginx-deployment --revision=2
...
 
# execute the rollback
$ kubectl rollout undo deployment/nginx --to-revision=1
deployment "nginx" rolled back
```
For more details about the deployment rollback functionality, see http://kubernetes.io/docs/user-guide/deployments/#rolling-back-a-deployment.
Creating services, deployments and pods from configuration files
Kubernetes resources can also be created from configuration files instead of via the command line. This makes it easy to put this kubernetes configuration in version control and maintain it from there.
```bash
$ kubectl delete svc nginx
service "nginx" deleted
 
$ kubectl delete deployment nginx
deployment "nginx" deleted
 
$ kubectl create -f nginx-deployment.yaml
replicationcontroller "nginx" created
 
$ kubectl create -f nginx-svc.yaml
service "nginx" created
```
Now edit the deployment yaml file to use a different image version (4->3)
```bash
$ kubectl replace -f nginx-deployment.yaml
```  

##Deploying a three tier application
Creating and claiming persisted volumes
The buildserver also hosts NFS service providing multiple volumes for mounting. In Kubernetes you can make a volume available for usage by creating a Persisted Volume.
Edit the nfs-pv.yaml file so that the nfs share path matches your node. Also change the PV name to a unique value.
```bash
$ kubectl create -f nfs-pv.yaml
persistentvolume "nfs-share-61" created
$ kubectl get pv
NAME           CAPACITY   ACCESSMODES   STATUS      CLAIM                    REASON    AGE
nfs-share-61   1Gi        RWO           Available                                      28s
```
Before the volume can be used it needs to be claimed for a certain application. This is done by creating a Persisted Volume Claim.
```bash
$ kubectl create -f cddb-pvc.yaml
persistentvolumeclaim "cddb-mysql-pv-claim" created
 
$ kubectl get pvc
NAME             STATUS    VOLUME         CAPACITY   ACCESSMODES   AGE
mysql-pv-claim   Bound     nfs-share-61   1Gi        RWO           12s
$ kubectl get pv
NAME           CAPACITY   ACCESSMODES   STATUS    CLAIM                        REASON    AGE
nfs-share-61   1Gi        RWO           Bound     rpi-node-61/mysql-pv-claim             5m
```
More information can be found here: http://kubernetes.io/docs/user-guide/persistent-volumes/

Some of the deployment and service yaml files in the "assignment-3" folder are incomplete. Open the files and lookup the missing values in the Kubernetes documentation http://kubernetes.io/docs/.

For the MySQL service, set an appropriate service type. Take some time to look at http://kubernetes.io/docs/user-guide/services/#publishing-services---service-types because services and their types are some of the most powerful and most important Kubernetes features.

For the MySQL deployment, create Kubernetes secrets. Take a look at http://kubernetes.io/docs/user-guide/secrets/#creating-a-secret-using-kubectl-create-secret for more info. The MySQL root password is "root_pw", the MySQL user is called "cddb_quintor" and the MySQL password is "quintor_pw". You can create secrets from the command line using:
kubectl create secret generic --from-literal=<field name>=<field value> <secret name>
so for example
kubectl create secret generic --from-literal=password=root_pw mysql-root-password

Choose a rollout strategy for the frontend deployment containers. Take a look at http://kubernetes.io/docs/user-guide/deployments/#strategy. It is not necessary to set the maxUnavailable and maxSurge fields but you can of course experiment with these values if you like.
For all three services (frontend, backend and MySQL), edit the *-service.yaml files and set the IP of your own node before creating them.
Now you can build the application from the ground up to the higher layers. Create the services and deployments for MySQL, backend and frontend:
 
```bash
$ kubectl create -f cddb-mysql-deployment.yaml
deployment "cddb-mysql" created
  
$ kubectl create -f cddb-mysql-service.yaml
service "cddb-mysql" created
 
$ kubectl create -f cddb-backend-deployment.yaml
deployment "cddb-backend" created
  
$ kubectl create -f cddb-backend-service.yaml
service "cddb-backend" created
 
$ kubectl create -f cddb-frontend-deployment.yaml
deployment "cddb-frontend" created
  
$ kubectl create -f cddb-backend-frontend.yaml
service "cddb-frontend" created
```
Test that the application is working using a browser and that it stores the data in the database. 
You can scale up the frontend and backend layer. But the mysql layer cannot be scaled. Though Kubernetes manages the persisted volumes and remounts them on a different node when needed. To test this find out on which node the mysql pod is running and kill the docker container on that node and look what happens.