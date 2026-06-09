# Guide complet — APIs Apicurio Registry 3.2.x

Ce guide couvre **toutes les familles d'endpoints** exposées par Apicurio Registry v3 :

- **Core REST API v3** — base `/apis/registry/v3`
- **Confluent Schema Registry compat (ccompat) v7** — base `/apis/ccompat/v7`
- **CNCF Schema Registry compat v0** — base `/apis/cncf/v0`
- **Endpoints système** — `/apis/registry/v3/system|users`
- **Endpoints d'administration** — `/apis/registry/v3/admin`
- **Endpoints de management (santé, métriques)** — port `9000`, `/q/health`, `/q/metrics`

Chaque endpoint est accompagné d'un exemple `curl` directement utilisable. On suppose que le déploiement [`k8s/apicurio.yaml`](../k8s/apicurio.yaml) est appliqué et que l'API est exposée :

```bash
# Via Ingress
export REG=http://apicurio.local

# OU via port-forward
kubectl -n apicurio port-forward svc/apicurio-registry 8080:8080 9000:9000
export REG=http://localhost:8080
export MGMT=http://localhost:9000
```

> **Auth** : le manifeste fourni laisse OIDC désactivé (défaut). Tous les exemples fonctionnent sans token. Si vous activez OIDC, ajoutez `-H "Authorization: Bearer $TOKEN"` à chaque requête.

---

## Table des matières

1. [Endpoints système](#1-endpoints-système)
2. [Endpoints de management (port 9000)](#2-endpoints-de-management-port-9000)
3. [Groupes](#3-groupes)
4. [Artefacts](#4-artefacts)
5. [Versions](#5-versions)
6. [Contenu et références](#6-contenu-et-références)
7. [Commentaires](#7-commentaires)
8. [Branches](#8-branches)
9. [Règles (artefact / groupe / global)](#9-règles)
10. [Recherche](#10-recherche)
11. [Lookups par ID](#11-lookups-par-id)
12. [Administration](#12-administration)
13. [Utilisateurs](#13-utilisateurs)
14. [API Confluent compatibility v7](#14-api-confluent-compatibility-v7)
15. [API CNCF compatibility v0](#15-api-cncf-compatibility-v0)
16. [Types d'artefacts supportés](#16-types-dartefacts-supportés)
17. [Scénario de test end-to-end](#17-scénario-de-test-end-to-end)

---

## 1. Endpoints système

| Méthode | Chemin                          | Usage                                                        |
| ------- | ------------------------------- | ------------------------------------------------------------ |
| GET     | `/apis/registry/v3/system/info` | Nom, version build, date de build                            |
| GET     | `/apis/registry/v3/system/uiConfig` | Configuration que la SPA UI charge au démarrage         |

```bash
curl -s "$REG/apis/registry/v3/system/info" | jq
# {
#   "name": "Apicurio Registry (SQL)",
#   "version": "3.2.5",
#   "builtOn": "2026-06-03T..."
# }

curl -s "$REG/apis/registry/v3/system/uiConfig" | jq
```

---

## 2. Endpoints de management (port 9000)

Depuis **Apicurio 3.2.0**, les endpoints de santé et métriques ne sont plus sur 8080 mais sur **9000** (port management séparé). Les probes Kubernetes du manifeste fourni utilisent ce port.

> **⚠️ Particularité Apicurio** : contrairement à un Quarkus standard où ces endpoints sont sous `/q/health` et `/q/metrics`, Apicurio les **remappe à la racine** via `quarkus.smallrye-health.root-path=/health` et `quarkus.micrometer.export.prometheus.path=/metrics`. Si vous suivez la doc Quarkus générique, vous obtiendrez systématiquement `<html><body><h1>Resource not found</h1></body></html>`.

| Méthode | Chemin (port 9000) | Usage                                                        |
| ------- | ------------------- | ------------------------------------------------------------ |
| GET     | `/health`           | Statut agrégé (DB + storage + readiness + liveness)          |
| GET     | `/health/live`      | Liveness probe                                               |
| GET     | `/health/ready`     | Readiness probe                                              |
| GET     | `/health/started`   | Startup probe                                                |
| GET     | `/metrics`          | Métriques Prometheus (HTTP, JVM, datasource, etc.)           |

```bash
curl -s "$MGMT/health" | jq
curl -s "$MGMT/health/ready" | jq
curl -s "$MGMT/health/live" | jq
curl -s "$MGMT/metrics" | head -20
```

Exemple de retour `/health/ready` sur un déploiement Postgres-backed :

```json
{
  "status": "UP",
  "checks": [
    { "name": "ResponseTimeoutReadinessCheck",    "status": "UP" },
    { "name": "ImportLifecycleReadinessCheck",    "status": "UP" },
    { "name": "PersistenceSimpleReadinessCheck",  "status": "UP" },
    { "name": "Database connections health check","status": "UP",
      "data": { "postgresql": "UP" } },
    { "name": "ElasticsearchIndexReadinessCheck", "status": "UP" }
  ]
}
```

> **Note pour la branche 2.x in-memory** : les paths sont les mêmes (`/health/*`, `/metrics`) MAIS exposés sur le **port 8080** — pas de port 9000 séparé avant la v3.2.

---

## 3. Groupes

Un **groupe** est un container logique d'artefacts (équivalent d'un namespace de schémas).

| Méthode | Chemin                              | Usage                                                                  |
| ------- | ----------------------------------- | ---------------------------------------------------------------------- |
| GET     | `/apis/registry/v3/groups`          | Liste les groupes (params : `limit`, `offset`, `order`, `orderby`)      |
| POST    | `/apis/registry/v3/groups`          | Crée un groupe                                                          |
| GET     | `/apis/registry/v3/groups/{groupId}`| Métadonnées du groupe                                                  |
| PUT     | `/apis/registry/v3/groups/{groupId}`| Met à jour `description` et `labels`                                    |
| DELETE  | `/apis/registry/v3/groups/{groupId}`| Supprime le groupe (uniquement si suppression de groupe activée)        |

```bash
# Lister
curl -s "$REG/apis/registry/v3/groups?limit=20&order=asc&orderby=createdOn" | jq

# Créer
curl -s -X POST "$REG/apis/registry/v3/groups" \
  -H "Content-Type: application/json" \
  -d '{
    "groupId": "orders",
    "description": "Schemas du domaine Orders",
    "labels": { "team": "platform", "env": "dev" }
  }' | jq

# Lire
curl -s "$REG/apis/registry/v3/groups/orders" | jq

# Mettre à jour
curl -s -X PUT "$REG/apis/registry/v3/groups/orders" \
  -H "Content-Type: application/json" \
  -d '{"description":"Domaine Orders v2","labels":{"team":"platform"}}'

# Supprimer
curl -s -X DELETE "$REG/apis/registry/v3/groups/orders"
```

---

## 4. Artefacts

Un **artefact** est un schéma/contrat (Avro, JSON Schema, OpenAPI, etc.) identifié par `(groupId, artifactId)`.

| Méthode | Chemin                                                          | Usage                                                                 |
| ------- | --------------------------------------------------------------- | --------------------------------------------------------------------- |
| GET     | `/apis/registry/v3/groups/{groupId}/artifacts`                  | Liste des artefacts du groupe                                          |
| POST    | `/apis/registry/v3/groups/{groupId}/artifacts`                  | Crée un artefact (avec `firstVersion` optionnel)                       |
| DELETE  | `/apis/registry/v3/groups/{groupId}/artifacts`                  | Supprime **tous** les artefacts du groupe                              |
| GET     | `/apis/registry/v3/groups/{groupId}/artifacts/{artifactId}`     | Métadonnées de l'artefact                                              |
| PUT     | `/apis/registry/v3/groups/{groupId}/artifacts/{artifactId}`     | Met à jour `name`, `description`, `labels`                             |
| DELETE  | `/apis/registry/v3/groups/{groupId}/artifacts/{artifactId}`     | Supprime l'artefact                                                    |

**Query params clés** pour `POST /artifacts` :

| Param      | Valeurs                                                  | Effet                                                                                 |
| ---------- | -------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| `ifExists` | `FAIL` (défaut), `CREATE_VERSION`, `FIND_OR_CREATE_VERSION` | Politique si l'artefact existe déjà                                                |
| `canonical`| `true`/`false`                                            | Si combiné avec `FIND_OR_CREATE_VERSION`, compare par hash canonique du contenu       |
| `dryRun`   | `true`/`false`                                            | Valide la requête sans persister                                                      |

```bash
# Créer un artefact AVRO + sa première version
curl -s -X POST "$REG/apis/registry/v3/groups/orders/artifacts?ifExists=CREATE_VERSION" \
  -H "Content-Type: application/json" \
  -d '{
    "artifactId": "Order",
    "artifactType": "AVRO",
    "name": "Order",
    "description": "Schema Avro pour un ordre client",
    "labels": { "owner": "data-platform" },
    "firstVersion": {
      "version": "1.0.0",
      "content": {
        "content": "{\"type\":\"record\",\"name\":\"Order\",\"namespace\":\"com.example.orders\",\"fields\":[{\"name\":\"order_id\",\"type\":\"string\"},{\"name\":\"amount\",\"type\":\"long\"}]}",
        "contentType": "application/json"
      }
    }
  }' | jq

# Lister
curl -s "$REG/apis/registry/v3/groups/orders/artifacts" | jq

# Lire métadonnées
curl -s "$REG/apis/registry/v3/groups/orders/artifacts/Order" | jq

# Mettre à jour
curl -s -X PUT "$REG/apis/registry/v3/groups/orders/artifacts/Order" \
  -H "Content-Type: application/json" \
  -d '{"name":"Order","description":"Updated","labels":{"owner":"data","status":"reviewed"}}'

# Test dry-run d'une création
curl -s -X POST "$REG/apis/registry/v3/groups/orders/artifacts?dryRun=true" \
  -H "Content-Type: application/json" \
  -d '{"artifactId":"OrderDraft","artifactType":"AVRO","firstVersion":{"content":{"content":"{\"type\":\"string\"}","contentType":"application/json"}}}'

# Supprimer
curl -s -X DELETE "$REG/apis/registry/v3/groups/orders/artifacts/Order"
```

---

## 5. Versions

Une **version** porte le contenu et son état (`ENABLED`, `DISABLED`, `DEPRECATED`, `DRAFT`). **En v3, seules les versions ont un état** — l'artefact n'en a plus.

`{versionExpression}` accepte :
- un literal : `1.0.0`
- un ordinal entier : `1`, `2`, `3`
- l'expression branche : `branch=latest`, `branch=drafts`

| Méthode | Chemin                                                                                       | Usage                                                  |
| ------- | -------------------------------------------------------------------------------------------- | ------------------------------------------------------ |
| GET     | `/apis/registry/v3/groups/{groupId}/artifacts/{artifactId}/versions`                         | Liste des versions                                     |
| POST    | `/apis/registry/v3/groups/{groupId}/artifacts/{artifactId}/versions`                         | Crée une nouvelle version                              |
| GET     | `/apis/registry/v3/groups/{groupId}/artifacts/{artifactId}/versions/{ver}`                   | Métadonnées de la version                              |
| PUT     | `/apis/registry/v3/groups/{groupId}/artifacts/{artifactId}/versions/{ver}`                   | Met à jour métadonnées **et état** (`ENABLED`, ...)    |
| DELETE  | `/apis/registry/v3/groups/{groupId}/artifacts/{artifactId}/versions/{ver}`                   | Supprime la version (si suppression activée)           |

```bash
# Créer une nouvelle version (1.0.1)
curl -s -X POST "$REG/apis/registry/v3/groups/orders/artifacts/Order/versions" \
  -H "Content-Type: application/json" \
  -d '{
    "version": "1.0.1",
    "content": {
      "content": "{\"type\":\"record\",\"name\":\"Order\",\"namespace\":\"com.example.orders\",\"fields\":[{\"name\":\"order_id\",\"type\":\"string\"},{\"name\":\"amount\",\"type\":\"long\"},{\"name\":\"currency\",\"type\":\"string\",\"default\":\"EUR\"}]}",
      "contentType": "application/json"
    }
  }' | jq

# Lister
curl -s "$REG/apis/registry/v3/groups/orders/artifacts/Order/versions" | jq

# Métadonnées d'une version par branche
curl -s "$REG/apis/registry/v3/groups/orders/artifacts/Order/versions/branch=latest" | jq

# Désactiver une version (état)
curl -s -X PUT "$REG/apis/registry/v3/groups/orders/artifacts/Order/versions/1.0.0" \
  -H "Content-Type: application/json" \
  -d '{"state":"DISABLED"}'

# Marquer une version comme dépréciée
curl -s -X PUT "$REG/apis/registry/v3/groups/orders/artifacts/Order/versions/1.0.0" \
  -H "Content-Type: application/json" \
  -d '{"state":"DEPRECATED","description":"Use 1.0.1+"}'
```

---

## 6. Contenu et références

| Méthode | Chemin                                                                                              | Usage                                       |
| ------- | --------------------------------------------------------------------------------------------------- | ------------------------------------------- |
| GET     | `/apis/registry/v3/groups/{groupId}/artifacts/{artifactId}/versions/{ver}/content`                  | Récupère le contenu brut                    |
| PUT     | `/apis/registry/v3/groups/{groupId}/artifacts/{artifactId}/versions/{ver}/content`                  | Remplace le contenu                         |
| GET     | `/apis/registry/v3/groups/{groupId}/artifacts/{artifactId}/versions/{ver}/references`               | Liste des références sortantes              |
| GET     | `/apis/registry/v3/groups/{groupId}/artifacts/{artifactId}/versions/{ver}/references/graph`         | Graphe complet des références               |

**Query param `references`** pour `GET .../content` :

- `PRESERVE` (défaut) — contenu tel que stocké
- `DEREFERENCE` — inline récursivement tous les `$ref`/imports (Avro, JSON Schema, OpenAPI, Protobuf, AsyncAPI)
- `REWRITE` — réécrit les URLs de référence pour pointer vers le registry

```bash
# Récupérer le schéma tel quel
curl -s "$REG/apis/registry/v3/groups/orders/artifacts/Order/versions/branch=latest/content"

# Dans le cas d'un schéma OpenAPI/JSON avec $ref : tout inliner
curl -s "$REG/apis/registry/v3/groups/orders/artifacts/Order/versions/1.0.1/content?references=DEREFERENCE"

# Voir les références sortantes
curl -s "$REG/apis/registry/v3/groups/orders/artifacts/Order/versions/1.0.1/references" | jq
```

---

## 7. Commentaires

Chaque version peut porter des commentaires (utiles en code-review de schémas).

| Méthode | Chemin                                                                                                  | Usage                |
| ------- | ------------------------------------------------------------------------------------------------------- | -------------------- |
| GET     | `/apis/registry/v3/groups/{groupId}/artifacts/{artifactId}/versions/{ver}/comments`                     | Liste                |
| POST    | `/apis/registry/v3/groups/{groupId}/artifacts/{artifactId}/versions/{ver}/comments`                     | Ajoute               |
| PUT     | `/apis/registry/v3/groups/{groupId}/artifacts/{artifactId}/versions/{ver}/comments/{commentId}`         | Modifie              |
| DELETE  | `/apis/registry/v3/groups/{groupId}/artifacts/{artifactId}/versions/{ver}/comments/{commentId}`         | Supprime             |

```bash
curl -s -X POST "$REG/apis/registry/v3/groups/orders/artifacts/Order/versions/1.0.1/comments" \
  -H "Content-Type: application/json" \
  -d '{"value":"Ajout du champ currency avec default EUR — compatible BACKWARD."}'

curl -s "$REG/apis/registry/v3/groups/orders/artifacts/Order/versions/1.0.1/comments" | jq
```

---

## 8. Branches

**Nouveau en v3.** Permet de grouper des versions par flux (latest, drafts, release-2024, etc.).

Branches built-in : `latest` (tip auto-géré), `drafts` (zone de travail mutable).

| Méthode | Chemin                                                                                                  | Usage                            |
| ------- | ------------------------------------------------------------------------------------------------------- | -------------------------------- |
| GET     | `/apis/registry/v3/groups/{groupId}/artifacts/{artifactId}/branches`                                    | Liste                            |
| POST    | `/apis/registry/v3/groups/{groupId}/artifacts/{artifactId}/branches`                                    | Crée                             |
| GET     | `/apis/registry/v3/groups/{groupId}/artifacts/{artifactId}/branches/{branchId}`                         | Lit                              |
| PUT     | `/apis/registry/v3/groups/{groupId}/artifacts/{artifactId}/branches/{branchId}`                         | Modifie                          |
| DELETE  | `/apis/registry/v3/groups/{groupId}/artifacts/{artifactId}/branches/{branchId}`                         | Supprime                         |
| GET     | `/apis/registry/v3/groups/{groupId}/artifacts/{artifactId}/branches/{branchId}/versions`                | Versions dans la branche         |
| POST    | `/apis/registry/v3/groups/{groupId}/artifacts/{artifactId}/branches/{branchId}/versions`                | Ajoute une version au tip        |

```bash
# Créer une branche release-2026
curl -s -X POST "$REG/apis/registry/v3/groups/orders/artifacts/Order/branches" \
  -H "Content-Type: application/json" \
  -d '{"branchId":"release-2026","description":"Versions livrées en 2026","versions":["1.0.0","1.0.1"]}'

# Lister les versions de cette branche
curl -s "$REG/apis/registry/v3/groups/orders/artifacts/Order/branches/release-2026/versions" | jq
```

---

## 9. Règles

Trois familles de règles, qui se cumulent selon une **priorité artefact > groupe > global**.

**Types** : `VALIDITY`, `COMPATIBILITY`, `INTEGRITY`.

**Configs possibles** :
- `VALIDITY` : `NONE`, `SYNTAX_ONLY`, `FULL`
- `COMPATIBILITY` : `NONE`, `BACKWARD`, `BACKWARD_TRANSITIVE`, `FORWARD`, `FORWARD_TRANSITIVE`, `FULL`, `FULL_TRANSITIVE`
- `INTEGRITY` : liste virgule-séparée de `NO_DUPLICATES`, `REFS_EXIST`, `ALL_REFS_MAPPED`, `NONE`, `FULL`

### 9.1 Règles globales

| Méthode | Chemin                                          | Usage                          |
| ------- | ----------------------------------------------- | ------------------------------ |
| GET     | `/apis/registry/v3/admin/rules`                 | Liste                          |
| POST    | `/apis/registry/v3/admin/rules`                 | Crée                           |
| DELETE  | `/apis/registry/v3/admin/rules`                 | Supprime toutes                |
| GET     | `/apis/registry/v3/admin/rules/{ruleType}`      | Lit                            |
| PUT     | `/apis/registry/v3/admin/rules/{ruleType}`      | Met à jour la config           |
| DELETE  | `/apis/registry/v3/admin/rules/{ruleType}`      | Supprime                       |

```bash
# Activer la compatibilité BACKWARD globale
curl -s -X POST "$REG/apis/registry/v3/admin/rules" \
  -H "Content-Type: application/json" \
  -d '{"ruleType":"COMPATIBILITY","config":"BACKWARD"}'

# Activer la validité syntaxique
curl -s -X POST "$REG/apis/registry/v3/admin/rules" \
  -H "Content-Type: application/json" \
  -d '{"ruleType":"VALIDITY","config":"SYNTAX_ONLY"}'

# Lister
curl -s "$REG/apis/registry/v3/admin/rules" | jq

# Renforcer en FULL
curl -s -X PUT "$REG/apis/registry/v3/admin/rules/COMPATIBILITY" \
  -H "Content-Type: application/json" \
  -d '{"ruleType":"COMPATIBILITY","config":"FULL"}'
```

### 9.2 Règles de groupe (nouveau en v3)

| Méthode | Chemin                                                       | Usage           |
| ------- | ------------------------------------------------------------ | --------------- |
| GET     | `/apis/registry/v3/groups/{groupId}/rules`                   | Liste           |
| POST    | `/apis/registry/v3/groups/{groupId}/rules`                   | Crée            |
| DELETE  | `/apis/registry/v3/groups/{groupId}/rules`                   | Supprime toutes |
| GET     | `/apis/registry/v3/groups/{groupId}/rules/{ruleType}`        | Lit             |
| PUT     | `/apis/registry/v3/groups/{groupId}/rules/{ruleType}`        | Met à jour      |
| DELETE  | `/apis/registry/v3/groups/{groupId}/rules/{ruleType}`        | Supprime        |

```bash
curl -s -X POST "$REG/apis/registry/v3/groups/orders/rules" \
  -H "Content-Type: application/json" \
  -d '{"ruleType":"INTEGRITY","config":"REFS_EXIST,NO_DUPLICATES"}'
```

### 9.3 Règles d'artefact

| Méthode | Chemin                                                                              | Usage           |
| ------- | ----------------------------------------------------------------------------------- | --------------- |
| GET     | `/apis/registry/v3/groups/{groupId}/artifacts/{artifactId}/rules`                   | Liste           |
| POST    | `/apis/registry/v3/groups/{groupId}/artifacts/{artifactId}/rules`                   | Crée            |
| DELETE  | `/apis/registry/v3/groups/{groupId}/artifacts/{artifactId}/rules`                   | Supprime toutes |
| GET     | `/apis/registry/v3/groups/{groupId}/artifacts/{artifactId}/rules/{ruleType}`        | Lit             |
| PUT     | `/apis/registry/v3/groups/{groupId}/artifacts/{artifactId}/rules/{ruleType}`        | Met à jour      |
| DELETE  | `/apis/registry/v3/groups/{groupId}/artifacts/{artifactId}/rules/{ruleType}`        | Supprime        |

```bash
# Forcer la compatibilité FORWARD_TRANSITIVE sur l'artefact Order
curl -s -X POST "$REG/apis/registry/v3/groups/orders/artifacts/Order/rules" \
  -H "Content-Type: application/json" \
  -d '{"ruleType":"COMPATIBILITY","config":"FORWARD_TRANSITIVE"}'

# Tester : poster une version cassante doit retourner 409
curl -i -X POST "$REG/apis/registry/v3/groups/orders/artifacts/Order/versions" \
  -H "Content-Type: application/json" \
  -d '{"content":{"content":"{\"type\":\"string\"}","contentType":"application/json"}}'
# => HTTP/1.1 409 Conflict
# {"error_code":409,"message":"... rule violation: COMPATIBILITY ..."}
```

---

## 10. Recherche

| Méthode | Chemin                                  | Usage                                                                                                          |
| ------- | --------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| GET     | `/apis/registry/v3/search/artifacts`    | Recherche d'artefacts par `name`, `description`, `labels`, `groupId`, `artifactType`, `globalId`, `contentId`  |
| POST    | `/apis/registry/v3/search/artifacts`    | Recherche **par contenu** (corps = contenu brut ; trouve les artefacts à hash canonique identique)             |
| GET     | `/apis/registry/v3/search/groups`       | Recherche de groupes                                                                                            |
| GET     | `/apis/registry/v3/search/versions`     | Recherche de versions                                                                                           |
| POST    | `/apis/registry/v3/search/versions`     | Recherche de versions par contenu                                                                               |

**Query params** communs : `limit`, `offset`, `order` (`asc`/`desc`), `orderby` (`name`, `createdOn`, `modifiedOn`, ...).

```bash
# Tous les artefacts AVRO du groupe orders
curl -s "$REG/apis/registry/v3/search/artifacts?groupId=orders&artifactType=AVRO&limit=50" | jq

# Recherche par label
curl -s "$REG/apis/registry/v3/search/artifacts?labels=owner:data-platform" | jq

# Recherche d'un artefact contenant exactement ce contenu
curl -s -X POST "$REG/apis/registry/v3/search/artifacts" \
  -H "Content-Type: application/json" \
  -d '{"type":"record","name":"Order","fields":[{"name":"order_id","type":"string"}]}' | jq
```

---

## 11. Lookups par ID

| Méthode | Chemin                                                       | Usage                                                              |
| ------- | ------------------------------------------------------------ | ------------------------------------------------------------------ |
| GET     | `/apis/registry/v3/ids/globalIds/{globalId}`                 | Contenu d'une version par son globalId                             |
| GET     | `/apis/registry/v3/ids/globalIds/{globalId}/references`      | Références sortantes par globalId                                  |
| GET     | `/apis/registry/v3/ids/contentIds/{contentId}`               | Contenu par contentId                                              |
| GET     | `/apis/registry/v3/ids/contentIds/{contentId}/references`    | Références par contentId                                           |
| GET     | `/apis/registry/v3/ids/contentHashes/{contentHash}`          | Contenu par SHA-256 canonique                                      |
| GET     | `/apis/registry/v3/ids/contentHashes/{contentHash}/references`| Références par hash                                                |

```bash
curl -s "$REG/apis/registry/v3/ids/globalIds/1" | jq
```

---

## 12. Administration

### 12.1 Export / Import (sauvegarde complète)

```bash
# Export complet (ZIP)
curl -s "$REG/apis/registry/v3/admin/export" --output apicurio-backup.zip

# Restore
curl -s -X POST "$REG/apis/registry/v3/admin/import" \
  -H "Content-Type: application/zip" \
  --data-binary @apicurio-backup.zip
```

### 12.2 Role mappings (quand RBAC activé avec source `application`)

```bash
# Créer un mapping
curl -s -X POST "$REG/apis/registry/v3/admin/roleMappings" \
  -H "Content-Type: application/json" \
  -d '{"principalId":"alice@example.com","role":"ADMIN"}'

# Lister
curl -s "$REG/apis/registry/v3/admin/roleMappings" | jq

# Modifier
curl -s -X PUT "$REG/apis/registry/v3/admin/roleMappings/alice@example.com" \
  -H "Content-Type: application/json" \
  -d '{"role":"DEVELOPER"}'

# Supprimer
curl -s -X DELETE "$REG/apis/registry/v3/admin/roleMappings/alice@example.com"
```

### 12.3 Propriétés dynamiques (modifiables à chaud)

```bash
curl -s "$REG/apis/registry/v3/admin/config/properties" | jq
curl -s "$REG/apis/registry/v3/admin/config/properties/apicurio.rest.deletion.artifact.enabled" | jq

# Activer la suppression d'artefact à chaud
curl -s -X PUT "$REG/apis/registry/v3/admin/config/properties/apicurio.rest.deletion.artifact.enabled" \
  -H "Content-Type: application/json" \
  -d '{"value":"true"}'
```

### 12.4 Types d'artefacts supportés

```bash
curl -s "$REG/apis/registry/v3/admin/config/artifactTypes" | jq
# [ "AVRO", "PROTOBUF", "JSON", "OPENAPI", "ASYNCAPI", "GRAPHQL",
#   "KCONNECT", "WSDL", "XSD", "XML", "OPENRPC",
#   "MODEL_SCHEMA", "PROMPT_TEMPLATE" ]
```

### 12.5 Snapshots (uniquement avec storage `kafkasql`)

```bash
curl -s -X POST "$REG/apis/registry/v3/admin/snapshots"
```

---

## 13. Utilisateurs

| Méthode | Chemin                       | Usage                                                          |
| ------- | ---------------------------- | -------------------------------------------------------------- |
| GET     | `/apis/registry/v3/users/me` | Identité du principal connecté + rôle (ADMIN/DEVELOPER/READ_ONLY) |

```bash
curl -s "$REG/apis/registry/v3/users/me" | jq
```

(Sans OIDC, renvoie un principal vide.)

---

## 14. API Confluent compatibility v7

Permet d'utiliser Apicurio comme **drop-in replacement de Confluent Schema Registry**. Base : `/apis/ccompat/v7`.

> **Note** : par défaut, les schémas vont dans le groupe `default` Apicurio. Pour cibler un autre groupe, ajoutez le header `X-Registry-GroupId: <groupName>` à chaque requête.

### 14.1 Subjects

| Méthode | Chemin                                                                 | Usage                                                                 |
| ------- | ---------------------------------------------------------------------- | --------------------------------------------------------------------- |
| GET     | `/apis/ccompat/v7/subjects`                                            | Liste les subjects                                                    |
| POST    | `/apis/ccompat/v7/subjects/{subject}`                                  | Lookup d'un schéma par contenu (renvoie l'ID existant ou 404)         |
| POST    | `/apis/ccompat/v7/subjects/{subject}/versions`                         | Enregistre une nouvelle version                                       |
| GET     | `/apis/ccompat/v7/subjects/{subject}/versions`                         | Liste des versions                                                    |
| GET     | `/apis/ccompat/v7/subjects/{subject}/versions/{version}`               | Version spécifique (`latest` accepté)                                 |
| GET     | `/apis/ccompat/v7/subjects/{subject}/versions/{version}/schema`        | Contenu brut                                                          |
| GET     | `/apis/ccompat/v7/subjects/{subject}/versions/{version}/referencedby`  | Subjects qui référencent ce schéma                                    |
| DELETE  | `/apis/ccompat/v7/subjects/{subject}`                                  | Supprime le subject (`?permanent=true` pour suppression dure)         |
| DELETE  | `/apis/ccompat/v7/subjects/{subject}/versions/{version}`               | Supprime une version                                                  |

```bash
# Enregistrer un schéma Avro
curl -s -X POST "$REG/apis/ccompat/v7/subjects/orders-value/versions" \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -d '{
    "schemaType": "AVRO",
    "schema": "{\"type\":\"record\",\"name\":\"Order\",\"namespace\":\"com.example.orders\",\"fields\":[{\"name\":\"order_id\",\"type\":\"string\"},{\"name\":\"amount\",\"type\":\"long\"}]}"
  }' | jq
# { "id": 1 }

# Récupérer la dernière version
curl -s "$REG/apis/ccompat/v7/subjects/orders-value/versions/latest" | jq

# Récupérer juste le contenu
curl -s "$REG/apis/ccompat/v7/subjects/orders-value/versions/latest/schema"

# Lister tous les subjects
curl -s "$REG/apis/ccompat/v7/subjects" | jq

# Lookup par contenu
curl -s -X POST "$REG/apis/ccompat/v7/subjects/orders-value" \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -d '{"schema":"{\"type\":\"record\",\"name\":\"Order\",\"namespace\":\"com.example.orders\",\"fields\":[{\"name\":\"order_id\",\"type\":\"string\"},{\"name\":\"amount\",\"type\":\"long\"}]}"}'
```

### 14.2 Schemas (par ID Confluent)

| Méthode | Chemin                                              | Usage                                              |
| ------- | --------------------------------------------------- | -------------------------------------------------- |
| GET     | `/apis/ccompat/v7/schemas`                          | Liste tous les schémas                             |
| GET     | `/apis/ccompat/v7/schemas/ids/{id}`                 | Schéma par ID                                      |
| GET     | `/apis/ccompat/v7/schemas/ids/{id}/schema`          | Contenu brut par ID                                |
| GET     | `/apis/ccompat/v7/schemas/ids/{id}/subjects`        | Subjects utilisant ce schéma                       |
| GET     | `/apis/ccompat/v7/schemas/ids/{id}/versions`        | Paires `{subject, version}` pour ce schéma         |
| GET     | `/apis/ccompat/v7/schemas/types`                    | Types supportés — renvoie `["AVRO","JSON","PROTOBUF"]` |

```bash
curl -s "$REG/apis/ccompat/v7/schemas/ids/1" | jq
curl -s "$REG/apis/ccompat/v7/schemas/types"
```

### 14.3 Compatibility check

| Méthode | Chemin                                                                          | Usage                                                                              |
| ------- | ------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| POST    | `/apis/ccompat/v7/compatibility/subjects/{subject}/versions/{version}`          | Test contre une version (`latest` accepté). Défaut `BACKWARD` si aucune règle.     |
| POST    | `/apis/ccompat/v7/compatibility/subjects/{subject}/versions`                    | Test contre toutes les versions                                                    |

```bash
# Vérifier si un nouveau schéma est compatible
curl -s -X POST "$REG/apis/ccompat/v7/compatibility/subjects/orders-value/versions/latest" \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -d '{
    "schema": "{\"type\":\"record\",\"name\":\"Order\",\"namespace\":\"com.example.orders\",\"fields\":[{\"name\":\"order_id\",\"type\":\"string\"},{\"name\":\"amount\",\"type\":\"long\"},{\"name\":\"currency\",\"type\":\"string\",\"default\":\"EUR\"}]}"
  }' | jq
# { "is_compatible": true }
```

### 14.4 Config (compatibilité globale / par subject)

| Méthode | Chemin                                | Usage                                                  |
| ------- | ------------------------------------- | ------------------------------------------------------ |
| GET     | `/apis/ccompat/v7/config`             | Config globale                                         |
| PUT     | `/apis/ccompat/v7/config`             | Définir la compatibilité globale                       |
| DELETE  | `/apis/ccompat/v7/config`             | Supprimer la config globale                            |
| GET     | `/apis/ccompat/v7/config/{subject}`   | Config par subject                                     |
| PUT     | `/apis/ccompat/v7/config/{subject}`   | Définir compat par subject                             |
| DELETE  | `/apis/ccompat/v7/config/{subject}`   | Supprimer                                              |

```bash
curl -s -X PUT "$REG/apis/ccompat/v7/config" \
  -H "Content-Type: application/json" \
  -d '{"compatibility":"BACKWARD"}'

curl -s -X PUT "$REG/apis/ccompat/v7/config/orders-value" \
  -H "Content-Type: application/json" \
  -d '{"compatibility":"FULL_TRANSITIVE"}'
```

### 14.5 Mode

| Méthode | Chemin                              | Usage                                                  |
| ------- | ----------------------------------- | ------------------------------------------------------ |
| GET     | `/apis/ccompat/v7/mode`             | Mode global                                            |
| PUT     | `/apis/ccompat/v7/mode`             | Modifier (`READWRITE` / `READONLY` / `IMPORT`)         |
| GET     | `/apis/ccompat/v7/mode/{subject}`   | Par subject                                            |
| PUT     | `/apis/ccompat/v7/mode/{subject}`   | Par subject                                            |

```bash
curl -s -X PUT "$REG/apis/ccompat/v7/mode" \
  -H "Content-Type: application/json" \
  -d '{"mode":"READWRITE"}'
```

### 14.6 Contexts

| Méthode | Chemin                              | Usage                                          |
| ------- | ----------------------------------- | ---------------------------------------------- |
| GET     | `/apis/ccompat/v7/contexts`         | Retourne `["."]` (stub Confluent)              |

---

## 15. API CNCF compatibility v0

Base : `/apis/cncf/v0`. Implémente l'ébauche CNCF Schema Registry (utile pour des clients qui parlent ce dialecte). Endpoints typiques :

- `GET /apis/cncf/v0/schemagroups`
- `POST /apis/cncf/v0/schemagroups/{group}/schemas/{id}?format=avro`
- `GET /apis/cncf/v0/schemagroups/{group}/schemas/{id}/versions`

```bash
curl -s "$REG/apis/cncf/v0/schemagroups" | jq
```

---

## 16. Types d'artefacts supportés

| Type            | Format                                           |
| --------------- | ------------------------------------------------ |
| `AVRO`          | Apache Avro                                      |
| `PROTOBUF`      | Google Protocol Buffers                          |
| `JSON`          | JSON Schema                                      |
| `OPENAPI`       | OpenAPI 2/3/3.1/3.2                              |
| `ASYNCAPI`      | AsyncAPI 2/3                                     |
| `GRAPHQL`       | GraphQL SDL                                      |
| `KCONNECT`      | Kafka Connect schema                             |
| `WSDL`          | WSDL                                             |
| `XSD`           | XML Schema                                       |
| `XML`           | XML générique                                    |
| `OPENRPC`       | OpenRPC (nouveau en 3.2.2)                       |
| `MODEL_SCHEMA`  | Contrats IA/ML (3.1.7+)                          |
| `PROMPT_TEMPLATE`| Templates de prompts LLM versionnés (3.1.7+)    |

Depuis 3.1.0, vous pouvez **ajouter vos propres types** sans recompiler le registry, via un JAR de plugin déposé au déploiement.

---

## 17. Scénario de test end-to-end

Cette section enchaîne les appels précédents pour valider un déploiement complet. À copier-coller pas à pas.

```bash
# Pré-requis : $REG pointe sur l'API
export REG=http://apicurio.local

# (1) Vérifier que tout est UP
curl -s "$REG/apis/registry/v3/system/info" | jq
curl -fsS "$MGMT/q/health/ready" && echo "ready"

# (2) Créer un groupe
curl -s -X POST "$REG/apis/registry/v3/groups" \
  -H "Content-Type: application/json" \
  -d '{"groupId":"demo","description":"demo group"}'

# (3) Activer la compatibilité BACKWARD sur le groupe
curl -s -X POST "$REG/apis/registry/v3/groups/demo/rules" \
  -H "Content-Type: application/json" \
  -d '{"ruleType":"COMPATIBILITY","config":"BACKWARD"}'

# (4) Créer un artefact AVRO + v1.0.0
curl -s -X POST "$REG/apis/registry/v3/groups/demo/artifacts" \
  -H "Content-Type: application/json" \
  -d '{
    "artifactId":"User",
    "artifactType":"AVRO",
    "firstVersion":{
      "version":"1.0.0",
      "content":{
        "content":"{\"type\":\"record\",\"name\":\"User\",\"fields\":[{\"name\":\"id\",\"type\":\"long\"},{\"name\":\"email\",\"type\":\"string\"}]}",
        "contentType":"application/json"
      }
    }
  }' | jq

# (5) Ajouter une v1.0.1 compatible (champ optionnel avec default)
curl -s -X POST "$REG/apis/registry/v3/groups/demo/artifacts/User/versions" \
  -H "Content-Type: application/json" \
  -d '{
    "version":"1.0.1",
    "content":{
      "content":"{\"type\":\"record\",\"name\":\"User\",\"fields\":[{\"name\":\"id\",\"type\":\"long\"},{\"name\":\"email\",\"type\":\"string\"},{\"name\":\"firstName\",\"type\":\"string\",\"default\":\"\"}]}",
      "contentType":"application/json"
    }
  }' | jq

# (6) Tenter une v2.0.0 INCOMPATIBLE — doit échouer avec 409
curl -i -X POST "$REG/apis/registry/v3/groups/demo/artifacts/User/versions" \
  -H "Content-Type: application/json" \
  -d '{
    "content":{
      "content":"{\"type\":\"record\",\"name\":\"User\",\"fields\":[{\"name\":\"id\",\"type\":\"string\"}]}",
      "contentType":"application/json"
    }
  }'

# (7) Lister les versions
curl -s "$REG/apis/registry/v3/groups/demo/artifacts/User/versions" | jq

# (8) Récupérer le contenu de la dernière version
curl -s "$REG/apis/registry/v3/groups/demo/artifacts/User/versions/branch=latest/content"

# (9) Tester via l'API Confluent compat
curl -s "$REG/apis/ccompat/v7/subjects" | jq

# (10) Export / backup complet
curl -s "$REG/apis/registry/v3/admin/export" --output /tmp/apicurio-backup.zip
ls -la /tmp/apicurio-backup.zip
```

Si toutes ces étapes passent (la (6) doit renvoyer un 409), le registry est pleinement fonctionnel.

---

## Annexes

### A. Codes de retour HTTP courants

| Code | Signification                                                              |
| ---- | -------------------------------------------------------------------------- |
| 200  | OK                                                                         |
| 201  | Created                                                                    |
| 204  | No Content (DELETE réussi)                                                 |
| 400  | Mauvaise requête (JSON invalide, schéma invalide en mode `SYNTAX_ONLY`)    |
| 404  | Groupe / artefact / version introuvable                                    |
| 405  | Méthode non autorisée                                                      |
| 409  | Conflit (artefact existe déjà, ou violation de règle compatibility)        |
| 415  | Type MIME non supporté                                                     |
| 422  | Contenu sémantiquement invalide                                            |
| 500  | Erreur serveur (regarder les logs `kubectl logs deploy/apicurio-registry`) |

### B. Activer l'authentification OIDC

Ajouter ces env vars au Deployment `apicurio-registry` :

```yaml
- name: QUARKUS_OIDC_TENANT_ENABLED
  value: "true"
- name: QUARKUS_OIDC_AUTH_SERVER_URL
  value: https://keycloak.example.com/realms/apicurio
- name: QUARKUS_OIDC_CLIENT_ID
  value: registry-api
- name: APICURIO_AUTH_ROLE_BASED_AUTHORIZATION
  value: "true"
- name: APICURIO_AUTH_ROLE_SOURCE
  value: token
- name: APICURIO_AUTH_ROLES_ADMIN
  value: sr-admin
- name: APICURIO_AUTH_ROLES_DEVELOPER
  value: sr-developer
- name: APICURIO_AUTH_ROLES_READONLY
  value: sr-readonly
```

Puis tous les `curl` ci-dessus deviennent :

```bash
TOKEN=$(curl -s -X POST \
  "https://keycloak.example.com/realms/apicurio/protocol/openid-connect/token" \
  -d "grant_type=client_credentials" \
  -d "client_id=svc-account" \
  -d "client_secret=$CLIENT_SECRET" | jq -r .access_token)

curl -s -H "Authorization: Bearer $TOKEN" "$REG/apis/registry/v3/groups"
```

### C. Clients officiels (alternatives à curl)

- **Java** : `io.apicurio:apicurio-registry-client:3.2.5`
- **Python** : `apicurioregistryclient` (3.x)
- **Go** : `github.com/Apicurio/apicurio-registry-client-go`
- **CLI** : `registry-cli` (release sur le repo GitHub apicurio)
