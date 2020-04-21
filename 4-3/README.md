
ODH Fraud Detection Demo:
https://www.youtube.com/watch?v=662FccIWeOE&feature=youtu.be


git clone https://github.com/tnscorcoran/odh-fraud-detection-tutorial

cd odh-fraud-detection-tutorial/4-3

Execute:
```
oc create -f 01-opendatahub_v1alpha1_opendatahub_crd.yaml
oc create -f 02-seldon-deployment-crd.yaml
oc new-project 00odh
```


On GUI:
- Install Open Data Hub operator 	- choose project 00odh
- Install Strimzi operator  		- choose project 00odh
- When Strimzi operator there, deploy a Kafka 
	- overwrite yaml with ./03-kafka-cluster.yaml

Execute these and wait till all Pods are ready:
```
oc get pods -w
```

In *04-strimzi-role-binding.yaml*, I use project *00odh* and user *opentlc-mgr*. If yours are different, modify as appropriate.
Also, FYI as we don't enable Kafka here, in 05-frauddetection_cr.yaml (seldon section) we already set: 
```
odh_deploy: true
```
Execute the following and wait till Jupyterhub, Spark, Prometheus etc Pods are all ready.
```
oc apply -f 04-strimzi-role-binding.yaml
oc apply -f 05-frauddetection_cr.yaml
oc get pods -w
```
In the this next section, I changed 07-rook-operator.yaml, setting the following, ensuring indentation is perfect. Just an FYI - no need to do this - it's done already:
```
name: FLEXVOLUME_DIR_PATH 
value: “/etc/kubernetes/kubelet-plugins/volume/exec”
name: ROOK_HOSTPATH_REQUIRES_PRIVILEGED 
value: “true” 
```
Execute these and wait till all Pods are ready:
```
oc create -f 06-scc.yaml
oc create -f 07-rook-operator.yaml
oc get pods -n rook-ceph-system -w
```
Execute the following and wait till all Pods are ready:
```
oc create -f 08-cluster.yaml
oc get pods -n rook-ceph -w
```

In 10-object.yaml, I changed port to 8080. Execute the following and wait till all Pods are ready:
```
oc create -f 09-toolbox.yaml
oc create -f 10-object.yaml
oc get pods -n rook-ceph -w
```

Execute the following and wait a few seconds:
```
oc create -f 11-object-user.yaml
```

Execute the following:
```
oc get secrets -n rook-ceph rook-ceph-object-user-my-store-my-user -o json
```
Copy the 2 fields in the secret: *AccessKey* and *SecretKey*. These are Base64 encodes values. We'll refer to their keys and values as
- AccessKey: *[base 64 encoded AccessKey]*
- SecretKey: *[base 64 encoded SecretKey]*

Decode them (e.g. at https://www.base64decode.org/) and record them. We'll refer to their keys and values as:
- AccessKey: *[decoded AccessKey]*
- SecretKey: *[decoded SecretKey]*

Update your ./12-s3-secretceph.yaml with correct key and secret (encoded) using vi or your favourite editor, e.g.
```
vi ./12-s3-secretceph.yaml
```

Now execute the following:
```
oc project 00odh
oc create -n 00odh -f ./12-s3-secretceph.yaml
```

On GUI, in the *rook-ceph* namespace, create an OpenShift route from service *rook-ceph-rgw-my-store*. To that, execute the following:
```
oc project rook-ceph
oc expose svc rook-ceph-rgw-my-store
oc get Route rook-ceph-rgw-my-store
```
Copy this value - it will be something like 
- *rook-ceph-rgw-my-store-rook-ceph.apps.cluster-ocp4-3-2657.ocp4-3-2657.example.opentlc.com*. Prefix it with *http* to become something like *http://rook-ceph-rgw-my-store-rook-ceph.apps.cluster-ocp4-3-2657.ocp4-3-2657.example.opentlc.com*
- Well refer to that later on as *[CEPH-URL]*. 


Install Fraud Detection Model
-----------------------------
Execute the following and wait till all Pods are ready:
```
oc create -n 00odh -f 13-modelfull.json
oc project 00odh
oc get seldondeployments
oc create -n 00odh -f 14-modelfull-route.yaml
```

On GUI, enable Prometheus. To do that, go to service *modelfull-modelfull*, open its yaml and add 2 prometheus annotations (indented below line 4 in line with *getambassador.io/config: |*)
```
  annotations:
    prometheus.io/path: /prometheus
    prometheus.io/scrape: 'true'
```

Rename *15-creditcard.csv*:
```
mv 15-creditcard.csv creditcard.csv
```

aws configure

	decoded AccessKey

	decoded SecretKey

	[None]

	[None]



aws s3 ls --endpoint-url [[CEPH-URL]]

It's empty

for your S3 Bucket name, use uppercase (TOMBUCKET)


aws s3api create-bucket --bucket TOMBUCKET --endpoint-url [[CEPH-URL]]

aws s3 cp creditcard.csv s3://TOMBUCKET/OPEN/uploaded/creditcard.csv --endpoint-url [[CEPH-URL]] --acl public-read-write

Verify

aws s3 ls s3://TOMBUCKET/OPEN/uploaded/ --endpoint-url [[CEPH-URL]]


Install Kafka Producer and Consumer
===================================

vi 16-ProducerDeployment.yaml

If necessary, change the s3endpoint ( where it says **http://rook-ceph-rgw-my-store-rook-ceph.apps-crc.testing** or **<*insert s3endpoint*>** - remember no trailing spaces in s3endpoint )
(btw, I forked referenced - in case it's ever needed https://github.com/tnscorcoran/fradudetection-producer-consumer):

oc process -f 16-ProducerDeployment.yaml | oc apply -f -



Remember - deploy kafka is off here:  05-frauddetection_cr.yaml


vi 17-ConsumerDeployment.yaml

If necessary, change the *seldon* value ( where it says **http://rook-ceph-rgw-my-store-rook-ceph.apps-crc.testing** or **seldon
              value: "[[insert seldon-core-seldon-apiserver route URL]]"** - remember no trailing spaces in *seldon* )

oc process -f 17-ConsumerDeployment.yaml | oc apply -f -


oc tag quay.io/odh-jupyterhub/jupyterhub-img:latest jupyterhub-img:latest


Grafana route - in my case https://grafana-00odh.apps-crc.testing

Dashboards -> Import -> add each of these 4
- 18-grafanaKafka.json
- 18-grafanaModelPrediction.json
- 18-grafanaSeldonCore.json
- 18-grafanaSparkMetrics.json






Jupyterhub and Fraud Detection Notebook - some examples in my case:

Jupyterhub - Route:	Sign in With OpenShift	https://jupyterhub-00odh.apps-crc.testing/

AWS_ACCESS_KEY_ID		decoded AccessKey above

AWS_SECRET_ACCESS_KEY	decoded SecretKey


ENDPOINT_URL (rook-ceph-rgw-my-store - above)
[[CEPH-URL]]
	

Ensure all placeholders for your URLs, projects, etc are updated to yours in your copy of :
./19-jupyterhub_frauddetection-notebook-template.ipynb

To that, make the following replacements:
- [[SELDON URL]] with your SELDON ROUTE URL (e.g. http://seldon-core-seldon-apiserver-00odh.apps-crc.testing)
- [[OPENSHIFT MASTER API URL]] - with yours (e.g. https://api.crc.testing:6443)
- [[ROOK CEPH URL - NO PROTOCOL]] - (e.g. rook-ceph-rgw-my-store-rook-ceph.apps-crc.testing)
 
Import ./19-jupyterhub_frauddetection-notebook-template.ipynb to Jupyter


You need JQ - get it from here: https://github.com/stedolan/jq/releases/tag/jq-1.6

In my case it's at:				https://github.com/stedolan/jq/releases/download/jq-1.6/jq-linux32 

Import them to jupyter by choosing: New -> Terminal on the Jupyter homepage

Then 
wget https://github.com/stedolan/jq/releases/download/jq-1.6/jq-linux32 

mv jq-linux32 jq

Finally - hit the Prmoetheus Route in your browser and get past the self signed cert security error. This will allow your Grafana dashboards to load.

Now run the demo

