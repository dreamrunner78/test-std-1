# Use-case complet — NiFi 2.9 + Apicurio Registry 3.2 + Kafka

Ce document décrit pas-à-pas un cas d'usage end-to-end qui valide l'intégration de **Apache NiFi 2.9.0** avec **Apicurio Registry 3.2.5** via **Kafka** (broker KRaft 3.9).

## Scénario fonctionnel

> Un fournisseur dépose des fichiers CSV de **commandes clients** dans un dossier surveillé. NiFi doit :
> 1. **Lire** chaque ligne CSV
> 2. **Valider** la structure contre un schéma Avro versionné enregistré dans Apicurio
> 3. **Publier** en Avro sur le topic Kafka `orders.v1` avec le framing Confluent (5 octets : `0x00` + ID du schéma)
> 4. Un second flow consomme `orders.v1`, **décode** l'Avro via le registry, **convertit** en JSON et **POST** vers un endpoint HTTPS

L'objectif est de prouver :
- la compatibilité de l'API ccompat d'Apicurio avec les clients Confluent (NiFi en l'occurrence)
- l'enforcement des règles de compatibilité de schéma à la publication
- la séparation producteur/consommateur autour d'un contrat unique versionné

---

## Architecture

```
            ┌──────────────────┐
            │  Fichier CSV     │
            └────────┬─────────┘
                     │
                     ▼
   ┌─────────────────────────────────┐
   │  NiFi — Flow PRODUCER           │
   │  GetFile → ConvertRecord        │
   │           (CSV→Avro)            │
   │  → PublishKafka                 │
   └──────┬──────────────────────────┘
          │ Avro + framing Confluent (5 octets)
          ▼
     ┌─────────┐         ┌──────────────────┐
     │ Kafka   │◀───────▶│ Apicurio (ccompat)│
     │ orders  │         │ schémas + règles  │
     │  .v1    │         │ /apis/ccompat/v7  │
     └────┬────┘         └──────────────────┘
          │
          ▼
   ┌─────────────────────────────────┐
   │  NiFi — Flow CONSUMER           │
   │  ConsumeKafka → ConvertRecord   │
   │              (Avro→JSON)        │
   │  → InvokeHTTP (POST)            │
   └─────────────────────────────────┘
```

---

## Pourquoi `ConfluentSchemaRegistry` plutôt que `ApicurioSchemaRegistry` ?

NiFi 2.x propose **deux controller services** pour les registres de schémas :

| Controller service          | Talk-to              | Avantages                                                              |
| --------------------------- | -------------------- | ---------------------------------------------------------------------- |
| `ApicurioSchemaRegistry`    | API native Apicurio v3 | Groupes, labels, branches natifs Apicurio                              |
| `ConfluentSchemaRegistry`   | API Confluent (Apicurio expose `/apis/ccompat/v7`) | **Framing wire Confluent** (5 octets) — interop avec tout client Kafka |

Pour un topic Kafka **partagé entre plusieurs producteurs/consommateurs** (NiFi, Java apps, Kafka Streams, ksqlDB...), il **faut** le framing Confluent. On utilise donc `ConfluentSchemaRegistry` pointé sur `/apis/ccompat/v7` d'Apicurio.

---

## Pré-requis

```bash
# Déployer la stack
kubectl apply -f k8s/apicurio.yaml
kubectl apply -f k8s/nifi.yaml

# Attendre que tout soit prêt
kubectl -n apicurio rollout status deploy/apicurio-registry --timeout=180s
kubectl -n apicurio rollout status deploy/apicurio-registry-ui --timeout=120s
kubectl -n nifi    rollout status statefulset/nifi --timeout=300s
kubectl -n kafka   rollout status statefulset/kafka --timeout=180s
```

Vérifier la connectivité réseau entre namespaces (CoreDNS doit résoudre `apicurio-registry.apicurio.svc` depuis le pod NiFi) :

```bash
kubectl -n nifi exec statefulset/nifi -- \
  curl -s http://apicurio-registry.apicurio.svc:8080/apis/registry/v3/system/info
```

---

## Étape 1 — Créer le topic Kafka

```bash
kubectl -n kafka exec statefulset/kafka -- \
  /opt/kafka/bin/kafka-topics.sh \
  --bootstrap-server kafka.kafka.svc:9092 \
  --create --topic orders.v1 \
  --partitions 3 --replication-factor 1
```

---

## Étape 2 — Enregistrer le schéma Avro dans Apicurio

```bash
# Port-forward Apicurio pour les curl depuis votre poste
kubectl -n apicurio port-forward svc/apicurio-registry 8080:8080 &

# Schéma initial orders v1
cat > /tmp/orders-value.json <<'EOF'
{
  "schemaType": "AVRO",
  "schema": "{\"type\":\"record\",\"name\":\"Order\",\"namespace\":\"com.example.orders\",\"fields\":[{\"name\":\"order_id\",\"type\":\"string\"},{\"name\":\"customer_id\",\"type\":\"string\"},{\"name\":\"amount_cents\",\"type\":\"long\"},{\"name\":\"currency\",\"type\":\"string\"},{\"name\":\"created_at\",\"type\":{\"type\":\"long\",\"logicalType\":\"timestamp-millis\"}}]}"
}
EOF

# Enregistrer le subject orders.v1-value
curl -s -X POST http://localhost:8080/apis/ccompat/v7/subjects/orders.v1-value/versions \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -d @/tmp/orders-value.json | jq
# => { "id": 1 }

# Activer la compatibilité BACKWARD sur ce subject
curl -s -X PUT http://localhost:8080/apis/ccompat/v7/config/orders.v1-value \
  -H "Content-Type: application/json" \
  -d '{"compatibility":"BACKWARD"}'

# Vérifier
curl -s http://localhost:8080/apis/ccompat/v7/subjects | jq
curl -s http://localhost:8080/apis/ccompat/v7/subjects/orders.v1-value/versions/latest | jq
```

L'ID renvoyé (par exemple `1`) est ce que NiFi écrira dans le préfixe Confluent des messages Kafka.

---

## Étape 3 — Configurer les controller services NiFi

Connectez-vous à l'UI NiFi (`https://localhost:8443/nifi/`, ignorer le warning TLS auto-signé, user `admin` / mot de passe défini dans le Secret).

> **⚠️ Où créer les Controller Services** : **clic droit sur le canvas vide
> → Configure → onglet Controller Services → +**. PAS via le menu hamburger
> "Controller Settings → Management Controller Services" — ce dernier est
> réservé aux Reporting Tasks et les processeurs (`ConsumeKafka`,
> `PublishKafka`, etc.) ne voient pas ces services.

> Note : tous les noms entre crochets `[...]` ci-dessous sont des **noms libres** à donner au controller service pour le retrouver dans les processeurs.

### 3.1 `StandardWebClientServiceProvider` — `[web-client]`

| Propriété         | Valeur         |
| ----------------- | -------------- |
| Connect Timeout   | `5 secs`       |
| Read Timeout      | `15 secs`      |

→ **Enable**.

### 3.2 `ConfluentSchemaRegistry` — `[apicurio-ccompat]`

| Propriété                | Valeur                                                             |
| ------------------------ | ------------------------------------------------------------------ |
| Schema Registry URLs     | `http://apicurio-registry.apicurio.svc:8080/apis/ccompat/v7`       |
| Authentication Type      | `NONE`                                                             |
| Cache Size               | `1000`                                                             |
| Cache Expiration         | `1 hour`                                                           |
| Communications Timeout   | `30 secs`                                                          |

→ **Enable**.

### 3.3 `KafkaConnectionService` (Kafka 3 Connection Service) — `[kafka-conn]`

| Propriété              | Valeur                          |
| ---------------------- | ------------------------------- |
| Bootstrap Servers      | `kafka.kafka.svc:9092`          |
| Security Protocol      | `PLAINTEXT`                     |
| Default Client ID      | `nifi-orders`                   |

→ **Enable**.

### 3.4 `CSVReader` — `[csv-reader]`

| Propriété                       | Valeur                              |
| ------------------------------- | ----------------------------------- |
| Schema Access Strategy          | `Use 'Schema Name' Property`        |
| Schema Registry                 | `apicurio-ccompat`                  |
| Schema Name                     | `orders.v1-value`                   |
| Treat First Line as Header      | `true`                              |
| CSV Format                      | `RFC 4180`                          |
| Value Separator                 | `,`                                 |

→ **Enable**.

### 3.5 `AvroRecordSetWriter` — `[avro-writer-ccompat]`

| Propriété                  | Valeur                                       |
| -------------------------- | -------------------------------------------- |
| Schema Write Strategy      | `Confluent Schema Registry Reference`        |
| Schema Access Strategy     | `Use 'Schema Name' Property`                 |
| Schema Registry            | `apicurio-ccompat`                           |
| Schema Name                | `orders.v1-value`                            |

→ **Enable**.

### 3.6 `AvroReader` — `[avro-reader-ccompat]`

| Propriété                  | Valeur                                          |
| -------------------------- | ----------------------------------------------- |
| Schema Access Strategy     | `Confluent Content-Encoded Schema Reference`    |
| Schema Registry            | `apicurio-ccompat`                              |
| Cache Size                 | `1000`                                          |

→ **Enable**.

### 3.7 `JsonRecordSetWriter` — `[json-writer]`

| Propriété                  | Valeur                       |
| -------------------------- | ---------------------------- |
| Schema Access Strategy     | `Inherit Record Schema`      |
| Schema Write Strategy      | `Do Not Write Schema`        |
| Pretty Print JSON          | `false`                      |

→ **Enable**.

---

## Étape 4 — Construire le flow PRODUCER

Drag & drop dans le canvas :

```
GetFile ──▶ ConvertRecord ──▶ PublishKafka
                                  │
                                  └──▶ failure → LogAttribute (ou DLQ)
```

### 4.1 `GetFile`

| Propriété         | Valeur                |
| ----------------- | --------------------- |
| Input Directory   | `/tmp/orders-in`      |
| Keep Source File  | `false`               |
| File Filter       | `.*\.csv`             |

### 4.2 `ConvertRecord`

| Propriété      | Valeur                  |
| -------------- | ----------------------- |
| Record Reader  | `csv-reader`            |
| Record Writer  | `avro-writer-ccompat`   |

### 4.3 `PublishKafka`

| Propriété                       | Valeur                          |
| ------------------------------- | ------------------------------- |
| Kafka Connection Service        | `kafka-conn`                    |
| Topic Name                      | `orders.v1`                     |
| Delivery Guarantee              | `Guarantee Replicated Delivery` |
| Transactions Enabled            | `true`                          |
| Record Reader                   | `avro-reader-ccompat`           |
| Record Writer                   | `avro-writer-ccompat`           |
| Publish Strategy                | `Use Content as Record Value`   |
| Message Key Field               | `/order_id`                     |
| Compression Type                | `snappy`                        |
| Failure Strategy                | `Route to Failure`              |

→ **Start** les 3 processeurs.

### 4.4 Tester en injectant un CSV

```bash
# Copier un CSV dans le pod NiFi
kubectl -n nifi cp /dev/stdin nifi-0:/tmp/orders-in/orders-001.csv <<'EOF'
order_id,customer_id,amount_cents,currency,created_at
ORD-0001,CUST-42,4999,EUR,1717590000000
ORD-0002,CUST-7,12500,USD,1717590060000
ORD-0003,CUST-42,899,EUR,1717590120000
EOF
```

Vérifier que les messages arrivent sur Kafka avec le framing Confluent :

```bash
kubectl -n kafka exec -it statefulset/kafka -- \
  /opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server kafka.kafka.svc:9092 \
  --topic orders.v1 --from-beginning --max-messages 3 \
  --property print.key=true | xxd | head
# Le premier octet doit être 0x00 (magic), les 4 suivants l'ID schéma big-endian (00 00 00 01 si l'ID est 1).
```

---

## Étape 5 — Construire le flow CONSUMER

```
ConsumeKafka ──▶ ConvertRecord ──▶ InvokeHTTP
                                       │
                                       └──▶ Response → LogAttribute
                                       └──▶ Failure  → PutFile (DLQ)
```

### 5.1 `ConsumeKafka`

| Propriété                  | Valeur                       |
| -------------------------- | ---------------------------- |
| Kafka Connection Service   | `kafka-conn`                 |
| Group ID                   | `nifi-orders-consumer`       |
| Topics                     | `orders.v1`                  |
| Topic Format               | `Names`                      |
| Auto Offset Reset          | `earliest`                   |
| Commit Offsets             | `true`                       |
| Processing Strategy        | `RECORD`                     |
| Output Strategy            | `Use Content as Value`       |
| Record Reader              | `avro-reader-ccompat`        |
| Record Writer              | `json-writer`                |
| Key Format                 | `String`                     |

### 5.2 `ConvertRecord` (ré-écriture clean → JSON)

| Propriété      | Valeur                  |
| -------------- | ----------------------- |
| Record Reader  | `avro-reader-ccompat`   |
| Record Writer  | `json-writer`           |

> Note : si `ConsumeKafka` écrit déjà du JSON via `json-writer`, ce `ConvertRecord` est facultatif. On le garde en exemple pédagogique.

### 5.3 `InvokeHTTP`

| Propriété              | Valeur                                       |
| ---------------------- | -------------------------------------------- |
| HTTP Method            | `POST`                                       |
| Remote URL             | `https://httpbin.org/post`                   |
| Content-Type           | `application/json`                           |
| SSL Context Service    | (créer un `StandardSSLContextService` minimal) |

→ **Start** les 3 processeurs.

---

## Étape 6 — Valider l'enforcement de la compatibilité

C'est le test le plus important : prouver que **Apicurio bloque les schémas cassants** quand on les pousse via ccompat.

### 6.1 Tentative de schéma **incompatible** — DOIT échouer

```bash
curl -i -X POST http://localhost:8080/apis/ccompat/v7/subjects/orders.v1-value/versions \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -d '{
    "schemaType":"AVRO",
    "schema":"{\"type\":\"record\",\"name\":\"Order\",\"namespace\":\"com.example.orders\",\"fields\":[{\"name\":\"order_id\",\"type\":\"long\"}]}"
  }'
```

Réponse attendue :

```
HTTP/1.1 409 Conflict
{ "error_code": 409, "message": "Incompatible schema, ..." }
```

### 6.2 Tentative de schéma **compatible** (ajout d'un champ avec default) — DOIT passer

```bash
curl -i -X POST http://localhost:8080/apis/ccompat/v7/subjects/orders.v1-value/versions \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -d '{
    "schemaType":"AVRO",
    "schema":"{\"type\":\"record\",\"name\":\"Order\",\"namespace\":\"com.example.orders\",\"fields\":[{\"name\":\"order_id\",\"type\":\"string\"},{\"name\":\"customer_id\",\"type\":\"string\"},{\"name\":\"amount_cents\",\"type\":\"long\"},{\"name\":\"currency\",\"type\":\"string\"},{\"name\":\"created_at\",\"type\":{\"type\":\"long\",\"logicalType\":\"timestamp-millis\"}},{\"name\":\"channel\",\"type\":\"string\",\"default\":\"web\"}]}"
  }'
```

Réponse attendue :

```
HTTP/1.1 200 OK
{ "id": 2 }
```

NiFi (`ConfluentSchemaRegistry`) rafraîchira son cache à expiration (1h par défaut, ou redémarrage du controller service pour forcer).

---

## Étape 7 — Observabilité

### 7.1 Métriques Apicurio (Prometheus)

```bash
kubectl -n apicurio port-forward svc/apicurio-registry 9000:9000 &
curl -s http://localhost:9000/q/metrics | grep -E 'apicurio|http_server'
```

Métriques utiles :
- `apicurio_storage_artifacts_total`
- `apicurio_storage_versions_total`
- `http_server_requests_seconds_count{method="POST",uri="/apis/ccompat/v7/subjects/{subject}/versions"}`

### 7.2 Provenance NiFi

UI NiFi → clic-droit sur un processeur → **View data provenance**. Permet de suivre une FlowFile de l'ingestion CSV au POST HTTPS, avec retournement automatique en cas d'échec.

### 7.3 Logs

```bash
kubectl -n nifi     logs -f nifi-0 | grep -E 'ERROR|WARN'
kubectl -n apicurio logs -f deploy/apicurio-registry | grep -E 'ERROR|registered'
kubectl -n kafka    logs -f kafka-0 | grep -E 'ERROR|orders.v1'
```

---

## Étape 8 — Évolution de schéma en production (workflow type)

1. Un développeur crée une **branche** Apicurio pour proposer un changement :

   ```bash
   curl -s -X POST "http://localhost:8080/apis/registry/v3/groups/default/artifacts/orders.v1-value/branches" \
     -H "Content-Type: application/json" \
     -d '{"branchId":"feature-add-channel","description":"Ajout du champ channel"}'
   ```

2. Il pousse une **version draft** dans cette branche.

3. Apicurio teste automatiquement la compatibilité via la règle `COMPATIBILITY=BACKWARD`.

4. Une fois validé, la branche `latest` est avancée et NiFi récupère automatiquement le nouveau schéma au prochain refresh de cache.

5. Les anciens consommateurs continuent de lire les anciens messages (l'ID schéma dans le préfixe Confluent les fait pointer vers la bonne version dans le registry).

---

## Annexe A — Alternative en CLI sans UI NiFi

NiFi a un client REST complet (`/nifi-api/`). On peut scripter tout le déploiement du flow avec [NiPyAPI](https://github.com/Chaffelson/nipyapi) ou directement en `curl`. Exemple : exporter un template de flow déjà construit, le commiter en git, le ré-importer dans un autre environnement.

```bash
# Export d'un Process Group en JSON (NiFi 2.x)
curl -k -u admin:ChangeMeNotForProd-12+ \
  "https://localhost:8443/nifi-api/process-groups/<PG-ID>/download" \
  -o orders-flow.json

# Re-import
curl -k -u admin:ChangeMeNotForProd-12+ \
  -X POST -F "file=@orders-flow.json" \
  "https://localhost:8443/nifi-api/process-groups/<TARGET-PG-ID>/upload"
```

---

## Annexe B — Diagnostic des erreurs courantes

| Symptôme                                                | Cause probable                                                         | Fix                                                                                       |
| ------------------------------------------------------- | ---------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| NiFi : `Unable to retrieve schema with id X`            | Cache stale OU registry inaccessible                                   | Disable+Enable `apicurio-ccompat` ; vérifier DNS depuis pod NiFi                          |
| NiFi : `Failed to find schema with name 'orders.v1-value'` | Subject pas encore enregistré                                        | Poster le schéma via ccompat (étape 2)                                                    |
| Apicurio : `409 Conflict ... rule violation: COMPATIBILITY` | Schéma cassant la règle                                              | C'est le comportement attendu ; corriger le schéma                                        |
| NiFi web UI : `400 An invalid request was attempted`    | `NIFI_WEB_PROXY_HOST` n'inclut pas le host utilisé                     | Mettre à jour la var d'env + rolling restart                                              |
| ConsumeKafka : messages illisibles                      | Reader configuré sans `Confluent Content-Encoded Schema Reference`     | Re-vérifier l'`AvroReader`                                                                |
| PublishKafka : `OutOfOrderSequenceException`            | Transactions activées sans `Guarantee Replicated Delivery`             | Cocher `Delivery Guarantee = Guarantee Replicated Delivery`                               |
| Kafka : `UNKNOWN_TOPIC_OR_PARTITION`                    | Topic non créé                                                         | `kafka-topics.sh --create --topic orders.v1`                                              |

---

## Annexe C — Variantes du use-case

### Variante 1 — `ApicurioSchemaRegistry` natif

Si **aucun client non-NiFi** ne consomme le topic, on peut court-circuiter ccompat :

- Remplacer `ConfluentSchemaRegistry` par `ApicurioSchemaRegistry`, propriété **Schema Registry URL** = `http://apicurio-registry.apicurio.svc:8080`, **Schema Group ID** = `orders`.
- Sur les Reader/Writer Avro, remettre `Schema Access Strategy = Use 'Schema Name' Property` et **Schema Name** = nom de l'artefact (`Order`).
- Inconvénient : les messages Kafka **n'auront pas** le préfixe Confluent — incompatible avec les consommateurs Confluent standards.

### Variante 2 — OpenAPI comme contrat de service

Apicurio n'est pas limité aux schémas Kafka. On peut publier un **contrat OpenAPI 3.x** et le faire valider en compatibility BACKWARD à chaque release :

```bash
curl -s -X POST "http://localhost:8080/apis/registry/v3/groups/api/artifacts" \
  -H "Content-Type: application/json" \
  -d '{
    "artifactId":"orders-api",
    "artifactType":"OPENAPI",
    "firstVersion":{
      "version":"1.0.0",
      "content":{"content":"<contenu yaml/json openapi>","contentType":"application/json"}
    }
  }'
```

NiFi peut ensuite récupérer le contrat via `InvokeHTTP` (`GET /apis/registry/v3/groups/api/artifacts/orders-api/versions/branch=latest/content?references=DEREFERENCE`) pour valider les payloads en entrée.

---

## Récapitulatif des fichiers

- `k8s/apicurio.yaml` — Apicurio Registry 3.2.5 (backend + UI + PostgreSQL)
- `k8s/nifi.yaml` — Apache NiFi 2.9.0 + Kafka 3.9 KRaft
- `docs/APICURIO_API_GUIDE.md` — Guide exhaustif des APIs Apicurio
- `docs/NIFI_APICURIO_USECASE.md` — Ce fichier
