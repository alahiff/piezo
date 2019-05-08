# Piezo deployment
Here we deploy a Kubernetes cluster on a single VM and deploy piezo on this cluster. Minio is also run on the Kubernetes cluster.

## Create a VM
Create a Ubuntu Bionic VM with at least 4 cores and 4 GB memory.

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
Now standard `kubectl` commands can be run, e.g. to see a list of all running pods in all namespaces:
```
kubectl get pods --all-namespaces
```

## Install a Minio server
Here we deploy a simple Minio server on Kubernetes. This should not be used in production! See https://docs.min.io/docs/deploy-minio-on-kubernetes.html for information on how to properly run Minio on Kubernetes.

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
Now run the following to create a service (i.e. a way of accessing Minio with a stable IP):
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
In this case it will be possible to access Minio from within the Kubernetes cluster and from the host using the IP address `10.152.183.149`. In order to access Minio externally an ingress controller is required, but for the moment this is not necessary.

## Create a secret containing the Minio credentials
A secret containing the access key and secret key is required by piezo so that jobs can read input files and write output to Minio using S3.

Copy the following into a file `minio-secret.yaml`:
```
apiVersion: v1
kind: Secret
metadata:
 name: local-minio
type: Opaque
data:
 accessKey: bWluaW8=
 secretKey: bWluaW8xMjM=
```
and then run:
```
kubectl apply -f minio-secret.yaml
```
The access key and secret key specified in `minio-secret.yaml` are base64 encoded. To create them run e.g.:
```
echo -n "<access_key>" | base64
```
The example `minio-secret.yaml` above contains the access key and secret key specified above when we deployed the test Minio server.

## Install the Minio client
The client can be used to check that the Minio server is running and also for uploading any required files necessary for testing piezo.

Download the client:
```
wget https://dl.min.io/client/mc/release/linux-amd64/mc
```
Create an environment variable specifying how to access the Minio server:
```
export MC_HOST_piezo=http://minio:minio123@10.152.183.149:9000
```
Here we have also specified an alias `piezo`.

Run:
```
./mc ls piezo
```
The first time this command is run some configuration files will be generated.
Now create a bucket called `piezo`:
```
./mc mb piezo/piezo
```

## Install Helm
Based on https://webcloudpower.com/use-kubernetics-locally-with-microk8s. We use snap to install Helm by running a single command:
```
snap install --classic helm
helm init
```
After a little while check that it works:
```
helm ls
```
The "tiller-deploy" pod needs to be running for this to work. If necessary you can check this by running:
```
kubectl get pods --all-namespaces
```
Initially the pod will be in the `creating` state but eventually should end up in the `running` state.

## Build the container images
The Docker images of course should be built on a machine with Docker installed. Firstly, clone the git repository:
```
git clone https://github.com/ukaea/piezo.git
```
Alternatively, the following images can be used:
* Spark: `alahiff/spark-piezo:latest`
* Piezo web app: `alahiff/piezo-web-app:latest`

### Piezo web app
Run the following to build and push the image to Docker Hub, replacing at least the username as appropriate:
```
cd piezo
docker build -t alahiff/piezo-web-app:latest .
docker push alahiff/piezo-web-app:latest
```

### Spark
Run the following to build and push the image to Docker Hub, replacing at least the username as appropriate:
```
cd Spark
docker build -t alahiff/spark-piezo:latest .
docker push alahiff/spark-piezo:latest
```

## Setup the priority class
A priority class is used so that under high resource pressures the core piezo components are the last to go down, i.e. less important pods will be killed first.
```
kubectl create -f piezo-priority-class.yaml
```

## Install an ingress controller
The ingress controller allows for external access to pods in the Kubernetes cluster.
```
helm install stable/nginx-ingress --set controller.service.nodePorts.http=31856 --set controller.priorityClassName=piezo-essential --set defaultBackend.priorityClassName=piezo-essential
```

## Install the Spark operator
Run the following to install the Spark operator (the Spark operator is a long-running pod which creates Spark driver pods and executors as necessary):
```
helm repo add incubator http://storage.googleapis.com/kubernetes-charts-incubator
helm install incubator/sparkoperator --namespace spark-operator --set enableWebhook=true --set metrics-labels=app_type
kubectl apply -f spark-rbac.yaml
kubectl create clusterrolebinding spark-role --clusterrole=edit --serviceaccount=default:spark --namespace=default
```

You can check that the Spark operator pod is running:
```
kubectl get pods --namespace=spark-operator
```
which should give, for example:
```
NAME                                           READY   STATUS    RESTARTS   AGE
mean-squirrel-sparkoperator-77bd58cf5d-q72ff   1/1     Running   0          24h
```
To look at the logs from the Spark operator, run for example:
```
kubectl logs -f --namespace=spark-operator mean-squirrel-sparkoperator-77bd58cf5d-q72ff
```
where the pod name should be replaced as necessary.

## Deploy the piezo web app
### Configuration
An example configuration file is provided in `piezo/piezo_web_app/PiezoWebApp`. An example is also here:  https://raw.githubusercontent.com/alahiff/piezo/master/configuration.ini. Create a copy of this file called `configuration.ini` and then run the following to create a ConfigMap:
```
kubectl create configmap piezo-web-app-config --from-file=configuration.ini
```

### Validation rules
An example validation rules JSON file is also provided in `piezo/piezo_web_app/PiezoWebApp` and an example is also here: https://raw.githubusercontent.com/alahiff/piezo/master/validation_rules.json. Create a copy of this file called `validation_rules.json` and make any required modifications. At the very least the container image name will need to be changed, then run:
```
kubectl create configmap validation-rules --from-file=validation_rules.json
```

### Deployment
Download a YAML file containing a deployment and service for the piezo web app:
```
wget https://github.com/alahiff/piezo/blob/master/piezo-deploy-svc.yaml
```
Make any changes as necessary, e.g. for the web app container image name. Then run:
```
kubectl -f piezo-deploy-svc.yaml
```

Now check that the container is running. After a while `kubectl get pods` should look something like:
```
NAME                                                              READY   STATUS    RESTARTS   AGE
minio-577ddb5db8-lfjwg                                            1/1     Running   0          5d19h
piezo-web-app-5fd865c769-x2fsv                                    1/1     Running   0          20h
...
```

### Final steps
Now download the following YAML file:
```
wget https://raw.githubusercontent.com/alahiff/piezo/master/piezo-extras.yaml
```
and replace all instances of `piezo-test01` with the hostname of your VM. Then run:
```
kubectl create -f piezo-extras.yaml
```
Note there is an error at the end which I haven't looked into it!

At this point you will see that there are a number of pods running:
```
root@piezo-test01:~# kubectl get pods --all-namespaces
NAMESPACE        NAME                                                              READY   STATUS    RESTARTS   AGE
default          minio-577ddb5db8-lfjwg                                            1/1     Running   0          5d19h
default          piezo-web-app-5fd865c769-x2fsv                                    1/1     Running   0          21h
default          pugnacious-termite-nginx-ingress-controller-798869f457-llqv5      1/1     Running   0          21h
default          pugnacious-termite-nginx-ingress-default-backend-8c67bf95dc6cl9   1/1     Running   0          21h
kube-system      heapster-v1.5.2-6b5d7b57f9-8kv7g                                  4/4     Running   0          5d21h
kube-system      kube-dns-6bfbdd666c-2wt8f                                         3/3     Running   0          5d21h
kube-system      kubernetes-dashboard-6fd7f9c494-5kpjx                             1/1     Running   0          5d21h
kube-system      monitoring-influxdb-grafana-v4-78777c64c8-c855g                   2/2     Running   0          5d21h
kube-system      tiller-deploy-c48485567-j7lwn                                     1/1     Running   0          24h
spark-operator   mean-squirrel-sparkoperator-77bd58cf5d-q72ff                      1/1     Running   0          24h
```

## Using the piezo web app
In the following examples the IP address of the piezo server should of course be changed as appropriate. To find out what IP address to use, run:
```
kubectl get svc piezo-app-service
```
which will give something like:
```
NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
piezo-app-service   ClusterIP   10.152.183.23   <none>        8888/TCP   20h
```

### Basic test that it's alive
First we just run the most basic check to see if the piezo REST API is alive:
```
# curl http://10.152.183.23:8888/piezo/
{"status": "success", "data": {"running": "true"}}
```

### Submit a job
Download the test script which calculates pi:
```
wget https://raw.githubusercontent.com/ukaea/piezo/2544c9176b5d93b6c0af4103d97674c30a41ab6b/SystemTests/roles/example_scripts/files/pi.py
```
and copy it to Minio:
```
./mc cp pi.py piezo/piezo/inputs/
```
Download the piezo JSON:
```
wget https://raw.githubusercontent.com/alahiff/piezo/master/demo-pi.json
```
and submit the job to piezo:
```
curl -X POST -H "Content-Type: application/json" -d@demo-pi.json http://10.152.183.23:8888/piezo/submitjob
```

### List jobs
```
curl -X GET -H "Content-Type: application/json" -d '{}' http://10.152.183.23:8888/piezo/getjobs
```
Example output:
```
{
   "status":"success",
   "data":{
      "message":"Found 1 spark jobs",
      "spark_jobs":{
         "python-pi-6c632":"SUBMITTED"
      }
   }
}
```

### Get the logs for a specific job
```
curl -X GET -H "Content-Type: application/json" -d '{"job_name":"python-pi-3b548"}' http://10.152.183.23:8888/piezo/getlogs
```

### Viewing log files for complete jobs
Log files are copied to Minio and can be seen by running:
```
./mc ls piezo/piezo/outputs
```
Example output:
```
[2019-05-08 11:13:29 UTC]      0B python-pi-29aff/
[2019-05-08 11:13:29 UTC]      0B python-pi-3b548/
[2019-05-08 11:13:29 UTC]      0B python-pi-6879b/
[2019-05-08 11:13:29 UTC]      0B python-pi-7394c/
[2019-05-08 11:13:29 UTC]      0B python-pi-ee602/
[2019-05-08 11:13:29 UTC]      0B python-pi-f2c80/
```
The output from each job is stored in a unique directory.

