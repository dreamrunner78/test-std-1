# Use-case — NiFi + Apicurio sans Kafka : validation CSV contre un schéma

Cas d'usage : un fichier **CSV** arrive dans un répertoire surveillé par NiFi.
Chaque ligne doit être validée contre un **schéma Avro versionné** publié dans
Apicurio Registry. Les lignes valides sont converties en JSON et écrites dans
un répertoire `valid/`. Les lignes invalides partent dans `invalid/` avec le
détail de l'erreur de validation en attributs FlowFile.

Aucune dépendance Kafka. C'est le scénario typique d'**ingestion fichier
gouvernée par contrat** : le schéma fait office de contrat, NiFi joue le rôle
de gateway de validation, et Apicurio sert de source de vérité versionnée.

---

## Architecture

```
   /tmp/csv-in/*.csv
         │
         ▼
   ┌───────────────────────────────────────────┐
   │  NiFi (namespace : apicurio)              │
   │                                           │
   │   GetFile ──▶ ValidateRecord              │
   │                  ├─ valid   ──▶ PutFile   │
   │                  └─ invalid ──▶ PutFile   │
   │                                           │
   └────────────┬──────────────────────────────┘
                │ HTTP GET /apis/registry/v3/groups/customers/
                │     artifacts/Customer/versions/branch=latest/content
                ▼
   ┌───────────────────────────┐
   │ Apicurio Registry 3.2.5   │
   │ (namespace : apicurio)    │
   │ Schéma Customer v1.0.0    │
   └───────────────────────────┘

   /tmp/csv-out/valid/*.json
   /tmp/csv-out/invalid/*.csv
```

---

## Pourquoi `ValidateRecord` ?

NiFi 2.x propose plusieurs processeurs de validation. Pour ce cas d'usage :

| Processeur          | Quand l'utiliser                                                               |
| ------------------- | ------------------------------------------------------------------------------ |
| `ValidateCsv`       | Validation **syntaxique** uniquement (nombre de colonnes, types primitifs). Pas d'accès au registre. |
| `ValidateRecord`    | Validation **record-aware** contre un schéma du registre. Reader/Writer = pipeline complet (CSV → JSON). |
| `ValidateJson`      | Pour JSON Schema en dur. Pas pour CSV.                                         |
| `ValidateXml`       | Pour XSD. Pas pour CSV.                                                        |

`ValidateRecord` est le bon choix : il lit le CSV avec un `CSVReader` qui va
chercher le schéma chez Apicurio, applique les règles de typage strict
(`Strict Type Checking`) et de champs non déclarés (`Allow Extra Fields`),
et route chaque record en `valid` ou `invalid` indépendamment des autres.

---

## Pré-requis

- Apicurio Registry 3.2.5 déployé dans le namespace `apicurio`
  (cf. [`k8s/apicurio.yaml`](../k8s/apicurio.yaml))
- NiFi 2.x déployé dans le **même namespace** `apicurio`
  (cf. [`k8s/nifi.yaml`](../k8s/nifi.yaml))
- Port-forwards locaux opérationnels (cf. guide v3 §1) :
  ```bash
  kubectl -n apicurio port-forward svc/apicurio-registry 8080:8080 9000:9000
  kubectl -n apicurio port-forward svc/nifi              8443:8443
  export REG=http://localhost:8080
  ```

> **Note** : Apicurio et NiFi sont dans le **même namespace** → la résolution DNS
> in-cluster utilise le nom court `apicurio-registry:8080`.

---

## Étape 1 — Enregistrer le schéma dans Apicurio

On va modéliser une CSV de clients avec 5 colonnes typées strictement.

### 1.1 Créer le groupe `customers`

```bash
curl -s -X POST "$REG/apis/registry/v3/groups" \
  -H "Content-Type: application/json" \
  -d '{
    "groupId": "customers",
    "description": "Schemas du domaine Customers",
    "labels": { "team": "data-platform" }
  }' | jq
```

### 1.2 Enregistrer l'artefact `Customer` avec sa v1.0.0

```bash
curl -s -X POST "$REG/apis/registry/v3/groups/customers/artifacts" \
  -H "Content-Type: application/json" \
  -d '{
    "artifactId": "Customer",
    "artifactType": "AVRO",
    "name": "Customer",
    "description": "Schema Avro pour un client",
    "firstVersion": {
      "version": "1.0.0",
      "content": {
        "content": "{\"type\":\"record\",\"name\":\"Customer\",\"namespace\":\"com.example.customers\",\"fields\":[{\"name\":\"customer_id\",\"type\":\"string\"},{\"name\":\"email\",\"type\":\"string\"},{\"name\":\"age\",\"type\":\"int\"},{\"name\":\"country\",\"type\":\"string\"},{\"name\":\"subscribed_at\",\"type\":{\"type\":\"long\",\"logicalType\":\"timestamp-millis\"}}]}",
        "contentType": "application/json"
      }
    }
  }' | jq
```

### 1.3 Activer la compatibilité BACKWARD sur l'artefact

Optionnel mais recommandé pour empêcher les versions futures cassantes :

```bash
curl -s -X POST "$REG/apis/registry/v3/groups/customers/artifacts/Customer/rules" \
  -H "Content-Type: application/json" \
  -d '{"ruleType":"COMPATIBILITY","config":"BACKWARD"}'
```

### 1.4 Vérifier

```bash
curl -s "$REG/apis/registry/v3/groups/customers/artifacts/Customer/versions/branch=latest/content" | jq
```

---

## Étape 2 — Préparer un CSV de test

À l'intérieur du pod NiFi (à faire dans l'étape 5) ou sur ton poste, prépare
deux fichiers : un valide, un invalide.

`customers-ok.csv` (valide) :

```csv
customer_id,email,age,country,subscribed_at
CUST-0001,alice@example.com,34,FR,1717590000000
CUST-0002,bob@example.com,29,US,1717590060000
CUST-0003,carla@example.com,42,DE,1717590120000
```

`customers-ko.csv` (lignes invalides — types et contenu cassés) :

```csv
customer_id,email,age,country,subscribed_at
CUST-0004,dave@example.com,thirty,UK,1717590180000
CUST-0005,,55,FR,1717590240000
CUST-0006,eve@example.com,28,IT,not-a-number
```

Causes d'échec :
- Ligne 1 : `age=thirty` n'est pas un `int`
- Ligne 2 : `email` vide (champ non nullable dans le schéma Avro)
- Ligne 3 : `subscribed_at=not-a-number` n'est pas un `long`

---

## Étape 3 — Configurer les controller services NiFi

Connecte-toi à l'UI NiFi (`https://localhost:8443/nifi/`, accepter le warning
TLS auto-signé, identifiants du Secret `nifi-credentials`).

Process Group root → ⚙️ → **Controller Services** → +.

### 3.1 `StandardWebClientServiceProvider` — `[web-client]`

| Propriété         | Valeur     |
| ----------------- | ---------- |
| Connect Timeout   | `5 secs`   |
| Read Timeout      | `15 secs`  |

→ **Enable**.

### 3.2 `ApicurioSchemaRegistry` — `[apicurio]`

| Propriété                     | Valeur                                       |
| ----------------------------- | -------------------------------------------- |
| Schema Registry URL           | `http://apicurio-registry:8080`              |
| Schema Group ID               | `customers`                                  |
| Cache Size                    | `1000`                                       |
| Cache Expiration              | `1 hour`                                     |
| Web Client Service Provider   | `web-client`                                 |

→ **Enable**.

> ⚠️ **N'ajoute pas** `/apis/registry/v3` à l'URL — le controller service le
> préfixe lui-même. Si tu mets l'URL complète, tu verras des erreurs
> 404 sur `/apis/registry/v3/apis/registry/v3/groups/...`.

### 3.3 `CSVReader` — `[csv-reader]`

| Propriété                       | Valeur                              |
| ------------------------------- | ----------------------------------- |
| Schema Access Strategy          | `Use 'Schema Name' Property`        |
| Schema Registry                 | `apicurio`                          |
| Schema Name                     | `Customer`                          |
| Schema Version                  | _(vide = latest)_                   |
| Treat First Line as Header      | `true`                              |
| Ignore CSV Header Column Names  | `false`                             |
| CSV Format                      | `RFC 4180`                          |
| Value Separator                 | `,`                                 |
| Quote Character                 | `"`                                 |
| Escape Character                | `\`                                 |
| Date Format                     | _(vide — on utilise timestamp-millis)_ |

→ **Enable**.

### 3.4 `JsonRecordSetWriter` — `[json-writer]`

| Propriété                  | Valeur                       |
| -------------------------- | ---------------------------- |
| Schema Write Strategy      | `Do Not Write Schema`        |
| Schema Access Strategy     | `Inherit Record Schema`      |
| Pretty Print JSON          | `true`                       |
| Output Grouping            | `Array`                      |
| Suppress Null Values       | `Never Suppress`             |
| Date Format                | _(vide)_                     |

→ **Enable**.

### 3.5 `CSVRecordSetWriter` — `[csv-writer]`

Pour ré-écrire les enregistrements invalides en CSV (avec les attributs
d'erreur sur la FlowFile) :

| Propriété                  | Valeur                       |
| -------------------------- | ---------------------------- |
| Schema Write Strategy      | `Do Not Write Schema`        |
| Schema Access Strategy     | `Inherit Record Schema`      |
| Include Header Line        | `true`                       |
| CSV Format                 | `RFC 4180`                   |

→ **Enable**.

---

## Étape 4 — Construire le flow

Drag & drop dans le canvas :

```
GetFile ──▶ ValidateRecord ──┬──▶ (valid)            ──▶ PutFile (csv-out/valid)
                             ├──▶ (invalid)          ──▶ PutFile (csv-out/invalid)
                             └──▶ (failure)          ──▶ LogAttribute
```

### 4.1 `GetFile`

| Propriété         | Valeur               |
| ----------------- | -------------------- |
| Input Directory   | `/tmp/csv-in`        |
| Keep Source File  | `false`              |
| File Filter       | `.*\.csv`            |
| Polling Interval  | `5 sec`              |

### 4.2 `ValidateRecord` — cœur du use-case

| Propriété                          | Valeur                                       |
| ---------------------------------- | -------------------------------------------- |
| Record Reader                      | `csv-reader`                                 |
| Record Writer                      | `json-writer` (pour la sortie valide)         |
| Schema Access Strategy             | `Use Schema From Record Reader`              |
| Allow Extra Fields                 | `false`                                      |
| Strict Type Checking               | `true`                                       |
| Coerce Types                       | `false` *(pas de conversion implicite)*       |
| Maximum Validation Details Length  | `1024`                                       |
| Failure Strategy                   | `Route to Failure` *(défaut)*                 |
| Validation Details Attribute Name  | `record.error`                               |

Relations sortantes :
- `valid` → vers le `PutFile` "valid"
- `invalid` → vers le second `ValidateRecord` ou directement vers `PutFile` "invalid"
- `failure` → vers `LogAttribute`

> **Note** : `ValidateRecord` route **chaque record** indépendamment. Si un
> CSV de 100 lignes contient 7 lignes invalides, on a deux FlowFiles en sortie :
> un sur `valid` (93 lignes JSON), un sur `invalid` (7 lignes — au format
> du Writer configuré sur le branchement invalide).

### 4.3 Premier `PutFile` (valid)

| Propriété                       | Valeur                          |
| ------------------------------- | ------------------------------- |
| Directory                       | `/tmp/csv-out/valid`            |
| Conflict Resolution Strategy    | `replace`                       |
| Create Missing Directories      | `true`                          |

### 4.4 Second `PutFile` (invalid)

| Propriété                       | Valeur                          |
| ------------------------------- | ------------------------------- |
| Directory                       | `/tmp/csv-out/invalid`          |
| Conflict Resolution Strategy    | `replace`                       |
| Create Missing Directories      | `true`                          |

> **Astuce** : avant le `PutFile` invalide, tu peux placer un `UpdateAttribute`
> pour ajouter l'attribut `original.filename = ${filename:append('.invalid')}`
> et rendre les fichiers en erreur immédiatement repérables.

### 4.5 `LogAttribute` (failure)

| Propriété              | Valeur     |
| ---------------------- | ---------- |
| Log Level              | `WARN`     |
| Log Payload            | `false`    |
| Attributes to Log      | _(vide = tous)_ |

---

## Étape 5 — Tester en injectant les CSV

Copie les fichiers de test dans le pod NiFi :

```bash
# Crée les répertoires dans le pod
kubectl -n apicurio exec nifi-0 -- mkdir -p /tmp/csv-in /tmp/csv-out/valid /tmp/csv-out/invalid

# Pousse le CSV valide
cat <<'EOF' | kubectl -n apicurio exec -i nifi-0 -- sh -c 'cat > /tmp/csv-in/customers-ok.csv'
customer_id,email,age,country,subscribed_at
CUST-0001,alice@example.com,34,FR,1717590000000
CUST-0002,bob@example.com,29,US,1717590060000
CUST-0003,carla@example.com,42,DE,1717590120000
EOF

# Pousse le CSV invalide
cat <<'EOF' | kubectl -n apicurio exec -i nifi-0 -- sh -c 'cat > /tmp/csv-in/customers-ko.csv'
customer_id,email,age,country,subscribed_at
CUST-0004,dave@example.com,thirty,UK,1717590180000
CUST-0005,,55,FR,1717590240000
CUST-0006,eve@example.com,28,IT,not-a-number
EOF
```

Vérifie quelques secondes plus tard :

```bash
kubectl -n apicurio exec nifi-0 -- ls -la /tmp/csv-out/valid /tmp/csv-out/invalid
kubectl -n apicurio exec nifi-0 -- cat /tmp/csv-out/valid/*.json
kubectl -n apicurio exec nifi-0 -- cat /tmp/csv-out/invalid/*.csv
```

Résultat attendu :
- `valid/*.json` contient 3 records de `customers-ok.csv` au format JSON
- `invalid/*.csv` contient 3 records de `customers-ko.csv`, et chaque
  FlowFile invalide porte un attribut `record.error` lisible dans
  l'UI NiFi (clic-droit → View FlowFile → Attributes).

Côté UI NiFi, sur `ValidateRecord` :
- Compteur `In` = 2 FlowFiles, `Out` = 2 ou plus selon le routage
- Onglet Provenance → on voit le résultat de validation par FlowFile

---

## Étape 6 — Faire évoluer le schéma et observer l'impact

Le grand intérêt d'Apicurio est de **versionner le contrat**. Ajoutons un champ
optionnel `loyalty_tier` (avec default) en v1.1.0 — c'est une évolution
backward-compatible.

```bash
curl -s -X POST "$REG/apis/registry/v3/groups/customers/artifacts/Customer/versions" \
  -H "Content-Type: application/json" \
  -d '{
    "version": "1.1.0",
    "content": {
      "content": "{\"type\":\"record\",\"name\":\"Customer\",\"namespace\":\"com.example.customers\",\"fields\":[{\"name\":\"customer_id\",\"type\":\"string\"},{\"name\":\"email\",\"type\":\"string\"},{\"name\":\"age\",\"type\":\"int\"},{\"name\":\"country\",\"type\":\"string\"},{\"name\":\"subscribed_at\",\"type\":{\"type\":\"long\",\"logicalType\":\"timestamp-millis\"}},{\"name\":\"loyalty_tier\",\"type\":\"string\",\"default\":\"bronze\"}]}",
      "contentType": "application/json"
    }
  }' | jq
```

Le `CSVReader` de NiFi continuera de fonctionner sur les anciens CSV (le champ
manquant prend la valeur `bronze` par défaut). Pour qu'il prenne la nouvelle
version sans attendre le TTL du cache :

- UI NiFi → Controller Services → `apicurio` → Disable, Enable
- ou attendre l'expiration du cache (1 heure par défaut)

Tente maintenant une évolution **cassante** — supprimer `email` (champ
obligatoire). La règle BACKWARD devrait l'empêcher :

```bash
curl -i -X POST "$REG/apis/registry/v3/groups/customers/artifacts/Customer/versions" \
  -H "Content-Type: application/json" \
  -d '{
    "content": {
      "content": "{\"type\":\"record\",\"name\":\"Customer\",\"namespace\":\"com.example.customers\",\"fields\":[{\"name\":\"customer_id\",\"type\":\"string\"},{\"name\":\"age\",\"type\":\"int\"},{\"name\":\"country\",\"type\":\"string\"},{\"name\":\"subscribed_at\",\"type\":{\"type\":\"long\",\"logicalType\":\"timestamp-millis\"}}]}",
      "contentType": "application/json"
    }
  }'
# HTTP/1.1 409 Conflict
# { "error_code": 409, "message": "Incompatible schema: missing field 'email'" }
```

---

## Variantes

### A. Validation avec JSON Schema au lieu d'Avro

Si tes données arrivent déjà en JSON, ou si tu préfères des règles de
validation plus expressives (`minimum`, `pattern`, `enum`), enregistre un
**JSON Schema** dans Apicurio :

```bash
curl -s -X POST "$REG/apis/registry/v3/groups/customers/artifacts" \
  -H "Content-Type: application/json" \
  -d '{
    "artifactId": "CustomerJson",
    "artifactType": "JSON",
    "firstVersion": {
      "version": "1.0.0",
      "content": {
        "content": "{\"$schema\":\"https://json-schema.org/draft-07/schema#\",\"type\":\"object\",\"required\":[\"customer_id\",\"email\",\"age\"],\"properties\":{\"customer_id\":{\"type\":\"string\",\"pattern\":\"^CUST-[0-9]{4}$\"},\"email\":{\"type\":\"string\",\"format\":\"email\"},\"age\":{\"type\":\"integer\",\"minimum\":18,\"maximum\":120},\"country\":{\"type\":\"string\",\"enum\":[\"FR\",\"US\",\"DE\",\"IT\",\"UK\"]}}}",
        "contentType": "application/json"
      }
    }
  }' | jq
```

Côté NiFi : remplace `ValidateRecord` par **`ValidateJson`** et configure
**Schema Access Strategy = Use Schema Name** pointant sur le controller
`ApicurioSchemaRegistry`.

### B. Validation OpenAPI d'un payload de requête

Si tu veux valider des payloads HTTP entrants contre un contrat OpenAPI :

1. Enregistre un OpenAPI dans Apicurio (type `OPENAPI`)
2. Côté NiFi, place un `HandleHttpRequest` en entrée
3. Récupère le schema via `InvokeHTTP` sur
   `$REG/apis/registry/v3/groups/api/artifacts/orders-api/versions/branch=latest/content?references=DEREFERENCE`
4. Valide le payload avec `ValidateJson` (configuré avec un `JsonTreeReader`)

### C. Notifier par mail / Slack en cas d'invalide

Sur la relation `invalid`, branche un `PutEmail` ou `InvokeHTTP` (vers un
webhook Slack) au lieu de `PutFile`.

### D. Conserver la traçabilité — écrire les erreurs côte à côte du fichier source

Avant `PutFile` invalide, place un `UpdateAttribute` qui pousse
`filename = ${filename}.error` puis un `AttributesToJSON` pour générer un
fichier `.error.json` à côté du CSV original avec :
- `record.error` (le détail de validation)
- `original.filename`
- timestamp

---

## Diagnostic — erreurs courantes

| Symptôme                                                              | Cause probable                                                        | Fix                                                                                 |
| --------------------------------------------------------------------- | --------------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| `Unable to retrieve schema with name 'Customer'`                      | Group ID erroné, ou schéma pas encore publié                          | Vérifier groupe = `customers`, vérifier `GET /groups/customers/artifacts`           |
| `Could not look up Schema Registry URL`                               | URL inclut `/apis/registry/v3` à tort                                 | Mettre uniquement `http://apicurio-registry:8080`                                   |
| Tout part en `failure` sur `ValidateRecord`                           | Le Reader CSV ne parse pas la première ligne                          | Cocher `Treat First Line as Header = true`                                          |
| `Field 'age' must be of type int but was string 'thirty'`             | C'est le résultat attendu — la ligne est routée en `invalid`          | Vérifier le routage des relations                                                   |
| Le compteur `valid` est à 0 même avec un CSV propre                   | Mismatch d'encodage, ou caractères BOM en tête de fichier             | Vérifier l'encodage UTF-8 sans BOM                                                   |
| Schéma v1.1.0 invisible côté NiFi                                     | Cache 1h du controller                                                | Disable + Enable du controller `apicurio` (purge le cache)                          |
| `Connection refused` vers Apicurio                                    | NiFi dans un autre namespace                                          | Mettre les 2 dans `apicurio`, ou utiliser le FQDN `apicurio-registry.apicurio.svc`  |

---

## Ce qu'on a démontré

- Apicurio peut servir comme **source de vérité de contrat** indépendamment
  de Kafka — n'importe quel CSV, JSON ou XML entrant peut être validé contre
  un schéma versionné via NiFi.
- `ValidateRecord` + `ApicurioSchemaRegistry` + `CSVReader` forment un
  pipeline de **validation déclarative** : changer le contrat = changer le
  comportement du flow sans toucher au flow.
- Les règles d'Apicurio (`COMPATIBILITY`, `VALIDITY`, `INTEGRITY`)
  protègent contre les évolutions cassantes en amont du déploiement.

Cas d'usage adjacents qui s'appuient sur le même pattern :
- Validation de fichiers EDIFACT/X12 (avec un schéma XML/XSD au lieu d'Avro)
- Validation de payloads webhooks (HandleHttpRequest + ValidateJson)
- Gateway de validation des sorties d'un système legacy avant chargement DWH

Pour la variante Kafka avec framing Confluent et propagation
producer/consumer, voir [`NIFI_APICURIO_USECASE.md`](./NIFI_APICURIO_USECASE.md).
