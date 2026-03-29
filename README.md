# Déploiement WordPress sur Kubernetes

Ce dépôt contient les manifestes Kubernetes pour déployer une stack WordPress complète avec MySQL et phpMyAdmin dans un namespace dédié (`wordpress-ns`).

## Architecture

```
                        ┌─────────────────┐
                        │   wordpress-ns  │
                        │                 │
  NodePort :30010 ──► │   wp-deploy     │ ──► pvc-wordpress (5Gi)
                        │   (3 replicas)  │
                        │        │        │
                        │        ▼        │
  ClusterIP ─────────► │   mysql-svc     │
                        │   StatefulSet   │ ──► pvc-mysql (10Gi)
                        │                 │
  LoadBalancer :30020 ►│   pma (Pod)     │
                        └─────────────────┘
```

## Contenu des fichiers

| Fichier | Description |
|--------|-------------|
| `0-wp-configmap.yml` | ConfigMap contenant la configuration MySQL, WordPress et phpMyAdmin |
| `1-wp-secret.yml` | Secret Kubernetes avec les mots de passe encodés en base64 |
| `2-all-pvc.yaml` | PersistentVolumeClaims : 10Gi pour MySQL, 5Gi pour WordPress |
| `3-mysql-db.yml` | StatefulSet MySQL 5.7 avec volume persistant |
| `4-wp-deploy.yml` | Deployment WordPress avec 3 réplicas |
| `5-pma-pod.yaml` | Pod phpMyAdmin pour administrer la base de données |
| `6-all-svc.yml` | Services : NodePort (WordPress), ClusterIP (MySQL), LoadBalancer (phpMyAdmin) |

## Prérequis

- Un cluster Kubernetes opérationnel
- `kubectl` configuré et connecté au cluster

## Déploiement

Appliquer les manifestes dans l'ordre :

```bash
kubectl create namespace wordpress-ns
kubectl apply -f 0-wp-configmap.yml
kubectl apply -f 1-wp-secret.yml
kubectl apply -f 2-all-pvc.yaml
kubectl apply -f 3-mysql-db.yml
kubectl apply -f 4-wp-deploy.yml
kubectl apply -f 5-pma-pod.yaml
kubectl apply -f 6-all-svc.yml
```

Ou en une seule commande :

```bash
kubectl create namespace wordpress-ns
kubectl apply -f .
```

## Accès aux applications

Le cluster étant un **Minikube**, utiliser les commandes suivantes :

**WordPress** (NodePort) :
```bash
minikube service wp-deploy -n wordpress-ns
# ou directement l'URL :
minikube service wp-deploy -n wordpress-ns --url
```

**phpMyAdmin** (LoadBalancer) :
```bash
minikube service pma-svc -n wordpress-ns
# ou directement l'URL :
minikube service pma-svc -n wordpress-ns --url
```

> Sur Minikube, les services LoadBalancer restent en `<pending>` pour l'EXTERNAL-IP.
> La commande `minikube service` crée un tunnel et ouvre automatiquement le navigateur.

| Application | Type de service | NodePort |
|------------|----------------|----------|
| WordPress | NodePort | 30010 |
| phpMyAdmin | LoadBalancer | 30020 |

## Choix `env` vs `envFrom` pour phpMyAdmin

Le pod phpMyAdmin utilise `env` avec des références ciblées (`valueFrom`) plutôt que `envFrom`.

**Pourquoi ?**

`envFrom` injecte **toutes** les variables d'un ConfigMap ou d'un Secret dans le conteneur. Pour phpMyAdmin, cela signifierait injecter des variables inutiles comme `MYSQL_DATABASE`, `MYSQL_USER`, `WORDPRESS_DB_HOST`, etc. qui ne le concernent pas.

Avec `env` + `valueFrom`, on injecte **uniquement** les variables nécessaires :
- `PMA_HOST` — adresse du serveur MySQL
- `PMA_PORT` — port MySQL
- `MYSQL_ROOT_PASSWORD` — mot de passe root pour la connexion

C'est une bonne pratique : **principe du moindre privilège** appliqué à la configuration.

| Approche | Variables injectées | Recommandée pour |
|---------|-------------------|-----------------|
| `envFrom` | Toutes (ConfigMap ou Secret entier) | Quand toutes les variables sont pertinentes |
| `env` + `valueFrom` | Seulement celles déclarées | Quand on veut cibler des variables précises |

## Vérification

```bash
kubectl get all -n wordpress-ns
kubectl get pvc -n wordpress-ns
```

## Suppression

```bash
kubectl delete namespace wordpress-ns
```

> **Note** : Ce dépôt est à des fins pédagogiques. Les secrets sont versionnés intentionnellement dans le cadre du cours.
