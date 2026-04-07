# Documentation Technico-Fonctionnelle — OP Replenishment

**Version :** 0.0.1-SNAPSHOT  
**Framework :** Spring Boot 3.5.7 / Java 21  
**Date :** 2026-04-07

---

## Table des matières

1. [Vue d'ensemble fonctionnelle](#1-vue-densemble-fonctionnelle)
2. [Architecture globale (C4 — Conteneurs)](#2-architecture-globale-c4--conteneurs)
3. [Architecture hexagonale — Modules](#3-architecture-hexagonale--modules)
4. [Modèle de données](#4-modèle-de-données)
5. [Flux principaux — Diagrammes de séquence](#5-flux-principaux--diagrammes-de-séquence)
   - 5.1 [Orchestration complète](#51-orchestration-complète)
   - 5.2 [Calcul des besoins (Stock Projection)](#52-calcul-des-besoins-stock-projection)
   - 5.3 [Évaluation des urgences (Emergency Assessment)](#53-évaluation-des-urgences-emergency-assessment)
   - 5.4 [Génération des demandes APP (Supply Core)](#54-génération-des-demandes-app-supply-core)
   - 5.5 [Scheduler — Création automatique des APP](#55-scheduler--création-automatique-des-app)
6. [Intégration Camel — Routes HTTP](#6-intégration-camel--routes-http)
7. [Sécurité](#7-sécurité)
8. [Déploiement](#8-déploiement)
9. [Paramètres métier](#9-paramètres-métier)

---

## 1. Vue d'ensemble fonctionnelle

Le microservice **OP Replenishment** gère l'optimisation du réapprovisionnement en pièces pour les techniciens de terrain. Il :

- Calcule les projections de stock quotidien par technicien sur un horizon configurable
- Évalue les niveaux d'urgence et de criticité pour chaque article manquant
- Génère automatiquement des demandes d'approvisionnement (APP) vers les distributeurs
- Orchestre l'ensemble du processus en agrégant les données de plusieurs microservices tiers
- Exécute un batch quotidien de création automatique des APP via un scheduler

```
┌──────────────────────────────────────────────────────────────────┐
│                     OP Replenishment                             │
│                                                                  │
│  1. Collecter données  →  2. Calculer besoins  →  3. Évaluer    │
│     (techniciens,            (projections stock       urgences   │
│      offres, stocks,          par technicien)        (scores     │
│      interventions)                                  criticité)  │
│                                           ↓                      │
│                              4. Générer demandes APP             │
│                                 (par technicien/distributeur)    │
└──────────────────────────────────────────────────────────────────┘
```

---

## 2. Architecture globale (C4 — Conteneurs)

```mermaid
C4Container
    title Architecture globale — OP Replenishment

    Person(user, "Utilisateur / Système appelant", "Appel REST ou scheduler interne")

    System_Boundary(replenishment, "OP Replenishment Microservice") {
        Container(api, "REST API", "Spring MVC\nPort 8054", "Expose les endpoints /v1/*")
        Container(orchestration, "OrchestrationService", "Spring Service", "Coordonne les 3 modules métier")
        Container(needsCalc, "Needs Calculation", "Module Maven", "Projections stock quotidien")
        Container(emergency, "Emergency Assessment", "Module Maven", "Scoring urgence & criticité")
        Container(supplyCore, "Supply Core", "Module Maven", "Génération demandes APP")
        Container(camel, "Camel Integration", "Apache Camel 4.10", "Agrégation parallèle HTTP")
        Container(scheduler, "AppProcessingScheduler", "Spring Scheduler", "Batch quotidien AUTO APP")
        ContainerDb(db, "Base de données", "MariaDB\n(H2 en dev)", "Proposals, paramètres métier, criticité")
        Container(cache, "Cache", "Caffeine\n500 items / 1h TTL", "ArticleTypes, SupplyParameters, etc.")
    }

    System_Ext(keycloak, "Keycloak", "OAuth2/JWT — Authentification")
    System_Ext(consul, "Consul", "Service Discovery & Config")
    System_Ext(opTechnician, "op-technician", "Liste des techniciens actifs")
    System_Ext(teamtool, "teamtool-install", "Offres distributeurs, articles requis/SAV")
    System_Ext(opRequest, "op-request", "Demandes APP en attente / création")
    System_Ext(opDevice, "op-device", "Stocks disponibles, interventions planifiées")

    Rel(user, api, "HTTPS/JSON", "OAuth2 JWT")
    Rel(api, orchestration, "appelle")
    Rel(orchestration, needsCalc, "calcul besoins")
    Rel(orchestration, emergency, "évalue urgences")
    Rel(orchestration, supplyCore, "génère APP")
    Rel(orchestration, camel, "intégration HTTP")
    Rel(scheduler, orchestration, "batch auto")
    Rel(camel, opTechnician, "POST /technicians")
    Rel(camel, teamtool, "POST /distributors\nPOST /required-articles\nPOST /sav-articles")
    Rel(camel, opRequest, "POST /pending-requests\nPOST create-app")
    Rel(camel, opDevice, "POST /planned-interventions\nPOST /available-stock")
    Rel(orchestration, db, "JPA/Liquibase")
    Rel(needsCalc, cache, "lecture paramètres")
    Rel(emergency, cache, "lecture niveaux criticité")
    Rel(api, keycloak, "validation JWT")
    Rel(api, consul, "discovery config")
```

---

## 3. Architecture hexagonale — Modules

```mermaid
graph TB
    subgraph MAIN["Module : main (orchestrateur)"]
        direction TB
        WEB["Adapters Inbound\nNeedsCalculationController\nEmergencyAssessmentController\nOrchestrationController"]
        ORCH["OrchestrationService"]
        SCHED["AppProcessingScheduler"]
        CAMEL["Adapters Outbound\nCamelMicroserviceAdapter\nOpTechnicianAdapter\nOpRequestAdapter"]
        SECUR["Security / Config\nKeycloak · Consul · OpenAPI"]
    end

    subgraph NC["Module : needs-calculation"]
        NC_PORT["Port Inbound\nStockProjectionPort"]
        NC_SVC["StockProjectionService"]
        NC_DOM["Domain\nDailyConsumptionBuilder\nTechnicianStockProjection\nSupplyParameters"]
    end

    subgraph EA["Module : emergency-assessment"]
        EA_PORT["Port Inbound\nEmergencyAssessmentPort"]
        EA_SVC["EmergencyAssessmentService"]
        EA_DOM["Domain\nEmergencyAssessment\nScoring Algorithm"]
    end

    subgraph SC["Module : supply-core"]
        SC_PORT["Port Inbound\nSupplyCorePort"]
        SC_SVC["SupplyCoreService"]
        SC_DOM["Domain\nBoxPacking\nPriorityArticle\nRequestBuilder"]
    end

    subgraph COMMON["Module : common (partagé)"]
        REPO["Ports Outbound\nRepositories JPA"]
        ENTITY["Entities JPA\nReplenishmentProposal\nArticleType\nSupplyParameter\n…"]
        DTO["DTOs / Value Objects\nDataIntegrationFinalOutput\nTechStockDailyConsumption\n…"]
    end

    WEB --> ORCH
    SCHED --> ORCH
    ORCH --> NC_PORT
    ORCH --> EA_PORT
    ORCH --> SC_PORT
    ORCH --> CAMEL

    NC_PORT --> NC_SVC
    NC_SVC --> NC_DOM
    NC_SVC --> REPO

    EA_PORT --> EA_SVC
    EA_SVC --> EA_DOM
    EA_SVC --> REPO

    SC_PORT --> SC_SVC
    SC_SVC --> SC_DOM
    SC_SVC --> REPO

    REPO --> ENTITY
    NC_DOM --> DTO
    EA_DOM --> DTO
    SC_DOM --> DTO

    style MAIN fill:#dbeafe,stroke:#3b82f6
    style NC fill:#dcfce7,stroke:#22c55e
    style EA fill:#fef9c3,stroke:#eab308
    style SC fill:#ffe4e6,stroke:#f43f5e
    style COMMON fill:#f3f4f6,stroke:#6b7280
```

---

## 4. Modèle de données

```mermaid
erDiagram
    REPLENISHMENT_PROPOSAL {
        bigint id PK
        varchar user_code
        date date_of_analysis
        int search_depth
        decimal contingency_factor
        varchar distributor
        int boxes_number
    }

    REPLENISHMENT_PROPOSAL_EQUIPMENT {
        bigint id PK
        bigint proposal_id FK
        varchar article_type
        int available_qty
        int total_gross_qty
        int qty_to_place
        int stock_min
        varchar criticality
    }

    REPLENISHMENT_PROPOSAL_INTER_ASSOC {
        bigint id PK
        bigint proposal_id FK
        varchar intervention_type
    }

    ARTICLE_TYPE {
        varchar code PK
        varchar device_type
        bool article_with_serial_number
        varchar description
    }

    SUPPLY_PARAMETER {
        varchar code PK
        varchar description
        decimal parameter_value
    }

    SUPPLY_ARTICLE_TYPE_CONSTRAINT {
        bigint id PK
        varchar article_code FK
        varchar constraint_ref FK
        decimal min_value
        decimal max_value
    }

    SUPPLY_CONSTRAINT {
        varchar ref PK
        varchar description
    }

    CRITICALITY_LEVEL {
        varchar code PK
        varchar description
        int score
        int order
    }

    BOX {
        varchar ref PK
        varchar designation
    }

    ARTICLE_TYPE_BOX_PARAMETER {
        bigint id PK
        varchar article_type FK
        varchar box_ref FK
        decimal occupancy_ratio
    }

    REPLENISHMENT_STRATEGY {
        bigint id PK
        varchar name
        varchar description
        bool is_activated
    }

    REPLENISHMENT_PROPOSAL ||--o{ REPLENISHMENT_PROPOSAL_EQUIPMENT : "contient"
    REPLENISHMENT_PROPOSAL ||--o{ REPLENISHMENT_PROPOSAL_INTER_ASSOC : "associe"
    ARTICLE_TYPE ||--o{ SUPPLY_ARTICLE_TYPE_CONSTRAINT : "contraint"
    SUPPLY_CONSTRAINT ||--o{ SUPPLY_ARTICLE_TYPE_CONSTRAINT : "définit"
    ARTICLE_TYPE ||--o{ ARTICLE_TYPE_BOX_PARAMETER : "occupe"
    BOX ||--o{ ARTICLE_TYPE_BOX_PARAMETER : "contient"
```

---

## 5. Flux principaux — Diagrammes de séquence

### 5.1 Orchestration complète

**Endpoint :** `POST /v1/orchestration/start?startDate=…&endDate=…`

```mermaid
sequenceDiagram
    autonumber
    actor Client
    participant API as OrchestrationController
    participant ORCH as OrchestrationService
    participant CAMEL as CamelMicroserviceAdapter
    participant opTech as op-technician
    participant teamtool as teamtool-install
    participant opReq as op-request
    participant opDev as op-device

    Client->>+API: POST /v1/orchestration/start
    API->>+ORCH: orchestrate(startDate, endDate)

    note over ORCH,CAMEL: Étape 1 — Techniciens & Offres (parallèle)
    ORCH->>+CAMEL: getTechnicianAndOffers()
    par
        CAMEL->>+opTech: POST /technicians
        opTech-->>-CAMEL: Liste techniciens
    and
        CAMEL->>+teamtool: POST /distributors
        teamtool-->>-CAMEL: Offres distributeurs
    end
    CAMEL-->>-ORCH: TechnicianDistributorDto

    note over ORCH,CAMEL: Étape 2 — Demandes & Interventions (parallèle)
    ORCH->>+CAMEL: getRequestsAndInterventions()
    par
        CAMEL->>+opReq: POST /pending-requests
        opReq-->>-CAMEL: Demandes APP en attente
    and
        CAMEL->>+opDev: POST /planned-interventions
        opDev-->>-CAMEL: Interventions planifiées
    end
    CAMEL-->>-ORCH: RequestInterventionResponseDto

    note over ORCH,CAMEL: Étape 3 — Articles & Stocks (parallèle)
    ORCH->>+CAMEL: getArticlesAndStock()
    par
        CAMEL->>+teamtool: POST /required-articles
        teamtool-->>-CAMEL: Articles requis
    and
        CAMEL->>+teamtool: POST /sav-articles
        teamtool-->>-CAMEL: Articles SAV
    and
        CAMEL->>+opDev: POST /available-stock
        opDev-->>-CAMEL: Stocks disponibles
    end
    CAMEL-->>-ORCH: ArticleStockResponseDto

    ORCH-->>-API: DataIntegrationFinalOutput
    API-->>-Client: 200 OK (JSON)
```

---

### 5.2 Calcul des besoins (Stock Projection)

**Endpoint :** `POST /v1/need-calculation`

```mermaid
sequenceDiagram
    autonumber
    actor Client
    participant API as NeedsCalculationController
    participant SVC as StockProjectionService
    participant REPO as Repositories (Cache)
    participant DOM as DailyConsumptionBuilder

    Client->>+API: POST /v1/need-calculation (DataIntegrationFinalOutput)
    API->>+SVC: calculateStockProjection(data)

    SVC->>+REPO: getArticleTypes()
    REPO-->>-SVC: Map<code, ArticleType>

    SVC->>+REPO: getSupplyParameters()
    REPO-->>-SVC: SDP (15j), IMP (coeff imprévu)

    loop Pour chaque technicien
        SVC->>+DOM: buildDailyConsumption(technicien, articles, SDP)
        DOM->>DOM: Filtrer articles avec numéro de série / TAG
        DOM->>DOM: Projeter stock jour par jour
        DOM->>DOM: Appliquer coefficient IMP
        DOM-->>-SVC: List<TechStockDailyConsumption>
    end

    SVC-->>-API: Map<techCode, List<TechStockDailyConsumption>>
    API-->>-Client: 200 OK (JSON)
```

**Paramètres clés :**

| Param | Code | Valeur défaut | Rôle |
|-------|------|---------------|------|
| Profondeur analyse | SDP | 15 jours | Horizon de projection |
| Facteur imprévu | IMP | 0 | Multiplicateur de sécurité |

---

### 5.3 Évaluation des urgences (Emergency Assessment)

**Endpoint :** `POST /v1/emergency-assessment/calculate`

```mermaid
sequenceDiagram
    autonumber
    participant API as EmergencyAssessmentController
    participant SVC as EmergencyAssessmentService
    participant DOM as EmergencyAssessment (domain)
    participant REPO as Repositories (Cache)

    API->>+SVC: buildEmergencyMap(projections)

    SVC->>+REPO: getCriticalityLevels()
    REPO-->>-SVC: CRITICAL_A/B · URGENT_A/B · SAFE

    SVC->>+REPO: getSupplyParameters()
    REPO-->>-SVC: CDS, CDE, UDS, UDE

    loop Pour chaque technicien / article
        SVC->>+DOM: assess(dailyConsumption, parameters)

        note over DOM: Étape 1 — Urgence quantitative
        DOM->>DOM: Stock restant < 0 dans SDP jours ?
        DOM->>DOM: Calculer score jours avant rupture

        note over DOM: Étape 2 — Urgence temporelle
        DOM->>DOM: Jours dans fenêtre criticité [CDS–CDE] ?
        DOM->>DOM: Jours dans fenêtre urgence [UDS–UDE] ?

        note over DOM: Étape 3 — Score combiné
        DOM->>DOM: Score = f(quantitatif, temporel, pondération)

        note over DOM: Étape 4 — Classification
        DOM->>DOM: Score ≥ 300 → CRITICAL_A
        DOM->>DOM: Score ≥ 250 → CRITICAL_B
        DOM->>DOM: Score ≥ 210 → URGENT_A
        DOM->>DOM: Score ≥ 160 → URGENT_B
        DOM->>DOM: Score < 160 → SAFE

        DOM-->>-SVC: TechArticleDetail (criticality, score)
    end

    SVC-->>-API: Map<techCode, List<TechArticleDetail>>
```

**Niveaux de criticité :**

| Code | Score | Priorité |
|------|-------|----------|
| CRITICAL_A | 300 | 1 — Critique Grade A |
| CRITICAL_B | 250 | 2 — Critique Grade B |
| URGENT_A   | 210 | 3 — Urgent Grade A |
| URGENT_B   | 160 | 4 — Urgent Grade B |
| SAFE       | 0   | 5 — Sécurisé |

---

### 5.4 Génération des demandes APP (Supply Core)

**Appelé par :** `OrchestrationService.calculate()`

```mermaid
sequenceDiagram
    autonumber
    participant ORCH as OrchestrationService
    participant SVC as SupplyCoreService
    participant REPO as Repositories
    participant DOM as RequestBuilder (domain)
    participant DB as Base de données
    participant EXT as op-request (externe)

    ORCH->>+SVC: generateAppRequests(emergencyMap, offers)

    SVC->>+REPO: getActivatedStrategy()
    REPO-->>-SVC: ReplenishmentStrategy (DEFAULT)

    SVC->>+REPO: getBoxesWithOccupancy()
    REPO-->>-SVC: Box + ArticleTypeBoxParameter (ratios)

    loop Pour chaque technicien
        SVC->>DOM: groupArticlesByDistributor(articles urgents)
        DOM->>DOM: Séparer urgents (score ≥ 210) / sûrs (score < 210)
        DOM->>DOM: Trier par priorité de criticité

        loop Pour chaque offre distributeur
            DOM->>DOM: Sélectionner articles par priorité
            DOM->>DOM: Calculer occupation boîtes (ratios)
            DOM->>DOM: Construire RequestDto (technicien, articles, boîte)
        end
    end

    SVC-->>-ORCH: List<RequestDto>

    ORCH->>+DB: persist(ReplenishmentProposal)
    DB-->>-ORCH: OK

    ORCH->>+EXT: submitRequests(List<RequestDto>)
    EXT-->>-ORCH: Confirmation

    ORCH-->>ORCH: return List<RequestDto>
```

---

### 5.5 Scheduler — Création automatique des APP

```mermaid
sequenceDiagram
    autonumber
    participant TIMER as Cron Scheduler
    participant SCHED as AppProcessingScheduler
    participant AUTH as UserAuthenticationService
    participant KC as Keycloak
    participant ORCH as OrchestrationService
    participant EXT as op-request

    TIMER-->>+SCHED: déclenchement cron (configurable)

    SCHED->>+AUTH: getServiceToken()
    AUTH->>+KC: POST /token (client_credentials)
    KC-->>-AUTH: access_token JWT
    AUTH-->>-SCHED: Bearer token

    SCHED->>+EXT: clearExistingRequests(token)
    EXT-->>-SCHED: OK

    SCHED->>+ORCH: calculate(startDate, endDate, token)
    note over ORCH: Exécute le flux complet\n(voir §5.1 → §5.4)
    ORCH-->>-SCHED: List<RequestDto> créées

    SCHED-->>-TIMER: fin du batch
```

---

## 6. Intégration Camel — Routes HTTP

```mermaid
flowchart TD
    START([direct:startMainOrchestrationRoute]) --> TECH_OFFER

    subgraph TECH_OFFER["Route: getTechOfferRoute (multicast parallèle)"]
        direction LR
        T1["HTTP POST\nop-technician\n/technicians"]
        T2["HTTP POST\nteamtool-install\n/distributors"]
        AGG1["TechOfferAggregationStrategy\n(merge résultats)"]
        T1 --> AGG1
        T2 --> AGG1
    end

    TECH_OFFER --> |TechnicianDistributorDto| REQ_INT

    subgraph REQ_INT["Route: getRequestInterventionRoute (multicast parallèle)"]
        direction LR
        R1["HTTP POST\nop-request\n/pending-requests"]
        R2["HTTP POST\nop-device\n/planned-interventions"]
        AGG2["RequestInterventionAggregationStrategy"]
        R1 --> AGG2
        R2 --> AGG2
    end

    REQ_INT --> |RequestInterventionResponseDto| ART_STOCK

    subgraph ART_STOCK["Route: getArticleStockRoute (multicast parallèle)"]
        direction LR
        A1["HTTP POST\nteamtool-install\n/required-articles"]
        A2["HTTP POST\nteamtool-install\n/sav-articles"]
        A3["HTTP POST\nop-device\n/available-stock"]
        AGG3["ArticleStockAggregationStrategy"]
        A1 --> AGG3
        A2 --> AGG3
        A3 --> AGG3
    end

    ART_STOCK --> |ArticleStockResponseDto| BUILD
    BUILD["buildFinalOutput()\nDataIntegrationFinalOutput"] --> END([Réponse JSON])

    subgraph ERRORS["Gestion erreurs globale"]
        ERR["onException\nHttpOperationFailedException\nSocketTimeoutException\nConnectException\nIOException"]
        ERR --> RETRY["Retry (max configurable)"]
        RETRY --> DLQ["Dead Letter Channel"]
    end
```

**Pool de connexions Camel :**

| Paramètre | Valeur |
|-----------|--------|
| Pool size total | 200 |
| Pool par route | 50 |
| Keep-alive | 60s |
| Socket timeout | 60s |
| Virtual Threads | Activés |

---

## 7. Sécurité

```mermaid
flowchart LR
    CLIENT["Client HTTP"] -->|"Bearer JWT"| SEC["SecurityFilterChain\nSpring Security"]

    SEC -->|"Public"| PUBLIC["/swagger-ui/**\n/v3/api-docs/**\n/actuator/**"]
    SEC -->|"Authentifié"| PROTECTED["/v1/**"]

    PROTECTED --> JWT["JwtDecoder\nvalidation issuer-uri"]
    JWT --> KC["Keycloak\n/realms/{realm}"]
    JWT --> ROLE["KeycloakRealmRoleConverter\nRBAC"]

    subgraph HEADERS["En-têtes de sécurité HTTP"]
        H1["HSTS (1 an)"]
        H2["Content-Security-Policy"]
        H3["X-XSS-Protection"]
        H4["X-Frame-Options: DENY"]
        H5["Referrer-Policy"]
    end
```

---

## 8. Déploiement

```mermaid
graph TB
    subgraph K8S["Kubernetes / Docker"]
        subgraph POD["Pod — OP Replenishment"]
            APP["JVM — Java 21\nZGC · Virtual Threads\nCDS archive"]
            APP_PORT["Port 8054\nAPI REST"]
            MGT_PORT["Port 8154\nActuator / Prometheus"]
        end

        subgraph DOCKER["Image Docker (multi-stage)"]
            BUILD_STG["Stage 1 : Build\nmaven:3.9.6-eclipse-temurin-21\nCompile + layered JAR"]
            RUN_STG["Stage 2 : Runtime\neclipse-temurin:21-jre-jammy\nUser non-root (spring:1001)"]
            BUILD_STG --> RUN_STG
        end
    end

    subgraph INFRA["Infrastructure"]
        MARIADB[("MariaDB\nPool HikariCP\n50 connexions max")]
        CONSUL["Consul\nService Discovery\n+ Config"]
        KEYCLOAK["Keycloak\nOAuth2 / JWT"]
        PROMETHEUS["Prometheus\n/actuator/prometheus"]
    end

    APP_PORT -.->|"HTTPS"| CLIENTS["Clients / API Gateway"]
    APP --> MARIADB
    APP --> CONSUL
    APP --> KEYCLOAK
    MGT_PORT --> PROMETHEUS

    subgraph JVM_OPTS["Options JVM"]
        O1["-XX:+UseZGC -XX:+ZGenerational"]
        O2["-XX:MaxRAMPercentage=70"]
        O3["-XX:SharedArchiveFile (CDS)"]
        O4["-Xss512k (virtual threads)"]
        O5["-XX:+ExitOnOutOfMemoryError"]
    end
```

**Health Check :**
```
GET http://localhost:8154/actuator/health
Interval: 30s | Timeout: 3s | StartPeriod: 60s | Retries: 3
```

---

## 9. Paramètres métier

### Paramètres de calcul (table `supply_parameter`)

| Code | Description | Valeur défaut | Usage |
|------|-------------|---------------|-------|
| **SDP** | Profondeur d'analyse (jours) | 15 | Horizon de projection stock |
| **IMP** | Facteur d'imprévu (coefficient) | 0 | Sécurité sur quantités calculées |
| **CDS** | Date début criticité (j) | 1 | Début fenêtre critique |
| **CDE** | Date fin criticité (j) | 5 | Fin fenêtre critique |
| **UDS** | Date début urgence (j) | 6 | Début fenêtre urgente |
| **UDE** | Date fin urgence (j) | 8 | Fin fenêtre urgente |

### Règle de scoring d'urgence

```mermaid
graph LR
    SCORE["Score d'urgence\n(combinaison quantitatif + temporel)"]
    SCORE -->|"≥ 300"| CA["CRITICAL_A\nPriorité 1"]
    SCORE -->|"≥ 250 et < 300"| CB["CRITICAL_B\nPriorité 2"]
    SCORE -->|"≥ 210 et < 250"| UA["URGENT_A\nPriorité 3"]
    SCORE -->|"≥ 160 et < 210"| UB["URGENT_B\nPriorité 4"]
    SCORE -->|"< 160"| SAFE["SAFE\nPriorité 5"]

    CA -->|"score ≥ 210"| PLACE["Articles à placer\nen commande APP"]
    CB --> PLACE
    UA --> PLACE
    UB --> PLACE
    SAFE -->|"score < 210"| WAIT["Articles surveillés\nnon commandés"]
```

### Stack technologique

| Couche | Technologie | Version |
|--------|-------------|---------|
| Runtime | Java / Eclipse Temurin | 21 LTS |
| Framework | Spring Boot | 3.5.7 |
| Build | Maven | 3.9.6 |
| Intégration | Apache Camel | 4.10.2 |
| Base de données | MariaDB / H2 | — |
| Migration | Liquibase | 4.20.0 |
| Cache | Caffeine | — |
| Auth | Keycloak / OAuth2 JWT | — |
| Discovery | Consul | — |
| API Docs | Springdoc OpenAPI | 2.8.14 |
| Observabilité | Micrometer / Prometheus | — |
| Conteneurisation | Docker | — |
