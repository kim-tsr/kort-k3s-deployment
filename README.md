# 🔗 Retex — Conteneurisation et déploiement d'un raccourcisseur d'URL

> **Retour d'expérience** sur la mise en conteneur puis le déploiement de l'application `kort`
> sur un cluster **K3s**. Ce document raconte le cheminement, les choix faits et les raisons
> derrière chaque décision, pas seulement le résultat final.

---

## 🗺️ Roadmap

- [x] **1. Conteneuriser l'application** — création des Dockerfiles et lancement en local, pour comprendre comment les composants interagissent entre eux.
- [x] **2. Premier déploiement basique** — manifests simples : `Deployment` / `StatefulSet` + `Service`.
- [x] **3. Élaboration d'un plan** — identifier les contraintes de chaque composant : nombre de replicas, avec ou sans état, besoins de connexion, tâches récurrentes (`cleaner`)…
- [x] **4. Mise en place des contraintes** — amélioration continue des conteneurs avec des mesures de consommation et ajout de probes adaptées : `liveness`, `readiness` et `startup`.
- [x] **5. Accès réseau externe** - mise en place de l'ingress et de la réécriture des routes
- [x] **6. Packaging de manifests** - Kustomize 
- [x] **7. GitOps** - Mise en place d'ArgoCD
---

## 🐳 1. Conteneuriser l'application

Cette étape s'est faite sur le dépôt où se trouve le code source :
👉 [kim-tsr/kort-URL-shorter](https://github.com/kim-tsr/kort-URL-shorter)
Les images Docker ont été poussées sur le registre public **Docker Hub**.

### Registry publique ou privée ?

J'ai hésité à héberger une registry privée et à pousser dessus, mais le choix du Docker Hub a
primé, pour deux raisons :

- je n'ai **aucune donnée confidentielle** et le code source est public ;
- le Docker Hub permet à **n'importe qui** de tester et de déployer l'application sur son propre
  cluster K3s.

### Construction des images

Les Dockerfiles sont en **multi-stage**, avec les layers les plus immuables en haut pour
optimiser la vitesse à chaque re-build (le cache Docker n'est invalidé qu'à partir de la couche
modifiée).

Autres choix :

- ajout d'un **utilisateur non-root** pour lancer l'application ;
- flag `--no-cache-dir` sur le `pip install` pour éviter le cache et obtenir une image plus légère.

Au final : **3 images basées sur `python:3.12-slim`** (api, worker, cleaner) et **une image
`nginx`** pour servir la page statique.

---

## ☸️ 2. Premier déploiement sur le cluster K3s

Création des premiers manifests, basiques : juste des `Deployment` et un `StatefulSet`, avec un
`Service` pour chacun.

Le but ici est **uniquement de faire tourner l'application** et d'avoir une version qui fonctionne
sur le cluster. Seul Postgres est déployé en `StatefulSet`. Pour l'instant, toutes les images
n'ont qu'**un seul replica**.

> [!NOTE]
> On ne cherche pas encore la résilience ni la performance : on cherche une base qui marche,
> sur laquelle itérer.

---

## 📊 3. Benchmark des besoins et amélioration continue des manifests

Une fois qu'on a des manifests qui fonctionnent, l'objectif devient d'avoir une application qui
fonctionne **bien**. Pour cela, il faut mieux comprendre les besoins métier de chaque composant.

Pour ça, je pars de ce tableau :

| Composant  | Type de charge             | Écoute     | Persistance  | Dépendances     |
|------------|----------------------------|------------|--------------|-----------------|
| `api`      | sans état, reçoit HTTP     | TCP 8000   | non          | postgres, redis |
| `web`      | statique, reçoit HTTP      | TCP 80     | non          | api (via `/api`)|
| `worker`   | sans état, ne reçoit rien  | aucun port | non          | postgres, redis |
| `postgres` | avec état                  | TCP 5432   | **oui**      | —               |
| `redis`    | cache / broker             | TCP 6379   | optionnelle  | —               |
| `cleaner`  | tâche ponctuelle           | aucun port | non          | postgres        |

*Tableau que l'on retrouve dans le `README.md` du code source.*

### ⏰ Le cas du `cleaner`

On passe le `cleaner` en **`CronJob`** : toutes les heures, un `Job` est créé, qui lance lui-même
un pod `cleaner` chargé de nettoyer les liens expirés.

### 🧪 Benchmark des ressources

Benchmark des ressources de chaque composant, en montant en charge et en observant le
comportement sous contrainte (*chaos engineering*). L'objectif : pouvoir fixer des `requests` et
des `limits` cohérentes pour chaque pod, plutôt que des valeurs choisies au hasard.

Je mesure avec `kubectl top pod -n kort --containers` et j'obtiens:

POD                       NAME       CPU(cores)   MEMORY(bytes)
api-868bb46955-77gjn      api        2m           50Mi
postgres-0                postgres   1m           57Mi
redis-7799d4bddd-qm8wq    redis      2m           11Mi
web-5547ff8855-lzslw      myapp      0m           4Mi
worker-6c65876cbd-4hq8d   worker     1m           24Mi

Puis je refais avec des testes de charge.

---

## 🔧 4. Mise à jour des manifests selon les contraintes trouvées

> 🚧 *Section en cours de rédaction — à compléter avec les valeurs de `requests`/`limits`
> retenues et les probes mises en place.*

- Ajout des 3 probes: startup, liveness et readiness.
- En fonction de la criticité des composants je choisis le QoS: Best Effort <
  Guaranteed < Burstable, en fais mes requests/limits de ressources en
  fonction.

  L'avantage des probes c'est eux qui vont determiner sur un pod est down, si
il faut le restart ou si il est en vie. Dans le cas de l'API, si ma DB postgres
est morte ça ne sert à rien de tuer mon pod même si sur /readyz ça renvoie une
erreur puisque Postgres est mort. Donc la readyness ne sera pas bonne mais la
liveness sera toujours bonne car le pod API est toujours en vie.

 Mise en place des 3 probes pour les composants où ça a du sens et d'un QoS
adapté.

## 5. Ingress et middleware

### Accès depuis l'exterieur

  Mise en place d'un IngressRoute plutot qu'un Ingress car IngressRoute permet
de matcher les regex.

  L'idée de base était de rediriger tout le flux /api -> api et / -> web. Sauf
que les redirections de se font via un slug, et ce slug doit être routé vers
l'api. Par exemple: http://kort.fr/uufxqf/ doit rediriger vers l'API, donc je
ne peux pas juste renvoyer /api -> api et / -> web. La question de rajouter
/api/uufxqf c'est posé mais le but est d'avoir des urls courtes, c'est la
valeur métier et l'infra ne doit pas la modifier.

Donc j'ai utilisé ingress route pour matcher `[a-z0-9]{6}`, les slugs ont
toujours 6 caractères alpha numérique.

### Réécriture de la route

J'ai utilisé un middleware pour réécrire les routes et enlever le /api. L'API
est en python avec le framework FastAPI et j'aurai pu rajouter le middleware de
coté mais le but ici est d'utiliser les composants K3s donc un middleware. Les
routes ne commencent pas par /api donc c'est juste un composant qui réécrit par
exmple /api/links -> /links


## 6. Packaging de manifests

  Split en 2 dossiers base + overlays avec un environnement de prod pour overide
les images plus facilement. Plus remplacement du ConfigMap par un
ConfigMapGenerator pour qu'a chaque modification des valeurs du ConfigMap ca en
recrée un nouveau avec un hash different ce qui force le Deployment a prendre
les nouvelles valeurs.


## 7. GitOps et ArgoCD
 
  On passe notre source de vérité à Git, **GitOps**, maintenant je push mes
manifests sur Git et mon infra les répliques sans que j'ai à apply à la main.
ArgoCD est sur mon cluster, il écoute mon repo et à chaque modification
l'applique.

  Permet les réconciliations en cas de drift, si par exemple je fais une
mauvais manip et fais: `kubectl scale deploy/api -n kort --replicas=10`,
ArgoCD va voir qu'il y a une différence.

### Secret

  Pour qu'ArgoCD puisse déployer l'infrastructure il faut qu'il est accès au
secret, mais pour cela il faut qu'ils soient dans le repository et ducoup tout
le monde y aurait accès.

  Pour répondre à cela on utilise du chiffrement asymétrique avec *sealed-secret*.
Une clé privé sur le cluster une clé publique chez moi, je chiffre avec la
clé publique les secrets et le cluster peut les déchiffrer avec la clé privé.