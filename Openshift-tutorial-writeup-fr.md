Apprentissage d'Openshift et Kubernetes avec installation depuis la ligne de commande, en suivant le tutoriel [Learn Kubernetes using the Developer Sandbox for Red Hat OpenShift](https://developers.redhat.com/developer-sandbox/activities/learn-kubernetes-using-red-hat-developer-sandbox-openshift)
## Installer OpenShift Local 2.31 et la ligne de commande Kubernetes
- N.b.: en raison d'une difficulté à trouver les identifiants pour se connecter au cluster du Developer Sandbox tournant sur l'infrastructure Red Hat (et utilisé dans le tutoriel), décision d'utiliser OpenShift Local, qui permet de faire tourner un cluster localement, sur sa propre machine. Rétrospectivement, les identifiants étaient probablement obtenables avec la commande `crc console --credentials`.
- Télécharger le paquet d'installation et et le "pull secret" depuis https://console.redhat.com/openshift/create/local (n.b.: un compte Red Hat est nécessaire; le "pull secret", qui permet de pull des images de Red Hat, est demandé lors de l'installation)
-  Installer la ligne de commande de Kubernetes (n.b.: certes pas vraiment nécessaire au final, les commandes étant les mêmes qu'avec `oc`, mais il s'agissait initialement de suivre un tutoriel à la lettre): `sudo port install kubectl-1.29` (&& `kubectl version --output=yaml`)

## Mise en place d'OpenShift Local
- `crc config set preset openshift` pour changer le preset à "Openshift"
- `crc setup`: télécharge 4.4 GiB de données, 31 GiB à la décompression data (n.b.: `crc` est l'interface de ligne de commande pour OpenShift Local, signifiant "Code-Ready Containers", son ancien nom)
- `crc start` pour lancer le cluster => demande le pull secret (disponible [à la page de téléchargement précédemment mentionnée](https://console.redhat.com/openshift/create/local))
- quelques notes pertinentes sur les spécificités d'OpenShift Local ([Guide officiel](https://access.redhat.com/documentation/en-us/red_hat_openshift_local/2.16/html-single/getting_started_guide/index#differences-from-production-openshift-install_gsg)):
>- It uses a single node, which behaves as both a control plane and worker node.
>- It disables the Cluster Monitoring Operator by default. This disabled Operator causes the corresponding part of the web console to be non-functional.
>- The OpenShift Container Platform cluster runs in a virtual machine known as an _instance_. This might cause other differences, particularly with external networking.
- vérifier la configuration de crc avec `crc config view` et son statut avec `crc status`
- Le cluster peut être visionné et contrôlé par une interface graphique via la console web d'OpenShift à l'adresse: https://console-openshift-console.apps-crc.testing
- Utiliser l'interface de ligne de commande d'OpenShift 'oc':
	- l'ajouter au PATH : `eval $(crc oc-env)`
	- se login au cluster : `oc login -u developer https://api.crc.testing:6443`

## Mise en place de l'application
(Suivant [Learn Kubernetes using the Developer Sandbox for Red Hat OpenShift](https://developers.redhat.com/developer-sandbox/activities/learn-kubernetes-using-red-hat-developer-sandbox-openshift))
- créer un nouveau projet: (`oc status` && `oc projects` pour jeter un coup d'oeil sur le contexte) `oc new-project sampleproject`
- télécharger les fichiers nécessaires à l'application:
	- `git clone https://github.com/redhat-developer-demos/quotesweb.git`
	- `git clone https://github.com/redhat-developer-demos/quotemysql.git`
	- `git clone https://github.com/redhat-developer-demos/qotd-python.git`
### Le backend "quotes" (une API REST livrant des citations)
dans `qotd-python/k8s` :
- créer un Deployment pour le Pod désiré : `kubectl create -f quotes-deployment.yaml`
- exposer le backend comme un Service au reste du cluster: `kubectl create -f service.yaml`
- créer une Route pour permettre l'accès au backend de l'extérieur du cluster: `kubectl create -f route.yaml`

- une fois terminé, 
	- `kubectl get pods` (ou `oc get pods`) pour vérifier si le pod tourne
	- `kubectl get routes`pour trouver l'URL, et enfin:
	- `curl http://<the-URL-from-above>/quotes` (+ avec `/random` à la fin) pour tester le backend
### Le frontend React 
Dans `quotesweb/k8s`:
- créer un Deployment: `kubectl create -f quotesweb-deployment.yaml`
- créer un Service: `kubectl create -f quotesweb-service.yaml`
- créer une Route: `kubectl create -f quotesweb-route.yaml`

Visiter le frontend: `kubectl get route` => http://quotesweb-kubectl-tutorial.apps-crc.testing. Coller l'URL du backend (voir ci-dessus) pour charger les citations choisies au hasard

- redéployer le backend "quotes" avec des répliques du Pod: `kubectl scale deployments/quotes --replicas=3` (vérifier avec `kubectl get pods`)
### Créer un backend MariaDB et l'utiliser
Dans `quotemysql/`:
- Créer un "Persistent Volume Claim" pour stocker les données pour la base de données même quand les Pods qui la font tourner sont supprimés: `kubectl create -f mysqlvolume.yaml`

- Créer un secret à être utilisé avec la base de données: `kubectl create -f mysql-secret.yaml`

- Enregistrer le nom du Pod dans une variable: `export PODNAME=$(a=$(kubectl get pods | grep 'mysql' | awk '{print $1}') && set – $a && echo $1)`

Créer la base de données: copie des commands dans le Pod
- `kubectl cp ./create_database_quotesdb.sql $PODNAME:/tmp/create_database_quotesdb.sql`
- `kubectl cp ./create_database.sh $PODNAME:/tmp/create_database.sh`

et exécution du script:
- `kubectl exec deploy/mysql -- /bin/bash ./tmp/create_database.sh`

Créer les tables de la base de données: copie des commandes dans le Pod
- `kubectl cp ./create_table_quotes.sql $PODNAME:/tmp/create_table_quotes.sql`
- `kubectl cp ./create_tables.sh $PODNAME:/tmp/create_tables.sh`

et exécution du script:
- `kubectl exec deploy/mysql -- /bin/bash ./tmp/create_tables.sh`

Remplir les tables de la base de données:
- `kubectl cp ./populate_table_quotes_BASH.sql $PODNAME:/tmp/populate_table_quotes_BASH.sql`
- `kubectl cp ./quotes.csv $PODNAME:/tmp/quotes.csv`
- `kubectl cp ./populate_tables_BASH.sh $PODNAME:/tmp/populate_tables_BASH.sh`
- `kubectl exec deploy/mysql -- /bin/bash ./tmp/populate_tables_BASH.sh`

Interroger la base de données:
- `kubectl cp ./query_table_quotes.sql $PODNAME:/tmp/query_table_quotes.sql`
- `kubectl cp ./query_table_quotes.sh $PODNAME:/tmp/query_table_quotes.sh`
- `kubectl exec deploy/mysql -- /bin/bash ./tmp/query_table_quotes.sh`

Utiliser notre base de données comme backend: 
- fixer la variable environnementale `kubectl set env deployment/quotes DB_SERVICE_NAME=mysql`
- changer l'image du backend "quotes" pour pointer vers notre nouveau backend `kubectl set image deploy quotes quotes=quay.io/donschenck/quotes:v2`

=> fonctionne, nous avons maintenant un site qui charge des citations au hasard depuis la base de données
## Gérer les Deployments
- `kubectl delete pod <pod-name>`: arrête un Pod, qui est ensuite normalement relancé automatiquement
- `kubectl delete deployment <deployment-name>`: arrête effectivement un Pod, sans qu'il soit relancé
- Dépannage des Pods "pending": https://learn.redhat.com/t5/DO280-Red-Hat-OpenShift/Pending-Pods/td-p/35635
- [Manage resources in Kubernetes](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)

## Monitoring
Dans Red Hat Openshift Local, le monitoring "core functionality" [nécessite 14 GiB of RAM](https://access.redhat.com/documentation/th-th/red_hat_openshift_local/2.5/html-single/getting_started_guide/index#starting-monitoring_gsg) (ou 15 GB), ce qui excède les capacités de notre système
