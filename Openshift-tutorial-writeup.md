Following [Learn Kubernetes using the Developer Sandbox for Red Hat OpenShift](https://developers.redhat.com/developer-sandbox/activities/learn-kubernetes-using-red-hat-developer-sandbox-openshift)
## Installing OpenShift Local 2.31 and Kubernetes command line

- N.b.: decided to use OpenShift Local instead of the Developer Sandbox, due to not knowing how to obtain the login credentials to log in to the Sandbox project. Retrospectively, credentials were likely availabel from running `crc console --credentials`)
- Download package and pull secret from https://console.redhat.com/openshift/create/local
- Run the installer .pkg (n.b.: no "Openshift Local" appears in "Applications" folder, contrary to installer message)
-  install kubectl: `sudo port install kubectl-1.29` (&& `kubectl version --output=yaml`)

## Setting up OpenShift locally
- `crc setup`: this downloads 4.4 GiB of data, uncompresses 31 GiB) (n.b.: `crc` is the command line interface for OpenShift Local, and stands for "Code-Ready Containers", its former name)
- `crc start` => prompts for pull secret (from [previous download page](https://console.redhat.com/openshift/create/local))
- a few notes of relevance on OpenShift Local:
>- It uses a single node, which behaves as both a control plane and worker node.
>- It disables the Cluster Monitoring Operator by default. This disabled Operator causes the corresponding part of the web console to be non-functional.
>- The OpenShift Container Platform cluster runs in a virtual machine known as an _instance_. This might cause other differences, particularly with external networking.
- check the crc config with `crc config view` and status with `crc status`
- The cluster can be accessed and controlled via the OpenShift web console : https://console-openshift-console.apps-crc.testing


## Cluster set up
- `crc config set preset openshift`+`crc setup` : to change preset to "Openshift"
- `crc start` to start running OpenShift
- Using the 'oc' command line interface:
	- add the oc CLI to the PATH : `eval $(crc oc-env)`
	- login to the cluster : `oc login -u developer https://api.crc.testing:6443`
## Project set up
Following [Learn Kubernetes using the Developer Sandbox for Red Hat OpenShift](https://developers.redhat.com/developer-sandbox/activities/learn-kubernetes-using-red-hat-developer-sandbox-openshift) 
- new project:  (`oc status`) `oc new-project sampleproject`
- download the sample project files:
	- `git clone https://github.com/redhat-developer-demos/quotesweb.git`
	- `git clone https://github.com/redhat-developer-demos/quotemysql.git`
	- `git clone https://github.com/redhat-developer-demos/qotd-python.git`
### Set up the "quotes" back-end (a REST API serving quotes)
in `qotd-python/k8s` :
- create a Deployment for the desired pod: `kubectl create -f quotes-deployment.yaml
- expose the app as a Service to the rest of the cluster: `kubectl create -f service.yaml`
- create a Route to enable access to the app from outside the cluster: `kubectl create -f route.yaml`

- when this is done, 
	- `kubectl get pods` (or `oc get pods`) to check if the pod is running
	- `kubectl get routes`to find the URL, and finally:
	- `curl http://<the-URL-from-above>/quotes` (+ with `/random` at the end) to test the backend
### Set up the React frontend 
In `quotesweb/k8s`:
- create Deployment: `kubectl create -f quotesweb-deployment.yaml`
- createa  Service: `kubectl create -f quotesweb-service.yaml`
- create a Route: `kubectl create -f quotesweb-route.yaml`

Browse the frontend website: `kubectl get route` => http://quotesweb-kubectl-tutorial.apps-crc.testing. Paste the quotes URL from above to load the random quotes

- scale the backend by create replicas of the "quotes" Pod: `kubectl scale deployments/quotes --replicas=3` (check with `kubectl get pods`)
### Create a MariaDB backend and switch to it
Create a Persistent Volume Claim to store the data files for the MariaDB app even when the pods running MariaDB are deleted : `kubectl create -f mysqlvolume.yaml`

Create a secret to be used with the database: `kubectl create -f mysql-secret.yaml`

Export the name of the pod running MariaDB: `export PODNAME=$(a=$(kubectl get pods | grep 'mysql' | awk '{print $1}') && set – $a && echo $1)`

Create the database (copy the commands into the pod and execute the script)
- `kubectl cp ./create_database_quotesdb.sql $PODNAME:/tmp/create_database_quotesdb.sql`
- `kubectl cp ./create_database.sh $PODNAME:/tmp/create_database.sh`

and execute the script:
- `kubectl exec deploy/mysql -- /bin/bash ./tmp/create_database.sh`

Create the database tables:
Copy the commands into the pod:
- `kubectl cp ./create_table_quotes.sql $PODNAME:/tmp/create_table_quotes.sql`
- `kubectl cp ./create_tables.sh $PODNAME:/tmp/create_tables.sh`
- `kubectl exec deploy/mysql -- /bin/bash ./tmp/create_tables.sh`

Populate the database table:
- `kubectl cp ./populate_table_quotes_BASH.sql $PODNAME:/tmp/populate_table_quotes_BASH.sql`
- `kubectl cp ./quotes.csv $PODNAME:/tmp/quotes.csv`
- `kubectl cp ./populate_tables_BASH.sh $PODNAME:/tmp/populate_tables_BASH.sh`
- `kubectl exec deploy/mysql -- /bin/bash ./tmp/populate_tables_BASH.sh`

Querying the database:
- `kubectl cp ./query_table_quotes.sql $PODNAME:/tmp/query_table_quotes.sql`
- `kubectl cp ./query_table_quotes.sh $PODNAME:/tmp/query_table_quotes.sh`
- `kubectl exec deploy/mysql -- /bin/bash ./tmp/query_table_quotes.sh`

Change the backend to our MariaDB: 
- set environmental variable `kubectl set env deployment/quotes DB_SERVICE_NAME=mysql`
- change image of "quotes" app to point to our new backend `kubectl set image deploy quotes quotes=quay.io/donschenck/quotes:v2`

=> it works, now loading random quotes from the MariaDB
## Managing deployments
- `kubectl delete pod <pod-name>`: removes a pod, which is then normally restarted automatically
- `kubectl delete deployment <deployment-name>`: actually removes a pod and prevents it from being restarted
## Monitoring
In Red Hat Openshift Local, core functionality monitoring [requires 14 GiB of RAM](https://access.redhat.com/documentation/th-th/red_hat_openshift_local/2.5/html-single/getting_started_guide/index#starting-monitoring_gsg) (or 15 GB), exceeding our system capacity