
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

	"AccessKey": "<decoded AccessKey>",

	"SecretKey": "decoded SecretKey"




Update ./12-s3-secretceph.yaml with correct key and secret (encoded)

vi ./12-s3-secretceph.yaml

oc project a-odh-4

oc create -n a-odh-4 -f ./12-s3-secretceph.yaml


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
	QInOwCEADuHhzOjAumZlrivTttRC3iy3lW6rjDqC
	jGb4HkmBRsPB3VKOVmPQWfFcWuUljjRd0yE0V67v
	[None]
	[None]


aws s3 ls --endpoint-url http://rook-ceph-rgw-my-store-rook-ceph.apps.cluster-6edf.6edf.sandbox106.opentlc.com
	# empty

S3 Bucket
USE UPPERCASE (TOMBUCKET)


aws s3api create-bucket --bucket TOMBUCKET --endpoint-url http://rook-ceph-rgw-my-store-rook-ceph.apps.cluster-6edf.6edf.sandbox106.opentlc.com

aws s3 cp creditcard.csv s3://TOMBUCKET/OPEN/uploaded/creditcard.csv --endpoint-url http://rook-ceph-rgw-my-store-rook-ceph.apps.cluster-6edf.6edf.sandbox106.opentlc.com --acl public-read-write

# verify
aws s3 ls s3://TOMBUCKET/OPEN/uploaded/ --endpoint-url http://rook-ceph-rgw-my-store-rook-ceph.apps.cluster-6edf.6edf.sandbox106.opentlc.com


Install Kafka Producer and Consumer
===================================

copied https://gitlab.com/opendatahub/fraud-detection-tutorial/-/raw/master/kafka/ProducerDeployment.yaml
to 
vi 16-ProducerDeployment.yaml

Change where it says <insert s3endpoint> 
	( remember no trailing spaces in s3endpoint )

oc process -f 16-ProducerDeployment.yaml | oc apply -f -



Remember - deploy kafka is off here
vi 5-frauddetection_cr.yaml


vi 17-ConsumerDeployment.yaml
	change seldon route (no trailing spaces)

oc process -f 17-ConsumerDeployment.yaml | oc apply -f -


oc tag quay.io/odh-jupyterhub/jupyterhub-img:latest jupyterhub-img:latest


	# Grafana route - https://grafana-a-odh-4.apps.cluster-6edf.6edf.sandbox106.opentlc.com/

		Dashboards -> Import -> add each of these 4






Jupyterhub and Fraud Detection Notebook
=======================================
Jupyterhub - Route:		https://jupyterhub-a-odh-4.apps.cluster-6edf.6edf.sandbox106.opentlc.com/
AWS_ACCESS_KEY_ID		QInOwCEADuHhzOjAumZlrivTttRC3iy3lW6rjDqC
AWS_SECRET_ACCESS_KEY	jGb4HkmBRsPB3VKOVmPQWfFcWuUljjRd0yE0V67v
ENDPOINT_URL			http://rook-ceph-rgw-my-store-rook-ceph.apps.cluster-ocp4-1-2578.ocp4-1-2578.open.redhat.com
						http://rook-ceph-rgw-my-store-rook-ceph.apps.cluster-6edf.6edf.sandbox106.opentlc.com
	
	
Downloaded
jupyterhub_frauddetection-notebook-template_TC_ORIG.ipynb
and
jupyterhub_jq
to./__my-files/

Import them to jupyter
