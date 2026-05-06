---
marp: true
theme: wave
paginate: true
footer: Dylan Decrulle - Séminaire du développement 2026
transition: fade 0.3s
---

# Du clic au back-end

### Piloter l'observabilité depuis le front-end avec OpenTelemetry

---

# Un bug en prod

<br>

> _"Monsieur Dupont n'arrive plus à soumettre son formulaire depuis ce matin."_

<br>

Le ticket arrive. Pas d'alerte. Pas de notification. Juste un mail.

---

# On l'a appris trop tard

<br>

```
Utilisateur bloqué  →  signalement  →  ticket  →  développeur
```

<br>

Combien d'autres ont abandonné sans dire un mot ?

---

# Et maintenant, on cherche

- Quel navigateur ? Quelle heure exacte ? Quel environnement ?
- Logs du service A → rien d'anormal
- Logs du service B → une erreur, peut-être ?
- Corrélation temporelle... à la main

---

## On ne détecte pas, on réagit et on cherche dans le noir.

---

# L'état des lieux

<div class="columns">
  <div>

**Surveillé ✅**

- Logs applicatifs des backends
- Métriques d'infra (CPU, mémoire)
- Erreurs serveur (5xx)

  </div>
<div>

**Angle mort ❌**

- Ce que fait le navigateur
- Erreurs silencieuses côté client
- Le lien entre les services — chaque monitoring est un silo
- **Aucune trace distribuée** front → back → base de données

</div>
</div>

---

![](images/otel.svg)

---

# OpenTelemetry

_"An open source observability framework for cloud native software."_

Né de la fusion d'OpenTracing et OpenCensus (CNCF, 2019).

[CNCF Projects](https://www.cncf.io/projects/) (comme Kubernetes, Argo, Helm, Keycloak etc...)

---

# OpenTelemetry

<div class="columns">
<div>

**Ce que c'est ✅**

- Standard ouvert (CNCF)
- Protocole : **OTLP**
- SDK pour 12+ langages
- 1000+ intégrations

</div>
<div>

**Ce que ce n'est PAS ❌**

- Un backend de stockage
- Un outil de visualisation
- Un concurrent d'Elastic

</div>
</div>

OTel = la **couche de collecte**. Elastic / Jaeger = les **backends**.

---

# Les signaux

<div style="display: grid; grid-template-columns: repeat(4, 1fr); gap: 1.2rem; margin-top: 1.5rem;">

<div style="background: rgba(128,179,255,0.12); border-radius: 12px; padding: 1.2rem; text-align: center;">
<img src="https://opentelemetry.io/img/homepage/signal-traces.svg" width="64"><br>
<strong>Traces</strong><br>
Traces distribuées
</div>

<div style="background: rgba(128,179,255,0.12); border-radius: 12px; padding: 1.2rem; text-align: center;">
<img src="https://opentelemetry.io/img/homepage/signal-metrics.svg" width="64"><br>
<strong>Métriques</strong><br>
Mesures dans le temps
</div>

<div style="background: rgba(128,179,255,0.12); border-radius: 12px; padding: 1.2rem; text-align: center;">
<img src="https://opentelemetry.io/img/homepage/signal-logs.svg" width="64"><br>
<strong>Logs</strong><br>
Événements horodatés
</div>

<div style="background: rgba(128,179,255,0.12); border-radius: 12px; padding: 1.2rem; text-align: center;">
<img src="https://opentelemetry.io/img/homepage/signal-baggage.svg" width="64"><br>
<strong>Baggage</strong><br>
Metadonnées contextuelles
</div>

</div>

---

# Trace & Span

<div class="columns">
<div>

Une **trace** = l'arbre complet d'une interaction.
Un **span** = une opération dans un service.

Tous les spans d'une trace partagent le même `trace_id`.

```
TraceID: abc-123
  ├─ browser  page load          [0ms → 340ms]
  └─ browser  POST /formulaire   [20ms → 118ms]
       ├─ api  validation        [25ms →  33ms] ✓
       └─ api  INSERT base       [35ms → 120ms] ❌
```

</div>
<div>

```json
{
  "name": "hello",
  "context": {
    "trace_id": "5b8aa5a2d2c872e8321cf37308d69df2",
    "span_id": "051581bf3cb55c13"
  },
  "parent_id": null,
  "start_time": "2022-04-29T18:52:58.114201Z",
  "end_time": "2022-04-29T18:52:58.114687Z",
  "attributes": {
    "http.route": "some_route1"
  },
  "events": [
    {
      "name": "Guten Tag!",
      "timestamp": "2022-04-29T18:52:58.114561Z",
      "attributes": {
        "event_attributes": 1
      }
    }
  ]
}
```

</div>
</div>

---

# Contexte & Propagation

Le contexte voyage dans le header HTTP `traceparent` (W3C TraceContext) :

```http
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             │  └────────── trace_id ───────────┘ └── span_id ───┘  │
           version                                               sampled
```

À chaque saut, le `trace_id` reste identique — seul le `span_id` change :

```
Browser  →  trace_id: 4bf92f...  span_id: 00f067...  (root span)
  API    →  trace_id: 4bf92f...  span_id: b9c7c9...  (parent: 00f067)
    DB   →  trace_id: 4bf92f...  span_id: a3ce92...  (parent: b9c7c9)
```

---

# Baggage

Un second header W3C, indépendant de `traceparent`, pour propager du **contexte métier** entre services.

```http
baggage: userId=dupont,env=prod,featureFlag=new-ui
```

<br/>

Utile pour corréler des traces avec un utilisateur, une session, ou un flag etc....

---

# Conventions sémantiques

OTel standardise les noms d'attributs — identiques dans tous les langages et tous les backends.

<div class="columns">
<div>

**HTTP**
`http.request.method` · `http.response.status_code`

**Base de données**
`db.system` · `db.query.text`

</div>
<div>

**Service**
`service.name` · `service.version`

**Navigateur**
`browser.platform` · `user_agent.original`

</div>
</div>

---

<!-- _class: part -->

# Pourquoi piloter l'observabilité depuis frontend ?

---

# Le browser orchestre tous les flux

Chaque flux **part du browser — et y revient** :

```
Browser  ──────────────────────►  Keycloak / Auth
         ◄──────────────────────  token

Browser  ──────────────────────►  API /formulaire
         ◄──────────────────────  réponse

Browser  ──────────────────────►  API /documents
         ◄──────────────────────  données
```

C'est le **seul point de passage commun** à tous les flux d'un utilisateur.

---

# Une trace, c'est quoi concrètement ?

Le **journal complet** d'une opération — de son clic jusqu'à la réponse de la base de données.

```
[clic "Soumettre"]
  └─► POST /formulaire           23ms
        ├─► validation métier     4ms  ✓
        └─► INSERT base          18ms  ✓
```

Le **`traceId`** c'est le numéro de dossier : un seul identifiant pour retrouver tout ce qui s'est passé, dans tous les services, au même moment.

---

<!-- _class: demo -->

## 🎯 Démo

# vite-insee-starter + OpenTelemetry

---

<!-- _class: demo -->

# Stack locale

```
Browser (localhost:5173)   todo-rest-api (localhost:8080)   Keycloak (localhost:8180)
          │ OTLP/HTTP                │ OTLP/HTTP                  │ OTLP/gRPC
          │                          │                             │
          ↓                          ↓                             ↓
          └──────────────────────────┴─────────────────────────────┘
                                     │
                                     ↓
                      OTel Collector (localhost:4317/4318)
                                     │
                                     │ OTLP/gRPC
                                     ↓
                           Jaeger UI (localhost:16686)
```

---

# Ce qu'on vient de voir

<br>

- Le browser produit des spans ✅
- Le `traceparent` est injecté dans chaque `fetch()` ✅
- Jaeger reçoit les données des deux côtés ✅
- La propagation de contexte **au sein du browser** reste imparfaite ⚠️

<br/>

Maintenant : comment on déploie ça en vrai ?

---

<!-- _class: part -->

# Ce qu'il faut mettre en place

### Collecteur, DMZ, et la chaîne des headers

---

# Le collecteur

Un service dédié avec trois responsabilités :

<div style="display: grid; grid-template-columns: repeat(3, 1fr); gap: 1.5rem; margin-top: 1rem;">
<div>

**Recevoir**

- Spans du browser
- Spans des servicesc
- Via OTLP/HTTP ou gRPC

</div>
<div>

**Transformer**

- Filtrer, échantillonner
- Enrichir (env, version…)
- Anonymiser, agréger

</div>
<div>

**Envoyer**

- Jaeger
- Loki
- Prometheus
- Elastic APM Server

</div>
</div>

---

# Le collecteur doit être joignable depuis le browser

Si notre frontend est exposé sur Internet, **le collecteur aussi**.

On peut en avoir plusieurs — ils parlent OTLP entre eux :

```
┌──────────────────────────────────────────────────────────────────┐
│  Internet / DMZ                                                   │
│  Browser  ──► Collector DMZ  (exposé, léger, CORS configuré)     │
└──────────────────────────────┬───────────────────────────────────┘
                               │ OTLP / réseau interne
┌──────────────────────────────┴───────────────────────────────────┐
│  Réseau interne              ↓                                   │
│  Services backend  ────────► Collector interne  ──► Elasticsearch│
└──────────────────────────────────────────────────────────────────┘
```

---

# La chaîne des headers

`traceparent` doit passer à travers **chaque couche** — sans se faire stripper.

| Couche                     | Ce qu'il faut configurer                                              |
| -------------------------- | --------------------------------------------------------------------- |
| **OTel Collector**         | CORS : `allowed_origins` + `allowed_headers: traceparent`             |
| **API Gateway (Gravitee)** | Policy : autoriser explicitement `traceparent` à transiter            |
| **Backend Java (Spring)**  | `@CrossOrigin` ou config CORS globale : header `traceparent` autorisé |

<br>

⚠️ Un header strippé silencieusement = les spans backend arrivent orphelins, sans trace parente.

---

# Ce qu'on retient

- Le **browser est le point d'entrée naturel** — c'est là que vit l'utilisateur, c'est là que la trace doit commencer
- OTel est un **standard ouvert** — l'instrumentation est pérenne, le backend (Elastic, Jaeger…) est un détail interchangeable
- C'est **compatible avec l'existant** — quelques lignes de config suffisent pour brancher OTel sur une stack ELK déjà en place

---

# Limites côté browser

- Le SDK browser est toujours **expérimental** — API susceptibles de changer
- Seules les **traces** sont supportées — pas de logs, pas de métriques
- La **propagation de contexte** au sein du browser reste imparfaite

---

# Merci

## Des Questions ?
