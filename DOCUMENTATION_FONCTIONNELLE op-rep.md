# Documentation Fonctionnelle — OP Replenishment

**Version :** 0.0.1-SNAPSHOT  
**Date :** 2026-04-07  
**Domaine :** Gestion du réapprovisionnement en pièces pour techniciens terrain

---

## Table des matières

1. [Présentation du domaine](#1-présentation-du-domaine)
2. [Acteurs et systèmes](#2-acteurs-et-systèmes)
3. [Glossaire métier](#3-glossaire-métier)
4. [Cas d'utilisation](#4-cas-dutilisation)
5. [Règles de gestion](#5-règles-de-gestion)
   - 5.1 [Projection de stock quotidien](#51-projection-de-stock-quotidien)
   - 5.2 [Scoring d'urgence et criticité](#52-scoring-durgence-et-criticité)
   - 5.3 [Génération de la demande APP](#53-génération-de-la-demande-app)
6. [Paramètres configurables](#6-paramètres-configurables)
7. [Référentiels articles et contraintes](#7-référentiels-articles-et-contraintes)
8. [Flux fonctionnels end-to-end](#8-flux-fonctionnels-end-to-end)
9. [Processus automatique (batch)](#9-processus-automatique-batch)
10. [Impacts et limites fonctionnelles](#10-impacts-et-limites-fonctionnelles)

---

## 1. Présentation du domaine

### Problématique

Les techniciens terrain interviennent quotidiennement chez des clients pour des opérations d'installation, de maintenance et de dépannage. Ils disposent chacun d'un stock embarqué de pièces détachées. Ce stock doit être suffisant pour couvrir les interventions planifiées sur un horizon glissant.

Sans système de réapprovisionnement proactif, les techniciens risquent :
- Des ruptures de stock pendant une intervention
- Des délais supplémentaires pour le client
- Des coûts de réexpédition urgente

### Objectif du microservice

**OP Replenishment** automatise la détection des besoins en réapprovisionnement et la création de demandes d'approvisionnement (APP) pour chaque technicien, en anticipant les consommations sur les 15 prochains jours (horizon configurable).

```mermaid
flowchart LR
    A["Stocks techniciens\n(situation actuelle)"]
    B["Interventions planifiées\n(à venir)"]
    C["Calcul des besoins\n(projection quotidienne)"]
    D["Évaluation d'urgence\n(scoring de criticité)"]
    E["Demandes APP\n(commandes automatiques)"]

    A --> C
    B --> C
    C --> D
    D --> E
```

---

## 2. Acteurs et systèmes

```mermaid
graph TB
    subgraph ACTEURS["Acteurs"]
        TECH["Technicien terrain\nBénéficiaire des commandes"]
        BATCH["Scheduler automatique\nCréateur des APP (batch)"]
        API_USER["Système tiers / API\nPeut déclencher manuellement"]
    end

    subgraph SYSTEMES["Systèmes contributeurs"]
        OP_TECH["op-technician\nListe des techniciens actifs"]
        TEAM["teamtool-install\nOffres distributeurs\nArticles requis et SAV"]
        OP_REQ["op-request\nDemandes APP existantes\nCréation nouvelles APP"]
        OP_DEV["op-device\nStocks actuels techniciens\nInterventions planifiées"]
    end

    subgraph SERVICE["OP Replenishment"]
        CORE["Calcul besoins\n+ Scoring urgence\n+ Génération APP"]
    end

    BATCH -->|"déclenchement quotidien"| SERVICE
    API_USER -->|"déclenchement manuel"| SERVICE
    SERVICE -->|"lecture"| SYSTEMES
    SERVICE -->|"création APP"| OP_REQ
    TECH -.->|"bénéficiaire"| OP_REQ
```

### Rôles

| Acteur | Rôle fonctionnel |
|--------|-----------------|
| **Technicien terrain** | Reçoit les pièces commandées ; son stock est projeté pour anticiper les besoins |
| **Scheduler automatique** | Déclenche chaque jour la création des APP en batch, après nettoyage des demandes précédentes |
| **Système tiers / API** | Peut déclencher le calcul ou les sous-étapes manuellement (via REST authentifié) |
| **op-technician** | Fournit la liste des techniciens actifs avec leur coefficient d'imprévu individuel |
| **teamtool-install** | Fournit les offres des distributeurs, la composition des bundles, les articles requis et SAV |
| **op-request** | Archive les demandes APP passées (évite les doublons) et reçoit les nouvelles commandes |
| **op-device** | Fournit les stocks réels par technicien et la liste des interventions planifiées |

---

## 3. Glossaire métier

| Terme | Définition |
|-------|------------|
| **APP** | *Approvisionnement* — Demande de réapprovisionnement automatique créée pour un technicien auprès d'un distributeur. |
| **Technicien** | Agent terrain identifié par un code unique (`userCode`), rattaché à un ou plusieurs distributeurs. |
| **Intervention** | Visite planifiée chez un client. Trois types : **SAV** (dépannage), **SS** (maintenance), **EX** (ajout d'option). |
| **Bundle** | Ensemble prédéfini d'articles nécessaires à une intervention donnée (identifié par une référence). Un bundle principal + bundles optionnels. |
| **Distributeur** | Fournisseur de pièces (ex. ORANGE, GROUPAMA). Identifié par un code propriétaire d'offre. |
| **Offre** | Contrat d'approvisionnement d'un distributeur (ex. GBH2, MPO2). Définit les articles disponibles et les contraintes associées. |
| **Stock min / max** | Seuils de stock définis par article et par offre. En dessous du min = déclenchement de la commande. La quantité commandée vise le milieu de la fourchette. |
| **SDP** | *Search Depth Parameter* — Horizon d'analyse en jours (défaut : 15 jours). |
| **IMP** | *Facteur d'imprévu* — Coefficient multiplicateur de sécurité appliqué aux consommations prévisionnelles (défaut : 0 = pas de marge). |
| **Criticité** | Niveau de priorité d'un besoin : CRITICAL_A, CRITICAL_B, URGENT_A, URGENT_B, SAFE. |
| **Score d'urgence** | Valeur numérique synthétisant l'urgence quantitative + temporelle + l'importance de l'article. |
| **Article requis** | Pièce nécessaire à une intervention de type SS ou EX, définie dans le bundle. |
| **Article SAV** | Pièce de remplacement pour une intervention de type SAV (dépannage), suivie individuellement (numéro de série). |
| **UrQ** | *Urgence Quantitative* — Score 0 ou 100 selon que le stock passe ou non sous le seuil minimum dans la fenêtre d'analyse. |
| **UrT** | *Urgence Temporelle* — Score 50, 100 ou 150 selon la date prévisionnelle de rupture par rapport aux fenêtres configurées. |
| **ImP** | *Importance* — Poids de l'article dans le calcul du score (100 pour les articles standard, 10 pour les badges TAG). |
| **Boîte (Box)** | Contenant physique d'expédition. Le nombre de boîtes est calculé en fonction du volume occupé par les articles à envoyer. |
| **Proposition de réapprovisionnement** | Enregistrement persisté en base de données résumant l'analyse pour un technicien (articles, quantités, criticité). |

---

## 4. Cas d'utilisation

```mermaid
graph LR
    BATCH(["Scheduler\nautomatique"])
    API(["Système tiers\n/ API"])
    TECH(["Technicien"])

    UC1["UC1 — Lancer l'orchestration\ncomplète"]
    UC2["UC2 — Calculer les projections\nde stock"]
    UC3["UC3 — Évaluer l'urgence\ndes besoins"]
    UC4["UC4 — Générer les demandes\nAPP"]
    UC5["UC5 — Création automatique\nbatch des APP"]
    UC6["UC6 — Consulter les offres\ntechniciens / distributeurs"]
    UC7["UC7 — Consulter les demandes\net interventions"]

    BATCH -->|"déclenche"| UC5
    UC5 -->|"inclut"| UC1
    UC1 -->|"inclut"| UC2
    UC1 -->|"inclut"| UC3
    UC1 -->|"inclut"| UC4
    API --> UC1
    API --> UC2
    API --> UC3
    API --> UC4
    API --> UC6
    API --> UC7
    TECH -.->|"bénéficiaire"| UC4
```

### Description des cas d'utilisation

#### UC1 — Lancer l'orchestration complète
- **Acteur principal :** Système tiers (API) ou scheduler
- **Pré-conditions :** Token OAuth2 valide
- **Description :** Agrège les données de tous les microservices externes, calcule les besoins, évalue l'urgence, génère et soumet les demandes APP
- **Post-conditions :** Propositions persistées en base, APP créées dans op-request

#### UC2 — Calculer les projections de stock
- **Acteur principal :** Système tiers ou orchestration interne
- **Description :** Pour chaque technicien, projette le niveau de stock article par article sur SDP jours en tenant compte des consommations prévisionnelles
- **Post-conditions :** Map de consommation quotidienne par technicien

#### UC3 — Évaluer l'urgence des besoins
- **Acteur principal :** Système tiers ou orchestration interne
- **Pré-conditions :** Projections de stock calculées (UC2)
- **Description :** Attribue un score d'urgence et un niveau de criticité à chaque article de chaque technicien
- **Post-conditions :** Map d'urgences par technicien avec criticité et quantité à commander

#### UC4 — Générer les demandes APP
- **Acteur principal :** Système tiers ou orchestration interne
- **Pré-conditions :** Map d'urgences calculée (UC3)
- **Description :** Groupe les articles par technicien/distributeur, calcule le conditionnement en boîtes, construit et soumet les RequestDto
- **Post-conditions :** APP soumises à op-request

#### UC5 — Création automatique batch des APP
- **Acteur principal :** Scheduler (cron configurable)
- **Description :** Supprime les APP existantes, s'authentifie sur Keycloak, lance l'orchestration complète (UC1)
- **Post-conditions :** APP du jour créées pour tous les techniciens actifs

#### UC6 — Consulter les offres techniciens / distributeurs
- **Acteur principal :** Système tiers
- **Description :** Retourne la liste combinée des techniciens actifs et des offres distributeurs disponibles
- **Endpoint :** `GET /v1/orchestration/technician-offer`

#### UC7 — Consulter les demandes et interventions
- **Acteur principal :** Système tiers
- **Description :** Retourne les demandes APP en attente et les interventions planifiées
- **Endpoint :** `POST /v1/orchestration/request-intervention`

---

## 5. Règles de gestion

### 5.1 Projection de stock quotidien

**Objectif :** Simuler le niveau de stock d'un technicien jour par jour sur les SDP prochains jours en tenant compte des consommations prévisionnelles issues de ses interventions planifiées.

```mermaid
flowchart TD
    START(["Début projection\npour un technicien"]) --> LOAD

    LOAD["Charger :\n- Types d'articles\n- Paramètres SDP et IMP\n- Bundles d'articles\n- Stocks actuels"]

    LOAD --> INIT["Initialiser tableau de consommation\n(SDP jours × articles) = 0"]

    INIT --> LOOP_INT

    subgraph LOOP_INT["Pour chaque intervention planifiée"]
        CALC_IDX["Calculer index_jour =\ndate_intervention - aujourd'hui\n(si > SDP → ignorer)"]
        CALC_IDX --> BUNDLE_MAIN["Parcourir le bundle principal :\npour chaque article\nconsommation[index_jour] += article.max × quantité_bundle"]
        BUNDLE_MAIN --> BUNDLE_OPT["Parcourir les bundles optionnels :\nconsommation[index_jour] += article.max × quantité_optionnel"]
        BUNDLE_OPT --> SAV_CHK["Intervention SAV ?\n→ Ajouter équipements SAV remplacés"]
    end

    LOOP_INT --> FILTER["Filtrer articles :\n- Exclure articles avec numéro de série\n- Exclure codes TAG\n- Exclure BAT (batteries)\n- Exclure CSM (consommables)"]

    FILTER --> APPLIMP["Appliquer facteur IMP :\nconsommation_finale[j] = ARRONDI_SUP(consommation[j] × (1 + IMP))"]

    APPLIMP --> PROJECT["Projeter le stock :\npour j de 1 à SDP :\n  stock_restant[j] = stock_restant[j-1] − consommation[j]\n  stock_restant[0] = stock_actuel"]

    PROJECT --> OUT(["Résultat :\nTechStockDailyConsumption\n(projection jour par jour)"])
```

**Règles spécifiques :**

| Règle | Description |
|-------|-------------|
| **RG-PROJ-01** | Les articles avec numéro de série individuel sont exclus de la projection (traités séparément) |
| **RG-PROJ-02** | Les badges TAG sont inclus mais avec un poids d'importance réduit (ImP = 10 vs 100) |
| **RG-PROJ-03** | Batteries (BAT) et consommables (CSM) sont exclus de la projection de stock |
| **RG-PROJ-04** | Le facteur IMP est appliqué article par article avec arrondi au supérieur |
| **RG-PROJ-05** | Si une intervention dépasse l'horizon SDP, elle est ignorée |
| **RG-PROJ-06** | La consommation SAV est calculée à partir des équipements réellement remplacés (1 unité par équipement) |

---

### 5.2 Scoring d'urgence et criticité

**Objectif :** Attribuer à chaque article d'un technicien un niveau de criticité permettant de prioriser les commandes. Le score est la somme de trois composantes.

#### Composantes du score

```mermaid
graph LR
    URQ["UrQ — Urgence Quantitative\n0 ou 100"]
    URT["UrT — Urgence Temporelle\n50, 100 ou 150"]
    IMP["ImP — Importance article\n100 (standard) ou 10 (TAG)"]

    URQ --> SCORE["Score total\n= UrQ + UrT + ImP"]
    URT --> SCORE
    IMP --> SCORE

    SCORE --> C1["≥ 300 → CRITICAL_A"]
    SCORE --> C2["≥ 250 → CRITICAL_B"]
    SCORE --> C3["≥ 210 → URGENT_A"]
    SCORE --> C4["≥ 160 → URGENT_B"]
    SCORE --> C5["< 160 → SAFE\n(pas de commande)"]
```

#### Calcul de l'urgence quantitative (UrQ)

```mermaid
flowchart TD
    Q1{"Le stock projeté\npasse-t-il sous\nle stock_min\ndans les SDP jours ?"}
    Q1 -->|"Non"| Q_NO["UrQ = 0\n(stock sécurisé)"]
    Q1 -->|"Oui"| Q_YES["UrQ = 100\n(stock insuffisant)"]

    Q_YES --> QTP["Calculer quantité à commander :\nqty_to_place = ARRONDI_SUP((stock_min + stock_max) / 2)\n              − stock_projeté[dernier_jour]"]
    QTP --> QTP_CHK{"qty_to_place > 0 ?"}
    QTP_CHK -->|"Non"| Q_SKIP["Article ignoré\n(stock suffisant en fin de période)"]
    QTP_CHK -->|"Oui"| Q_OK["Article éligible à la commande"]
```

#### Calcul de l'urgence temporelle (UrT)

*S'applique uniquement si UrQ = 100.*

```mermaid
flowchart TD
    UT_START["Trouver le 1er jour D\noù stock < stock_min"] --> UT_CHK1

    UT_CHK1{"D ≤ CDE ?\n(dans fenêtre criticité\njours 1 à 5)"}
    UT_CHK1 -->|"Oui"| UT_150["UrT = 150\nRupture IMMINENTE"]
    UT_CHK1 -->|"Non"| UT_CHK2

    UT_CHK2{"D ≥ UDS ET D ≤ UDE ?\n(dans fenêtre urgence\njours 6 à 8)"}
    UT_CHK2 -->|"Oui"| UT_100["UrT = 100\nRupture PROCHAINE"]
    UT_CHK2 -->|"Non"| UT_50["UrT = 50\nRupture LOINTAINE\n(jours 9 à SDP)"]
```

#### Fenêtres temporelles (valeurs par défaut)

```
Aujourd'hui  J+1   J+2   J+3   J+4   J+5   J+6   J+7   J+8   J+9 … J+15
             ├─────────────────────────┤  ╔══════════════╗  ╔───────────────╗
             │  Fenêtre CRITIQUE (UrT=150) ║ Urgence (UrT=100) ║  Faible (UrT=50)
             │  CDS=1 ────────── CDE=5 │  ╚UDS=6 ─── UDE=8╝  ╚───────────────╝
```

#### Règle de synchronisation SIM / GAT

| Règle | Description |
|-------|-------------|
| **RG-EMERG-01** | Si un technicien a à la fois un besoin en SIM et en GAT, la quantité à commander pour SIM est alignée sur celle de GAT (les deux articles sont toujours commandés en même nombre) |

#### Tableau de criticité

| Niveau | Score minimum | Priorité | Commande créée ? |
|--------|--------------|----------|-----------------|
| **CRITICAL_A** | 300 | 1 — Critique Grade A | Oui |
| **CRITICAL_B** | 250 | 2 — Critique Grade B | Oui |
| **URGENT_A** | 210 | 3 — Urgent Grade A | Oui |
| **URGENT_B** | 160 | 4 — Urgent Grade B | Oui |
| **SAFE** | < 160 | 5 — Sécurisé | Non |

> **Règle de seuil de commande (RG-EMERG-02)** : Seuls les articles avec un score ≥ 160 (URGENT_B ou supérieur) font l'objet d'une demande APP.

#### Exemples de scores

| UrQ | UrT | ImP | Score | Criticité | Interprétation |
|-----|-----|-----|-------|-----------|----------------|
| 100 | 150 | 100 | **350** → 300 cap | CRITICAL_A | Rupture dans 1-5j, article standard |
| 100 | 100 | 100 | **300** | CRITICAL_A | Rupture dans 6-8j, article standard |
| 100 | 50 | 100 | **250** | CRITICAL_B | Rupture dans 9-15j, article standard |
| 100 | 150 | 10 | **260** | CRITICAL_B | Rupture imminente, badge TAG |
| 100 | 100 | 10 | **210** | URGENT_A | Rupture prochaine, badge TAG |
| 100 | 50 | 10 | **160** | URGENT_B | Rupture lointaine, badge TAG |
| 0 | — | — | **0** | SAFE | Aucune rupture prévue |

---

### 5.3 Génération de la demande APP

**Objectif :** Construire, pour chaque technicien et chaque distributeur, une demande d'approvisionnement structurée avec la liste des articles et le nombre de boîtes nécessaires.

#### Structure d'une demande APP

```mermaid
graph TB
    APP["Demande APP"]
    APP --> INFO["Informations générales\n• Type : APP\n• Statut initial : AV (disponible)\n• Créateur : batch\n• Code technicien\n• Code offre distributeur"]
    APP --> BOITES["Nombre de boîtes\n(calculé par conditionnement)"]
    APP --> ARTICLES["Liste des articles\n(equipments)"]
    ARTICLES --> ART1["Article 1\n• Code article (ex: PIR)\n• Quantité à envoyer\n• Objet d'envoi : Envoi\n• Type requête : APP"]
    ARTICLES --> ART2["Article 2\n…"]
```

#### Algorithme de conditionnement en boîtes

```mermaid
flowchart TD
    INPUT["Articles urgents triés par priorité\n(CRITICAL_A > CRITICAL_B > URGENT_A > URGENT_B > SAFE)"]

    INPUT --> CALC_BOX["Calculer le nombre de boîtes nécessaires :\nnb_boites = ARRONDI_SUP(SOMME(quantité × ratio_occupation))"]

    CALC_BOX --> PLACE_PRIO["Placer les articles PRIORITAIRES\n(score ≥ 210 : CRITICAL_A, CRITICAL_B, URGENT_A)\nen round-robin sur les boîtes\n(du plus grand ratio au plus petit)"]

    PLACE_PRIO --> PLACE_URG["Placer les articles URGENTS\n(URGENT_B, score ≥ 160)\ndans l'espace restant des boîtes"]

    PLACE_URG --> PLACE_SAFE["Placer les articles SÉCURISÉS (SAFE)\ndans l'espace restant si disponible"]

    PLACE_SAFE --> OUT(["RequestDto finalisé\nprêt à soumettre"])
```

**Ratios d'occupation par article (exemples) :**

| Article | Ratio occupation boîte | Interprétation |
|---------|----------------------|----------------|
| SIM | 0,001 | Très faible encombrement |
| GAT | 0,125 | 12,5 % du volume d'une boîte |
| PIR | 0,010 | 1 % du volume d'une boîte |
| OSD | Configurable | Capteur de porte |
| TAG | Configurable | Badge RFID |
| CAE | Configurable | Terminal/clavier |

**Règles de conditionnement :**

| Règle | Description |
|-------|-------------|
| **RG-BOX-01** | Le nombre de boîtes est calculé par arrondi supérieur du volume total occupé par les articles prioritaires uniquement |
| **RG-BOX-02** | Les articles prioritaires (score ≥ 210) sont toujours placés en premier |
| **RG-BOX-03** | La distribution en boîtes se fait en round-robin pour équilibrer le chargement |
| **RG-BOX-04** | Les articles SAFE peuvent ne pas être inclus si les boîtes sont pleines |
| **RG-BOX-05** | Les articles URGENT_B et SAFE ne génèrent jamais de boîte supplémentaire — ils complètent uniquement l'espace restant |
| **RG-BOX-06** | Si aucune boîte n'a assez d'espace pour un article donné, celui-ci est abandonné (placement partiel autorisé) |

#### Critères de sélection des articles URGENT_B pour compléter les boîtes

Lorsque les articles prioritaires (CRITICAL_A, CRITICAL_B, URGENT_A) ont été placés, l'espace restant dans les boîtes est d'abord proposé aux articles **URGENT_B**. Ceux-ci sont triés par un **score composite** qui détermine l'ordre de placement :

```
score_URGENT_B = efficacité_occupation + écart_au_stock_minimum
```

| Composante | Formule | Logique métier |
|---|---|---|
| **Efficacité d'occupation** | `(1 / ratio_occupation) × poids_occupation` | Favorise les articles qui **prennent peu de place** dans la boîte, afin de maximiser le nombre d'articles insérés dans l'espace restant |
| **Écart au stock minimum** | `((stock_min − quantité_actuelle) / stock_min) × poids_écart` | Favorise les articles dont le stock est **le plus en dessous** du seuil minimum, donc les plus critiques à réapprovisionner |

Les deux **poids** (`poids_occupation` et `poids_écart`) sont configurables et permettent d'ajuster l'équilibre entre remplissage optimal des boîtes et urgence de réapprovisionnement.

```mermaid
flowchart TD
    START["Articles URGENT_B\n(score criticité ≥ 160 et < 210)"]

    START --> SCORE["Calculer le score composite :\nscore = efficacité_occupation + écart_stock_min"]

    SCORE --> SORT["Trier par score décroissant\n(meilleur candidat en premier)"]

    SORT --> LOOP["Pour chaque article (par unité)"]
    LOOP --> FIND{"Trouver une boîte\navec espace restant\n≥ ratio_occupation ?"}
    FIND -->|"Oui"| PLACE["Placer l'unité\n(round-robin sur les boîtes)"]
    PLACE --> LOOP
    FIND -->|"Non"| STOP["Arrêt — article abandonné\n(placement partiel possible)"]
```

**Exemple :** Entre un article PIR (ratio 0,010 ; stock à 50 % du min) et un article CAE (ratio 0,125 ; stock à 80 % du min), le PIR sera placé en priorité car il est plus petit et plus en dessous de son seuil.

#### Critères de sélection des articles SAFE pour compléter les boîtes

Si de l'espace reste après le placement des URGENT_B, les articles **SAFE** (score < 160, aucune rupture prévue) sont candidats au remplissage. Ils passent par un **pré-filtre** puis un **score de tri**.

**Pré-filtre — Éligibilité :**

| Étape | Règle | Formule |
|---|---|---|
| 1 | Calculer le stock cible | `stock_cible = (stock_min + stock_max) / 2` |
| 2 | Exclure les articles déjà au niveau cible | Retenir uniquement si `quantité_actuelle < stock_cible` |
| 3 | Calculer la quantité à placer | `quantité = stock_cible − quantité_actuelle` |

**Score de tri :**

```
score_SAFE = (quantité_actuelle − stock_cible) / stock_cible
```

| Valeur du score | Signification | Priorité de placement |
|---|---|---|
| Très négatif (ex : −0,8) | Stock très en dessous de la cible | **Haute** — placé en premier |
| Légèrement négatif (ex : −0,1) | Stock proche de la cible | **Basse** — placé en dernier |
| Zéro ou positif | Stock au niveau ou au-dessus de la cible | **Exclu** par le pré-filtre |

```mermaid
flowchart TD
    START["Articles SAFE\n(score criticité < 160)"]

    START --> FILTER["Pré-filtre :\nstock_cible = (stock_min + stock_max) / 2\nRetenir si quantité_actuelle < stock_cible"]

    FILTER --> CALC_QTY["Quantité à placer =\nstock_cible − quantité_actuelle"]

    CALC_QTY --> SCORE["Score = (quantité_actuelle − stock_cible) / stock_cible\n(plus négatif = plus prioritaire)"]

    SCORE --> SORT["Trier par score croissant\n(plus grand écart en premier)"]

    SORT --> LOOP["Pour chaque article (par unité)"]
    LOOP --> FIND{"Trouver une boîte\navec espace restant\n≥ ratio_occupation ?"}
    FIND -->|"Oui"| PLACE["Placer l'unité\n(round-robin sur les boîtes)"]
    PLACE --> LOOP
    FIND -->|"Non"| STOP["Arrêt — plus d'espace\ndans aucune boîte"]
```

**Exemple :** Un technicien a un stock de PIR à 10 unités (cible : 22). Score = (10 − 22) / 22 = −0,55. Un autre article TAG est à 20 unités (cible : 19) → exclu car déjà au-dessus de la cible. Le PIR sera placé si de l'espace reste dans les boîtes.

---

## 6. Paramètres configurables

Ces paramètres sont stockés en base de données (table `supply_parameter`) et modifiables sans redéploiement.

```mermaid
graph LR
    subgraph PARAMS["Paramètres supply_parameter"]
        SDP["SDP = 15 jours\nHorizon d'analyse"]
        IMP_P["IMP = 0\nFacteur d'imprévu"]
        CDS["CDS = 1 jour\nDébut fenêtre critique"]
        CDE["CDE = 5 jours\nFin fenêtre critique"]
        UDS["UDS = 6 jours\nDébut fenêtre urgente"]
        UDE["UDE = 8 jours\nFin fenêtre urgente"]
    end

    SDP -->|"influe sur"| PROJ["Projection de stock"]
    IMP_P -->|"influe sur"| PROJ
    CDS & CDE -->|"définissent"| CRIT_WIN["Fenêtre critique\nUrT = 150"]
    UDS & UDE -->|"définissent"| URG_WIN["Fenêtre urgente\nUrT = 100"]
```

| Code | Description | Valeur défaut | Impact d'une modification |
|------|-------------|---------------|--------------------------|
| **SDP** | Horizon d'analyse (jours) | 15 | Plus grand → plus d'interventions prises en compte → plus de besoins détectés |
| **IMP** | Facteur d'imprévu (0 à 1) | 0 (0 %) | 0,15 → +15 % sur les quantités prévisionnelles, commandes plus importantes |
| **CDS** | Début fenêtre critique (jours) | 1 | Avancer → élargit la zone à UrT=150 |
| **CDE** | Fin fenêtre critique (jours) | 5 | Reculer → élargit la zone à UrT=150 |
| **UDS** | Début fenêtre urgente (jours) | 6 | Doit être CDE+1 pour cohérence |
| **UDE** | Fin fenêtre urgente (jours) | 8 | Au-delà → UrT=50 (faible urgence) |

---

## 7. Référentiels articles et contraintes

### Types d'articles traités

| Code | Description | Imprévu (ImP) | Suivi sériel |
|------|-------------|---------------|-------------|
| **SIM** | Carte SIM | 100 | Non |
| **GAT** | Passerelle GSM | 100 | Non |
| **PIR** | Détecteur de mouvement | 100 | Non |
| **TAG** | Badge RFID | **10** | Non |
| **CAE** | Terminal / clavier | 100 | Non |
| **OSD** | Détecteur porte/optique | 100 | Non |
| **REM** | Télécommande | 100 | Non |

> Les articles à suivi sériel sont exclus de la projection de stock automatique.

### Contraintes par article et par offre

Les seuils min/max de stock sont définis par couple (article, offre distributeur) :

| Article | ORANGE (GBH2) — min/max | GROUPAMA (MPO2) — min/max |
|---------|------------------------|--------------------------|
| SIM | 12 / 20 | 12 / 24 |
| GAT | 12 / 20 | 12 / 24 |
| PIR | 17 / 28 | 24 / 48 |
| TAG | 14 / 24 | 24 / 48 |
| CAE | 2 / 4 | 3 / 6 |
| OSD | 32 / 54 | 35 / 70 |
| REM | 4 / 6 | 1 / 3 |

> **RG-CONTRAINTE-01** : La quantité à commander vise le milieu de la fourchette min/max, calculée à partir du stock projeté en fin de période.

---

## 8. Flux fonctionnels end-to-end

### Vue d'ensemble du flux complet

```mermaid
flowchart TD
    TRIGGER(["Déclenchement\n(batch ou API)"])

    subgraph COLLECT["Phase 1 — Collecte des données (parallèle)"]
        C1["Techniciens actifs\n← op-technician"]
        C2["Offres distributeurs\n← teamtool-install"]
        C3["Demandes APP en attente\n← op-request"]
        C4["Interventions planifiées\n← op-device"]
        C5["Articles requis + SAV\n← teamtool-install"]
        C6["Stocks actuels techniciens\n← op-device"]
    end

    subgraph CALC["Phase 2 — Calcul des besoins"]
        PROJ["Projection de stock\nquotidien par technicien\n(SDP jours)"]
    end

    subgraph ASSESS["Phase 3 — Évaluation d'urgence"]
        SCORE["Scoring urgence\n(UrQ + UrT + ImP)"]
        CLASS["Classification\n(CRITICAL_A → SAFE)"]
        SCORE --> CLASS
    end

    subgraph GENERATE["Phase 4 — Génération des APP"]
        GROUP["Grouper par\ntechnicien / distributeur"]
        BOX["Calculer conditionnement\n(boîtes)"]
        BUILD["Construire RequestDto"]
        GROUP --> BOX --> BUILD
    end

    subgraph PERSIST["Phase 5 — Persistance et soumission"]
        SAVE["Sauvegarder proposition\nen base de données"]
        SUBMIT["Soumettre APP\nà op-request"]
    end

    TRIGGER --> COLLECT
    C1 & C2 & C3 & C4 & C5 & C6 --> CALC
    CALC --> ASSESS
    ASSESS --> GENERATE
    GENERATE --> PERSIST
```

### Flux de décision par article

```mermaid
flowchart TD
    ART["Article d'un technicien"]

    ART --> CHK_SERIAL{"Article avec\nnuméro de série ?"}
    CHK_SERIAL -->|"Oui"| EXCL["Exclu de la projection\nautomatique"]
    CHK_SERIAL -->|"Non"| CHK_BAT{"BAT ou CSM ?"}
    CHK_BAT -->|"Oui"| EXCL
    CHK_BAT -->|"Non"| PROJ["Inclus dans la\nprojection de stock"]

    PROJ --> CHK_MIN{"Stock projeté\n< stock_min ?"}
    CHK_MIN -->|"Non"| SAFE_OUT["SAFE — Pas de commande"]
    CHK_MIN -->|"Oui"| CHK_QTY{"qty_to_place > 0 ?"}
    CHK_QTY -->|"Non"| SAFE_OUT
    CHK_QTY -->|"Oui"| CALC_SCORE["Calculer score\n(UrQ + UrT + ImP)"]

    CALC_SCORE --> CHK_SCORE{"Score ≥ 160 ?"}
    CHK_SCORE -->|"Non"| SAFE_OUT
    CHK_SCORE -->|"Oui"| INCLUDE["Inclus dans\nla demande APP"]

    INCLUDE --> PRIO{"Niveau\nde criticité"}
    PRIO -->|"≥ 210"| PRIO_BOX["Article prioritaire\n(placé en premier)"]
    PRIO -->|"160-209"| URG_BOX["Article urgent\n(placé en second)"]
```

---

## 9. Processus automatique (batch)

Le scheduler exécute quotidiennement le cycle complet de réapprovisionnement selon le cron configuré dans `opreplenishment.scheduler.clear-create-auto-app.cron`.

```mermaid
sequenceDiagram
    autonumber
    participant CRON as Scheduler (Cron)
    participant AUTH as Authentification Keycloak
    participant CLEAN as Nettoyage APP existantes
    participant ORCH as Orchestration complète
    participant DB as Base de données
    participant DEST as op-request

    CRON->>AUTH: Obtenir un token de service (client_credentials)
    AUTH-->>CRON: Bearer JWT valide

    CRON->>CLEAN: Supprimer toutes les APP en attente
    CLEAN-->>CRON: OK (table nettoyée)

    note over CRON,DEST: Lancement de l'orchestration complète

    CRON->>ORCH: calculate(aujourd'hui+1, aujourd'hui+SDP)
    ORCH-->>DB: Sauvegarder les propositions
    ORCH-->>DEST: Créer les nouvelles APP

    ORCH-->>CRON: Liste des APP créées
```

**Règles du batch :**

| Règle | Description |
|-------|-------------|
| **RG-BATCH-01** | Le batch efface TOUTES les APP précédentes avant d'en créer de nouvelles |
| **RG-BATCH-02** | La période analysée commence le lendemain (J+1) jusqu'à J+SDP |
| **RG-BATCH-03** | Le batch s'authentifie de manière autonome via Keycloak (client_credentials) |
| **RG-BATCH-04** | En cas d'échec d'un appel externe, Camel tente une nouvelle fois avant de basculer en dead-letter |

---

## 10. Impacts et limites fonctionnelles

### Règles de cohérence

| Règle | Description |
|-------|-------------|
| **RG-COH-01** | SIM et GAT sont toujours commandés en même quantité pour un même technicien |
| **RG-COH-02** | Une APP n'est créée que si au moins un article a un score ≥ 160 |
| **RG-COH-03** | Les contraintes (stock_min, stock_max, ImP) dépendent de l'offre distributeur : un même article peut avoir des seuils différents selon le distributeur |
| **RG-COH-04** | Le facteur IMP global (paramètre SDP) peut être surchargé par un facteur individuel par technicien |
| **RG-COH-05** | Les interventions dont la date est au-delà de l'horizon SDP sont ignorées dans le calcul |

### Limites connues

| Limite | Description |
|--------|-------------|
| Pas de commande partielle | Si le score d'un article est < 160, l'article n'est pas commandé même si son stock est proche du minimum |
| Articles sériels exclus | Les articles suivis individuellement (numéro de série) ne sont pas couverts par le réapprovisionnement automatique |
| Horizon fixe | La projection est limitée à SDP jours ; les interventions au-delà ne sont pas anticipées |
| Dépendance aux données externes | La qualité du calcul dépend entièrement de l'exactitude des stocks fournis par op-device et des interventions planifiées |
| Batch destructif | Le batch supprime TOUTES les APP en attente avant de les recréer ; toute APP modifiée manuellement entre deux exécutions sera perdue |

---

## Incohérences et anomalies détectées

> Anomalies entre les règles fonctionnelles documentées et le comportement réel du code ou du système.

---

### IC-FONC-01 — Le seuil de score 160 n'est pas paramétrable

**Règle documentée :** Un article est commandé si son score total (UrQ + UrT + ImP) ≥ 160.

**Code réel :** La valeur `160` est codée en dur dans la logique de scoring. Elle n'est pas exposée comme paramètre dans la table `supply_parameter` et ne peut pas être ajustée sans déploiement.

**Impact :** Impossible d'adapter le seuil de déclenchement selon le contexte métier (saisonnalité, stratégie commerciale) sans modifier et redéployer le code.

---

### IC-FONC-02 — Les statuts de demande diffèrent entre le domaine supply-core et les routes d'orchestration

**Règle documentée :** Les demandes de réapprovisionnement suivent un cycle de vie défini (À Faire → Validée → etc.).

**Code réel :** supply-core crée des demandes en statut `AV` (À Valider), mais les routes Camel d'orchestration filtrent sur le statut `F` (À Faire). Une demande créée par supply-core ne sera jamais traitée par les routes d'orchestration puisque son statut (`AV`) ne correspond pas au filtre attendu (`F`).

**Impact :** Blocage systématique du flux de réapprovisionnement automatique — les demandes générées par le calcul ne déclenchent jamais les routes d'expédition.

---

### IC-FONC-03 — Le facteur IMP individuel par technicien n'est pas décrit dans les règles métier

**Règle documentée :** Le paramètre `IMP` global est configurable.

**Code réel :** Un facteur `IMP` individuel peut surcharger le facteur global pour un technicien spécifique via la table de paramètres. Cette surcharge n'est mentionnée dans aucune règle métier documentée (RG-CDS, RG-UDS etc.) et n'a aucun processus de gestion décrit (qui peut le modifier ? avec quelles contraintes ?).

**Impact :** Comportement de scoring opaque — deux techniciens avec les mêmes stocks et interventions peuvent avoir des propositions différentes sans que l'utilisateur comprenne pourquoi.

---

### IC-FONC-04 — Le batch détruit les APP manuelles sans avertissement

**Règle documentée :** Le batch supprime toutes les APP avant de les recréer (mentionné dans les limites).

**Impact non documenté :** Si un logisticien modifie manuellement une APP (ajoute un article, change une quantité) entre deux exécutions du scheduler, sa modification est perdue sans aucun avertissement. Aucun mécanisme de verrouillage ("APP en cours de modification") ni de notification n'est prévu.

**Impact :** Perte silencieuse de modifications manuelles. Le logisticien peut croire que son ajustement est pris en compte alors qu'il a été écrasé.

---

### IC-FONC-05 — Les interventions passées proches de la limite SDP gonflent artificiellement le score

**Règle documentée :** Seules les interventions dans l'horizon SDP (N prochains jours) sont prises en compte.

**Code réel :** Le filtre temporel applique une fenêtre `[aujourd'hui, aujourd'hui + SDP]`. Les interventions dont la date est **dans le passé récent** mais enregistrées dans le système avec une date future ne sont pas exclues. De plus, si SDP est grand (ex: 30 jours), des interventions déjà réalisées (date passée) mais encore dans la fenêtre sont comptabilisées, gonflant le score inutilement.

**Impact :** Sur-estimation du besoin de réapprovisionnement sur des articles dont les interventions prévues ont déjà eu lieu.

---

### IC-FONC-06 — Absence de gestion des retours d'équipements dans le calcul de besoin

**Règle documentée :** Le calcul tient compte du stock actuel pour chaque technicien.

**Impact non documenté :** Les équipements en cours de retour (statut `R` ou `RE` dans teamtool-logistique) ne sont pas pris en compte dans le stock projeté. Un technicien qui a initié un retour de 3 centrales apparaît comme en ayant encore 3 dans son stock, ce qui sous-estime son besoin réel de réapprovisionnement.

**Impact :** Calcul de besoin inexact pour les techniciens avec des retours en cours. Les APP générées peuvent être insuffisantes.

---

### IC-FONC-07 — Aucune règle métier documentée sur la gestion des erreurs Chronopost en batch

**Règle documentée :** Le batch génère des APP et des bons Chronopost.

**Code réel :** Si la génération Chronopost échoue pour un technicien (service SOAP indisponible, adresse invalide), le comportement n'est pas documenté. Le batch continue-t-il pour les autres techniciens ? La demande est-elle créée sans Chronopost ? L'erreur est-elle notifiée ?

**Impact :** En cas de panne Chronopost partielle pendant le batch nocturne, certains techniciens reçoivent leur APP et d'autres non, sans que la logistique en soit informée.
