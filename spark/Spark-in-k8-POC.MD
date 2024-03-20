# This is a step by step instruction to run spark pi example in k8 using spark-operator

## Prerequesites

You need to install and configure following software 

### Install docker
[Docker install instructions](https://docs.docker.com/desktop/install/mac-install/)

### Install kubectl
For our test we are running on a localhost so we are not connecting to external cluster but running on a local host container (minikube)
Follow [Minikube installation instructions](https://minikube.sigs.k8s.io/docs/start/)

### Install HELM
[HELM install instructions](https://helm.sh/docs/intro/install/)

### Download Spark
Current instruction is using spark-3.1.3 . We will use `spark-3.1.3-bin-hadoop3.2/examples/jars/spark-examples_2.12-3.1.3.jar`
```shell
cd ~

wget https://archive.apache.org/dist/spark/spark-3.1.3/spark-3.1.3-bin-hadoop3.2.tgz  
tar -xvzf spark-3.1.3-bin-hadoop3.2.tgz

vi ~/.zshrc
export SPARK_HOME=~/spark-3.1.3-bin-hadoop3.2
alias spark-shell="$SPARK_HOME/bin/spark-shell"

source ~/.zshrc
```
#### Install spark operator with helm chart
We need to add spark-operator repo to helm. Below is the most recent repo (it has recently moved)
```shell
helm repo add spark-operator https://kubeflow.github.io/spark-operator/
```
Install Spark operator. Pay attention to [Version Matrix](https://github.com/kubeflow/spark-operator/tree/v1beta2-1.3.8-3.1.1?tab=readme-ov-file#version-matrix) and install corresponding version
```shell

helm install my-release spark-operator/spark-operator --namespace spark-operator --create-namespace --set sparkJobNamespace=default --set webhook.enable=true --set image.tag=v1beta2-1.3.3-3.1.1
```
We have added `--set webhook.enable=true` to let spark modify pods, mount secrets to it, ConfigMap, etc.

#### Create Service Account Role
```shell
#create service account, role and rolebinding for spark
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: spark
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: spark-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["*"]
- apiGroups: [""]
  resources: ["services"]
  verbs: ["*"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: spark-role-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: spark
  namespace: default
roleRef:
  kind: Role
  name: spark-role
  apiGroup: rbac.authorization.k8s.io
EOF
```
* Check it's created: `kubectl get ns`
We should see `spark-operator` in active state. 
* Add spark image to minikube. 
```shell
minikube image load apache/spark:v3.1.3
```
* Clone example manifest repo
```shell
git clone git@github.com:kubeflow/spark-operator.git
```
* Update `spark-operator/examples/spark-pi.yaml` to match following template
```shell
apiVersion: "sparkoperator.k8s.io/v1beta2"
kind: SparkApplication
metadata:
  name: spark-pi
  namespace: default
spec:
  type: Scala
  mode: cluster
  image: "apache/spark:v3.1.3"
  imagePullPolicy: Always
  mainClass: org.apache.spark.examples.SparkPi
  mainApplicationFile: "local:///opt/spark/examples/jars/spark-examples_2.12-3.1.3.jar"
  sparkVersion: "3.1.3"
  restartPolicy:
    type: Never
  volumes:
    - name: "test-volume"
      hostPath:
        path: "/tmp"
        type: Directory
  driver:
    cores: 1
    coreLimit: "1200m"
    memory: "512m"
    labels:
      version: 3.1.3
    serviceAccount: spark
    volumeMounts:
      - name: "test-volume"
        mountPath: "/tmp"
  executor:
    cores: 1
    instances: 1
    memory: "512m"
    labels:
      version: 3.1.3
    volumeMounts:
      - name: "test-volume"
        mountPath: "/tmp"
```
* We would need to add clusterrole binding permissions
```shell
kubectl create clusterrolebinding spark-role --clusterrole=edit --serviceaccount=default:spark --namespace=default
```
* To run the app we must apply our manifest file 
```shell
kubectl apply -f spark-operator/examples/spark-pi.yaml
```
* Check pod is creating (should have status `ContainerCreating`)
```shell
kubectl get pods
```
* Check the logs
```shell
kubectl logs spark-pi-driver | grep 3.1
```
* we may get list of all spark applications
```shell
kubectl get sparkapplications.sparkoperator.k8s.io
```
* We may check cluster info
```shell
kubectl describe sparkapplications.sparkoperator.k8s.io spark-pi
```
