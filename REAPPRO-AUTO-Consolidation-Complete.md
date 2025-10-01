# 🚀 SYSTÈME DE RÉAPPROVISIONNEMENT AUTOMATIQUE - Documentation Consolidée

**Version :** 1.0
**Date :** 2025-09-30
**Tickets Jira :** TOP-6187, TOP-6188, TOP-6XXX

---

## 📋 Table des Matières

1. [Vue d'Ensemble du Système](#vue-densemble-du-système)
2. [Module 1 : Calcul de Besoin](#module-1--calcul-de-besoin)
3. [Module 2 : Évaluation d'Urgence](#module-2--évaluation-durgence)
4. [Module 3 : Core Moteur d'Optimisation](#module-3--core-moteur-doptimisation)
5. [Architecture Globale](#architecture-globale)
6. [Flux de Données Complet](#flux-de-données-complet)
7. [Règles de Gestion Transversales](#règles-de-gestion-transversales)
8. [Critères d'Acceptation Consolidés](#critères-dacceptation-consolidés)

---

## 🎯 1. Vue d'Ensemble du Système

### Objectif Global
Mettre en place un système de réapprovisionnement automatique des techniciens avec optimisation mathématique pour :
- ✅ **Garantir zéro rupture** pour les matériels critiques
- ✅ **Optimiser le nombre de colis** tout en respectant les priorités métier
- ✅ **Anticiper les besoins** sur un horizon temporel paramétrable
- ✅ **Prioriser automatiquement** selon plusieurs critères d'urgence

### Architecture 3 Modules

```mermaid
graph TB
    subgraph " 📊 MODULE 1: CALCUL DE BESOIN "
        M1[Calcul Stock Initial J0] --> M2[Calcul Consommation Prévisionnelle]
        M2 --> M3[Projection Stock jour par jour]
        M3 --> OUTPUT1[Matrice Projection Stock]
    end

    subgraph " 🚨 MODULE 2: ÉVALUATION URGENCE "
        OUTPUT1 --> E1[Calcul Urgence Quantitative UrQ]
        OUTPUT1 --> E2[Calcul Urgence Temporelle UrT]
        E1 --> E3[Calcul Importance ImP]
        E2 --> E3
        E3 --> E4[Urgence Totale UrTT]
        E4 --> E5[Classification Criticité]
        E5 --> OUTPUT2[Articles Classifiés par Grade]
    end

    subgraph " 🎯 MODULE 3: OPTIMISATION COLIS "
        OUTPUT2 --> O1[Classification Articles]
        O1 --> O2[Phase 1: Court-Circuit Critiques]
        O2 --> O3[Phase 2: Complétion URGENT_B]
        O3 --> O4[Phase 3: Knapsack SAFE]
        O4 --> O5[Phase 4: Validation]
        O5 --> OUTPUT3[Proposition APP Optimisée]
    end

    style OUTPUT1 fill:#e1f5fe
    style OUTPUT2 fill:#fff8e1
    style OUTPUT3 fill:#e8f5e9
```

### Niveaux de Criticité

| Grade | Score UrTT | Description | Traitement |
|-------|-----------|-------------|------------|
| **CRITIQUE_A** | 300 | Scannable rupture imminente J0-J5 | Court-circuit Phase 1 |
| **CRITIQUE_B** | 210 | Déclarable/Consommable rupture J0-J5 | Court-circuit Phase 1 |
| **URGENT_A** | 250 | Scannable rupture modérée J6-J8 | Court-circuit Phase 1 |
| **URGENT_B** | 160 | Déclarable/Consommable rupture J6-J8 | Complétion Phase 2 |
| **SAFE** | 0 | Pas de criticité détectée | Knapsack Phase 3 |

---

## 📊 2. MODULE 1 : Calcul de Besoin

### 2.1 Contexte et Objectif

**En tant que** logisticien
**Je souhaite** mettre en place un module de calcul de besoin en matériel pour chaque technicien dans un horizon de X jours
**Afin de** pouvoir créer un APP en cohérence avec le besoin réel du technicien

### 2.2 Architecture du Module

```mermaid
sequenceDiagram
    participant SYS as Système Principal
    participant ALG2 as Algorithme Stock Initial
    participant ALG1 as Algorithme Consommation
    participant ALG3 as Algorithme Projection
    participant DB as Base de Données

    Note over SYS: Début Calcul Besoin - Horizon X jours

    SYS->>ALG2: Initialiser calcul stock initial
    ALG2->>DB: Récupérer stock_actuel
    ALG2->>DB: Récupérer colis en transit
    ALG2->>DB: Récupérer colis pending
    DB-->>ALG2: Données stock
    ALG2->>ALG2: stock_initial = stock_actuel + transit + pending
    ALG2-->>SYS: Matrice stock_projection[articles][J0]

    Note over SYS: Stock Initial Calculé

    SYS->>ALG1: Calculer consommation prévisionnelle
    loop Pour chaque technicien
        loop Jour J+1 à J+search_depth
            ALG1->>DB: Lister interventions du jour
            DB-->>ALG1: Liste interventions
            loop Pour chaque intervention
                ALG1->>ALG1: Articles requis + QTE_Brute
            end
            ALG1->>ALG1: Splitter GBH vs ORG
            ALG1->>ALG1: QTE_Brute_Final = ARRONDI_SUP(QTE_Brute × Imprev_Fact)
        end
    end
    ALG1-->>SYS: Matrice consommation[articles][jours]

    Note over SYS: Consommation Prévisionnelle Calculée

    SYS->>ALG3: Lancer projection stock
    loop Pour chaque article
        loop Jour 1 à search_depth
            ALG3->>ALG1: CalculerConsommationPrevue(article, jour)
            ALG1-->>ALG3: consommation_prevue
            ALG3->>ALG3: stock[jour] = stock[jour-1] - consommation_prevue
        end
    end
    ALG3-->>SYS: Matrice projection complète

    Note over SYS: Projection Terminée
```

### 2.3 Règles de Gestion

#### **RG01 - Calcul Stock Initial (J0)**

```pseudocode
FONCTION StockInitial(liste_articles_types, periode_jours)
DEBUT
    stock_projection = MATRICE[TAILLE(liste_articles_types), periode_jours + 1]

    POUR i DE 0 A TAILLE(liste_articles_types) FAIRE
        article_type = liste_articles_types[i]

        stock_initial = article_type.stock_actuel +
                       ObtenirQuantiteColisEnTransit(article_type) +
                       ObtenirQuantiteColisPending(article_type)

        stock_projection[i][0] = stock_initial
    FIN_POUR

    RETOURNER stock_projection
FIN
```

**Précisions :**
- ✅ Stock initial calculé pour chaque type d'article dans la BDD
- ❌ **Exclusions** : guides, stickers (signalétiques), piles (lot ultérieur)
- ✅ APP statut "En transit" → comptabilisées dans stock J0
- ✅ APP "Pending" (créées non expédiées) → comptabilisées dans stock J0

#### **RG02 - Calcul de Besoin avec Facteur d'Imprévu**

**Paramètres Configurables :**

| Paramètre | Description | Calcul | Exemple |
|-----------|-------------|--------|---------|
| `search_depth` | Horizon temporel d'analyse | Paramètre Consul | 15 jours |
| `Imprev_Fact` | Facteur d'imprévu/incertitude | `1 + (nbr_heures_libre / totale_heures_shift)` | 1.2 pour 24h libres/120h |

**Formule de Correction :**
```
QTE_BRUTE_FINALE = ARRONDI_SUP(QTE_BRUTE × Imprev_Fact)
```

**Exemple :**
- QTE_BRUTE = 12 unités
- Imprev_Fact = 1.6
- QTE_BRUTE_FINALE = ARRONDI_SUP(12 × 1.6) = ARRONDI_SUP(19.2) = **20 unités**

**Calcul du Facteur d'Imprévu (Lot 1) :**

```pseudocode
Imprev_Fact = 1 + (nbr_heures_libre / totale_heures_shift_dispo)

Exemple :
- Shift: 09h-19h → 10h/jour
- search_depth: 15 jours
- Jours ouvrables: 12 jours
- totale_heures_shift: 12 × 10 = 120h
- heures_libres_continues ≥ 1h: 24h détectées
- Imprev_Fact = 1 + (24/120) = 1.2

Si aucune heure libre continue ≥ 1h:
- Imprev_Fact = 1.0
```

**Output Structure :**
```
Besoin ORG:
  Article_Type1: QTE_Brute_Final_1
  Article_Type2: QTE_Brute_Final_2
  ...

Besoin GBH:
  Article_Type1: QTE_Brute_Final_1
  Article_Type2: QTE_Brute_Final_2
  ...
```

**Notes Importantes :**
- ✅ 1 Centrale = 1 Carte SIM (associés)
- ✅ 1 Cam EXT = 1 Panneau solaire (associés)
- ✅ Seuls les SS, SAV, EX sont analysés (pas les VE)

#### **RG03 - Projection Stock Jour par Jour**

```pseudocode
FONCTION ProjectionStockInitial(liste_articles_types, periode_jours)
DEBUT
    POUR i DE 0 A TAILLE(liste_articles_types) - 1 FAIRE
        article_type = liste_articles_types[i]

        POUR jour DE 1 A periode_jours FAIRE
            consommation_prevue = CalculerConsommationPrevue(article_type, jour)

            stock_projection[i][jour] = stock_projection[i][jour-1] - consommation_prevue
        FIN_POUR
    FIN_POUR

    RETOURNER stock_projection
FIN
```

**Exemple de Projection :**
```
Article avec stock_initial = 25 unités
Consommation_prevue: [3, 1, 4, 2, 0, 5, 1]

Résultat:
J0 = 25
J1 = 25 - 3 = 22
J2 = 22 - 1 = 21
J3 = 21 - 4 = 17
J4 = 17 - 2 = 15
J5 = 15 - 0 = 15
J6 = 15 - 5 = 10
J7 = 10 - 1 = 9
```

---

## 🚨 3. MODULE 2 : Évaluation d'Urgence

### 3.1 Contexte et Objectif

**En tant que** DSI
**Je souhaite** évaluer le niveau d'urgence des besoins matériels identifiés
**Afin de** prioriser les réapprovisionnements selon plusieurs critères d'urgence et classifier automatiquement les matériels par niveau de criticité

### 3.2 Architecture du Module

```mermaid
flowchart TD
    START[📊 Matrice Projection Stock] --> LOOP{Pour chaque Article}

    LOOP --> URQ[🔍 Calcul Urgence Quantitative UrQ]
    URQ --> CHECK_URQ{UrQ > 0 ?}

    CHECK_URQ -->|Non| SAFE[✅ Article SAFE]
    CHECK_URQ -->|Oui| URT[⏰ Calcul Urgence Temporelle UrT]

    URT --> IMP[⭐ Déterminer Importance ImP]

    IMP --> URTT[🎯 Calcul Urgence Totale UrTT]
    URTT --> CLASS[📋 Classification Grade]

    CLASS --> CRIT_A{UrTT = 300?}
    CLASS --> URG_A{UrTT = 250?}
    CLASS --> CRIT_B{UrTT = 210?}
    CLASS --> URG_B{UrTT = 160?}

    CRIT_A -->|Oui| GRADE_CA[🔴 CRITIQUE_A]
    URG_A -->|Oui| GRADE_UA[🟠 URGENT_A]
    CRIT_B -->|Oui| GRADE_CB[🟡 CRITIQUE_B]
    URG_B -->|Oui| GRADE_UB[🟢 URGENT_B]

    SAFE --> OUTPUT[📋 Output Classifié]
    GRADE_CA --> OUTPUT
    GRADE_UA --> OUTPUT
    GRADE_CB --> OUTPUT
    GRADE_UB --> OUTPUT

    OUTPUT --> LOOP

    style CRIT_A fill:#ff6b6b
    style URG_A fill:#ffa07a
    style CRIT_B fill:#ffd93d
    style URG_B fill:#95e1d3
    style SAFE fill:#a8dadc
```

### 3.3 Algorithme Principal

```pseudocode
FONCTION CalculerUrgencesMateriels(liste_materiels)
DEBUT
    resultats = LISTE_VIDE()

    POUR CHAQUE materiel DANS liste_materiels FAIRE
        // 1. Projection stock (issue Module 1)
        projections = ProjectionStock(materiel, search_depth)

        // 2. Urgence quantitative pour chaque jour
        urgences_quanti = TABLEAU taille search_depth
        POUR jour DE 1 A search_depth FAIRE
            SI projections[jour] <= materiel.stock_min ALORS
                urgences_quanti[jour] = 100
            SINON
                urgences_quanti[jour] = 0
            FIN_SI
        FIN_POUR

        // 3. Urgence temporelle
        premier_jour_critique = TrouverPremiereRupture(projections, materiel.stock_min)
        urgence_tempo = CalculerUrgenceTemporelle(premier_jour_critique)

        // 4. Importance
        importance = DeterminerImportance(materiel.type)

        // 5. Urgence totale avec court-circuit
        SI urgence_tempo = 0 OU TOUS(urgences_quanti) = 0 ALORS
            urgence_totale = 0
        SINON
            urgence_totale = urgence_tempo + MAX(urgences_quanti) + importance
        FIN_SI

        // 6. Stockage résultat
        resultat = CréerMaterielAvecUrgence(materiel, urgence_totale,
                                           urgences_quanti, urgence_tempo, importance)
        AJOUTER resultat A resultats
    FIN_POUR

    // 7. Tri par urgence décroissante
    TRIER resultats PAR urgence_totale DÉCROISSANT

    RETOURNER resultats
FIN
```

#### **Diagramme de Séquence : CalculerUrgencesMateriels**

```mermaid
sequenceDiagram
    participant CALLER as Appelant
    participant CALC as CalculerUrgencesMateriels
    participant PROJ as ProjectionStock
    participant TEMPO as CalculerUrgenceTemporelle
    participant IMP as DeterminerImportance
    participant RESULT as Liste Résultats

    CALLER->>CALC: CalculerUrgencesMateriels(liste_materiels)
    CALC->>RESULT: Créer liste_vide()

    loop Pour chaque matériel
        Note over CALC: Étape 1: Projection Stock
        CALC->>PROJ: ProjectionStock(materiel, search_depth)
        PROJ-->>CALC: projections[1...search_depth]

        Note over CALC: Étape 2: Urgence Quantitative
        loop Pour jour = 1 à search_depth
            alt stock_projection[jour] ≤ stock_min
                CALC->>CALC: urgences_quanti[jour] = 100
            else stock_projection[jour] > stock_min
                CALC->>CALC: urgences_quanti[jour] = 0
            end
        end

        Note over CALC: Étape 3: Urgence Temporelle
        CALC->>CALC: TrouverPremiereRupture(projections, stock_min)
        CALC->>TEMPO: CalculerUrgenceTemporelle(premier_jour_critique)

        alt Rupture J1-J5 (imminent)
            TEMPO-->>CALC: urgence_tempo = 100
        else Rupture J6-J8 (modéré)
            TEMPO-->>CALC: urgence_tempo = 50
        else Rupture J9+ ou pas de rupture
            TEMPO-->>CALC: urgence_tempo = 0
        end

        Note over CALC: Étape 4: Importance
        CALC->>IMP: DeterminerImportance(materiel.type)

        alt Scannable (quotidien)
            IMP-->>CALC: importance = 100
        else Déclarable/Consommable
            IMP-->>CALC: importance = 10
        end

        Note over CALC: Étape 5: Urgence Totale (Court-Circuit)
        alt urgence_tempo = 0 OU tous urgences_quanti = 0
            CALC->>CALC: urgence_totale = 0 (SAFE)
            Note right of CALC: Court-circuit:<br/>Pas de criticité
        else Sinon
            CALC->>CALC: urgence_totale = urgence_tempo<br/>+ MAX(urgences_quanti)<br/>+ importance
            Note right of CALC: Valeurs possibles:<br/>160, 210, 250, 300
        end

        Note over CALC: Étape 6: Créer Résultat
        CALC->>RESULT: Ajouter materiel_avec_urgence(urgence_totale,<br/>urgences_quanti, urgence_tempo, importance)
    end

    Note over CALC: Étape 7: Tri par Urgence
    CALC->>RESULT: Trier par urgence_totale DÉCROISSANT

    RESULT-->>CALLER: Liste matériels classifiés<br/>(CRITIQUE_A, URGENT_A,<br/>CRITIQUE_B, URGENT_B, SAFE)
```

**Légende des valeurs d'urgence_totale :**
- **300** = CRITIQUE_A (scannable, rupture J1-J5)
- **250** = URGENT_A (scannable, rupture J6-J8)
- **210** = CRITIQUE_B (déclarable, rupture J1-J5)
- **160** = URGENT_B (déclarable, rupture J6-J8)
- **0** = SAFE (pas de rupture ou rupture J9+)

---

### 3.4 Règles de Gestion

#### **RG1 - Urgence Quantitative (UrQ)**

**Formule :**
```
Pour chaque jour j de 1 à search_depth:

urgence_quantitative[jour_j] = {
    100  si stock_projection[jour_j] ≤ stock_min
    0    sinon
}

UrQ = MAX(urgence_quantitative[jour_1...jour_search_depth])
```

**Exemple :**
```
Stock Initial J0: 25 DO
Stock minimum: 16 DO
Consommation: [3, 1, 4, 2, 0, 5, 10, 1, 1, 2]

Projection:
J0: 25 → J1: 22 → J2: 21 → J3: 18 → J4: 16 → J5: 16
→ J6: 11 → J7: 1 → J8: 0 → J9: -1 → J10: -3

Urgence quantitative résultante par jour:
[0, 0, 0, 100, 100, 100, 100, 100, 100, 100]

UrQ = MAX = 100
```

#### **RG2 - Urgence Temporelle (UrT)**

```pseudocode
FONCTION CalculerUrgenceTemporelle(stock_projection, stock_min, search_depth)
DEBUT
    // Trouver premier jour de rupture
    premier_jour_critique = -1

    POUR jour DE 1 A search_depth FAIRE
        SI stock_projection[jour] <= stock_min ALORS
            premier_jour_critique = jour
            SORTIR_BOUCLE
        FIN_SI
    FIN_POUR

    // Calculer urgence selon fenêtre temporelle
    SI premier_jour_critique = -1 ALORS
        RETOURNER 0  // Pas de rupture
    SINON_SI premier_jour_critique >= 1 ET premier_jour_critique <= 5 ALORS
        RETOURNER 100  // Rupture imminente J0-J5
    SINON_SI premier_jour_critique >= 6 ET premier_jour_critique <= 8 ALORS
        RETOURNER 50   // Rupture modérée J6-J8
    SINON
        RETOURNER 0    // Rupture lointaine J9-J10
    FIN_SI
FIN
```

**Table de Décision :**

| Premier Jour Critique | Fenêtre | UrT | Interprétation |
|----------------------|---------|-----|----------------|
| J1 - J5 | Imminent | **100** | Rupture critique imminente |
| J6 - J8 | Modéré | **50** | Rupture modérée proche |
| J9 - J10 | Lointain | **0** | Rupture lointaine acceptable |
| Pas de rupture | - | **0** | Stock suffisant |

**⚠️ Optimisation :** Le calcul de UrT ne se fait **que si UrQ = 100**

#### **RG3 - Importance du Matériel (ImP)**

**Règle Lot 1 :**

| Type Matériel | Importance (ImP) | Justification |
|--------------|------------------|---------------|
| **Scannables** | **100** | Utilisés quotidiennement SS/SAV/EX |
| **Déclarables/Consommables** | **10** | Moins critiques opérationnellement |

**Note :** Les piles ne sont pas gérées dans le lot 1.

**Lot 2 (Future):** Valorisation dynamique selon critères plus pertinents.

#### **RG4 - Urgence Totale (UrTT)**

**Formule avec Court-Circuit :**
```
UrTT(materiel) = {
    0                                      si UrT × UrQ = 0 (court-circuit)
    UrT + UrQ + ImP                        sinon
}

Avec:
• UrT ∈ {0, 50, 100}
• UrQ ∈ {0, 100}
• ImP ∈ {10, 100}

Donc: UrTT ∈ {0, 160, 210, 250, 300}
```

**Justification du Court-Circuit :**
1. **UrT = 0** (rupture > J8 ou pas de rupture) → Horizon temporel acceptable
2. **UrQ = 0** (stock suffisant tous les jours) → Pas de criticité quantitative

#### **RG5 - Classification Finale**

```pseudocode
FONCTION ClassifierArticle(UrTT)
DEBUT
    SELON UrTT FAIRE
        CAS 300:
            RETOURNER "CRITIQUE_GRADE_A"
        CAS 250:
            RETOURNER "URGENT_GRADE_A"
        CAS 210:
            RETOURNER "CRITIQUE_GRADE_B"
        CAS 160:
            RETOURNER "URGENT_GRADE_B"
        DÉFAUT:
            RETOURNER "SAFE"
    FIN_SELON
FIN
```

**Table de Classification Complète :**

| Grade | Score UrTT | UrT | UrQ | ImP | Interprétation Métier |
|-------|-----------|-----|-----|-----|----------------------|
| **CRITIQUE_A** | 300 | 100 | 100 | 100 | Scannable rupture imminente J0-J5 |
| **URGENT_A** | 250 | 50 | 100 | 100 | Scannable rupture modérée J6-J8 |
| **CRITIQUE_B** | 210 | 100 | 100 | 10 | Déclarable/Consommable rupture J0-J5 |
| **URGENT_B** | 160 | 50 | 100 | 10 | Déclarable/Consommable rupture J6-J8 |
| **SAFE** | 0 | - | - | - | Court-circuit activé (pas de criticité) |

---

## 🎯 4. MODULE 3 : Core Moteur d'Optimisation

### 4.1 Contexte et Objectif

**En tant que** DSI
**Je souhaite** implémenter le cœur du moteur d'optimisation des propositions APP
**Afin de :**
- ✅ Implémenter la politique **"zéro rupture"**
- ✅ Garantir que les techniciens disposent toujours du stock nécessaire
- ✅ Réduire le nombre de colis tout en respectant les priorités métier

### 4.2 Architecture du Module

```mermaid
flowchart TD
    INPUT[📋 Articles Classifiés par Grade] --> INIT[🎯 Initialisation Context]

    INIT --> CLASS[📊 Classification Articles]
    CLASS --> PRIO[🔴 articles_prioritaires<br/>CRITIQUE_A/B + URGENT_A]
    CLASS --> URG[🟠 articles_urgent_b]
    CLASS --> SAFE[🟢 articles_safe]

    PRIO --> STRAT{Quelle Stratégie ?}
    URG --> STRAT
    SAFE --> STRAT

    STRAT -->|DEFAULT| DEFAULT[📦 Stratégie Standard Lot1]
    STRAT -->|Lot2| LOT2[📦 Stratégie Lot2]
    STRAT -->|Lot3| LOT3[🎯 Stratégie Lot3]

    DEFAULT --> ANALYZE{🔍 Analyser Composition<br/>Articles SAFE présents ?}

    ANALYZE -->|COMPOSITION_COMPLETE<br/>Prio+UrgB+Safe| P1[🚨 Phase 1: Prioritaires]
    ANALYZE -->|PRIORITAIRES_ET_SAFE<br/>Prio+Safe| P1A[🚨 Phase 1: Prioritaires]
    ANALYZE -->|URGENT_B_ET_SAFE<br/>UrgB+Safe| P2B[⚡ Phase 1: UrgentB]
    ANALYZE -->|SAFE_SEULEMENT<br/>ou Aucun SAFE| EMPTY[❌ Pas de proposition]

    P1 --> P2[⚡ Phase 2: Complétion URGENT_B<br/>si espace dispo]
    P1A --> P3A[🎯 Phase 2: Knapsack SAFE<br/>si espace dispo]
    P2 --> P3[🎯 Phase 3: Knapsack SAFE<br/>si espace dispo]
    P2B --> P3B[🎯 Phase 2: Knapsack SAFE<br/>si espace dispo]

    P3 --> P4[✅ Validation & Rapport]
    P3A --> P4
    P3B --> P4
    EMPTY --> P4

    P4 --> RESULT[📦 PackingResult Final]

    style P1 fill:#ffcccb
    style P1A fill:#ffcccb
    style P2 fill:#fff8dc
    style P2B fill:#fff8dc
    style P3 fill:#d4f1d4
    style P3A fill:#d4f1d4
    style P3B fill:#d4f1d4
    style P4 fill:#e0e0ff
    style EMPTY fill:#f5f5f5
```

### 4.3 Modèles de Données

#### **OptimisationContext**

```java
CLASS OptimisationContext {
    List<Article> articles_input;           // Articles à traiter
    CartonConstraints contraintes_carton;   // Contraintes physiques (BDD)
    Map<ArticleType, Double> coefficients_occupation; // BDD
    int search_depth;                       // Paramètre Consul
    Strategy strategy;                      // DEFAULT, Lot2, Lot3
}

// Méthodes
getArticlesByGrade(grade) : List<Article>
getTotalArticleCount() : int
validateConfiguration() : boolean
```

#### **PackingResult**

```java
CLASS PackingResult {
    List<Carton> cartons_finaux;           // Cartons optimisés
    MetriquesOptimisation metriques;       // KPIs performance
    boolean validation_success;            // État validation
}

// Méthodes
getTauxSatisfactionGlobal() : double
getNombreCartonsTotal() : int
generateSummaryReport() : Report
```

#### **Article**

```java
CLASS Article {
    ArticleType type;                      // BDD
    String distributeur;                   // GBH ou ORG
    GradeCriticite grade;                  // CRITIQUE_A → SAFE
    int quantite;                          // Quantité à placer
    double coefficient_occupation;          // BDD ArticleType_Carton_parameters
    int[] stock_projections;               // Depuis Module 1
    int stock_minimum, stock_maximum;      // BDD
}

// Méthodes
getOccupationTotale() : double            // quantite × coefficient
isCritique() : boolean                    // grade in (A, B, URGENT_A)
isUrgentB() : boolean
isSafe() : boolean
calculateStockObjective() : int           // (min + max) / 2
```

#### **Carton**

```java
CLASS Carton {
    String id;
    double occupation_actuelle;            // 0.0 à 1.0
    List<Article> articles_contenus;
    double capacite_maximale = 1.0;        // Constante
    boolean est_finalise;
}

// Méthodes
getCapaciteRestante() : double            // 1.0 - occupation_actuelle
peutAccueillir(article) : boolean
ajouterArticle(article) : void
finaliser() : void
getTauxOccupation() : double              // Pour affichage
```

### 4.4 Algorithme d'Orchestration

```pseudocode
ALGORITHME OptimiserColis(context)
DEBUT
    // === INITIALISATION ===
    strategie ← context.strategie_name
    articles_input ← context.articles_input
    cartons_resultats ← LISTE_VIDE()

    // === CLASSIFICATION INITIALE ===
    articles_prioritaires ← FiltrerParGrade(articles_input, [CRITIQUE_A, CRITIQUE_B, URGENT_A])
    articles_urgent_b ← FiltrerParGrade(articles_input, [URGENT_B])
    articles_safe ← FiltrerParGrade(articles_input, [SAFE])

    // === SÉLECTION STRATÉGIE ===
    SELON strategie FAIRE
        CAS "DEFAULT":
            RETOURNER ExecuterStrategieStandard(articles_prioritaires, articles_urgent_b, articles_safe)
        CAS "Lot2":
            RETOURNER ExecuterStrategieLot2(...)
        CAS "Lot3":
            RETOURNER ExecuterStrategieLot3(...)
        PAR_DEFAUT:
            RETOURNER ExecuterStrategieStandard(...)
    FIN_SELON
FIN
```

#### **Stratégie Standard (Lot 1)**

```pseudocode
ALGORITHME ExecuterStrategieStandard(articles_prioritaires, articles_urgent_b, articles_safe)
DEBUT
    cartons_resultats ← LISTE_VIDE()

    composition ← AnalyserComposition(articles_prioritaires, articles_urgent_b, articles_safe)

    SELON composition FAIRE
        CAS "COMPOSITION_COMPLETE":
            // Phase 1: Articles prioritaires (CRITIQUE_A/B + URGENT_A)
            cartons_resultats ← TraiterArticlesPrioritaires(articles_prioritaires)
            // Phase 2: S'il y a de l'espace dispo restant à l'issue de la phase 1
            cartons_resultats ← CompleterAvecUrgentB(cartons_resultats, articles_urgent_b)
            // Phase 3: S'il y a de l'espace dispo restant à l'issue de la phase 2 (Knapsack)
            cartons_resultats ← OptimiserAvecSafe(cartons_resultats, articles_safe)

        CAS "PRIORITAIRES_ET_SAFE":
            // Phase 1: Articles prioritaires (CRITIQUE_A/B + URGENT_A)
            cartons_resultats ← TraiterArticlesPrioritaires(articles_prioritaires)
            // Phase 2: S'il y a de l'espace dispo restant à l'issue de la phase 1 (Knapsack)
            cartons_resultats ← OptimiserAvecSafe(cartons_resultats, articles_safe)

        CAS "URGENT_B_ET_SAFE":
            // Phase 1: Articles URGENT_B
            cartons_resultats ← TraiterArticlesUrgentB(articles_urgent_b)
            // Phase 2: S'il y a de l'espace dispo restant à l'issue de la phase 1 (Knapsack)
            cartons_resultats ← OptimiserAvecSafe(cartons_resultats, articles_safe)

        CAS "SAFE_SEULEMENT":
            cartons_resultats ← LISTE_VIDE()  // Pas de proposition

        CAS "AUCUN_ARTICLE":
            cartons_resultats ← LISTE_VIDE()
    FIN_SELON

    RETOURNER ValiderEtGenererRapport(cartons_resultats)
FIN
```

#### **Analyse de Composition**

**Principe de simplification :**
L'orchestrateur a été simplifié pour ne traiter que les cas où des articles SAFE sont présents. Cela évite de créer des propositions APP pour des articles non critiques seuls (PRIORITAIRES_SEULEMENT, URGENT_B_SEULEMENT), conformément à la règle métier "pas de proposition si SAFE_SEULEMENT".

```pseudocode
ALGORITHME AnalyserComposition(articles_prioritaires, articles_urgent_b, articles_safe)
DEBUT
    a_prioritaires ← (articles_prioritaires.taille > 0)
    a_urgent_b ← (articles_urgent_b.taille > 0)
    a_safe ← (articles_safe.taille > 0)

    // Seuls les cas avec articles SAFE sont traités
    SI a_prioritaires ET a_urgent_b ET a_safe ALORS
        RETOURNER "COMPOSITION_COMPLETE"
    SINON_SI a_prioritaires ET a_safe ALORS
        RETOURNER "PRIORITAIRES_ET_SAFE"
    SINON_SI a_urgent_b ET a_safe ALORS
        RETOURNER "URGENT_B_ET_SAFE"
    SINON_SI a_safe ALORS
        RETOURNER "SAFE_SEULEMENT"
    SINON
        // Cas sans SAFE: pas de proposition APP
        RETOURNER "AUCUN_ARTICLE"
    FIN_SI
FIN
```

**Justification :**
- ✅ Articles PRIORITAIRES et URGENT_B seuls → déjà traités en urgence via d'autres mécanismes
- ✅ Articles SAFE → nécessitent optimisation pour atteindre stock cible
- ✅ Évite propositions APP redondantes pour matériel déjà expédié en urgence

---

#### **Diagramme de Séquence : OptimiserColis + ExecuterStrategieStandard**

```mermaid
sequenceDiagram
    participant CALLER as Module 3 - Entrée
    participant OPTIM as OptimiserColis
    participant EXEC as ExecuterStrategieStandard
    participant ANALYZE as AnalyserComposition
    participant PRIO as TraiterArticlesPrioritaires
    participant URGB as CompleterAvecUrgentB
    participant SAFE as OptimiserAvecSafe
    participant VALID as ValiderEtGenererRapport

    CALLER->>OPTIM: OptimiserColis(context)
    Note over OPTIM: Initialisation
    OPTIM->>OPTIM: strategie = context.strategie_name<br/>articles_input = context.articles_input

    Note over OPTIM: Classification Initiale
    OPTIM->>OPTIM: articles_prioritaires = Filtrer([CRITIQUE_A, CRITIQUE_B, URGENT_A])
    OPTIM->>OPTIM: articles_urgent_b = Filtrer([URGENT_B])
    OPTIM->>OPTIM: articles_safe = Filtrer([SAFE])

    Note over OPTIM: Sélection Stratégie
    alt strategie = "DEFAULT"
        OPTIM->>EXEC: ExecuterStrategieStandard(articles_prioritaires,<br/>articles_urgent_b, articles_safe)
    else strategie = "Lot2" | "Lot3"
        OPTIM->>OPTIM: ExecuterStrategieLot2/3(...)
        Note right of OPTIM: Stratégies futures
    end

    Note over EXEC: Analyse Composition
    EXEC->>ANALYZE: AnalyserComposition(articles_prioritaires,<br/>articles_urgent_b, articles_safe)

    alt Prioritaires + UrgentB + Safe
        ANALYZE-->>EXEC: "COMPOSITION_COMPLETE"

        Note over EXEC: Phase 1: Prioritaires (100% garanti)
        EXEC->>PRIO: TraiterArticlesPrioritaires(articles_prioritaires)
        PRIO-->>EXEC: cartons_resultats (avec CRITIQUE_A/B + URGENT_A)
        Note right of PRIO: Crée PLAFOND(occupation_totale) cartons<br/>Garantie 100% placement

        Note over EXEC: Phase 2: Complétion URGENT_B
        EXEC->>URGB: CompleterAvecUrgentB(cartons_resultats, articles_urgent_b)
        Note over URGB: 1. Valoriser articles (stock_min)<br/>2. Trier par valeur décroissante<br/>3. Placer dans espace dispo
        URGB-->>EXEC: cartons_resultats mis à jour

        Note over EXEC: Phase 3: Optimisation SAFE
        EXEC->>SAFE: OptimiserAvecSafe(cartons_resultats, articles_safe)
        Note over SAFE: 1. Calculer quantités vers stock_cible<br/>2. Appliquer Knapsack Multi-Contraintes<br/>3. Maximiser valeur métier
        SAFE-->>EXEC: cartons_resultats optimisés

    else Prioritaires + Safe
        ANALYZE-->>EXEC: "PRIORITAIRES_ET_SAFE"

        Note over EXEC: Phase 1: Prioritaires
        EXEC->>PRIO: TraiterArticlesPrioritaires(articles_prioritaires)
        PRIO-->>EXEC: cartons_resultats

        Note over EXEC: Phase 2: Optimisation SAFE
        EXEC->>SAFE: OptimiserAvecSafe(cartons_resultats, articles_safe)
        SAFE-->>EXEC: cartons_resultats optimisés

    else UrgentB + Safe
        ANALYZE-->>EXEC: "URGENT_B_ET_SAFE"

        Note over EXEC: Phase 1: URGENT_B
        EXEC->>URGB: TraiterArticlesUrgentB(articles_urgent_b)
        URGB-->>EXEC: cartons_resultats

        Note over EXEC: Phase 2: Optimisation SAFE
        EXEC->>SAFE: OptimiserAvecSafe(cartons_resultats, articles_safe)
        SAFE-->>EXEC: cartons_resultats optimisés

    else Safe Seulement | Aucun Article
        ANALYZE-->>EXEC: "SAFE_SEULEMENT" | "AUCUN_ARTICLE"
        EXEC->>EXEC: cartons_resultats = LISTE_VIDE()
        Note right of EXEC: Pas de proposition APP<br/>Règle métier
    end

    Note over EXEC: Validation & Rapport
    EXEC->>VALID: ValiderEtGenererRapport(cartons_resultats)
    VALID-->>EXEC: PackingResult validé
    EXEC-->>OPTIM: PackingResult
    OPTIM-->>CALLER: PackingResult final

    Note over CALLER: Résultat:<br/>- Cartons optimisés<br/>- Métriques (taux remplissage, etc.)<br/>- Articles placés/non placés
```

**Points clés du diagramme :**

1. **Classification initiale** : Tri des articles en 3 catégories (Prioritaires, URGENT_B, SAFE)
2. **Analyse de composition** : Détermine la stratégie à appliquer selon les articles présents
3. **Traitement en phases** :
   - Phase 1 : Articles prioritaires (garantie 100%)
   - Phase 2 : Complétion URGENT_B (si espace dispo)
   - Phase 3 : Optimisation SAFE via Knapsack (si espace dispo)
4. **Court-circuit** : Pas de proposition si SAFE_SEULEMENT ou AUCUN_ARTICLE
5. **Validation finale** : Génération du rapport avec métriques

---

### 4.5 Règles de Gestion

#### **RG01 - Coefficient d'Occupation**

**Définition :**
Le coefficient d'occupation désigne le **taux du volume du carton** occupé par un article.

**Formule :**
```
Coefficient_Occupation = 1 / Quantité_Max_Par_Carton
```

**Exemple :**
- Carton pouvant contenir max 10 centrales
- Coefficient = 1/10 = **0.1**

**Calcul Nombre de Cartons :**
```
Nombre_Cartons = ARRONDI_SUP(Σ(quantité_article × coefficient_type))
```

**Exemple Complet :**
```
Articles à mettre (Critiques + Urgent A):
- 5 Centrales (coeff 0.5)
- 2 DO (coeff 0.2)
- 6 DFO (coeff 0.25)

Occupation totale:
= 5×0.5 + 2×0.2 + 6×0.25
= 2.5 + 0.4 + 1.5
= 4.4

Nombre de cartons = ARRONDI_SUP(4.4) = 5 cartons
```

#### **RG02 - Hiérarchie de Criticité**

```
(CRITIQUE_A = CRITIQUE_B = URGENT_A) > URGENT_B > SAFE
```

**Traitement :**
- ✅ **Prioritaires (CRITIQUE_A/B + URGENT_A)** : Court-circuit obligatoire (100% dans APP)
- ⚠️ **URGENT_B** : Complétion si espace disponible
- 🎯 **SAFE** : Optimisation knapsack espace résiduel

**Note Lot 2 :** Possibilité de knapsack complet sans garantie 100% critiques (avec justifications).

#### **RG03 - Complétion URGENT_B avec Priorisation Intelligente**

**Principe :**
Après inclusion des articles prioritaires (Phase 1), compléter avec URGENT_B **SANS créer de nouveaux cartons**. Comme l'espace est limité, une fonction de valorisation détermine quels articles URGENT_B prioriser.

### 🤔 Problématique de la Complétion URGENT_B

Après Phase 1, il reste un **espace limité** dans les cartons. Tous les articles URGENT_B ne peuvent pas forcément être placés. La fonction de valorisation détermine **quels articles URGENT_B prioriser**.

**Situation typique après Phase 1 :**
```
Espace disponible restant :
- Carton 1 : 25% libre
- Carton 2 : 10% libre
- Carton 3 : 35% libre

Articles URGENT_B candidats (tous en rupture J6-J8) :
- Article X (Câbles) : coeff 0.2, stock 8/15 (min), fréquence 70%
- Article Y (Badges) : coeff 0.1, stock 4/10 (min), fréquence 40%
- Article Z (Kits) : coeff 0.4, stock 2/8 (min), fréquence 60%

❓ Lequel prioriser pour éviter la rupture ?
```

### 📐 Fonction de Valorisation URGENT_B

**Différence clé avec SAFE :**
- **URGENT_B** : Objectif = atteindre `stock_min` (éviter la rupture imminente J6-J8)
- **SAFE** : Objectif = atteindre `stock_cible = (min + max) / 2` (optimiser vers l'idéal)

**Algorithme :**
```pseudocode
FONCTION CalculerValeurValorisationUrgentB(article)
DEBUT
    stock_min = article.stock_min
    stock_actuel = article.stock_actuel

    // Facteur 1: Écart au stock minimum (normalisé) - 50%
    ecart_normalise = (stock_min - stock_actuel) / stock_min
    poids_ecart = 50.0

    // Facteur 2: Efficacité d'occupation (favorise petits articles) - 50%
    efficacite_occupation = 1.0 / article.coefficient_occupation
    poids_efficacite = 50.0

    // Calcul valeur composite
    valeur = (ecart_normalise × poids_ecart) +
             (efficacite_occupation × poids_efficacite)

    RETOURNER valeur
FIN
```

### 🎯 Explication des 2 Facteurs

#### **Facteur 1 : Écart au Stock Minimum (50%)**

**Formule (différente de SAFE) :**
```
écart_normalisé = (stock_min - stock_actuel) / stock_min
```

**Exemple avec nos 3 articles URGENT_B :**
```
Article X (Câbles) : (15 - 8) / 15 = 0.47  (moyennement urgent)
Article Y (Badges) : (10 - 4) / 10 = 0.60  (plus urgent)
Article Z (Kits)   : (8 - 2) / 8 = 0.75   (très urgent !)
```

**Pourquoi stock_min et pas stock_cible ?**
Les articles URGENT_B sont en **rupture modérée J6-J8**. L'objectif est d'**éviter la rupture complète**, pas d'optimiser vers un stock idéal. On vise donc le minimum de sécurité.

**Pourquoi 50% ?** L'écart au stock minimum est le critère le plus important car il mesure directement l'urgence de la rupture.

#### **Facteur 2 : Efficacité d'Occupation (50%)**

**Formule :**
```
efficacité = 1.0 / coefficient_occupation
```

**Exemple avec nos 3 articles URGENT_B :**
```
Article X (Câbles) : 1 / 0.2 = 5.0  (assez efficace)
Article Y (Badges) : 1 / 0.1 = 10.0 (très efficace, petit)
Article Z (Kits)   : 1 / 0.4 = 2.5  (peu efficace, gros)
```

**Pourquoi 50% ?** Favoriser les petits articles permet de mettre **plus de diversité** dans l'espace limité restant après Phase 1.

**Comparaison URGENT_B vs SAFE :**
```
Article Kits : stock_actuel = 2, stock_min = 8, stock_max = 20

URGENT_B : écart = (8 - 2) / 8 = 0.75 (vise le min)
SAFE     : écart = (14 - 2) / 14 = 0.86 (vise la cible 14)

→ URGENT_B est plus tolérant, vise juste éviter la rupture
→ SAFE est plus exigeant, vise l'optimum
```

### 💡 Exemple de Calcul Complet

**Article X (Câbles) :**
```
Valeur_X = (0.47 × 50) + (5.0 × 50)
        = 23.5 + 250
        = 273.5 points
```

**Article Y (Badges) :**
```
Valeur_Y = (0.60 × 50) + (10.0 × 50)
        = 30 + 500
        = 530 points ← GAGNANT grâce à efficacité + urgence
```

**Article Z (Kits) :**
```
Valeur_Z = (0.75 × 50) + (2.5 × 50)
        = 37.5 + 125
        = 162.5 points
```

**Résultat :** Article Y (Badges) est priorisé car :
- ✅ Très loin du stock min (60% d'écart)
- ✅ Très petit (coeff 0.1) → efficacité maximale (10.0)
- ✅ Article Z a le plus fort écart (75%) mais pénalisé par sa taille

**Stratégie :** Si espace limité, mettre des Badges (Y) d'abord (530 pts), puis Câbles (X) (273.5 pts), puis Kits (Z) (162.5 pts).

### 🔄 Algorithme Complet de Complétion

```pseudocode
ALGORITHME CompleterAvecUrgentB(cartons_existants, articles_urgent_b)
DEBUT
    // 1. Calculer les valeurs de priorisation
    POUR CHAQUE article DANS articles_urgent_b FAIRE
        article.valeur = CalculerValeurValorisationUrgentB(article)
    FIN_POUR

    // 2. Trier par valeur décroissante (plus urgent en premier)
    TRIER articles_urgent_b PAR valeur DÉCROISSANT

    // 3. Placer dans l'espace disponible
    POUR CHAQUE article DANS articles_urgent_b FAIRE
        quantite_restante = article.quantite_a_placer

        POUR CHAQUE carton DANS cartons_existants FAIRE
            SI carton.peutAccueillir(article) ET quantite_restante > 0 ALORS
                capacite_restante = carton.getCapaciteRestante()
                quantite_possible = PLANCHER(capacite_restante / article.coefficient)
                quantite_a_ajouter = MIN(quantite_possible, quantite_restante)

                carton.ajouterArticle(article, quantite_a_ajouter)
                quantite_restante = quantite_restante - quantite_a_ajouter
            FIN_SI
        FIN_POUR

        // Note : quantite_restante peut être > 0 si espace insuffisant
        SI quantite_restante > 0 ALORS
            LoggerPlacementPartiel(article, quantite_restante)
        FIN_SI
    FIN_POUR

    RETOURNER cartons_existants
FIN
```

#### **Diagramme de Séquence : CompleterAvecUrgentB + CalculerValeurValorisationUrgentB**

```mermaid
sequenceDiagram
    participant EXEC as ExecuterStrategieStandard
    participant COMPLET as CompleterAvecUrgentB
    participant VALOR as CalculerValeurValorisationUrgentB
    participant CARTONS as Cartons Existants

    EXEC->>COMPLET: CompleterAvecUrgentB(cartons_existants, articles_urgent_b)
    Note over COMPLET: Étape 1: Valoriser tous les articles

    loop Pour chaque article URGENT_B
        COMPLET->>VALOR: CalculerValeurValorisationUrgentB(article)

        Note over VALOR: Récupérer données article
        VALOR->>VALOR: stock_min = article.stock_min<br/>stock_actuel = article.stock_actuel

        Note over VALOR: Facteur 1 (50%) : Écart au stock_min
        VALOR->>VALOR: ecart_normalise = (stock_min - stock_actuel) / stock_min
        VALOR->>VALOR: score_ecart = ecart_normalise × 50
        Note right of VALOR: Plus l'article est loin<br/>du stock_min, plus il<br/>est urgent

        Note over VALOR: Facteur 2 (50%) : Efficacité occupation
        VALOR->>VALOR: efficacite = 1.0 / article.coefficient_occupation
        VALOR->>VALOR: score_efficacite = efficacite × 50
        Note right of VALOR: Favorise petits articles<br/>pour maximiser diversité

        Note over VALOR: Calcul valeur composite
        VALOR->>VALOR: valeur = score_ecart + score_efficacite

        VALOR-->>COMPLET: valeur (ex: 273.5, 530, 162.5 points)
        COMPLET->>COMPLET: article.valeur = valeur
    end

    Note over COMPLET: Étape 2: Trier par valeur décroissante
    COMPLET->>COMPLET: TRIER articles_urgent_b PAR valeur DÉCROISSANT
    Note right of COMPLET: Articles triés:<br/>1. Article Y (530 pts)<br/>2. Article X (273.5 pts)<br/>3. Article Z (162.5 pts)

    Note over COMPLET: Étape 3: Placement séquentiel

    loop Pour chaque article (ordre décroissant)
        COMPLET->>COMPLET: quantite_restante = article.quantite_a_placer

        loop Pour chaque carton
            COMPLET->>CARTONS: carton.peutAccueillir(article) ?

            alt Espace disponible ET quantite_restante > 0
                CARTONS-->>COMPLET: OUI, capacite_restante

                COMPLET->>COMPLET: quantite_possible = PLANCHER(capacite_restante / coeff)
                COMPLET->>COMPLET: quantite_a_ajouter = MIN(quantite_possible, quantite_restante)

                COMPLET->>CARTONS: ajouterArticle(article, quantite_a_ajouter)
                CARTONS-->>COMPLET: Article ajouté

                COMPLET->>COMPLET: quantite_restante -= quantite_a_ajouter
                Note right of COMPLET: Ex: Carton1 += 3 Badges

            else Pas d'espace OU quantite_restante = 0
                CARTONS-->>COMPLET: NON
                Note right of COMPLET: Passer au carton suivant
            end
        end

        alt quantite_restante > 0
            COMPLET->>COMPLET: LoggerPlacementPartiel(article, quantite_restante)
            Note right of COMPLET: ⚠️ Placement partiel<br/>Espace insuffisant
        end
    end

    COMPLET-->>EXEC: cartons_existants mis à jour

    Note over EXEC: Résultat:<br/>- Articles URGENT_B placés par priorité<br/>- Maximisation valeur métier<br/>- Placement partiel si espace insuffisant
```

**Points clés du diagramme :**

1. **Valorisation intelligente** : Chaque article URGENT_B reçoit un score basé sur :
   - Écart au stock_min (50%) : Plus il est loin, plus c'est urgent
   - Efficacité d'occupation (50%) : Favorise petits articles pour diversité

2. **Tri par priorité** : Articles triés par valeur décroissante avant placement

3. **Placement séquentiel** : Les articles les plus valorisés sont placés en premier dans l'espace disponible

4. **Gestion placement partiel** : Si espace insuffisant, placement partiel accepté et loggé

---

### 📊 Comparaison URGENT_B vs SAFE

| Critère | URGENT_B (Phase 2) | SAFE (Phase 3) |
|---------|-------------------|---------------|
| **Objectif stock** | `stock_min` | `(stock_min + stock_max) / 2` |
| **Urgence métier** | Rupture J6-J8 (modérée) | Pas de rupture (optimisation) |
| **Priorité placement** | Haute (après critiques) | Basse (espace résiduel) |
| **Quantités partielles** | Acceptées | Acceptées |
| **Création nouveaux cartons** | ❌ Non | ❌ Non |
| **Facteur 1 (écart)** | Distance au stock_min | Distance au stock_cible |
| **Impact opérationnel** | Critique (éviter rupture) | Confort (stock optimal) |

**Exemple concret :**
```
Article : stock_actuel = 5, stock_min = 10, stock_max = 30

En URGENT_B (rupture J7) :
- Écart = (10 - 5) / 10 = 0.50
- Objectif : placer 5 unités pour atteindre 10 (le minimum)

En SAFE (pas de rupture) :
- Écart = (20 - 5) / 20 = 0.75
- Objectif : placer 15 unités pour atteindre 20 (l'optimal)
```

#### **RG04 - Optimisation SAFE (Knapsack)**

**Objectif :**
Optimiser vers le stock cible `(stock_min + stock_max) / 2` dans l'espace restant **SANS créer de nouveaux cartons**.

**Algorithme :**
```pseudocode
ALGORITHME OptimiserArticlesSafe(articles_safe, cartons_existants)
DEBUT
    // Calculer quantités SAFE requises pour atteindre stock cible
    POUR CHAQUE article DANS articles_safe FAIRE
        stock_cible = (article.stock_min + article.stock_max) / 2
        stock_actuel_projete = article.stock_actuel + article.quantite_deja_placee

        SI stock_actuel_projete < stock_cible ALORS
            article.quantite_a_placer = stock_cible - stock_actuel_projete
        SINON
            article.quantite_a_placer = 0
            RETIRER article DE articles_safe
        FIN_SI
    FIN_POUR

    // Appliquer Knapsack Multi-Contraintes sur espace résiduel
    RETOURNER KnapsackMultiContraintes(articles_safe, cartons_existants)
FIN
```

#### **RG05 - Fonction de Valorisation SAFE (pour Knapsack)**

**Objectif :**
Déterminer la valeur d'un article SAFE pour prioriser dans l'optimisation knapsack.

### 🤔 Problématique du Knapsack

Après les phases 1 et 2, il reste un **espace limité** dans les cartons. Tous les articles SAFE ne peuvent pas être placés. La fonction de valorisation détermine **quels articles SAFE prioriser**.

**Situation typique après Phase 2 :**
```
Espace disponible restant :
- Carton 1 : 30% libre
- Carton 2 : 15% libre
- Carton 3 : 40% libre

Articles SAFE candidats :
- Article A (Centrales) : coeff 0.5, stock 5/15 (cible), fréquence 80%
- Article B (Badges) : coeff 0.1, stock 15/19 (cible), fréquence 30%
- Article C (DO) : coeff 0.3, stock 3/10 (cible), fréquence 60%

❓ Lequel prioriser ? On ne peut pas tout mettre !
```

### 📐 Fonction de Valorisation Composite

**Algorithme :**
```pseudocode
FONCTION CalculerValeurValorisationSafe(article)
DEBUT
    stock_cible = (article.stock_min + article.stock_max) / 2
    stock_actuel = article.stock_actuel

    // Facteur 1: Écart au stock cible (normalisé) - 50%
    ecart_normalise = (stock_cible - stock_actuel) / stock_cible
    poids_ecart = 50.0

    // Facteur 2: Efficacité d'occupation (favorise petits articles) - 50%
    efficacite_occupation = 1.0 / article.coefficient_occupation
    poids_efficacite = 50.0

    // Calcul valeur composite
    valeur = (ecart_normalise × poids_ecart) +
             (efficacite_occupation × poids_efficacite)

    RETOURNER valeur
FIN
```

### 🎯 Explication des 2 Facteurs

#### **Facteur 1 : Écart au Stock Cible (50%)**

**Formule :**
```
écart_normalisé = (stock_cible - stock_actuel) / stock_cible
```

**Exemple avec nos 3 articles :**
```
Article A (Centrales) : (15 - 5) / 15 = 0.67
Article B (Badges)    : (19 - 15) / 19 = 0.21
Article C (DO)        : (10 - 3) / 10 = 0.70
```

**Pourquoi 50% ?** Plus l'article est loin de son stock optimal, plus il est urgent de le réapprovisionner. C'est le critère principal avec un poids égal à l'efficacité d'occupation.

#### **Facteur 2 : Efficacité d'Occupation (50%)**

**Formule :**
```
efficacité = 1.0 / coefficient_occupation
```

**Exemple avec nos 3 articles :**
```
Article A (Centrales) : 1 / 0.5 = 2.0  (gros, peu efficace)
Article B (Badges)    : 1 / 0.1 = 10.0 (petit, très efficace)
Article C (DO)        : 1 / 0.3 = 3.3  (moyen)
```

**Pourquoi 50% ?** Favoriser les petits articles permet de mettre **plus de diversité** dans les cartons. Mieux vaut 5 types d'articles différents (même en petite quantité) que 2 types gros qui remplissent tout l'espace. Ce critère a un poids égal à l'écart au stock cible.

**Exemple concret :**
- Option 1 : Mettre 1 Centrale (occupe 50% d'un carton)
- Option 2 : Mettre 1 DO + 2 Badges (occupe 50% mais 2 types différents)
→ **Option 2 préférée** car plus de diversité

### 💡 Exemple de Calcul Complet

**Article A (Centrales) :**
```
Valeur_A = (0.67 × 50) + (2.0 × 50)
        = 33.5 + 100
        = 133.5 points
```

**Article B (Badges) :**
```
Valeur_B = (0.21 × 50) + (10.0 × 50)
        = 10.5 + 500
        = 510.5 points ← GAGNANT (efficacité excellente)
```

**Article C (DO) :**
```
Valeur_C = (0.70 × 50) + (3.3 × 50)
        = 35 + 165
        = 200 points
```

**Résultat :** Article B (Badges) est priorisé car :
- ✅ Excellente efficacité d'occupation (coeff 0.1 = 10.0 points)
- ✅ Permet de maximiser la diversité d'articles dans l'espace limité
- ⚠️ Bien que l'écart soit faible (21%), l'efficacité compense largement

### 🔧 Alternatives de Valorisation

La fonction composite (50%-50%) est **paramétrable**. Voici d'autres stratégies possibles :

| Stratégie | Facteur Principal | Cas d'Usage |
|-----------|------------------|-------------|
| **Besoin pur** | 100% écart stock cible | Prioriser uniquement le besoin quantitatif |
| **Diversité max** | 100% efficacité occupation | Maximiser nombre de types d'articles différents |
| **Équilibrée** (recommandée) | 50%-50% | Mix optimal des 2 critères (écart + efficacité) |

**Note :** Les poids (50-50) peuvent être ajustés dans les paramètres Consul selon les priorités métier.

#### **RG06 - Algorithme TraiterArticlesPrioritaires**

```pseudocode
ALGORITHME TraiterArticlesPrioritaires(articles_prioritaires)
DEBUT
    occupation_totale ← CalculerOccupationRequise(articles_prioritaires)
    nombre_cartons ← PLAFOND(occupation_totale)

    cartons ← CreerCartons(nombre_cartons)

    // Distribution homogène (round-robin possible)
    POUR chaque carton DANS cartons FAIRE
        TANT_QUE carton.occupation_actuelle < 1.0 FAIRE
            Ajouter un article prioritaire
        FIN_TANT_QUE
    FIN_POUR

    RETOURNER cartons
FIN
```

**Garantie :** `ARRONDI_SUP(occupation_totale)` assure mathématiquement 100% placement.

#### **Diagramme de Séquence : TraiterArticlesPrioritaires**

```mermaid
sequenceDiagram
    participant EXEC as ExecuterStrategieStandard
    participant TRAITER as TraiterArticlesPrioritaires
    participant CALC as CalculerOccupationRequise
    participant FACTORY as CreerCartons
    participant CARTONS as Liste Cartons

    EXEC->>TRAITER: TraiterArticlesPrioritaires(articles_prioritaires)
    Note over TRAITER: Articles prioritaires:<br/>CRITIQUE_A, CRITIQUE_B, URGENT_A

    Note over TRAITER: Étape 1: Calculer occupation totale
    TRAITER->>CALC: CalculerOccupationRequise(articles_prioritaires)

    loop Pour chaque article prioritaire
        CALC->>CALC: occupation = article.quantite × article.coefficient
        Note right of CALC: Ex: 5 Centrales × 0.5 = 2.5<br/>2 DO × 0.2 = 0.4<br/>6 DFO × 0.25 = 1.5
    end

    CALC->>CALC: occupation_totale = Σ(toutes les occupations)
    CALC-->>TRAITER: occupation_totale (ex: 4.4)
    Note right of CALC: Exemple:<br/>2.5 + 0.4 + 1.5 = 4.4

    Note over TRAITER: Étape 2: Calculer nombre de cartons requis
    TRAITER->>TRAITER: nombre_cartons = PLAFOND(occupation_totale)
    Note right of TRAITER: PLAFOND(4.4) = 5 cartons<br/>✅ Garantie 100% placement

    Note over TRAITER: Étape 3: Créer les cartons
    TRAITER->>FACTORY: CreerCartons(nombre_cartons)

    loop i = 1 à nombre_cartons
        FACTORY->>CARTONS: Nouveau Carton(id=i, occupation=0.0, capacite_max=1.0)
        Note right of CARTONS: Carton vide initialisé
    end

    FACTORY-->>TRAITER: cartons[] (5 cartons créés)

    Note over TRAITER: Étape 4: Distribution homogène (Round-Robin)

    loop Pour chaque carton
        loop Tant que carton.occupation_actuelle < 1.0
            alt Il reste des articles prioritaires à placer
                TRAITER->>TRAITER: article = ProchainArticlePrioritaire()
                TRAITER->>TRAITER: quantite_a_ajouter = Calculer quantité pour ce carton

                TRAITER->>CARTONS: ajouterArticle(carton, article, quantite_a_ajouter)
                Note right of CARTONS: Ex: Carton1 += 2 Centrales<br/>occupation: 0.0 → 1.0

                CARTONS->>CARTONS: occupation_actuelle += quantite × coefficient
                CARTONS-->>TRAITER: Article ajouté, occupation mise à jour

                Note right of TRAITER: Distribution round-robin<br/>pour répartir équitablement
            else Plus d'articles à placer
                TRAITER->>TRAITER: Passer au carton suivant
                Note right of TRAITER: Carton peut être<br/>partiellement rempli
            end
        end
    end

    Note over TRAITER: Étape 5: Vérification garantie 100%
    TRAITER->>TRAITER: Vérifier tous les articles prioritaires placés

    alt Tous les articles placés
        Note right of TRAITER: ✅ Garantie respectée<br/>PLAFOND() assure mathématiquement<br/>l'espace suffisant
    else Certains articles non placés (impossible mathématiquement)
        Note right of TRAITER: ❌ Erreur logique<br/>Ne devrait jamais arriver
    end

    TRAITER-->>EXEC: cartons[] (5 cartons avec articles prioritaires)

    Note over EXEC: Résultat:<br/>- Articles CRITIQUE_A/B + URGENT_A placés à 100%<br/>- Cartons créés = PLAFOND(occupation)<br/>- Distribution homogène<br/>- Prêt pour Phase 2 (URGENT_B)
```

**Points clés du diagramme :**

1. **Calcul de l'occupation totale** : Somme des `quantité × coefficient` de tous les articles prioritaires

2. **Garantie mathématique 100%** : En créant `PLAFOND(occupation_totale)` cartons, on garantit mathématiquement qu'il y a assez d'espace
   - Exemple : occupation = 4.4 → 5 cartons → espace total = 5.0 > 4.4 ✅

3. **Distribution homogène (Round-Robin)** : Les articles sont répartis équitablement entre les cartons pour éviter qu'un carton soit très plein et d'autres vides

4. **Pas de priorisation** : Tous les articles prioritaires (CRITIQUE_A/B + URGENT_A) sont traités avec la même priorité (court-circuit = 100% garanti)

5. **Aucun rejet** : Contrairement aux phases 2 et 3, aucun article prioritaire ne peut être rejeté

**Exemple concret :**
```
Articles prioritaires:
- 5 Centrales (coeff 0.5) = 2.5
- 2 DO (coeff 0.2) = 0.4
- 6 DFO (coeff 0.25) = 1.5
Total = 4.4

Cartons créés = PLAFOND(4.4) = 5 cartons

Distribution possible:
- Carton 1: 2 Centrales (1.0) ← plein
- Carton 2: 2 Centrales (1.0) ← plein
- Carton 3: 1 Centrale (0.5) + 2 DO (0.4) = 0.9
- Carton 4: 3 DFO (0.75)
- Carton 5: 3 DFO (0.75)
```

---

#### **RG07 - Algorithme Knapsack Multi-Contraintes**

```pseudocode
ALGORITHME KnapsackMultiContraintes(articles_safe, cartons_existants)
DEBUT
    POUR CHAQUE carton DANS cartons_existants FAIRE
        capacite_restante ← 1.0 - carton.occupation_actuelle
        capacite_discretisee ← ARRONDI(capacite_restante × 100)
        articles_candidats ← FiltrerParCapacite(articles_safe, capacite_restante)

        // Programmation Dynamique
        dp ← InitialiserTableDP(articles_candidats.size, capacite_discretisee)

        POUR i DE 1 À articles_candidats.size FAIRE
            POUR w DE 0 À capacite_discretisee FAIRE
                article ← articles_candidats[i-1]
                cout_occupation_discret ← ARRONDI(article.quantite × article.coefficient × 100)
                valeur_stock ← CalculerValeurValorisationSafe(article)

                SI cout_occupation_discret <= w ALORS
                    dp[i][w] ← MAX(dp[i-1][w],
                                   dp[i-1][w-cout_occupation_discret] + valeur_stock)
                SINON
                    dp[i][w] ← dp[i-1][w]
                FIN_SI
            FIN_POUR
        FIN_POUR

        solution ← ReconstruireSolution(dp, articles_candidats, capacite_discretisee)
        AppliquerSolution(carton, solution)
    FIN_POUR

    RETOURNER cartons_existants
FIN
```

#### **Diagramme de Séquence : OptimiserArticlesSafe + CalculerValeurValorisationSafe**

```mermaid
sequenceDiagram
    participant EXEC as ExecuterStrategieStandard
    participant OPTIM as OptimiserArticlesSafe
    participant VALOR as CalculerValeurValorisationSafe
    participant KNAP as KnapsackMultiContraintes
    participant CARTONS as Cartons Existants

    EXEC->>OPTIM: OptimiserArticlesSafe(articles_safe, cartons_existants)
    Note over OPTIM: Étape 1: Calculer quantités requises

    loop Pour chaque article SAFE
        OPTIM->>OPTIM: stock_cible = (article.stock_min + article.stock_max) / 2
        OPTIM->>OPTIM: stock_actuel_projete = stock_actuel + quantite_deja_placee

        alt stock_actuel_projete < stock_cible
            OPTIM->>OPTIM: article.quantite_a_placer = stock_cible - stock_actuel_projete
            Note right of OPTIM: Article nécessite<br/>réapprovisionnement
        else stock_actuel_projete ≥ stock_cible
            OPTIM->>OPTIM: article.quantite_a_placer = 0
            OPTIM->>OPTIM: RETIRER article DE articles_safe
            Note right of OPTIM: Stock déjà optimal
        end
    end

    Note over OPTIM: Étape 2: Lancer Knapsack Multi-Contraintes
    OPTIM->>KNAP: KnapsackMultiContraintes(articles_safe, cartons_existants)

    loop Pour chaque carton
        KNAP->>CARTONS: Récupérer occupation_actuelle
        CARTONS-->>KNAP: occupation_actuelle (ex: 0.70 = 70%)

        KNAP->>KNAP: capacite_restante = 1.0 - occupation_actuelle
        KNAP->>KNAP: capacite_discretisee = ARRONDI(capacite_restante × 100)
        Note right of KNAP: Ex: 0.30 → 30 unités

        Note over KNAP: Filtrer articles candidats
        KNAP->>KNAP: articles_candidats = FiltrerParCapacite(articles_safe, capacite_restante)

        Note over KNAP: Valoriser chaque article candidat
        loop Pour chaque article candidat
            KNAP->>VALOR: CalculerValeurValorisationSafe(article)

            Note over VALOR: Récupérer données
            VALOR->>VALOR: stock_cible = (stock_min + stock_max) / 2<br/>stock_actuel = article.stock_actuel

            Note over VALOR: Facteur 1 (50%) : Écart au stock_cible
            VALOR->>VALOR: ecart_normalise = (stock_cible - stock_actuel) / stock_cible
            VALOR->>VALOR: score_ecart = ecart_normalise × 50
            Note right of VALOR: Plus il est loin du<br/>stock optimal, plus il<br/>est prioritaire

            Note over VALOR: Facteur 2 (50%) : Efficacité occupation
            VALOR->>VALOR: efficacite = 1.0 / article.coefficient_occupation
            VALOR->>VALOR: score_efficacite = efficacite × 50
            Note right of VALOR: Favorise petits articles<br/>pour diversité

            Note over VALOR: Calcul valeur composite
            VALOR->>VALOR: valeur = score_ecart + score_efficacite

            VALOR-->>KNAP: valeur (ex: 133.5, 510.5, 200 points)
            KNAP->>KNAP: article.valeur_stock = valeur
        end

        Note over KNAP: Programmation Dynamique
        KNAP->>KNAP: InitialiserTableDP(nb_articles, capacite_discretisee)

        loop i = 1 à nb_articles
            loop w = 0 à capacite_discretisee
                KNAP->>KNAP: cout = ARRONDI(article[i].qte × coeff × 100)
                KNAP->>KNAP: valeur = article[i].valeur_stock

                alt cout ≤ w
                    KNAP->>KNAP: dp[i][w] = MAX(<br/>  dp[i-1][w],<br/>  dp[i-1][w-cout] + valeur<br/>)
                    Note right of KNAP: Choix optimal:<br/>Inclure ou exclure
                else cout > w
                    KNAP->>KNAP: dp[i][w] = dp[i-1][w]
                    Note right of KNAP: Article ne rentre pas
                end
            end
        end

        Note over KNAP: Reconstruction Solution Optimale
        KNAP->>KNAP: solution = ReconstruireSolution(dp, articles_candidats, capacite)
        Note right of KNAP: Backtracking depuis dp[N][cap]

        KNAP->>CARTONS: AppliquerSolution(carton, solution)
        CARTONS-->>KNAP: Articles SAFE ajoutés
        Note right of CARTONS: Ex: Carton1 += 3 Badges (510.5 pts)<br/>+ 1 DO (200 pts)
    end

    KNAP-->>OPTIM: cartons_existants optimisés
    OPTIM-->>EXEC: cartons_existants avec SAFE placés

    Note over EXEC: Résultat:<br/>- Articles SAFE placés optimalement<br/>- Maximisation valeur métier totale<br/>- Contrainte d'espace respectée
```

**Points clés du diagramme :**

1. **Calcul quantités requises** : Détermine pour chaque article SAFE la quantité nécessaire pour atteindre le stock cible `(stock_min + stock_max) / 2`

2. **Valorisation SAFE** : Chaque article reçoit un score basé sur :
   - Écart au stock_cible (50%) : Plus il est loin de l'optimal, plus prioritaire
   - Efficacité d'occupation (50%) : Favorise petits articles pour diversité

3. **Programmation dynamique** : Pour chaque carton, résout le problème du sac à dos :
   - Table DP pour mémoriser les solutions optimales
   - Choix binaire pour chaque article : inclure ou exclure
   - Maximise la valeur totale sous contrainte d'espace

4. **Backtracking** : Reconstruit la solution optimale depuis la table DP

5. **Résultat optimal** : Les articles SAFE placés maximisent la valeur métier dans l'espace résiduel disponible

---

## 🏗️ 5. Architecture Globale

### 5.1 Diagramme de Flux Global

```mermaid
sequenceDiagram
    participant USER as Utilisateur
    participant SYS as Système Réappro
    participant M1 as Module 1<br/>Calcul Besoin
    participant M2 as Module 2<br/>Évaluation Urgence
    participant M3 as Module 3<br/>Optimisation
    participant DB as Base de Données
    participant REPORT as Rapport Final

    USER->>SYS: Lancer Réapprovisionnement(technicien, search_depth)

    Note over SYS,M1: 📊 MODULE 1: CALCUL BESOIN
    SYS->>M1: CalculerBesoin(technicien, search_depth)
    M1->>DB: Récupérer stock_actuel + transit + pending
    M1->>DB: Récupérer planning interventions
    M1->>M1: Calculer stock_initial (J0)
    M1->>M1: Calculer consommation avec Imprev_Fact
    M1->>M1: Projeter stock jour par jour
    M1-->>SYS: Matrice Projection[articles][jours]

    Note over SYS,M2: 🚨 MODULE 2: ÉVALUATION URGENCE
    SYS->>M2: CalculerUrgences(projection_stock)
    loop Pour chaque article
        M2->>M2: Calculer UrQ (urgence quantitative)
        alt UrQ > 0
            M2->>M2: Calculer UrT (urgence temporelle)
            M2->>M2: Déterminer ImP (importance)
            M2->>M2: Calculer UrTT = UrT + UrQ + ImP
        else UrQ = 0
            M2->>M2: UrTT = 0 (court-circuit)
        end
        M2->>M2: Classifier Grade (CRITIQUE_A→SAFE)
    end
    M2-->>SYS: Articles Classifiés par Grade

    Note over SYS,M3: 🎯 MODULE 3: OPTIMISATION COLIS
    SYS->>M3: OptimiserColis(articles_classifies)
    M3->>M3: Classification initiale (critiques, urgent_b, safe)
    M3->>M3: AnalyserComposition()

    alt Composition Complete
        M3->>M3: Phase 1: TraiterArticlesCritiques()
        M3->>M3: Phase 2: CompleterAvecUrgentB()
        M3->>M3: Phase 3: KnapsackMultiContraintes()
    else Autres Compositions
        M3->>M3: Traitement selon composition détectée
    end

    M3->>M3: Phase 4: ValiderEtGenererRapport()
    M3-->>SYS: PackingResult avec cartons optimisés

    Note over SYS,REPORT: 📋 GÉNÉRATION RAPPORT
    SYS->>REPORT: GénérerRapport(PackingResult)
    REPORT->>DB: Sauvegarder proposition APP
    REPORT-->>USER: Proposition APP avec métriques

    Note over USER: Logisticien valide/modifie/refuse
```

### 5.2 Flux de Données Intégré

```mermaid
graph TB
    subgraph " INPUT "
        I1[Planning Technicien]
        I2[Stock Actuel]
        I3[Colis Transit/Pending]
        I4[Paramètres Consul]
    end

    subgraph " MODULE 1: CALCUL BESOIN "
        M1A[Stock Initial J0]
        M1B[Consommation Prévisionnelle]
        M1C[Facteur Imprévu]
        M1D[Projection Stock]

        I1 --> M1B
        I2 --> M1A
        I3 --> M1A
        I4 --> M1C
        M1A --> M1D
        M1B --> M1D
        M1C --> M1B
    end

    subgraph " MODULE 2: ÉVALUATION URGENCE "
        M2A[Urgence Quantitative UrQ]
        M2B[Urgence Temporelle UrT]
        M2C[Importance ImP]
        M2D[Urgence Totale UrTT]
        M2E[Classification Grade]

        M1D --> M2A
        M1D --> M2B
        M2A --> M2D
        M2B --> M2D
        M2C --> M2D
        M2D --> M2E
    end

    subgraph " MODULE 3: OPTIMISATION "
        M3A[Classification Articles]
        M3B[Phase 1: Critiques]
        M3C[Phase 2: URGENT_B]
        M3D[Phase 3: Knapsack SAFE]
        M3E[Validation]

        M2E --> M3A
        M3A --> M3B
        M3B --> M3C
        M3C --> M3D
        M3D --> M3E
    end

    subgraph " OUTPUT "
        O1[Proposition APP]
        O2[Cartons Optimisés]
        O3[Métriques KPI]
        O4[Rapport Validation]

        M3E --> O1
        M3E --> O2
        M3E --> O3
        M3E --> O4
    end

    style M1D fill:#e1f5fe
    style M2E fill:#fff8e1
    style M3E fill:#e8f5e9
```

---

### 5.3 Diagramme de Classes

```mermaid
classDiagram
    %% ========== ENUMERATIONS ==========
    class GradeCriticite {
        <<enumeration>>
        CRITIQUE_A : 300
        URGENT_A : 250
        CRITIQUE_B : 210
        URGENT_B : 160
        SAFE : 0
    }

    class Strategy {
        <<enumeration>>
        DEFAULT
        LOT2
        LOT3
    }

    class ArticleType {
        <<enumeration>>
        CENTRALE
        BADGE
        DO
        DFO
        CAMERA_EXT
        CABLES
        KITS
    }

    %% ========== ENTITÉS MÉTIER ==========
    class Materiel {
        +String id
        +ArticleType type
        +String distributeur
        +int stock_actuel
        +int stock_min
        +int stock_max
        +int[] stock_projections
        +int quantite_requise
        +double coefficient_occupation
        +UrQ urgence_quantitative
        +UrT urgence_temporelle
        +ImP importance
        +UrTT urgence_totale
        +getStockCible() int
        +calculerEcartStockMin() double
        +calculerEcartStockCible() double
        +isScannable() boolean
    }

    class Article {
        +String id
        +ArticleType type
        +String distributeur
        +GradeCriticite grade
        +int quantite_a_placer
        +int quantite_deja_placee
        +double coefficient_occupation
        +int stock_min
        +int stock_max
        +int stock_actuel
        +double valeur_priorisation
        +getOccupationTotale() double
        +isCritique() boolean
        +isUrgentB() boolean
        +isSafe() boolean
        +calculateStockObjective() int
    }

    class Carton {
        +String id
        +double occupation_actuelle
        +List~Article~ articles_contenus
        +double capacite_maximale
        +boolean est_finalise
        +getCapaciteRestante() double
        +peutAccueillir(Article) boolean
        +ajouterArticle(Article, int) void
        +finaliser() void
        +getTauxOccupation() double
    }

    %% ========== CONTEXTE & RÉSULTAT ==========
    class OptimisationContext {
        +List~Article~ articles_input
        +CartonConstraints contraintes_carton
        +Map~ArticleType, Double~ coefficients_occupation
        +int search_depth
        +Strategy strategy
        +String strategie_name
        +getArticlesByGrade(GradeCriticite) List~Article~
        +getTotalArticleCount() int
        +validateConfiguration() boolean
    }

    class PackingResult {
        +List~Carton~ cartons_finaux
        +MetriquesOptimisation metriques
        +boolean validation_success
        +List~Article~ articles_non_places
        +getTauxSatisfactionGlobal() double
        +getNombreCartonsTotal() int
        +generateSummaryReport() Report
    }

    class MetriquesOptimisation {
        +double taux_occupation_moyen
        +int nombre_articles_places
        +int nombre_articles_total
        +double taux_satisfaction
        +Map~GradeCriticite, Integer~ repartition_par_grade
        +calculerTauxSatisfaction() double
    }

    class CartonConstraints {
        +double capacite_maximale
        +int nombre_max_cartons
        +Map~ArticleType, Integer~ quantite_max_par_type
        +validateCarton(Carton) boolean
    }

    %% ========== SERVICES/CALCULATEURS ==========
    class CalculBesoinService {
        <<service>>
        +calculerStockInitial(Materiel) int
        +projeterStock(Materiel, int) int[]
        +calculerFacteurImprevus(Planning) double
        +determinerQuantiteRequise(Materiel) int
    }

    class EvaluationUrgenceService {
        <<service>>
        +calculerUrgencesMateriels(List~Materiel~) List~Materiel~
        +calculerUrgenceQuantitative(Materiel) int
        +calculerUrgenceTemporelle(Materiel) int
        +determinerImportance(ArticleType) int
        +calculerUrgenceTotale(int, int, int) int
        +classifierParGrade(Materiel) GradeCriticite
    }

    class OptimisationService {
        <<service>>
        +optimiserColis(OptimisationContext) PackingResult
        +executerStrategieStandard(List~Article~) PackingResult
        +analyserComposition(List~Article~) String
        +traiterArticlesPrioritaires(List~Article~) List~Carton~
        +completerAvecUrgentB(List~Carton~, List~Article~) List~Carton~
        +optimiserAvecSafe(List~Carton~, List~Article~) List~Carton~
        +validerEtGenererRapport(List~Carton~) PackingResult
    }

    class ValorisationService {
        <<service>>
        +calculerValeurValorisationUrgentB(Article) double
        +calculerValeurValorisationSafe(Article) double
        +calculerEcartNormalise(Article, int) double
        +calculerEfficaciteOccupation(Article) double
    }

    class KnapsackService {
        <<service>>
        +knapsackMultiContraintes(List~Article~, List~Carton~) List~Carton~
        +initialiserTableDP(int, int) double[][]
        +reconstruireSolution(double[][], List~Article~, int) List~Article~
        +appliquerSolution(Carton, List~Article~) void
    }

    %% ========== RELATIONS ==========
    Materiel --> ArticleType : type
    Materiel --> GradeCriticite : urgence_totale classée en

    Article --> ArticleType : type
    Article --> GradeCriticite : grade

    Carton "1" *-- "0..*" Article : contient

    OptimisationContext --> Strategy : strategy
    OptimisationContext "1" *-- "0..*" Article : articles_input
    OptimisationContext --> CartonConstraints : contraintes_carton

    PackingResult "1" *-- "0..*" Carton : cartons_finaux
    PackingResult --> MetriquesOptimisation : metriques
    PackingResult "1" *-- "0..*" Article : articles_non_places

    CalculBesoinService ..> Materiel : calcule
    EvaluationUrgenceService ..> Materiel : évalue
    EvaluationUrgenceService ..> Article : produit

    OptimisationService ..> OptimisationContext : utilise
    OptimisationService ..> PackingResult : produit
    OptimisationService ..> Article : traite
    OptimisationService ..> Carton : crée

    ValorisationService ..> Article : valorise
    KnapsackService ..> Article : optimise
    KnapsackService ..> Carton : remplit

    OptimisationService ..> ValorisationService : appelle
    OptimisationService ..> KnapsackService : appelle

    %% Annotations
    note for Materiel "Sortie Module 1 & 2\nAvant classification"
    note for Article "Sortie Module 2\nAprès classification par grade"
    note for Carton "Module 3\nConteneur de placement"
    note for PackingResult "Résultat final\nProposition APP"
```

**Description des classes principales :**

| Classe | Responsabilité | Module |
|--------|----------------|--------|
| **Materiel** | Représente un article avec son historique de stock et ses projections | Module 1 & 2 |
| **Article** | Matériel classifié avec grade de criticité et prêt pour optimisation | Module 3 |
| **Carton** | Conteneur physique avec contrainte de capacité (0.0 à 1.0) | Module 3 |
| **OptimisationContext** | Contexte d'exécution avec paramètres et stratégie | Module 3 |
| **PackingResult** | Résultat final avec cartons optimisés et métriques | Module 3 |
| **CalculBesoinService** | Calcule les besoins en stock et projections | Module 1 |
| **EvaluationUrgenceService** | Évalue l'urgence et classifie les matériels | Module 2 |
| **OptimisationService** | Orchestre l'optimisation des cartons | Module 3 |
| **ValorisationService** | Calcule les valeurs de priorisation | Module 3 |
| **KnapsackService** | Résout le problème du sac à dos multi-contraintes | Module 3 |

**Relations clés :**
- Un **Materiel** devient un **Article** après classification
- Un **Carton** contient plusieurs **Articles**
- Un **PackingResult** contient plusieurs **Cartons**
- Les **Services** manipulent les entités métier pour produire les résultats

---

## 📏 6. Règles de Gestion Transversales

### RT-01 : Séparation GBH vs ORG

**Principe :** À **toute étape** du système, distinction entre stock MPO et stock GBH.

**Application :**
- Module 1 : Calcul séparé GBH/ORG
- Module 2 : Urgences séparées par distributeur
- Module 3 : APP divisées en 2 (APP GBH + APP MPO)

### RT-02 : Exclusions Matériels

**Exclus du Lot 1 :**
- ❌ Guides (non gérés en stock)
- ❌ Stickers/Signalétiques (non gérés en stock)
- ❌ Piles (lot ultérieur)

### RT-03 : Associations Matériels

**Règles d'Association :**
- 1 Centrale = 1 Carte SIM (toujours associées)
- 1 Cam EXT = 1 Panneau solaire (toujours associés)

### RT-04 : Types d'Interventions Analysées

**Inclus :** SS, SAV, EX
**Exclus :** VE (pas de remplacement équipement)

### RT-05 : Jours Ouvrables

**Règle :** Seuls les jours ouvrables comptent dans `search_depth`.

**Exemple :**
- search_depth = 15 jours calendaires
- Jours ouvrables effectifs = 12 jours (hors week-ends)

### RT-06 : Paramètres Configurables (Consul)

| Paramètre | Description | Valeur Défaut | Module |
|-----------|-------------|---------------|--------|
| `search_depth` | Horizon temporel analyse (jours) | 15 | Module 1 |
| `Imprev_Fact` | Facteur d'imprévu | Calculé dynamiquement | Module 1 |
| `seuil_UrT_imminent` | J0-J5 → UrT=100 | [1,5] | Module 2 |
| `seuil_UrT_modere` | J6-J8 → UrT=50 | [6,8] | Module 2 |
| `importance_scannable` | Importance scannables | 100 | Module 2 |
| `importance_declarable` | Importance déclarables | 10 | Module 2 |

---

## 📋 6.7 Tableau Récapitulatif des Règles de Gestion

| Numéro RG | Contenu de la Règle de Gestion |
|-----------|--------------------------------|
| **MODULE 1 : CALCUL DE BESOIN** | |
| **M1-RG01** | **Calcul Stock Initial (J0)** : `stock_initial = stock_actuel + colis_en_transit + colis_pending`. Exclusions : guides, stickers, piles. |
| **M1-RG02** | **Calcul de Besoin avec Facteur d'Imprévu** : `QTE_BRUTE_FINALE = ARRONDI_SUP(QTE_BRUTE × Imprev_Fact)` où `Imprev_Fact = 1 + (heures_libres / totale_heures_shift)`. Paramètre `search_depth` configurable (Consul). |
| **M1-RG03** | **Projection Stock Jour par Jour** : `stock[jour] = stock[jour-1] - consommation_prevue[jour]`. Matrice de projection sur `search_depth` jours. |
| **MODULE 2 : ÉVALUATION URGENCE** | |
| **M2-RG1** | **Urgence Quantitative (UrQ)** : `UrQ = 100` si `stock_projection[jour] ≤ stock_min`, sinon `UrQ = 0`. Valeur finale = `MAX(UrQ[jour_1...jour_search_depth])`. |
| **M2-RG2** | **Urgence Temporelle (UrT)** : `UrT = 100` si rupture J1-J5 (imminent), `UrT = 50` si rupture J6-J8 (modéré), `UrT = 0` si rupture J9-J10 ou pas de rupture. |
| **M2-RG3** | **Importance du Matériel (ImP)** : `ImP = 100` pour scannables (utilisés quotidiennement), `ImP = 10` pour déclarables/consommables. |
| **M2-RG4** | **Urgence Totale (UrTT)** : `UrTT = UrT + UrQ + ImP` si `UrT × UrQ ≠ 0`, sinon `UrTT = 0` (court-circuit). Résultats possibles : {0, 160, 210, 250, 300}. |
| **M2-RG5** | **Classification Finale** : `CRITIQUE_A` (300), `URGENT_A` (250), `CRITIQUE_B` (210), `URGENT_B` (160), `SAFE` (0). |
| **MODULE 3 : CORE MOTEUR OPTIMISATION** | |
| **M3-RG01** | **Coefficient d'Occupation** : `Coefficient = 1 / Quantité_Max_Par_Carton`. Calcul nombre cartons : `Nombre_Cartons = ARRONDI_SUP(Σ(quantité × coefficient))`. |
| **M3-RG02** | **Hiérarchie de Criticité** : `(CRITIQUE_A = CRITIQUE_B = URGENT_A) > URGENT_B > SAFE`. Traitement : Prioritaires (100% garanti) > URGENT_B (complétion) > SAFE (knapsack). |
| **M3-RG03** | **Complétion URGENT_B avec Priorisation Intelligente** : Fonction de valorisation `CalculerValeurValorisationUrgentB` avec 2 facteurs : (1) Écart au `stock_min` (50%), (2) Efficacité occupation (50%). Tri par valeur décroissante avant placement. **SANS créer nouveaux cartons**. |
| **M3-RG04** | **Optimisation SAFE (Knapsack)** : Calculer quantités requises pour atteindre `stock_cible = (stock_min + stock_max) / 2`. Appliquer algorithme Knapsack Multi-Contraintes sur espace résiduel. **SANS créer nouveaux cartons**. |
| **M3-RG05** | **Fonction de Valorisation SAFE** : Fonction composite avec 2 facteurs : (1) Écart au `stock_cible` (50%), (2) Efficacité occupation (50%). Priorise articles les plus loin du stock optimal et favorise diversité dans l'espace limité. |
| **M3-RG06** | **Algorithme TraiterArticlesPrioritaires** : Calculer `occupation_totale`, créer `PLAFOND(occupation_totale)` cartons, distribuer articles prioritaires (CRITIQUE_A/B + URGENT_A). **Garantie 100% placement**. |
| **M3-RG07** | **Algorithme Knapsack Multi-Contraintes** : Programmation dynamique pour optimiser placement SAFE. Utilise `CalculerValeurValorisationSafe` pour priorisation. Reconstruction solution optimale. |

---

## 📚 7. Références et Liens

### Tickets Jira Associés

- **TOP-6187** : Module Calcul de Besoin
- **TOP-6188** : Module Évaluation Urgence
- **TOP-6219** : Reporting et Métriques
- **TOP-6180** : MDD Tables Paramétrables

### Documentation Technique

- [Diagrammes Complets Mermaid](https://github.com/Singularity-Institute/PTL-r-appro-/blob/main/Diagrammes-OptimiserColis-Visuels.md)
- Tables BDD : `Cartons`, `Citicity_levels`, `ArticleType_Carton_parameters`, `AutoAPP_Parameters`

### Glossaire

| Terme | Définition |
|-------|------------|
| **APP** | Application de Paiement (Colis technicien) |
| **GBH** | Distributeur GBH |
| **ORG** | Distributeur ORG |
| **UrQ** | Urgence Quantitative (0 ou 100) |
| **UrT** | Urgence Temporelle (0, 50 ou 100) |
| **ImP** | Importance Matériel (10 ou 100) |
| **UrTT** | Urgence Totale (0, 160, 210, 250 ou 300) |
| **search_depth** | Horizon temporel analyse (jours) |
| **Imprev_Fact** | Facteur d'imprévu planification |
| **Court-circuit** | Bypass knapsack pour critiques |
| **Knapsack** | Algorithme optimisation contrainte |

---

## 🎯 8. Vue Synthétique pour Management

### 8.1 Vision d'Ensemble du Système

Le système de réapprovisionnement automatique garantit que chaque technicien dispose du matériel nécessaire au bon moment, tout en minimisant les coûts logistiques.

```mermaid
graph LR
    subgraph "INPUT 📥"
        A[Planning<br/>Technicien]
        B[Stock<br/>Actuel]
        C[Colis en<br/>Transit]
    end

    subgraph "PROCESSUS INTELLIGENT 🧠"
        D[📊 MODULE 1<br/>Calcul Besoins<br/>Futurs]
        E[🚨 MODULE 2<br/>Évaluation<br/>Urgences]
        F[🎯 MODULE 3<br/>Optimisation<br/>Colis]
    end

    subgraph "OUTPUT 📤"
        G[Proposition APP<br/>Optimisée]
    end

    A --> D
    B --> D
    C --> D
    D --> E
    E --> F
    F --> G

    style D fill:#e1f5fe
    style E fill:#fff8e1
    style F fill:#e8f5e9
    style G fill:#c8e6c9
```

### 8.2 Valeur Métier du Système

```mermaid
mindmap
  root((Système<br/>Réappro<br/>Auto))
    **Bénéfices Opérationnels**
      Zéro rupture matériel critique
      Techniciens toujours équipés
      Anticipation 15 jours
    **Bénéfices Économiques**
      Réduction nombre colis
      Optimisation transport
      Moins de surstockage
    **Bénéfices Qualité**
      Décisions data-driven
      Traçabilité complète
      Métriques en temps réel
    **Bénéfices Stratégiques**
      Satisfaction client
      Performance techniciens
      Agilité logistique
```

### 8.3 Fonctionnement en 3 Étapes

```mermaid
flowchart TB
    START([🎬 Déclenchement pour<br/>un Technicien]) --> M1

    subgraph STEP1["<b>ÉTAPE 1: PRÉVISION DES BESOINS</b> 📊"]
        M1[Analyse du planning<br/>sur 15 jours]
        M1 --> M2[Calcul consommation<br/>prévisionnelle]
        M2 --> M3[Intègre facteur<br/>d'imprévu]
        M3 --> M4[Projection stock<br/>jour par jour]
    end

    M4 --> S2

    subgraph STEP2["<b>ÉTAPE 2: PRIORISATION INTELLIGENTE</b> 🚨"]
        S2[Détection des ruptures<br/>de stock futures]
        S2 --> S3{Rupture<br/>détectée ?}
        S3 -->|Oui| S4[Classification par<br/>niveau d'urgence]
        S3 -->|Non| S5[Article SAFE]
        S4 --> S6[CRITIQUE_A: 300 pts<br/>Scannable J0-J5]
        S4 --> S7[URGENT_A: 250 pts<br/>Scannable J6-J8]
        S4 --> S8[CRITIQUE_B: 210 pts<br/>Déclarable J0-J5]
        S4 --> S9[URGENT_B: 160 pts<br/>Déclarable J6-J8]
    end

    S6 --> STEP3
    S7 --> STEP3
    S8 --> STEP3
    S9 --> STEP3
    S5 --> STEP3

    subgraph STEP3["<b>ÉTAPE 3: OPTIMISATION COLIS</b> 🎯"]
        O1[Phase 1<br/>Articles PRIORITAIRES<br/>100% garantis]
        O1 --> O2[Phase 2<br/>Complétion URGENT_B<br/>si espace disponible]
        O2 --> O3[Phase 3<br/>Optimisation SAFE<br/>algorithme knapsack]
        O3 --> O4[Validation &<br/>Génération rapport]
    end

    O4 --> END([✅ Proposition APP<br/>Optimisée])

    style STEP1 fill:#e1f5fe
    style STEP2 fill:#fff8e1
    style STEP3 fill:#e8f5e9
    style S6 fill:#ffcdd2
    style S7 fill:#ffccbc
    style S8 fill:#ffe0b2
    style S9 fill:#fff9c4
    style END fill:#c8e6c9
```

### 8.4 Système de Priorisation

```mermaid
graph TD
    START[Article à<br/>réapprovisionner] --> EVAL{Évaluation<br/>Urgence}

    EVAL -->|UrTT=300| CRIT_A[🔴 CRITIQUE_A<br/>Scannable<br/>Rupture J0-J5]
    EVAL -->|UrTT=250| URG_A[🟠 URGENT_A<br/>Scannable<br/>Rupture J6-J8]
    EVAL -->|UrTT=210| CRIT_B[🟡 CRITIQUE_B<br/>Déclarable<br/>Rupture J0-J5]
    EVAL -->|UrTT=160| URG_B[🟢 URGENT_B<br/>Déclarable<br/>Rupture J6-J8]
    EVAL -->|UrTT=0| SAFE[⚪ SAFE<br/>Pas de rupture<br/>prévue]

    CRIT_A --> TREAT1[Traitement:<br/>Court-circuit<br/>100% garanti]
    URG_A --> TREAT1
    CRIT_B --> TREAT1

    URG_B --> TREAT2[Traitement:<br/>Complétion<br/>Si espace dispo]

    SAFE --> TREAT3[Traitement:<br/>Optimisation<br/>Knapsack]

    TREAT1 --> RESULT[Proposition APP]
    TREAT2 --> RESULT
    TREAT3 --> RESULT

    style CRIT_A fill:#ff6b6b,color:#fff
    style URG_A fill:#ffa07a
    style CRIT_B fill:#ffd93d
    style URG_B fill:#95e1d3
    style SAFE fill:#e0e0e0
    style RESULT fill:#4caf50,color:#fff
```

### 8.5 Métriques Clés & Indicateurs de Performance

| Indicateur | Description | Objectif |
|------------|-------------|----------|
| **Taux de Rupture** | % articles critiques en rupture | **0%** |
| **Taux d'Optimisation** | Réduction nombre colis vs. baseline | **-30%** |
| **Taux de Remplissage** | Occupation moyenne des cartons | **> 85%** |
| **Anticipation Moyenne** | Jours d'avance détection rupture | **> 5 jours** |
| **Précision Prévision** | Écart consommation réelle vs. prévue | **< 15%** |

### 8.6 ROI & Impact Business

**Matrice Impact vs Complexité :**

| Module | Complexité | Impact Business | Catégorie |
|--------|-----------|-----------------|-----------|
| **Module 2 - Évaluation Urgence** | ⭐⭐ Moyenne | ⭐⭐⭐⭐⭐ Très Fort | 🎯 Quick Win |
| **Module 1 - Calcul Besoin** | ⭐⭐⭐ Élevée | ⭐⭐⭐⭐⭐ Très Fort | 🎯 Quick Win |
| **Intégration GBH/ORG** | ⭐ Faible | ⭐⭐⭐⭐ Fort | 🎯 Quick Win |
| **Module 3 - Optimisation** | ⭐⭐⭐⭐ Très Élevée | ⭐⭐⭐⭐⭐ Très Fort | 🚀 Stratégique |
| **Reporting & Métriques** | ⭐ Faible | ⭐⭐⭐ Moyen | 🔧 Optimisation Future |

**Stratégie de déploiement recommandée :**
1. **Phase 1 (Quick Wins)** : Module 2 → Module 1 → Intégration GBH/ORG
2. **Phase 2 (Stratégique)** : Module 3 avec optimisation avancée
3. **Phase 3 (Amélioration)** : Reporting & Métriques avancées

**Gains estimés (annuels) :**
- 💰 **Réduction coûts transport** : -25% (moins de colis)
- ⏱️ **Gain productivité techniciens** : +15% (moins d'arrêts matériel)
- 📉 **Réduction stock immobilisé** : -20% (juste nécessaire)
- 🎯 **Amélioration satisfaction client** : +30% (interventions réussies)

---

**Document consolidé créé le :** 2025-09-30
**Auteurs :** Équipe Logistique PTL
**Version :** 1.0 - Lot 1

---

**🎯 Ce document constitue la référence complète pour l'implémentation du système de réapprovisionnement automatique.**
