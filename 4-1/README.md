
Demo:
https://www.youtube.com/watch?v=662FccIWeOE&feature=youtu.be


cd ~

git clone https://github.com/tnscorcoran/odh-fraud-detection-tutorial

cd odh-fraud-detection-tutorial/4-1

oc create -f 01-opendatahub_v1alpha1_opendatahub_crd.yaml

oc create -f 02-seldon-deployment-crd.json

oc new-project a-odh-4



On GUI
Install Open Data Hub Operator 	- choose project a-odh-4

Install Strimzi operator  		- choose project a-odh-4

On GUI
	when Strimzi operator there, deploy a Kafka 
		- overwrite yaml with 
		./03-kafka-cluster.yaml
		(in chat with him)

In 05-frauddetection_cr.yaml we did the following:

	# only:
		  seldon:
    		odh_deploy: true
    # don't enable Kafka here


Set proper cluster role and binding - ***** NOTE ASSUMES project a-odh-4 and user opentlc-mgr

oc apply -f 04-strimzi-role-binding.yaml



oc apply -f 05-frauddetection_cr.yaml

oc get pods -w

Wait till they're all ready  


Made the following changes to 07-rook-operator.yaml
	name: FLEXVOLUME_DIR_PATH 
	value: “/etc/kubernetes/kubelet-plugins/volume/exec”
	name: ROOK_HOSTPATH_REQUIRES_PRIVILEGED 
	value: “true” 


	***** ensure indentation is perfect ******


oc create -f 06-scc.yaml

oc create -f 07-rook-operator.yaml

oc get pods -n rook-ceph-system -w

oc create -f 08-cluster.yaml

oc get pods -n rook-ceph -w


oc create -f 09-toolbox.yaml

Made the following changes to 10-object.yaml

8080

oc create -f 10-object.yaml

oc get pods -n rook-ceph -w



oc create -f 11-object-user.yaml

Wait a few seconds

oc get secrets -n rook-ceph rook-ceph-object-user-my-store-my-user -o json

Interesting parts encoded

	"AccessKey": "<base 64 encoded AccessKey>",

	"SecretKey": "<base 64 encoded SecretKey>"

Decode them (e.g. at https://www.base64decode.org/) and record them - you'll need them below - 

	"AccessKey": "decoded AccessKey",

	"SecretKey": "decoded SecretKey"




Update ./12-s3-secretceph.yaml with correct key and secret (encoded)

vi ./12-s3-secretceph.yaml

oc project a-odh-4

oc create -n a-odh-4 -f ./12-s3-secretceph.yaml

On GUI, in the rook-ceph namespace, expose service rook-ceph-rgw-my-store - in my case it becomes:

http://rook-ceph-rgw-my-store-rook-ceph.apps.cluster-989f.989f.sandbox1049.opentlc.com


Install Fraud Detection Model
-----------------------------

oc create -n a-odh-4 -f 13-modelfull.json

oc get seldondeployments

oc get pods


oc create -n a-odh-4 -f 14-modelfull-route.yaml

Enable prometheus - on GUI, go to modelfull-modelfull service and add 2 annotations

  annotations:

    prometheus.io/path: /prometheus

    prometheus.io/scrape: 'true'

mv 15-creditcard.csv creditcard.csv

aws configure

	decoded AccessKey

	decoded SecretKey

	[None]

	[None]



aws s3 ls --endpoint-url http://rook-ceph-rgw-my-store-rook-ceph.apps.cluster-989f.989f.sandbox1049.opentlc.com

It's empty

for your S3 Bucket name, use uppercase (TOMBUCKET)


aws s3api create-bucket --bucket TOMBUCKET --endpoint-url http://rook-ceph-rgw-my-store-rook-ceph.apps.cluster-989f.989f.sandbox1049.opentlc.com

aws s3 cp creditcard.csv s3://TOMBUCKET/OPEN/uploaded/creditcard.csv --endpoint-url http://rook-ceph-rgw-my-store-rook-ceph.apps.cluster-989f.989f.sandbox1049.opentlc.com --acl public-read-write

Verify

aws s3 ls s3://TOMBUCKET/OPEN/uploaded/ --endpoint-url http://rook-ceph-rgw-my-store-rook-ceph.apps.cluster-989f.989f.sandbox1049.opentlc.com


Install Kafka Producer and Consumer
===================================

vi 16-ProducerDeployment.yaml

Change where it says [[insert s3endpoint]] to the **rook-ceph-rgw-my-store** route above - in my case **http://rook-ceph-rgw-my-store-rook-ceph.apps.cluster-989f.989f.sandbox1049.opentlc.com**
	( remember no trailing spaces in s3endpoint )

oc process -f 16-ProducerDeployment.yaml | oc apply -f -



Remember - deploy kafka is off here:  05-frauddetection_cr.yaml


vi 17-ConsumerDeployment.yaml

change where it says *[[insert seldon-core-seldon-apiserver route URL]]* seldon-core-seldon-apiserver route in your ODH project namesapce (in my case a-odh-4) 
In my case this URL is *http://seldon-core-seldon-apiserver-a-odh-4.apps.cluster-989f.989f.sandbox1049.opentlc.com*
( remember - no trailing spaces)

oc process -f 17-ConsumerDeployment.yaml | oc apply -f -


oc tag quay.io/odh-jupyterhub/jupyterhub-img:latest jupyterhub-img:latest


Grafana route - in my case https://grafana-a-odh-4.apps.cluster-989f.989f.sandbox1049.opentlc.com

Dashboards -> Import -> add each of these 4
- 18-grafanaKafka.json
- 18-grafanaModelPrediction.json
- 18-grafanaSeldonCore.json
- 18-grafanaSparkMetrics.json






Jupyterhub and Fraud Detection Notebook - some examples in my case:

Jupyterhub - Route:		Sign in With OpenShift	https://jupyterhub-a-odh-4.apps.cluster-989f.989f.sandbox1049.opentlc.com

AWS_ACCESS_KEY_ID		decoded AccessKey above

AWS_SECRET_ACCESS_KEY	decoded SecretKey


ENDPOINT_URL (rook-ceph-rgw-my-store - above)
http://rook-ceph-rgw-my-store-rook-ceph.apps.cluster-989f.989f.sandbox1049.opentlc.com
	
	
Import ./19-jupyterhub_frauddetection-notebook-template.ipynb to Jupyter


You need JQ - get it from here: https://github.com/stedolan/jq/releases/tag/jq-1.6

In my case it's at:				https://github.com/stedolan/jq/releases/download/jq-1.6/jq-linux32 

Import them to jupyter by choosing: New -> Terminal on the Jupyter homepage

Then 
wget https://github.com/stedolan/jq/releases/download/jq-1.6/jq-linux32 

mv jq-linux32 jq

Finally - hit the Prmoetheus Route in your browser and get past the self signed cert security error. This will allow your Grafana dashboards to load.

Now the demo - but ensure all placeholders for your URLs, projects, etc are updated to yours in your copy of :
./19-jupyterhub_frauddetection-notebook-template.ipynb

To that, make the following replacements:
- [[SELDON URL]] with your SELDON ROUTE URL
- [[OPENSHIFT MASTER API URL]] - with yours (not the URL used to access in a browser - rather with the CLI)
- [[ROOK CEPH URL - NO PROTOCOL]] - in the format **rook-ceph-rgw-my-store-rook-ceph.apps.......**
 

