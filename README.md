# Tom's mods to https://gitlab.com/opendatahub/fraud-detection-tutorial/-/blob/master/README.md


# Open Data Hub Fraud Detection Use Case Tutorial
Please follow the instructions below to install and run Open Data Hub Fraud Detection usecase. For more details on the use case and
a high level architecture please visit [here](usecase/README.md)


## Requirements
* Openshift 3.11 or 4.x
* Cluster Admin. Permissions

## Install Open Data Hub with Kafka and Seldon Metrics Enabled

````
git clone [https://gitlab.com/opendatahub/opendatahub-operator](https://gitlab.com/opendatahub/opendatahub-operator)
cd opendatahub-operator
````
Deploy the ODH and Seldon custom resource based on the sample template. This step requires cluster-admin permissions. 
````
oc create -f deploy/crds/opendatahub_v1alpha1_opendatahub_crd.yaml
oc create -f  deploy/crds/seldon-deployment-crd.json
````
Create a new project to deploy Open Data Hub into. This project name will be used later in many installation commands
````
oc new-project <projectname>
````
Create the services and RBAC policy for the service account the operator will run as. This step at a minimum requires namespace admin rights.
````
oc create -f deploy/service_account.yaml
oc create -f deploy/role.yaml
oc create -f deploy/role_binding.yaml
oc adm policy add-role-to-user admin -z opendatahub-operator
````
Deploy the operator image into your project
````
oc create -f deploy/operator.yaml
````
Check the operator pod. Do not proceed until the operator pod is ready
````
oc get pods
````
We need to edit the ODH custom resource yaml file to enable Seldon and Kafka. We also provide a modified [frauddetection_cr.yaml](opendatahub/frauddetection_cr.yaml) for convenience
```` 
cp deploy/crds/opendatahub_v1alpha1_opendatahub_cr.yaml frauddetection_cr.yaml
vi frauddetection_cr.yaml
# Seldon Deployment
seldon:
odh_deploy: true

kafka:
odh_deploy: true
kafka_cluster_name: odh-message-bus
kafka_broker_replicas: 3
kafka_zookeeper_replicas: 3
````

Kafka installation requires special setup, the following steps are to configure Kafka. Add your username to the kafka_admins list
````
vi deploy/kafka/vars/vars.yaml
kafka_admins:
- admin
- system:serviceaccount:{{ NAMESPACE }}:opendatahub-operator
- <INSERT USERNAME>
````

Deploy Kafka Operator
````
cd deploy/kafka/ 
pipenv install
pipenv run ansible-playbook deploy_kafka_operator.yaml -e kubeconfig=$HOME/.kube/config -e NAMESPACE=<namespace>
````
Check that the strimzi Kafka operator is ready before proceeding
````
oc get pods
opendatahub-operator-86c5cb8b4b-g9249       1/1     Running   0          6m44s
strimzi-cluster-operator-57864db849-224b7   0/1     Running   0          17s
````
Deploy the ODH custom resource based on the sample template
````
oc create -f frauddetection_cr.yaml
````
To check successful installation output of command below should be similar. This step takes a couple of minutes so be patient
````
oc get pods
NAME READY STATUS RESTARTS AGE
grafana-68b7d6bd68-swwvk                              2/2     Running     0          4m32s
jupyterhub-1-deploy                                   0/1     Completed   0          5m12s
jupyterhub-1-tzjwh                                    1/1     Running     0          5m4s
jupyterhub-db-1-6cptv                                 1/1     Running     0          5m5s
jupyterhub-db-1-deploy                                0/1     Completed   0          5m14s
opendatahub-operator-86c5cb8b4b-k57dh                 1/1     Running     0          9m43s
prometheus-0                                          4/4     Running     0          5m1s
seldon-core-redis-master-0                            1/1     Running     0          3m1s
seldon-core-seldon-apiserver-5d6b5d9665-n2klz         1/1     Running     0          3m7s
seldon-core-seldon-cluster-manager-5dcfdfb667-96mxf   1/1     Running     0          3m5s
spark-operator-6884cbc6f6-ncxks                       1/1     Running     0          5m4s
````

## Install Rook-Ceph
This installation of Rook-Ceph assumes your OCP 3.11 or 4.x cluster has at least 3 worker nodes.
Download the files for Rook Ceph v0.9.3 and  modify the source Rook Ceph files directly, clone the Rook operator and checkout the v0.9.3 branch. For convenience we 
also included the [modified files](rook-ceph)
````
git clone https://github.com/rook/rook.git
cd rook
git checkout -b rook-0.9.3 v0.9.3
cd cluster/examples/kubernetes/ceph/
````
Edit operator.yaml and set the environment variables for FLEXVOLUME_DIR_PATH and ROOK_HOSTPATH_REQUIRES_PRIVILEGED to allow the Rook operator to use OpenShift hostpath storage.
````
name: FLEXVOLUME_DIR_PATH 
value: “/etc/kubernetes/kubelet-plugins/volume/exec”
name: ROOK_HOSTPATH_REQUIRES_PRIVILEGED 
value: “true” 
````
The following steps require cluster wide permissions. Configure the necessary security contexts, and deploy the rook operator, this will create a new namespace, rook-ceph-system, and deploy the pods in it.
````
oc create -f scc.yaml
oc create -f operator.yaml
oc get pods -n rook-ceph-system
NAME READY STATUS RESTARTS AGE
rook-ceph-agent-j4zms 1/1 Running 0 33m
rook-ceph-agent-qghgc 1/1 Running 0 33m
rook-ceph-agent-tjzv6 1/1 Running 0 33m
rook-ceph-operator-567f8cbb6-f5rsj 1/1 Running 0 33m
rook-discover-gghsw 1/1 Running 0 33m
rook-discover-jd226 1/1 Running 0 33m
rook-discover-lgfrx 1/1 Running 0 33m
````
Once the operator is ready, you can create a Ceph cluster, and Ceph object service. The toolbox service is also handy to deploy for checking the health of the Ceph cluster. This step takes a couple of minutes, please be patient
````
oc create -f cluster.yaml
````
Check the pods and wait for this pods to finish before proceeding
````
oc get pods -n rook-ceph
rook-ceph-mgr-a-66db78887f-5pt7l              1/1     Running     0          108s
rook-ceph-mon-a-69c8b55966-mtb47              1/1     Running     0          3m19s
rook-ceph-mon-b-59699948-4zszh                1/1     Running     0          2m44s
rook-ceph-mon-c-58f4744f76-r8prn              1/1     Running     0          2m11s
rook-ceph-osd-0-764bbd9694-nxjpz              1/1     Running     0          75s
rook-ceph-osd-1-85c8df76d7-5bdr7              1/1     Running     0          74s
rook-ceph-osd-2-8564b87d6c-lcjx2              1/1     Running     0          74s
rook-ceph-osd-prepare-ip-10-0-136-154-mzf66   0/2     Completed   0          87s
rook-ceph-osd-prepare-ip-10-0-153-32-prf94    0/2     Completed   0          87s
rook-ceph-osd-prepare-ip-10-0-175-183-xt4jm   0/2     Completed   0          87s
````
Edit the object.yaml file and replace port 80 with 8080
````
vi object.yaml
-----------
gateway:
# type of the gateway (s3)
type: s3
# A reference to the secret in the rook namespace where the ssl certificate is stored
sslCertificateRef:
# The port that RGW pods will listen on (http)
port: 8080
----------
oc create -f toolbox.yaml
oc create -f object.yaml
oc get pods -n rook-ceph
rook-ceph-mgr-a-5b6fcf7c6-cx676 1/1 Running 0 6m56s
rook-ceph-mon-a-54d9bc6c97-kvfv6 1/1 Running 0 8m38s
rook-ceph-mon-b-74699bf79f-2xlzz 1/1 Running 0 8m22s
rook-ceph-mon-c-5c54856487-769fx 1/1 Running 0 7m47s
rook-ceph-osd-0-7f4c45fbcd-7g8hr 1/1 Running 0 6m16s
rook-ceph-osd-1-55855bf495-dlfpf 1/1 Running 0 6m15s
rook-ceph-osd-2-776c77657c-sgf5n 1/1 Running 0 6m12s
rook-ceph-osd-3-97548cc45-4xm4q 1/1 Running 0 5m58s
rook-ceph-osd-prepare-ip-10-0-138-84-gc26q 0/2 Completed 0 6m29s
rook-ceph-osd-prepare-ip-10-0-141-184-9bmdt 0/2 Completed 0 6m29s
rook-ceph-osd-prepare-ip-10-0-149-16-nh4tm 0/2 Completed 0 6m29s
rook-ceph-osd-prepare-ip-10-0-173-174-mzzhq 0/2 Completed 0 6m28s
rook-ceph-rgw-my-store-d6946dcf-q8k69 1/1 Running 0 5m33s
rook-ceph-tools-cb5655595-4g4b2 1/1 Running 0 8m46s
````
Next, you will need to create a set of S3 credentials, the resulting credentials will be stored in a secret file under the rook-ceph namespace. There isn’t currently a way to cross-share secrets between OpenShift namespaces, so you will need to copy the secret to the namespace running Open Data Hub operator.
````
oc create -f object-user.yaml
oc get secrets -n rook-ceph rook-ceph-object-user-my-store-my-user -o json
````
Create a secret in your deployment namespace that includes the secret and key for S3 interface. Make sure to copy the accesskey and secretkey from the command output above and download the [secret yaml file](opendatahub/s3-secretceph.yaml)
````
oc create -n <project name> -f s3-secretceph.yaml
````
From the Openshift console, create a route to the rook service, rook-ceph-rgw-my-store, in the rook-ceph namespace to expose the endpoint. This endpoint url will be used to access the S3 interface from the example notebooks.

## Install Fraud Detection Model
Deploy fraud detection fully trained model by downloading [modelfull.json](model/modelfull.json)
````
oc create -n <project name> -f modelfull.json
````
Check and make sure the model is created, this step will take a couple of minutes.
````
oc get seldondeployments
oc get pods
````
Create a route to the model by downloading [modelfull-route.yaml](model/modelfull-route.yaml)
````
oc create -n <project name> -f modelfull-route.yaml
````
Enable Prometheus metric scraping by editing modelfull-modelfull service from the portal and add these two lines under annotations
````
apiVersion: v1
kind: Service
metadata:
annotations:
prometheus.io/path: /prometheus
prometheus.io/scrape: 'true'

````
## Upload the Credit Card Transaction Data to Rook-Ceph

Make sure to decode the key and secret copied from the rook installation by using the following commands
````
base64 -d
Paste secret
Ctrl-D
````
Download the credit card transaction: [creditcard.csv](data) file. From a command line use the aws tool to upload the file to rook-ceph data store
````
aws configure
````
Only enter key and secret, leave all other fields as default. Check if connection is working.
````
aws s3 ls --endpoint-url <Insert URL to Rook-Ceph Service>
````
Create a bucket and upload the file.
````
aws s3api create-bucket --bucket <Insert Bucket Name> --endpoint-url <Insert URL to Rook-Ceph Service>

aws s3 cp creditcard.csv s3://<Insert Bucket Name>/OPEN/uploaded/creditcard.csv --endpoint-url <Insert URL to Rook-Ceph Service> --acl public-read-write
````

Verify File is uploaded
````
aws s3 ls s3://<Insert Bucket Name>/OPEN/uploaded/ --endpoint-url <Insert URL to Rook-Ceph Service>
````

## Install Kafka Producer and Consumer
Kafka Producer needs specific parameters to read from S3 interface and call the model REST prediction endpoint. Download [ProducerDeployment.yaml](kafka/ProducerDeployment.yaml). Edit the file to specify namespace and  your rook-ceph url, your bucket name (this need to point to the location of the creditcard.csv file in the rook-ceph data store)
````
vi ProducerDeployment.yaml

- name: NAMESPACE
description: The OpenShift project in use
value: <Project name>


- name: s3endpoint
value: "<add_s3_endpoint_here>:443"
- name: s3bucket
value: "<add_bucket_name>"
- name: filename
value: "OPEN/uploaded/creditcard.csv"
- name: bootstrap
````

Create the producer pod
````
oc process -f ProducerDeployment.yaml | oc apply -f -
````

Edit and deploy the consumer by downloading [ConsumerDeployment.yaml](kafka/ConsumerDeployment.yaml). Edit the file to make changes to these fields, specify the project name and the route to seldon-core-seldon-apiserver
````
vi ConsumerDeployment.yaml

- name: NAMESPACE
description: The OpenShift project in use
value: <Insert Project name>


- name: seldon
value: "<add_seldon_endpoint_here>"
````

Create the consumer pod
````
oc process -f ConsumerDeployment.yaml | oc apply -f -
````

## Upload Grafana Dashboards

From the Openshift portal click on the Prometheus route and explore some of the metrics. To launch Grafana dashboard click on the Grafana route. Download the [Grafana Boards](grafana) and upload them to the dashboard. The following is a list of the boards

* Kafka
* Seldon Model
* Seldon Core
* Spark Metrics

## Jupyterhub and Fraud Detection Notebook

To Launch Jupyterhub go to the route from Openshift portal and click on it. Sign in with your  openshift credentials
Keep everything default but enter AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY from the last step in the rook-ceph installation
Hit the launch button
This step takes a while, until you see jupyterhub folder ready
Download the [notebook](jupyterhub/frauddetection-notebook-template.ipynb) and [jq](jupyterhub/jq) tool then upload them to the Jupyterhub folder. Click on the notebook to launch it. From there you will need to enter some specific information regarding your environment to run some cells such as:
* Check that you entered AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY correctly
* URL to your seldon core API server
* URL to log into your Openshift Cluster
* Check that your bucket name and file key are consistent
* Follow the [mymodel instructions](model/mymodel) to customize and build mymodel to work with your S3 data store and bucket

## TroubleShooting

* Kafka pods are not available: Restart the Open Data Hub operator pod
* Spark Session from the notebook cannot connect to S3: Make sure your bucket name is all CAPS, known issue for hadoop-aws:2.7.3
* Spark Cluster is not launching: Delete any limitranges specified for your namespace/project

## Helpful Links

* Open Data Hub [site](https://opendatahub.io/) and [gitlab](https://gitlab.com/opendatahub/opendatahub-operator)
* Fraud Detection Demo Video in [Openshift Commons](https://youtu.be/662FccIWeOE) 



















