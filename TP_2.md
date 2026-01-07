# Rapport du TP 2

# Sommaire

1. [Introduction et Centralisation](#1-introduction-et-centralisation)
2. [Architecture du Pipeline](#2-architecture-du-pipeline)
3. [Détails techniques des étapes](#3-détails-techniques-des-étapes)
    - 3.1 [Stage Install : Double Environnement Virtuel](#31-stage-install--double-environnement-virtuel)
    - 3.2 [Stage Test : Audit et Conformité](#32-stage-test--audit-et-conformité)
    - 3.3 [Stage Build : Packaging et Persistance](#33-stage-build--packaging-et-persistance)
    - 3.4 [Stage Deploy : Livraison Sécurisée](#34-stage-deploy--livraison-sécurisée)
4. [Gestion des branches](#4-gestion-des-branches)
5. [Conclusion](#5-conclusion)

# 1. Introduction

L'objectif de ce TP est de migrer une architecture statique vers une application dynamique **Python Flask**.

**Point clé de l'infrastructure :** Pour ce projet, nous avons appliqué le principe d'**exfiltration de la configuration**. Le fichier `.gitlab-ci.yml` n'est pas stocké dans le dépôt de l'application mais dans un dépôt externe nommé **"Gitlab CI FLASK"**.

* **Configuration :** Dans les paramètres du projet, le chemin pointe vers `.gitlab-ci.yml@cicd/gitlab-ci-yml`.
* **Avantage :** Cela permet une gestion centralisée et uniforme de la CI sur toutes les branches du projet, tout en protégeant la logique de déploiement des modifications accidentelles dans le code source.

# 2. Architecture du pipeline

Le pipeline est organisé en **4 stages** permettant une isolation stricte des responsabilités :

1. **Install** : Création des environnements isolés.
2. **Test** : Audit de qualité et sécurité.
3. **Build** : Minification et archivage incluant les dépendances.
4. **Deploy** : Transfert vers la machine de destination.

# 3. Détails techniques des étapes

## 3.1 Stage Install : Double environnement virtuel

Une évolution majeure a été apportée via l'utilisation de deux VirtualEnvs distincts :

* **`venv` (Production) :** Contient uniquement les librairies nécessaires au fonctionnement de l'application (Flask, etc.). C'est cet environnement qui sera déployé.
* **`venv_test` (Audit) :** Contient les outils de test et de sécurité.
* **Justification :** Cette séparation garantit que les outils de développement (Pylint, Bandit, etc.) ne polluent pas le livrable final, respectant les contraintes de sécurité et de poids des archives.

## 3.2 Stage Test : Audit et conformité

Le job `py_lint` utilise exclusivement le `venv_test`. Nous y exécutons :

* **Pylint & Black** : Pour la propreté du code.
* **Bandit** : Pour le scan de vulnérabilités.
* **Pytest** : Pour les tests fonctionnels.
Le succès de ce stage est impératif pour passer à la phase de build.

## 3.3 Stage Build : Packaging et persistance

Les jobs de build (`build_zip` et `build_tar_gz`) préparent le dossier final :

1. **Minification** : Le JavaScript est compressé via `uglifyjs`.
2. **Préparation du dossier `build/**` : Nous y copions le code source (`hello.py`, `templates`, etc.) **ET** le dossier `venv/` de production. Ainsi, l'application est "prête à l'emploi" dès son extraction.
3. **Gestion des Artifacts** : L'archive générée est conservée **7 jours**.
* **Justification** : L'artifact est indispensable pour que le stage `Deploy` puisse accéder au fichier. Le délai de 7 jours permet un historique de déploiement sans saturer le stockage GitLab.



## 3.4 Stage Deploy : Livraison sécurisée

Le déploiement récupère l'archive (`.zip` pour la preprod, `.tar.gz` pour la prod).

* **Sécurité** : Authentification par clé privée via `$SSH_PRIVATE_KEY`.
* **Fiabilité** : Utilisation de `$FINGERPRINT_SSH` pour valider l'identité du serveur `192.168.56.30` sans désactiver les vérifications de sécurité SSH.

# 4. Gestion des branches

Le comportement du pipeline s'adapte automatiquement à la branche :

* **`main`** : Branche de développement. Seuls les tests sont joués.
* **`preprod`** : Exécute les tests, minifie, build en **.zip** (incluant le venv) et déploie.
* **`prod`** : Exécute les tests, minifie, build en **.tar.gz** (incluant le venv) et déploie.

# 5. Conclusion

Ce TP nous a permis de mettre en place une stratégie CI/CD de niveau industriel. La séparation des environnements de test et de production au sein même du pipeline, couplée à une externalisation de la configuration CI, offre une robustesse et une sécurité optimales. L'inclusion du `venv` dans les archives garantit une portabilité parfaite de l'application Flask vers la VM de déploiement.
