## Question 3-1

L'inventaire liste les serveurs à gérer (ici un seul, dans le groupe `prod`)
et les variables de connexion : l'utilisateur SSH (`admin`) et le chemin de la
clé privée. La clé elle-même n'est pas versionnée.

### Commandes

| Commande | Rôle |
|---|---|
| `ansible all -i inventories/setup.yml -m ping` | Vérifie la connexion, l'utilisateur et l'authentification. Résultat attendu : `pong` |
| `ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"` | Récupère les *facts* : distribution et version du système |
| `ansible all -i inventories/setup.yml -m apt -a "name=apache2 state=present" --become` | Installe Apache (en root grâce à `--become`) |
| `ansible all -i inventories/setup.yml -m shell -a 'echo blabla > /var/www/html/index.html' --become` | Crée la page d'accueil |
| `ansible all -i inventories/setup.yml -m service -a "name=apache2 state=started" --become` | Démarre le service Apache |
| `ansible all -i inventories/setup.yml -m apt -a "name=apache2 state=absent" --become` | Désinstalle Apache |

- `-i` indique l'inventaire
- `-m` le module
- `-a` ses arguments
- `--become` exécute la commande en administrateur.

Ansible décrit un état voulu plutôt qu'une action : relancer la
commande de désinstallation donne `changed: false` car Apache est déjà absent.

## Question 3-2

Un playbook décrit, en YAML, les tâches à appliquer sur les machines de
l'inventaire. Le playbook principal (`playbook.yml`) cible tous les hôtes
(`hosts: all`), collecte les facts (`gather_facts: true`) et passe en
administrateur (`become: true`). Il se contente d'appeler les rôles, dans
l'ordre : `docker`, `network`, `database`, `app`, `proxy`.

L'installation de Docker est dans le rôle `roles/docker/tasks/main.yml`
(créé avec `ansible-galaxy init roles/docker`, en ne gardant que `tasks`).

### Tâches du rôle `docker`

| Tâche | Module | Rôle |
|---|---|---|
| Set Docker architecture | `set_fact` | Détermine l'architecture (`amd64` ou `arm64`) pour le dépôt |
| Install required packages | `apt` | Installe les prérequis (`ca-certificates`, `curl`, `gnupg`, `python3-venv`...) |
| Add Docker GPG key | `apt_key` | Ajoute la clé officielle de Docker, pour vérifier les paquets |
| Add Docker APT repository | `apt_repository` | Ajoute le dépôt stable de Docker pour la version de Debian détectée (fact `distribution_release`) |
| Install Docker | `apt` | Installe `docker-ce`, `docker-ce-cli`, `containerd.io` et le plugin compose |
| Install Python3 and pip3 | `apt` | Installe Python et pip |
| Create a virtual environment | `command` | Crée `/opt/docker_venv` (`creates` évite de le refaire) |
| Install Docker SDK | `command` | Installe le SDK Python `docker` utilisé par les modules `docker_*` d'Ansible |
| Make sure Docker is running | `service` | Vérifie que Docker tourne |

### Commandes

| Commande | Rôle |
|---|---|
| `ansible-playbook -i inventories/setup.yml playbook.yml --syntax-check` | Vérifie la syntaxe sans rien exécuter |
| `ansible-playbook -i inventories/setup.yml playbook.yml` | Exécute le playbook |
| `ansible-galaxy init roles/docker` | Crée le squelette d'un rôle |

## Question 3-3

Chaque partie de l'application est un rôle qui utilise le module
docker_container. Les variables communes sont dans
group_vars/all.yml (images, réseaux, identifiants de base). Les modules
docker_* utilisent l'interpréteur Python du virtualenv
(ansible_python_interpreter: "{{ docker_python }}"), car c'est là que le SDK
Docker est installé.

### Configuration des tâches

| Rôle | Module | Configuration |
|---|---|---|
| `network` | `docker_network` | Crée `proxy-net` (httpd et API) et `backend-net` (API et base) |
| `database` | `docker_container` | Image `tp1-database`, variables d'environnement `POSTGRES_*` (équivalent du `.env`), volume `db-data` pour les données, réseau `backend-net` seulement, aucun port publié |
| `app` | `docker_container` | Image `tp1-backend`, variables `SPRING_DATASOURCE_*` (surchargent `application.yml`, la base est jointe par le nom `database`), réseaux `backend-net` et `proxy-net`, aucun port publié |
| `proxy` | `docker_container` | Image `tp1-httpd`, port `80:80` publié (seule entrée sur le serveur), réseau `proxy-net` |

Options communes :
- `restart_policy: unless-stopped` : le conteneur redémarre après un crash ou un reboot ;
- `state: started` : l'état voulu est « conteneur lancé » (idempotent) ;
- le tag d'image (`image_tag`) est une variable, ce qui permet de déployer une version précise.

Le backend est dans les deux réseaux et sert d'intermédiaire : le proxy ne peut pas joindre la base.

Vérification : `curl http://<adresse-du-serveur>/` doit répondre via httpd puis l'API.

## Déploiement continu : est-il sûr de déployer automatiquement chaque nouvelle image ?

Non, pas sans point de sécurité car une image non relue peut poser problème.
Une dépendance vulnérable pourrait arriver directement en production. Le tag `latest`
change sans qu'on sache quelle version tourne, et les secrets (clé SSH) donnent
un accès direct au serveur.

Pour sécuriser :
- protéger `main` et exiger des pull requests relues ;
- ne déployer qu'après des tests et un Quality Gate réussis (`workflow_run` avec `conclusion == 'success'`) ;
- demander une approbation manuelle (environnement GitHub `production`) ;
- utiliser des tags précis (SHA du commit ou version) plutôt que `latest` ;
- scanner les images et limiter les droits des secrets (clé SSH dédiée, utilisateur non root, `ansible-vault` pour les mots de passe) ;
- vérifier l'application après déploiement et prévoir un retour en arrière.


Le job de déploiement est désactivé (en commentaire) : le serveur du TP n'existe plus.