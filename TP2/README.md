## Question 1 

Testcontainers est une bibliothèque Java qui lance de vrais conteneurs Docker pendant l'exécution des tests, puis les 
supprime à la fin. Ici, elle démarre une base PostgreSQL temporaire à laquelle l'application se connecte pendant les 
tests d'intégration. On teste ainsi avec une vraie base, identique à celle de la production, sans rien installer à la 
main, et sans que les tests dépendent d'une base partagée : chaque exécution repart d'un environnement propre et 
reproductible, en local comme dans le pipeline CI.

## Question 2

Elles permettent d'utiliser des identifiants (mot de passe, token, clé d'API) dans le pipeline sans les écrire dans le 
code. Le fichier main.yml est versionné et visible par toute personne ayant accès au dépôt, et un dépôt public l'est par
tout le monde : des identifiants y seraient récupérés par des robots en quelques minutes, puis réutilisés pour publier 
de fausses images ou accéder à tes comptes. Un secret est chiffré, masqué dans les logs, et accessible seulement au 
pipeline. On peut aussi le changer ou le révoquer sans toucher au code.

## Question 3


Par défaut, les jobs s'exécutent en parallèle. Sans needs, la construction des images démarrerait en même temps que 
les tests, et on pourrait livrer une image d'une version dont les tests échouent. Avec needs, le job de livraison
attend la fin du job de tests et ne se lance que s'il a réussi : si un test casse, rien n'est construit ni publié. 
Seul du code testé arrive donc en livraison.

## Question 4

Pousser une image sur un registre comme Docker Hub la rend disponible en dehors de la machine du pipeline. La machine 
de GitHub Actions est détruite après chaque exécution, donc une image construite mais non poussée disparaît avec elle. 
Une fois publiée, elle peut être récupérée avec docker pull par un collègue, un serveur de test ou de production, et 
chaque commit sur main produit automatiquement une image à jour : c'est la partie livraison (CD) du pipeline. Tout le
monde déploie ainsi exactement la même image, déjà testée, et le registre garde l'historique des versions.

## Quality Gate (SonarCloud)

Le workflow lance une analyse SonarCloud après les tests, avec
`mvn verify sonar:sonar`. Le projet est identifié par sa clé et son
organisation, et l'authentification se fait avec le secret `SONAR_TOKEN`.
L'option `-Dsonar.qualitygate.wait=true` fait échouer le pipeline si le
Quality Gate n'est pas atteint : du code de mauvaise qualité ne passe donc pas.

Le Quality Gate est un ensemble de seuils appliqués au nouveau code. Le gate
« Sonar way » vérifie notamment : 
couverture minimale, duplication maximale, notes de fiabilité, de sécurité
et de maintenabilité.

## Pipelines séparés

| Workflow | Déclencheur | Rôle |
|---|---|---|
| `test-backend.yml` | push sur `main` et `develop`, pull requests | Tests Maven et analyse SonarCloud |
| `build-and-push.yml` | `workflow_run` : fin de « Test backend » sur `main` | Construit et publie les images sur Docker Hub |

Le second workflow ne démarre que si le premier a réussi
(`conclusion == 'success'`) : une image n'est donc publiée que si les tests
passent. Séparer les workflows permet d'avoir des déclencheurs différents
(tests partout, publication seulement sur `main`), de relancer une partie
sans refaire l'autre, et de limiter l'accès aux secrets Docker Hub au seul
workflow de publication. À noter que pour l'instant durant un push sur une autre branche le test ne fonctionnera pas car
le test Sonar ne fonctionne que sur la branche main. Limite de SonarCloud.