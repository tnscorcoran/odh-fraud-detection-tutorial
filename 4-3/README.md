
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

In the *rook-ceph* namespace, create an OpenShift route from service *rook-ceph-rgw-my-store*. To that, execute the following:
```
oc project rook-ceph
oc expose svc rook-ceph-rgw-my-store
oc get Route rook-ceph-rgw-my-store
```
Copy this value - it will be something like 
- *rook-ceph-rgw-my-store-rook-ceph.apps.cluster-ocp4-3-2657.ocp4-3-2657.example.opentlc.com*. Prefix it with *http* to become something like *http://rook-ceph-rgw-my-store-rook-ceph.apps.cluster-ocp4-3-2657.ocp4-3-2657.example.opentlc.com*
- We'll refer to that later on as *[CEPH-URL]*. 


Install Fraud Detection Model
-----------------------------
Execute the following and wait till all Pods are ready:
```
oc create -n 00odh -f 13-modelfull.json
oc project 00odh
oc get seldondeployments
oc create -n 00odh -f 14-modelfull-route.yaml
oc get pods -n 00odh -w
```

On GUI, enable Prometheus. To do that, go to service *modelfull-modelfull*, open its yaml and add 2 prometheus annotations (indented below line 4 above *getambassador.io/config*)
```
  annotations:
    prometheus.io/path: /prometheus
    prometheus.io/scrape: 'true'
```

Rename *15-creditcard.csv*:
```
mv 15-creditcard.csv creditcard.csv
```

In this section, you'll need the *aws* CLI. Download if necessary.
```
aws configure
- enter [decoded AccessKey] from above
- enter [decoded AccessKey] from above
- enter [None]
- enter [None]
```
Substitute your *[CEPH-URL]* from earlier and execute (it will be empty):
```
aws s3 ls --endpoint-url [CEPH-URL]
```
In executing the following, for your S3 Bucket name, _*use and uppercase value*_ (I used *TOMBUCKET*) as uppercase will be needed for it to work later - due to some anomoly in Jupyter. 
```
aws s3api create-bucket --bucket TOMBUCKET --endpoint-url [CEPH-URL]
aws s3 cp creditcard.csv s3://TOMBUCKET/OPEN/uploaded/creditcard.csv --endpoint-url [CEPH-URL] --acl public-read-write
```
Execute the same *ls* again to verify file is present:
```
aws s3 ls s3://TOMBUCKET/OPEN/uploaded/ --endpoint-url [CEPH-URL]
```

Install Kafka Producer and Consumer
-----------------------------------

Recall earlier in 05-frauddetection_cr.yaml we did not enable Kafka. We do it with the producer and consumer next. Using your favourite editor, replace *[CEPH-URL]* in this file with yours. I use *vi* : 
```
vi 16-ProducerDeployment.yaml
```
Now process it:
```
oc process -f 16-ProducerDeployment.yaml | oc apply -f -
```

On the GUI, with project *00odh* selected, copy the value of the OpenShift Route, *seldon-core-seldon-apiserver*. In my case it's *http://seldon-core-seldon-apiserver-00odh.apps.cluster-ocp4-3-2657.ocp4-3-2657.example.opentlc.com*. We'll refer to it as *[seldon-core-seldon-apiserver route URL]* (_without the trailing slash_)

Again, using your favourite editor, substitute the *[seldon-core-seldon-apiserver route URL]* placeholder with your actual URL (without trailing /). I use vi :
```
vi 17-ConsumerDeployment.yaml
```
Now process it:
```
oc process -f 17-ConsumerDeployment.yaml | oc apply -f -
```
Now you'll need to tag this Quay.io image:
```
oc tag quay.io/odh-jupyterhub/jupyterhub-img:latest jupyterhub-img:latest
```

On the GUI, with project 00odh selected, open your Prometheus OpenShift Route (just to get past any self-signed cert issues). 

Next, also on the GUI, open your Grafana route - in my case https://grafana-00odh.apps.cluster-ocp4-3-2657.ocp4-3-2657.example.opentlc.com

Navigate to: Dashboards -> Import -> add import each of these 4 - using the 4 JSON files in this repo.
- 18-grafanaKafka.json
- 18-grafanaModelPrediction.json
- 18-grafanaSeldonCore.json
- 18-grafanaSparkMetrics.json

On the GUI, with project 00odh selected, open your Jupyterhub OpenShift Route. Sign In with OpenShift. Click *Allow selected permissions*.

Under *Environment Variables*:
- for AWS_ACCESS_KEY_ID, enter your [decoded AccessKey] created above
- for AWS_SECRET_ACCESS_KEY, enter your [decoded SecretKey] created above
- under *ENDPOINT_URL*, enter the value you recorded above in placeholder *[CEPH-URL]*

Now we're going to create a number of new Environment Variables. We have to do them one at a time.  For the first one, enter the following:
- for Variable name: enter *ENDPOINT_URL*
- for Variable value: enter the value you recorded above in placeholder *[CEPH-URL]*

Next, ensure all placeholders for your URLs, projects, etc are updated to yours in your copy of:
./19-jupyterhub_frauddetection-notebook-template.ipynb

At a minimum, make the following replacements:
- [[SELDON URL]] with your SELDON ROUTE URL (e.g. http://seldon-core-seldon-apiserver-00odh.apps-crc.testing)
- [[OPENSHIFT MASTER API URL]] - with yours (e.g. https://api.crc.testing:6443)
- [[ROOK CEPH URL - NO PROTOCOL]] - (e.g. rook-ceph-rgw-my-store-rook-ceph.apps-crc.testing)
 
Import ./19-jupyterhub_frauddetection-notebook-template.ipynb to Jupyter

You need JQ - available from here: https://github.com/stedolan/jq/releases/tag/jq-1.6
Run the following:
```
wget https://github.com/stedolan/jq/releases/download/jq-1.6/jq-linux32 
mv jq-linux32 jq
```
Now open the file *./19-jupyterhub_frauddetection-notebook-template.ipynb* and run the demo
