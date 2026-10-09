# TP 1 DOCKER Victor LI YIM

## DATABASE PostgreSQL

Les scripts sql sont copiés avec **COPY** dans /docker-entrypoint-initdb.d/
PostgreSQL exécute automatiquement ces scripts quand la base de données est initialisée pour la première fois.

Création de l'image :
```bash
docker build -t my-postgres .
docker network create app-network
```
Lancement PostgreSQL
```bash
docker run --name my-postgres \
--network app-network \
-e POSTGRES_DB=db \
-e POSTGRES_USER=usr \
-e POSTGRES_PASSWORD=pwd \
-v ~/postgres-data:/var/lib/postgresql/data \
-d my-postgres
```

**Adminer**

Adminer fournit une interface web pour accéder et gérer la base de donnée PostgreSQL.
Les paramètres de connexion sont les suivants :
System: PostgreSQL
Server: my-postgres
Username: usr
Password: pwd
Database: db

```bash
docker run \
-p "8090:8080" \
--network app-network \
--name adminer \
-d \
adminer
```

Adminer est accessible via:

http://localhost:8090

**Question 1**

Il vaut mieux utiliser -e car cela évite de mettre des informations sensibles, comme les mots de passe, directement
#dans le Dockerfile. Cela permet aussi de changer les variables d\'environnement sans modifier l\'image
#Docker.

**Question 2**

Sans volume les données sont stockées dans le conteneur, donc si on supprime le conteneur les données le sont avec.
Stocker les données dans le volume permet d'écarter les données du cycle de vie du conteneur

## Backend API

**Question 3**

On utilise le multistage build car ça permet de séparer la compilation et l'exécution ce qui nous permettra de rendre 
l'image finale plus légère.

**Les différentes étapes du Dockerfile (dans simpleapi)**

- Compilation: FROM eclipse-temurin:21-jdk-alpine AS javaapp
- Définition de dossier de travail : WORKDIR /opt/javaapp
- Installation du Maven : RUN apk add --no-cache maven
- Copie les dossiers dans le conteneur : COPY pom.xml ., COPY src ./src
- Compilation du projet pour génération du .jar : RUN mvn package -DskipTests
- Étape du Run : FROM eclipse-temurin:21-jre-alpine
- Dossier de travail : WORKDIR /opt/javaapp
  - Récupération du .jar : COPY --from=javaapp-build /opt/javaapp/target/*.jar app.jar
- Port exposé : EXPOSE 8080
- Lancement de l'application ENTRYPOINT ["java", "-jar", "app.jar"]

Comment le tester :
depuis le dossier simpleapi (cd simpleapi) lancez les commandes: 
- docker build -t javaapp .
- docker run --rm -p 8080:8080 javaapp
- Test basique : http://localhost:8080
- Pour tester le paramètre dans la fonction greeting : http://localhost:8080/?name=Victor

L'API retournera un fichier json de ce type :
{
"id": 1,
"content": "Hello, Victor!"
}

### Connecter le backend à PostgreSQL avec Docker

Le backend Spring Boot et PostgreSQL doivent être connectés au même réseau Docker.

Le réseau utilisé est `app-network`.

PostgreSQL est lancé avec :

```bash
docker run --name my-postgres \
  --network app-network \
  -e POSTGRES_DB=db \
  -e POSTGRES_USER=usr \
  -e POSTGRES_PASSWORD=pwd \
  -d my-postgres
```
Ensuite on lance le backend avec:

```bash
docker run --rm \
  --network app-network \
  -p 8080:8080 \
  javaapp
```
Dans application.yml, le backend utilise le nom du conteneur PostgreSQL pour se connecter à la base :
```bash
spring:
datasource:
url: jdbc:postgresql://my-postgres:5432/db
username: usr
password: pwd
driver-class-name: org.postgresql.Driver
```

Le backend peut ainsi communiquer avec PostgreSQL grâce au réseau Docker app-network.
Api : http://localhost:8080
Exemple d'endpoint : http://localhost:8080/departments/IRC/students

## HTTP Server

Test avec la configuration par défaut (avant d'ajouter le proxy) :

```bash
docker build -t my-httpd .
docker run -d --name my-httpd -p 8080:80 my-httpd
curl http://localhost:8080
docker stats --no-stream my-httpd
docker inspect my-httpd
docker logs my-httpd 
```

Configuration : récupérer le httpd.conf par défaut
```bash
docker cp my-httpd:/usr/local/apache2/conf/httpd.conf ./httpd.conf
```
Reverse proxy

Pour lancer l'ensemble

```bash
docker rm -f my-httpd
docker network create mynet

# base de données (le nom my-postgres est attendu par l'API)
docker run -d --name my-postgres --network mynet \
  -e POSTGRES_DB=db -e POSTGRES_USER=usr -e POSTGRES_PASSWORD=pwd postgres:17

# API
docker build -t simple-api ./simpleapi
docker run -d --name backend --network mynet simple-api

# Apache
docker build -t my-httpd .
docker run -d --name my-httpd --network mynet -p 80:80 my-httpd
```

Vérification

```bash
curl -i http://localhost/
```

Nettoyage

```bash
docker rm -f my-httpd backend my-postgres
docker network rm mynet
```
Question 5

Un reverse proxy est un serveur placé devant une ou plusieurs applications. Les clients ne parlent qu'à lui, et c'est lui qui relaie les requêtes vers les bons serveurs. On en a besoin pour plusieurs raisons :

- Point d'entrée unique : le client utilise une seule adresse et un seul port, sans connaître l'organisation interne (nombre de services, ports, noms de conteneurs).
- Sécurité : les applications ne sont pas exposées directement sur Internet. Le proxy peut filtrer les requêtes, cacher les détails internes et limiter les abus.
- Répartition de charge : le proxy peut répartir les requêtes entre plusieurs instances de l'application. Si l'une tombe, les autres continuent de répondre.
- Contenu statique et cache : il peut servir lui-même les fichiers statiques (comme un front-end) et mettre des réponses en cache, ce qui soulage l'application.
- Routage : selon le chemin ou le nom de domaine, il envoie vers différents services (par exemple /api vers un backend et / vers le front-end).

## Linked Application

Question 6

Docker Compose est important parce qu'il permet de décrire toute l'application (les conteneurs, leurs réseaux, leurs 
volumes et leurs variables d'environnement) dans un seul fichier, au lieu de lancer et configurer chaque conteneur à la
main avec de longues commandes docker run. Une seule commande, docker compose up, crée et démarre l'ensemble dans le bon
ordre, et une autre, docker compose down, l'arrête proprement. Comme ce fichier peut être versionné et partagé, toute 
l'équipe obtient exactement la même installation, sur n'importe quelle machine. Compose gère aussi les dépendances entre 
services (par exemple l'API attend que la base soit prête), le redémarrage automatique en cas de panne, et permet aux 
conteneurs de se joindre par leur nom grâce aux réseaux. Il simplifie ainsi le développement, les tests et le déploiement 
d'applications composées de plusieurs services.

Question 7

| Commande | Rôle |
|---|---|
| `docker compose up -d` | Crée et démarre les services en arrière-plan |
| `docker compose up -d --build` | Idem, en reconstruisant les images |
| `docker compose down` | Arrête et supprime les conteneurs et les réseaux |
| `docker compose down -v` | Idem, et supprime aussi les volumes (données perdues) |
| `docker compose ps` | Liste les services et leur état |
| `docker compose logs -f [service]` | Affiche les logs en continu |
| `docker compose build` | Reconstruit les images sans démarrer |
| `docker compose restart [service]` | Redémarre un service |
| `docker compose stop` / `docker compose start` | Arrête / relance les services sans les supprimer |
| `docker compose exec <service> <commande>` | Exécute une commande dans un conteneur en cours d'exécution |
| `docker compose config` | Valide le fichier et affiche la configuration finale |

Question 8

- database : image construite depuis ./database (PostgreSQL 17). Identifiants lus dans .env. 
Les données sont stockées dans le volume db-data. Le healthcheck pg_isready indique quand la base accepte les connexions. 
Réseau : backend-net uniquement, aucun port publié.
- backend : l'API Spring Boot construite depuis ./simpleapi. Elle reçoit l'URL et les identifiants de la base par 
variables d'environnement. Elle attend que database soit saine. Son healthcheck interroge /actuator/health. 
Réseaux : backend-net (vers la base) et front-net (vers Apache). Aucun port publié.
- HTMl : Apache construit depuis ./httpd, reverse proxy vers backend:8080. Seul service exposé (80:80).
Il démarre quand backend est sain. Réseau : front-net uniquement.
- networks : proxy-net et backend-net séparent la couche d'entrée de la couche de données.
- volumes : db-data assure la persistance de la base.

Pour tester l'API : 
```bash
curl http://localhost/department
curl http://localhost/students
```

Question 9

## Publication sur Docker Hub
```bash
docker login
docker tag tp1-database victorliy/tp1-database:1.0
docker tag tp1-backend  victorliy/tp1-backend:1.0
docker tag tp1-httpd    victorliy/tp1-httpd:1.0
docker push victorliy/tp1-database:1.0
docker push victorliy/tp1-backend:1.0
docker push victorliy/tp1-httpd:1.0
```

| Image | Rôle | Port |
|---|---|---|
| `victorliy/tp1-database:1.0` | PostgreSQL 17 avec scripts d'initialisation | 5432 (interne) |
| `victorliy/tp1-backend:1.0` | API Spring Boot | 8080 (interne) |
| `victorliy/tp1-httpd:1.0` | Apache, reverse proxy | 80 |

Lien : https://hub.docker.com/u/victorliy

Question 10

Un registre en ligne rend les images accessibles de partout : un collègue ou un serveur peut les récupérer avec docker
pull sans avoir le code source ni reconstruire l'image. Cela garantit que tout le monde exécute exactement la même 
image, avec une version identifiée par un tag (1.0, 1.1), et permet de revenir à une version précédente si besoin. C'est
aussi la base du déploiement : les serveurs de production et les pipelines CI/CD tirent les images depuis le registre. 
Enfin, il sert de sauvegarde et de point de partage central, avec la possibilité de limiter l'accès (dépôts privés, 
registre auto-hébergé en entreprise).