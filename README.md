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

## Deploy the piezo web app


## Using the piezo web app
In the following examples the IP address of the piezo server should of course be changed as appropriate.

### Basic test that it's alive
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

