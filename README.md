# Docker-B3-CT-2018-TP2

# B3 - Conteneurisation 

**RAPPELS :

* Désactiver SELinux:
* Laisser activé le firewall et le configurer si besoin:
* inspirez-vous du TP précédent pour certains détails (install + conf Docker)
* un hostname correctement défini (et le fichier `/etc/hosts` rempli)
  * pour TOUS les hôtes (si vous en utilisez plusieurs, comme pour la partie Docker Swarm)
  * dans TOUTES les parties du TP
  
![img](/1.png)
![img](/2.png)
![img](/3.png)

## Part 1 : Gitlab
Ici, seront déployés, sur une nouvelle VM :  

* un gitlab-ce 
* un gitlab-runner
* un démon Docker
* un registry Docker

Le but va être de monter une pipeline de CI/CD Gitlab. Un morceau de code vous sera fourni, il devra être hébergé par Gitlab, testé lors des commits, puis déployé sur un serveur de "staging".

### 1. Ajout d'un disque

* ajouter un disque à la machine (30Go)
* utiliser LVM : ajouter ce disque à un volume group nouvellement créé, et s'en servir pour monter une partition `/data` de 15 Go
* cette partition `/data` stockera... les données de vos application
  * vous utiliserez `fstab` pour que le volume soit monté automatiquement au boot
  
> Nous ajoutons un disque physique depuis notre hypviseur.

> Nous utilisons la commande pvdisplay afin de montrer les disques physique présent dans la machine.

![img](/3,5.png)

> Ensuite nous allons ajouter ce disque à un volume group appeler mvg.

![img](/5.png)

> Ont convertie se disque physique en volume logique.

![img](/6.png)

> Aprés on va crée une partition avec un filesystem ext4. 

![img](/7.png)

![img](/8.png)

> Création d'un répertoire `/Data` à la racine et montage de la partition Vol1 dans le répertoire `/Data` .

![img](/9.png)

> Ensuite ont vérifie la la configuration à l'aide de la commande `df -h`.

![img](/10.png)

> Et pour finir nous ajoutons notre configuration dans le fichier `/etc/fstab` qui va sauvegarder la conf au prochain boot.

![img](/11.png)



### 2. Installer Docker

* suivre la doc officielle
* utiliser comme répertoire de données `/data/docker`

> Nous suivont la documentation officiel

> Les prérequis

![img](/12.png)

> Nous ajoutons le mirroire docker pour avoir accés au mise à jour.

![img](/13.png)

> Puis nous installons docker.

![img](/14.png)

> Aprés cela nous allons changer le réperoire de donner de docker.

> Il faut d'abord crée une fichier daemon.json dans `/etc/docker/`

![img](/15.png)

> Puis redémarrer le service avec `systemctl`

![img](/16.png)

> On peut prouver que la configuration fonctionne en inspectant le répertoire `/Data/docker/`

cat ![img](/17.png)



### 3. Installer gitlab-ce
On va utiliser l'installation "omnibus" de Gitlab. N'hésitez pas à vous référer à [la doc officielle](https://about.gitlab.com/installation/#centos-7?version=ce). 
    
```
sudo yum install -y curl policycoreutils-python openssh-server openssh-clients
sudo systemctl enable sshd
sudo systemctl start sshd

sudo firewall-cmd --permanent --add-service=http
sudo systemctl reload firewalld

sudo yum install postfix
sudo systemctl enable postfix
sudo systemctl start postfix

curl -sS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.rpm.sh | sudo bash
sudo EXTERNAL_URL="http://gitlab.example.com" yum install -y gitlab-ce
```

* une fois terminé, il vous faudra activer le TLS
    * vous pouvez générer une paire de clé avec la commande : `openssl req -new -newkey rsa:2048 -days 365 -nodes -x509 -keyout server.key -out server.crt`
    * déplacer le certificat dans `/etc/gitlab/ssl/<HOSTNAME>.crt`
    * déplacer la clé dans `/etc/gitlab/ssl/<HOSTNAME>.key`
    * modifier le fichier `/etc/gitlab/gitlab.rb` :
        * `external_url 'https://<HOSTNAME>'`
    * redémarrer gitlab complètement avec `gitlab-ctl reconfigure`
    * n'oubliez pas de reconfigurer le firewall ! (je veux voir votre config firewall dans le compte-rendu)
* vous devrez pouvoir vous connecter sur `https://<HOSTNAME>`

### 4. Docker registry 

* créer un projet de test sur l'interface Gitlab, et récupérer le dépôt Git en local

* on va aussi activer le registre Docker
    * modifier le fichier `/etc/gitlab/gitlab.rb` :
        * `registry_external_url 'https://<FQDN>:<PORT>'`
    * redémarrer gitlab complètement avec `gitlab-ctl reconfigure`
* vous devriez pouvoir avoir accès au registre dans votre projet (sur l'interface web)
* et vous devriez pouvoir vous logger avec `docker login <FQDN>:<PORT>`
    * sauf que votre registe est 'insecure' : le certificat est auto-signé
    * trouvez un moyen de l'accepter malgré tout (et expliquez dans le compte-rendu)

* push une image sur l'espace du registre dédié à ce projet
    * le nom de l'image doit se conformer à une syntaxe précise, par exemple, pour push une image alpine
```
# On récupère une image alpine sur le Hub
docker pull alpine

# On la retag en suivant la convention de nommage (obligatoire)
docker tag alpine <REGISTRY_URL>:<PORT>/<GITLAB_USER>/<GITLAB_REPO>/<IMAGE>:<TAG>

# Par exemple
docker tag alpine gitlab.b3.ingesup:9999/root/test-project/alpine:mine

# On se log à notre registre
docker login <HOSTNAME>:<PORT>
# Par exemple
docker login gitlab.b3.ingesup:9999

# On peut push 
docker push gitlab.b3.ingesup:9999/root/test-project/alpine:mine 
```

* Expliquer comment mettre en place un check de syntaxe sur les Dockerfiles et `docker-compose.yml` AVANT d'accepter un push
  * **(optionnel)** Mettre en place ce contrôle de syntaxe 
   
### 5. Gitlab-runner

Gitlab-runner est le nom de l'utilitaire permettant de transformer une machine en Runner pour Gitlab. Awi, qu'est-ce qu'un Runner ? De façon simple, un Runner est un hôte que Gitlab peut utiliser pour lancer des actions.  
Exemple : vous développez une appli Python. Vous faites des push vers le dépôt Gitlab. Automatiquement, des actions sont exécutées pour tester le bon fonctionnement de l'app Python. C'est le Runner qui exécute ces actions.  

* installer un runner (il lancera vos tests)
    * se référer à [cette doc](https://docs.gitlab.com/runner/install/linux-repository.html)
* enregistrer le runner pour votre projet
    * vous trouverez le token sur l'interface graphique, dans votre projet, `Settings > CI/CD > Runner Settings`
    * **important** : utilisez le tag 'docker' quand le prompt vous le demandera
    * **important** : utilisez l'executor 'docker' quand le prompt vous le demander
```
sudo gitlab-runner register \
  --tls-ca-file /etc/gitlab/ssl/<HOSTNAME>.crt
```

* **important** : relancer la commande et préciser 'shell' pour le tag et l'executor

* un fichier a été généré et automatiquement configuré dans `/etc/gilab-runner/config.toml`
    * éditer le fichier
    * dans la clause [[runners.docker]] ajoutez les lignes :
* `extra_hosts = ["<HOSTNAME>:<IP>"]`

La configuration du Runner est OK, testons-le : 

* clonez un dépôt de test depuis votre gitlab

* créer un fichier `.gitlab-ci.yml` à la racine du dépôt de votre projet
    * mettez en place un build simpliste, par exemple :
```
image: 
  name: debian:8-slim

sleepytest-in-docker:
  script:
    - ls
    - whoami
    - df -h
    - /bin/sleep 3
  tags:
    docker
```
* une fois add, commit et push, le build devrait ête visible sur l'interface graphique de Gitlab :)

* vous pourrez trouver une bonne doc sur le contenu d'un build Gitlab à [cette url](https://docs.gitlab.com/ee/ci/yaml)

* je vous conseille de jouer un peu avec les builds avant de vous attaquer à la suite (essayez un peu l'executor `shell`, par exemple)

* **NB** : certaines configurations (simples) seront nécessaires pour que tout ça marche. Les messages d'erreur devraient êter explicites. Cherchez par vous-mêmes avant tout, mais n'hésitez pas à m'appeler. 

### 6. Build, build, build

Ici on va pousser un peu plus la partie Gitlab + Docker. Nous mettrons en place des configurations pour répondre à deux besoins très courants : 
* **Besoin 1 :** des dévs sont dans votre boîte, ils utilisent Docker pour faciliter leurs développements. Ils ont besoin d'un endroit où héberger leurs images Docker ; dans l'idéal cet endroit est étroitement lié à leurs dépôts `git` (Gitlab pour nous).
  * hébergement d'image Docker avec le Gitlab Registry, qui est nativement lié à chacun des projets
  * mise en place d'un `docker build` et `docker push` automatisé
* **Besoin 2 :** maintenant l'appli est packagée puis poussée sur une plateforme centrale (le Registry) automatiquement. Il serait bon de tester le bon fonctionnement de l'appli. 
  * récupération d'image Docker depuis le Gitlab Registry 
  * lancer une commande de test dans le conteneur pour vérifier que l'app tourne

#### Besoin 1 : automatiser le packaging et la mise à disposition d'une image

* créer un nouveau projet 
* créer un fichier `.gitlab-ci.yml` à la racine du projet
  * vous utiliserez le concept de [`stage`](https://docs.gitlab.com/ee/ci/yaml/#stages) 
  * vous utiliserez le tag `shell`
* créer un Dockerfile à la racine du projet
  * il se base sur une Alpine
  * `python3` doit être installé
  * l'image doit exécuter`python3 -m http.server 8888` 
* le but est donc de créer un tâche (dans le stage `build`) qui se log au registry, effectue un `docker build` puis push l'image construite

* **HINT :** pour vous faciliter le tâche, certaines variables sont disponibles dans le fichier `.gitlab-ci.yml`, [voir ici](https://docs.gitlab.com/ee/ci/variables/README.html). Par exemple :

```shell
  script:
    - /bin/docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" $CI_REGISTRY
```
* **HINT :** avant de passer des commandes `docker` dans un `.gitlab-ci.yml` on s'assure d'avoir `docker`. Bonne pratique : faisant un `docker info`. ([cette clause](https://docs.gitlab.com/ee/ci/yaml/#before_script-and-after_script) peut être utile)

#### Besoin 2 : tester l'application packagée

On va rester très simple ici pour la partie testing : 
* ajoutez le binaire `curl` à l'image précédente
* dans une tâche du stage `test` 
  * utilisez le tag `docker` 
  * utilisez l'image précédemment construite (son nom est déjà stockée dans une variable ;) )
  * exécutez un `curl` local pour vérifier la disponibilité de l'appli
  
* **HINT** : utilisez l'executor Docker sur l'image poussée précédemment. Votre image doit permettre d'utiliser un shell dedans (=> pas d'`ENTRYPOINT` et tout en `CMD`). Dans des cas réels, on voit souvent deux images construites puis poussées : une de test, qui est testée (tout en `CMD`), puis une deuxième, de prod (tout en `ENTRYPOINT`).
  
#### Besoin 3 : packaging avancé

Mettre en place une pipeline de build + tests similaire en packageant l'application Python du TP1 (interface web + Redis). 

## Part 2 : Docker swarm

### 1. Création du Swarm

Partez sur de nouvelles VMs, la VM Gitlab est devenue lourde, et vous éviterez les effets de bord. 
* Il vous en faudra 3 avec `docker` installé. Je vous conseille d'en créer une, installer `docker` et clone après :)
* 1Go de RAM suffira amplement pour chacune d'elle.
* toujours le répertoire `/data` en LVM obligatoire (vous pouvez aussi faire cette opération AVANT de cloner)
* vos 3 hôtes devront pouvoir se joindre avec leurs hostnames (configurez le fichier `/etc/hosts`)

Création du Swarm

* créer un swarm avec 3 noeuds
    * sur chaque noeud, Docker devra être installé
    * modifier la configuration du démon pour utiliser 
        * `metrics-addr: 0.0.0.0:9323` afin de récolter des métriques sur le swarm plus tard (expérimental encore)
        * `experimental: true` à true
    * faites attention à votre firewall (voir [ici](https://docs.docker.com/engine/swarm/swarm-tutorial/#the-ip-address-of-the-manager-machine) pour plus de renseignements sur les ports à ouvrir)
    * créez le swarm en suivant [la doc](https://docs.docker.com/engine/swarm/swarm-tutorial/create-swarm/)
 
 
Jouez un peu avec le swarm : 
* testez le en lançant un service répliqué, puis en éteignant les VMs (ou juste couper `docker.service`, ou en tuant les conteneurs), pour voir comment il réagit : 
```shell
 $ docker service create --replicas 5 --name nginx-test --constraint node.role=worker nginx
 ```
* utilisez l'option `docker service create --publish` pour pouvoir joindre votre application
* utilisez `docker service scale` pour augmenter/réduire le nombre de replicas

* **HINT :** pour empêcher un hôte de recevoir des ordres (bonne idée pour les masters, dans de gros Swarm,), on peut le positionner en situation [`DRAIN`](https://docs.docker.com/engine/swarm/swarm-tutorial/drain-node/).

### 2. Lancer une app sur le Swarm

* le moyen le plus simple de déployer des images custom sur tout le cluster est d'utiliser un Registry. Vous pouvez vous familiariser avec le principe [ici](https://docs.docker.com/engine/swarm/stack-deploy/) (sérieux, cette doc rend mon cours obsolète d'année en année :( ).
  * vous devrez ajouter la clause `insecure-registries` dans votre `/etc/docker/daemon.json` dans le cas où vous ne configurez pas de HTTPS

* vous allez réutiliser le `docker-compose.yml` du TP1
  * l'app Python + Redis avec NGINX en front
  * adapter le compose pour pouvoir le déployer sur le swarm :
    * ne déployer que sur les workers
    * déployer plusieurs réplicas de l'app python
  * Vous aurez besoin de choses comme : 
```
deploy:
  mode: replicated
  replicas: 5
  labels:
    label1: 'value1'
  placement:
    constraints: 
      - node.role == worker
```

* **HINT** : `docker-compose build` permet de build toutes les images d'un compose et `docker-compose push` de pousser toutes les images buildées :)

* Réflechissez un peu sur l'utilité d'utiliser NGINX ici. Est-ce toujours pertinent maintenant que nous avons déployé notre application dans un swarm ? Imaginez et proposez une solution.

* Réfléchissez sur l'emploi d'une base de données avec plusieurs replicas. Pourrait-on faire scale la base de données (Redis ici) de façon aussi simple et transparente que l'app Python ? (problème de cohérence de données ?). Imaginez et proposez une solution. 

### 3. Faire joujou : obtenir de la visualisation et de métriques

* lancer un [Portainer](https://portainer.io/) pour piloter le swarm (en tant que service swarm, pas juste un docker run :) )

* utiliser [Weave Cloud](https://www.weave.works/docs/cloud/latest/install/docker-swarm/) pour monitorer votre déploiement Swarm
  * il vous faudra un compte Weave
  * inscrivez-vous (n'hésitez pas à utiliser un mail jetable :) )
  * demandez une connexion à une instance de Docker Swarm pour récupérer votre token
  * utilisez un conteneur Docker Weave([toujours la même page](https://www.weave.works/docs/cloud/latest/install/docker-swarm/)) avec votre token
  * le conteneur Weave va utiliser votre swarm pour lancer des services. Regardez et expliquez un peu ce qu'il vient de se passer. 
  * une fois lancé, RDV sur l'interface graphique de Weave pour voir la magie. Explorez un peu y'a une tonne d'infos. 
  
* on pourrait aussi lancer un p'tit [Prometheus](https://prometheus.io/) à la main pour obtenir des courbes personnalisables (Prometheus fait plein de trucs en vrai). C'est un des outils les plus utilisés avec Swarm ou Kubernetes pour la partie monitoring. (PS : Weave l'a installé pour s'en servir ehe).

## Part 3 (bonus) : Kubernetes

* **caractéristiques**
    * orchestration de conteneurs (comme Swarm)
    * projet de Google
    * réputé très robuste
    * profondément ancré dans le monde de la conteneurisation/cloud/microservices

Ici est décrite une installation extrêmement simplifiée, sans détails, visant juste à mettre en place un petit cluster Kubernetes (pas très très sécurisé, mais fonctionnel :) ).

On va l'installer avec `kubeadm`. 
Préparez deux machines : 
	
* la première avec un petit 2/3Go de RAM ça fait pas de mal, et 2 coeurs proc
* centOS7
* swap désactivée (`swapoff` et/ou `/etc/fstab`)
* docker installé
    * le démon utilise le driver cgroup `cgroupfs` (`--exec-opt native.cgroupdriver=cgroupfs` sur la ligne `dockerd`)
* les éléments nécessaire à Kubernetes installés, suivre la doc [ici](https://kubernetes.io/docs/setup/independent/install-kubeadm/)
*   * le kubelet utilise aussi le driver cgroup `cgroupfs` (dans le fichier `/etc/systemd/system/kubelet.service.d/10-kubeadm.conf`)
    * ces deux configs permettent d'évite des problèmes plus tard. L'idée est surtout de mettre le même driver des deux côtés (`docker` et `kubelet`). `cgroupfs` sera celui qui vous posera le moins de soucis de configuration.

* lisez ce qu'il y a en dessous **d'abord** puis suivre la doc [ici](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/)
    * on va utiliser Calico comme driver réseau. Donc pour l'initialisation du cluster (ça peut prendre un p'tit moment)   avec `kubeadm init` utilisez : 
```
kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address <VM_IP>
```
* après l'init, attendez que les pods soient en état de marche avant de dérouler la documentation (seul les DNS doivent restés éteints)
   * une fois le cluster mis en place (quand vous faites un `kubectl get pods --all-namespace` tout le monde devrait être OK) déployez le dashboard Kubernetes avec :
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```
* pour que vous puissiez bypass le login (c'est sale, mais on veut juste que ça marche :) ) : 
    
    
```
cat <<EOF | kubectl create -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system
EOF
```
* pour pouvoir y accéder depuis un navigateur de votre hôte : 
```
kubectl proxy --address <VM_IP> --accept-hosts='.*'
```
* go sur `http://<VM_IP>:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/`
* essayez de déployer un service de test (en plus du dashboard)
* on peut monter une stack avec un Heapster(+InfluxDB +Grafana) pour avoir des métriques dans le dashboard Kubernetes, ou aussi un Weave, ou du Prometheus !
