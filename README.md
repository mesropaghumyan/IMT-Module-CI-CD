# Rapport de TP : Mise en œuvre d'une chaîne CI/CD avec GitLab

| Date | Matière | Etudiants |
|-|-|-|
| 07/01/2026 | INTÉ DEPLOIEMENT CONTINUS | - Mesrop AGHUMYAN<br>- Ian BERTIN |

# 1. Introduction

L'objectif de ce TP est de mettre en place une chaîne de livraison continue.

L'infrastructure repose sur trois piliers :
- GitLab Server : Gestion du code et orchestration.
- GitLab Runner : Exécuteur des tâches (Build/Test) dans des conteneurs Docker.
- Serveur de déploiement : Serveur web final recevant les livrables.

# 2. Mise en place de l'infrastructure

## 2.1 Configuration réseau

Après l'import des VMs, nous avons configuré la résolution de noms locale pour simuler un environnement réel.

Fichier hosts (Machine hôte) : Ajout de 192.168.56.10 gitlab.example.com pour accéder à l'interface.

Fichier hosts (Runner) : Modification identique pour que le Runner puisse contacter le serveur.

## 2.2 Liaison entre le GitLab et le Runner

Pour permettre au Runner Docker de communiquer avec GitLab, nous avons édité `/etc/gitlab-runner/config.toml` :

```toml
[runners.docker]
  extra_hosts = ["gitlab.example.com:192.168.56.10"]
```

Validation : L'état du Runner est passé au vert dans l'interface d'administration GitLab.

[Capture d'écran : Etat du Runner au vert]

# 3. Phase de test

Nous avons créé un premier job pour vérifier la qualité du code HTML avec l'outil `tidy`.

- Fichier `index.html` : Contient initialement une erreur (balise `<meta` non fermée).

```html
<!DOCTYPE html>
<html lang="fr">
    <head>
        <meta charset="utf-8"
        <title>Cours CI/CD</title>
    </head>
    <body>
        <h1>Page de test !</h1>
    </body>
</html>
```

- Résultat : Le job `lint_html` échoue comme prévu, bloquant la suite de la chaîne.

[Capture d'écran : Pipeline en échec montrant l'erreur de syntaxe détectée par tidy]

# 4. Génération de livrables

Une fois le code corrigé, nous avons ajouté des étapes de transformation (minification et compression).

## 4.1 Jobs de transformation

| Job | Outil utilisé | Action |
|-|-|-|
| compress_pictures | Imagemagick | Réduction de la qualité JPEG à 70% |
| minify_css | Uglifycss (Node:24) | Suppression des espaces et commentaires du CSS | 

## 4.2 Gestion des artifacts

Pour conserver les fichiers générés et les passer d'un job à l'autre, nous avons utilisé les `artifacts` :

```yaml
artifacts:
  paths:
    - "output/"
  expire_in: 7 days
```

Vérification : En téléchargeant l'artifact depuis l'interface GitLab, nous avons constaté que le fichier style.css est passé de 3 lignes à 1 seule ligne compacte.

# 5. Orchestration par stages

Pour organiser l'exécution, nous avons défini des `stages`. Cela permet de s'assurer que les tests passent avant de lancer le build.

```yaml
stages:
  - test
  - build
  - bundle
  - deploy
```

- Observations : Les jobs `compress_pictures` et `minify_css` s'exécutent en parallèle car ils appartiennent au même stage (`build`), optimisant ainsi le temps total de livraison.

# 6. Déploiement continu (CD)

## 6.1 Sécurisation (Variables CI/CD)

Pour permettre au Runner de se connecter au serveur de déploiement sans intervention humaine, nous avons utilisé une variable d'environnement masquée : `SSH_PRIVATE_KEY`.

- Cela évite de stocker des clés privées en clair dans le dépôt Git.

## 6.2 Le Job de Déploiement

Le job `deploy` effectue les actions suivantes :
- Installation du client OpenSSH.
- Configuration de la clé privée dans le conteneur.
- Transfert des fichiers vers `/var/www/html/` sur la VM `192.168.56.30`.

Script de déploiement final :

```yaml
deploy:
  stage: deploy
  script:
    - apt update && apt install -y openssh-client
    - mkdir ~/.ssh && chmod 700 ~/.ssh
    - echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa && chmod 400 ~/.ssh/id_rsa
    - scp -o StrictHostKeyChecking=no -r output/* root@192.168.56.30:/var/www/html/
```

# 7. Gestion multi-branches et déploiement dynamique

Cette dernière étape introduit la gestion d'environnements distincts (Staging vs Production) et l'utilisation des métadonnées de commit pour assurer la traçabilité des déploiements.

## 7.1 Stratégie de branches

Nous avons créé une branche de développement nommée `dev` pour isoler les travaux en cours avant leur fusion sur la branche principale.

- Modification du code : Ajout d'une balise `<p>Infos</p>` dans le fichier `index.html` servant de marqueur pour l'injection de données.

- Contrôle d'exécution (`only`) : Nous avons restreint le job `minify_css` à la branche `main` uniquement. Cela permet d'alléger le pipeline de développement en évitant des tâches de production non nécessaires en phase de test.

## 7.2 Utilisation des variables prédéfinies de GitLab

Pour assurer la traçabilité de chaque déploiement, nous utilisons les variables d'environnement de GitLab CI pour modifier dynamiquement le contenu du site.

Dans le job `build_artifacts`, nous avons injecté les informations de commit via la commande `sed` :

```bash
# Remplacement de la chaîne "Infos" par les données du commit
- sed -i "s/Infos/Auteur : $CI_COMMIT_AUTHOR | Commit : $CI_COMMIT_SHORT_SHA/g" index.html
```

## 7.3 Déploiement ciblé par environnement

Afin de ne pas écraser la version de production lors des tests sur la branche dev, nous avons configuré le job de déploiement pour cibler des répertoires différents sur le serveur de destination.

Script de déploiement dynamique :

```yaml
deploy:
  stage: deploy
  script:
    - apt update && apt install -y openssh-client
    - mkdir ~/.ssh && chmod 700 ~/.ssh
    - echo "$SSH_PRIVATE_KEY" > ~/.ssh/id_rsa && chmod 400 ~/.ssh/id_rsa
    - |
      if [ "$CI_COMMIT_BRANCH" == "main" ]; then
        DEST_DIR="/var/www/html/main"
      else
        DEST_DIR="/var/www/html/dev"
      fi
    - ssh -o StrictHostKeyChecking=accept-new root@192.168.56.30 "mkdir -p $DEST_DIR"
    - scp -r output/* root@192.168.56.30:$DEST_DIR/
```

Résultat : Les modifications poussées sur dev sont visibles sur http://192.168.56.30/dev, tandis que la version stable reste accessible sur http://192.168.56.30/main.

# 8. Centralisation de la configuration CI/CD

Dans une optique de gouvernance et de sécurité, nous avons extrait la logique du pipeline du dépôt de code pour la centraliser dans un dépôt dédié.

## 8.1 Exfiltration du fichier .gitlab-ci.yml

Nous avons créé un nouveau projet nommé Gitlab CI YML. Ce dépôt ne contient aucune page web, mais uniquement la configuration technique du pipeline.

- Sécurisation : La fonctionnalité CI/CD a été désactivée dans les paramètres de ce dépôt de configuration pour éviter des exécutions inutiles.

- Externalisation : Le fichier `.gitlab-ci.yml` original y a été déplacé.

## 8.2 Liaison distante

Dans notre projet principal (TP CI CD), nous avons modifié le chemin de configuration dans Settings > CI/CD > General pipelines.

Syntaxe utilisée : `.gitlab-ci.yml@cicd/gitlab-ci-yml`

- Vérification : Après avoir supprimé le fichier `.gitlab-ci.yml` local de toutes les branches du projet de travail, nous avons constaté que les pipelines continuaient de se déclencher normalement.

- Avantage technique : Cette méthode permet de protéger la configuration du pipeline (le développeur ne peut plus la modifier par erreur) et de standardiser les processus de déploiement pour plusieurs projets simultanément.

# 9. Automatisation de la maintenance avec RenovateBot

La gestion des dépendances et des versions d'images est un enjeu critique. Nous avons mis en place RenovateBot pour automatiser cette surveillance.

## 9.1 Configuration du Bot

Nous avons créé un dépôt RenovateBot utilisant un template officiel via l'instruction `include:remote`.

- Authentification : Un Personal Access Token (PAT) avec les `scopes api` et `write_repository` a été généré et stocké dans la variable `RENOVATE_TOKEN`.

- Autodiscover : L'utilisation du flag `--autodiscover=true` permet au bot de scanner automatiquement tous les projets accessibles par l'utilisateur pour y proposer des mises à jour.

## 9.2 Planification (Pipeline Schedules)

Contrairement aux jobs précédents déclenchés par un push, Renovate fonctionne de manière temporelle via un Cron.

- Fréquence : Quotidienne (Every day).

- Action : Le bot analyse les fichiers de configuration (comme les versions d'images Docker) et ouvre automatiquement des Merge Requests (MR) si une version plus récente est disponible.

## 9.3 Cas pratique : Mise à jour d'image

Nous avons forcé l'utilisation d'une image spécifique dans notre dépôt de configuration :

```yaml
lint_html:
  image: debian:12
```

- Résultat : Lors de l'exécution manuelle du schedule, RenovateBot a détecté que `debian:13` était disponible et a généré une Merge Request sur le dépôt Gitlab CI YML pour suggérer la mise à jour.

# 10. Conclusion finale

Ce TP complet nous a permis de passer d'une simple vérification de syntaxe à une infrastructure DevOps industrielle et mature. Nous avons maîtrisé :

- La chaîne de build complète : Du linting à la génération d'artifacts optimisés.

- Le déploiement sécurisé : Utilisation de clés SSH et de variables d'environnement pour isoler le développement de la production.

- L'industrialisation : Centralisation des pipelines pour une meilleure maintenance et sécurité.

- La maintenance préventive : Automatisation du cycle de vie des dépendances avec RenovateBot.

L'ensemble de ces pratiques garantit non seulement la vélocité des déploiements, mais surtout la stabilité et la sécurité du système d'information sur le long terme.