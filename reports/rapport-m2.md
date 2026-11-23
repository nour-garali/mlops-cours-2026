# Rapport comparatif — Kubeflow Pipelines vs Airflow (Lab 2)

**Binôme :** Nour Garali et Ghofrane Khadhar · **Date :** 23/11/2026 · **Machine :** 4 CPU, 12 Go de RAM alloués à la VM, Ubuntu 24.04 (VMware) · **Dataset :** german-credit

## 1. Contexte

Nous avons migré le pipeline DVC du Lab 1 vers deux orchestrateurs : Kubeflow Pipelines (KFP 2.17) et Airflow 3.3. Le pipeline compte 6 steps : ingest, validate, prepare, train, evaluate et register. Il traite German Credit (1 000 demandes de crédit) et chaque step s'exécute dans un pod du cluster kind. Les deux outils utilisent la même image `localhost:5001/scoring-pipeline:1.0.0`, donc exactement le même code du Lab 1, ainsi que le même serveur MLflow. Chaque run y laisse une trace tagée `launched_by=kfp` ou `launched_by=airflow`, ce qui permet de comparer les deux dans la même expérience `scoring`.

## 2. Résultats quantitatifs

| Mesure | KFP | Airflow |
|---|---|---|
| Installation de zéro (min, estimation) | 17 | 5 |
| Temps jusqu'au premier run réussi (min, hors écriture du code) | 24 | 8 |
| Durée d'un run complet (min) | 6,6 (premier run, images à télécharger) | 3,1 |
| Relance avec les mêmes paramètres (min) | 3,8 (3 steps sur 6 servis par le cache) | 3,1 (pas de cache) |
| Hot-reload : modifier un step et voir le résultat (min, estimation) | 5 | 4 |
| Lignes de code propres à l'orchestrateur | 282 | 70 |
| Mémoire consommée par l'outil (Go, `docker stats`) | 4,8 (3 nœuds kind) | 1,8 (7 conteneurs) |

## 3. Analyse qualitative

### Developer experience

KFP impose un cycle plus long : écrire les composants, compiler le YAML, l'uploader dans l'interface, puis créer un run. En échange, il fournit des artefacts typés (Dataset, Model, Metrics) et un cache automatique : lors du second run, seuls train, evaluate et register ont été relancés, ce qui a ramené la durée de 6,6 à 3,8 minutes. Airflow est plus direct : le DAG est un simple fichier Python, repris par le scheduler en une trentaine de secondes. En revanche, aucun fichier n'y circule entre les tasks : nous avons dû passer par un dossier de travail sur S3 et de petites valeurs en XCom. Le DAG est aussi quatre fois plus court (70 lignes contre 282).

### Scaling

Dans les deux cas, un step correspond à un pod avec ses ressources déclarées (`set_memory_limit` pour KFP, `container_resources` pour Airflow), et le parallélisme est natif. KFP est beaucoup plus lourd : 14 pods permanents, environ 4,8 Go de mémoire et 17 minutes d'installation, contre environ 1,8 Go et 5 minutes pour Airflow en Docker Compose. Sur notre VM de 12 Go, nous ne pouvions pas faire tourner les deux en même temps sans que tout ralentisse.

### Debugging

Les logs sont consultables par step ou par task dans les deux interfaces. Côté KFP, l'UI a affiché un bandeau rouge MLMD alors que les runs réussissaient : il a fallu vérifier l'état réel avec `kubectl get workflows` et la liste des pods. Le lineage est en revanche très bon : le tag `kfp_run_id` relie un modèle MLflow au run KFP qui l'a produit, et le md5 des données correspond à celui de `dvc.lock`. Côté Airflow, l'état de chaque task se lit d'un coup d'œil dans la vue Graph.

## 4. Recommandation

Nous choisirions KFP pour un projet centré sur le ML : plusieurs modèles, des expériences fréquentes, un besoin de traçabilité des artefacts et de cache, et une équipe qui dispose déjà d'un cluster Kubernetes. Nous choisirions Airflow pour des pipelines de données planifiés, qui relient plusieurs systèmes, dans une équipe qui connaît déjà l'outil et veut une installation légère. Pour notre mini-projet, KFP est le meilleur choix, car le lineage et le cache y sont directement utiles, à condition de disposer de la mémoire nécessaire.

## 5. Limites de la comparaison

Cette comparaison repose sur un seul dataset de 1 000 lignes, un seul poste et un seul run chronométré par outil. Nous n'avons pas testé KFP sur un vrai cluster cloud, ni Airflow avec le KubernetesExecutor : il tournait en Docker Compose. La mémoire de KFP inclut aussi MLflow et SeaweedFS du cours, qui tournent dans les mêmes nœuds kind. Les temps d'installation et de hot-reload sont des estimations. Les durées dépendent de l'état de la VM et ne reflètent pas un usage en production.
