## Question 3-1 

L'inventaire liste les serveurs à gérer (ici un seul, dans le groupe `prod`)
et les variables de connexion : l'utilisateur SSH (`admin`) et le chemin de la
clé privée. La clé elle-même n'est pas versionnée.

### Commandes

| Commande | Rôle |
|---|---|
| `ansible all -i inventories/setup.yml -m ping` | Vérifie la connexion, l'utilisateur et l'authentification résultat attendu : pong|
| `ansible -m setup -a "filter=ansible_distribution*"` | Récupère les *facts* : distribution et version du système |
| `ansible -m apt -a "name=apache2 state=present" --become` | Installe Apache (en root grâce à `--become`) |
| `ansible -m shell -a 'echo blabla > /var/www/html/index.html' --become` | Crée la page d'accueil |
| `ansible -m service -a "name=apache2 state=started" --become` | Démarre le service Apache |
| `ansible -m apt -a "name=apache2 state=absent" --become` | Désinstalle Apache |

- `-i` indique l'inventaire
- `-m` le module
- `-a` ses arguments
- `--become` exécute la commande en administrateur.

Ansible décrit un état voulu plutôt qu'une action : relancer la
commande de désinstallation donne `changed: false` car Apache est déjà absent.