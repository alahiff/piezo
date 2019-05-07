# Piezo deployment
Here we deploy a Kubernetes cluster on a single VM and deploy piezo on this cluster. Minio is also run on the Kubernetes cluster.

## Create a VM
Create a Ubuntu Bionic VM.

## Install Kubernetes
[MicroK8s](https://microk8s.io/) is a very quick and easy way install Kubernetes on a single VM for testing. Run:
```
snap install microk8s --classic
```
Once installation is complete and Kubernetes is running, install the DNS add-on:
```
microk8s.enable dns
```
and then install the dashboard add-on:
```
microk8s.enable dashboard
```
Run the following to create an alias for `kubectl`:
```
snap alias microk8s.kubectl kubectl
```

## Install a Minio server
Create a deployment containing a single instance of Minio:
```
kubectl create -f https://raw.githubusercontent.com/alahiff/piezo/master/minio-standalone-deployment.yaml
```
After a little while you should have a running Minio pod, e.g.
```
# kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
minio-577ddb5db8-lfjwg   1/1     Running   0          18m
```
Now run the following to create a service (i.e. a way of accessing Minio):
```
kubectl create -f https://raw.githubusercontent.com/alahiff/piezo/master/minio-standalone-service.yaml
```
You can run the following command to see Minio's IP address:
```
# kubectl get svc
NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
kubernetes      ClusterIP   10.152.183.1     <none>        443/TCP    179m
minio-service   ClusterIP   10.152.183.149   <none>        9000/TCP   18m
```

## Install the Minio client
Download the client:
```
wget https://dl.min.io/client/mc/release/linux-amd64/mc
```
Create an environment variable specifying how to access the Minio server:
```
export MC_HOST_piezo=http://minio:minio123@10.152.183.149:9000
```
Run:
```
./mc ls piezo
```
Create a bucket called `piezo`:
```
./mc mb piezo/piezo
```

## Install Helm
Based on https://webcloudpower.com/use-kubernetics-locally-with-microk8s

```
snap install --classic helm
helm init
```
After a little while check that it works:
```
helm ls
```
The "tiller-deploy" pod needs to be running for this to work. If necessary you can check this by running `kubectl get pods --all-namespaces`.

## Setup the priority class
```
kubectl create -f piezo-priority-class.yaml
```

## Install an ingress controller
```
helm install stable/nginx-ingress --set controller.service.nodePorts.http=31856 --set controller.priorityClassName=piezo-essential --set defaultBackend.priorityClassName=piezo-essential
```

## Install the Spark operator
```
helm repo add incubator http://storage.googleapis.com/kubernetes-charts-incubator
helm install incubator/sparkoperator --namespace spark-operator --set enableWebhook=true --set metrics-labels=app_type
kubectl apply -f spark-rbac.yaml
kubectl create clusterrolebinding spark-role --clusterrole=edit --serviceaccount=default:spark --namespace=default
```

## Test the piezo web app
```
# curl http://10.152.183.23:8888/piezo/
{"status": "success", "data": {"running": "true"}}
```




