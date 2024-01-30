Ce document récapitule le tutoriel [Learn Kubernetes using the Developer Sandbox for Red Hat OpenShift](https://developers.redhat.com/developer-sandbox/activities/learn-kubernetes-using-red-hat-developer-sandbox-openshift), mais en utilisant *OpenShift Local*, qui permet le déploiement local d'un cluster OpenShift, au lieu du *Developer Sandbox*, un cluster OpenShift géré dans le cloud de Red Hat. Ce choix résulte d'une difficulté à trouver les identifiants pour se connecter depuis le Terminal de ma machine au cluster du *Developer Sandbox* une fois celui-ci mis en place. Rétrospectivement, ces identifiants pouvaient probablement être obtenus à l'aide de la commande `crc console --credentials`.
 Quelques notes pertinentes sur les spécificités d'OpenShift Local, tirées du [guide officiel](https://access.redhat.com/documentation/en-us/red_hat_openshift_local/2.16/html-single/getting_started_guide/index#differences-from-production-openshift-install_gsg):
>- It uses a single node, which behaves as both a control plane and worker node.
>- It disables the Cluster Monitoring Operator by default. This disabled Operator causes the corresponding part of the web console to be non-functional.
>- The OpenShift Container Platform cluster runs in a virtual machine known as an _instance_. This might cause other differences, particularly with external networking.
## Installation et configuration initiale d'OpenShift Local 2.31 
- Télécharger le paquet d'installation et et le "*pull secret*" depuis https://console.redhat.com/openshift/create/local (n.b.: un compte Red Hat est nécessaire; le "pull secret", qui permet de pull des images de Red Hat, est demandé lors de l'installation)
- L'outil de ligne de commande pour la configuration d'*OpenShift Local* est nommée `crc` d'après *Code-Ready Containers*, l'ancien nom du projet. Il permet de configurer et lancer un cluster, après quoi nous utiliserons la commande `kubectl` pour gérer ce dernier.
- Avant de mettre en place le cluster, il faut changer le *preset* à "Openshift":
`crc config set preset openshift` 
- Configuration:
`crc setup`(dans mon cas, télécharge 4.4 GiB de données, 31 GiB à la décompression)
- Démarrage du cluster:
`crc start` => à ce stade, on nous demande le *pull secret* (disponible [à la page de téléchargement précédemment mentionnée](https://console.redhat.com/openshift/create/local))
- Vérification de la configuration du cluster et de son statut:
`crc config view`
`crc status`
- Le cluster peut être visionné et contrôlé par une interface graphique via la console web OpenShift à l'adresse: https://console-openshift-console.apps-crc.testing
- Utiliser l'outil CLI d'OpenShift `oc`:
	- l'ajouter au PATH:
	`eval $(crc oc-env)`
	- se login au cluster:
	`oc login -u developer https://api.crc.testing:6443`
## Déploiement de l'application quotes
- créer un nouveau projet:
(`oc status` && `oc projects` peuvent être utilisées pour jeter un coup d'oeil sur le contexte d'abord)
`oc new-project sampleproject`
Suivant [Learn Kubernetes using the Developer Sandbox for Red Hat OpenShift](https://developers.redhat.com/developer-sandbox/activities/learn-kubernetes-using-red-hat-developer-sandbox-openshift):
- télécharger les fichiers nécessaires à l'application:
	- `git clone https://github.com/redhat-developer-demos/quotesweb.git`
	- `git clone https://github.com/redhat-developer-demos/quotemysql.git`
	- `git clone https://github.com/redhat-developer-demos/qotd-python.git`

## Installation de l'outil de ligne de commande Kubernetes
- En plus de l'outil CLI d'Openshift `oc`, le tutoriel utilise l'outil (CLI) standard de Kubernetes `kubectl` pour gérer le cluster, et `kubectl` est utilisable également avec *OpenShift Local*. À noter que cet outil n'est pas vraiment nécessaire au final, les commandes étant les mêmes qu'avec `oc`, l'outil CLI d'OpenShift, mais il s'agissait initialement de suivre un tutoriel à la lettre. Pour installer `kubectl` (sur OS X, pour les autres systèmes utiliser les commandes usuelles):
`sudo port install kubectl-1.29` (&& `kubectl version --output=yaml`)
### Le backend "quotes" (une API REST livrant des citations)
dans `qotd-python/k8s` :
- créer un Deployment pour le Pod désiré : `kubectl create -f quotes-deployment.yaml`
- exposer le backend comme un Service au reste du cluster: `kubectl create -f service.yaml`
- créer une Route pour permettre l'accès au backend de l'extérieur du cluster: `kubectl create -f route.yaml`

- une fois terminé, 
	- `kubectl get pods` (ou `oc get pods`) pour vérifier si le pod tourne
	- `kubectl get routes`pour trouver l'URL, et enfin:
	- `curl http://<the-URL-from-above>/quotes` (+ avec `/random` à la fin) pour tester le backend
### Le frontend web
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
Dans Red Hat Openshift Local, le monitoring "core functionality" [nécessite 14 GiB of RAM](https://access.redhat.com/documentation/th-th/red_hat_openshift_local/2.5/html-single/getting_started_guide/index#starting-monitoring_gsg) (ou 15 GB), ce qui excède les capacités de notre système.

## Connaissances acquises
### Appris:
- Mettre un place un cluster et s'authentifier
- Comment le Deployments des Pods sont configurés (notamment les fichiers .yaml) et comment les Pods fonctionnent ensemble (avec en plus l'ajout de Repliques)
- Comment visualiser le cluster à travers la console web (vue développeur, vue administrateur)
- Comment diagnostiquer des problèmes basiques (dans mon cas, le manque de mémoire empêchant un Pod d'être relancé)

### Pas appris:
- Les Nodes: il n'y a qu'un seul Node dans OpenShift Local, et pas de séparation des rôles entre le plan de contrôle et les worker Nodes
- Monitoring: pas de possibilité de faire un réel monitoring du cluster et de ses composantes (manque de RAM pour activer le Cluster Monitoring Operator); tout au plus, une constatation d'un manque de mémoire via `oc describe pod <pod-name>`
- Usage approfondi de la console web: Pas d'usage de la console web pour effectuer des modifications ou opérations importantes sur le cluster; seulement pour observer le résultat de mes commandes dans le Terminal