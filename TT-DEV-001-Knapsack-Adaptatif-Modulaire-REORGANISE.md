# TT-DEV-001 - Implémentation Algorithme Knapsack Adaptatif Modulaire

## 📋 1. CONTEXTE ET OBJECTIFS MÉTIER

### Epic : Optimisation Automatique des Colis
**Type :** Feature | **Priorité :** High | **Estimation :** 13 points

### Description Fonctionnelle
Implémenter un système d'optimisation de colis utilisant un algorithme de knapsack adaptatif et modulaire pour gérer différents niveaux de criticité des matériels avec des contraintes d'occupation par type d'article.

### Valeur Métier
- 🎯 **Garantir 100%** des matériels critiques dans les colis (zéro rupture)
- 📦 **Optimiser l'utilisation** de l'espace selon la criticité des articles
- ⚡ **Réduire le nombre** de colis tout en respectant les priorités métier
- 📈 **Améliorer la satisfaction** client par une livraison prioritaire intelligente

---

## 📊 2. DONNÉES MÉTIER ET CONFIGURATION

### Niveaux de Criticité (Par Ordre de Priorité Décroissante)
```
1. CRITIQUE_A    (Criticité Maximale)
2. CRITIQUE_B    (Criticité Élevée)
3. URGENT_A      (Urgent Important)
4. URGENT_B      (Urgent Standard)
5. SAFE          (Sécurisé - Optimisable)
```

### Coefficients d'Occupation par Type d'Article
```
TYPE_1 = 0.20  (20% de l'espace carton)
TYPE_2 = 0.25  (25% de l'espace carton)
TYPE_3 = 0.10  (10% de l'espace carton)
CAPACITE_CARTON = 1.0  (100% - Limite absolue)
```

### Formule de Base
```
occupation_article = quantité × coefficient_type
occupation_carton = Σ(occupation_article) ≤ 1.0
```

---

## 🎯 3. STRATÉGIE ALGORITHMIQUE GLOBALE

### Vue d'Ensemble - 4 Phases Séquentielles

```mermaid
flowchart LR
    A[📋 Articles Input] --> B[🚨 Phase 1: CRITIQUES]
    B --> C[⚡ Phase 2: URGENT_B]
    C --> D[🎯 Phase 3: SAFE]
    D --> E[✅ Phase 4: Validation]

    B --> F[Court-Circuit<br/>Garantie 100%]
    C --> G[Complétion<br/>Quantités Partielles]
    D --> H[Knapsack<br/>Optimisation]
    E --> I[Contrôles<br/>Métriques]

    style B fill:#ff9999
    style C fill:#ffcc99
    style D fill:#99ff99
    style E fill:#99ccff
```

### Logique de Décision par Phase

| Phase | Articles Traités | Algorithme | Objectif | Court-Circuit |
|-------|-----------------|------------|----------|---------------|
| **Phase 1** | CRITIQUE_A/B + URGENT_A | **Court-Circuit** | Garantie 100% | ✅ Oui |
| **Phase 2** | URGENT_B | **Complétion Intelligente** | Max satisfaction | ❌ Non |
| **Phase 3** | SAFE | **Knapsack Multi-Contraintes** | Valorisation stock | ❌ Non |
| **Phase 4** | Résultat Global | **Validation & Métriques** | Qualité finale | ❌ Non |

---

## 📝 4. RÈGLES DE GESTION PRÉCISES

### RG-001 : Classification et Priorités
**Règle :** `CRITIQUE_A > CRITIQUE_B > URGENT_A > URGENT_B > SAFE`
**Application :** Les articles critiques (A/B + URGENT_A) **court-circuitent** l'algorithme de knapsack

### RG-002 : Calcul d'Occupation des Cartons
**Formule :** `occupation_totale = Σ(quantité_article × coefficient_type)`
**Contrainte Absolue :** `occupation_totale ≤ 1.0`

### RG-003 : Stratégie Court-Circuit (Articles Critiques)
**Règle :** Utiliser `PLAFOND(occupation_requise)` cartons pour garantir 100% des critiques
**Justification :** Aucun compromis acceptable sur les articles critiques

### RG-004 : Algorithme Knapsack (Articles SAFE)
**Objectif :** Maximiser valorisation stock avec cible `(stock_min + stock_max) / 2`
**Contrainte :** Respecter les capacités restantes des cartons existants

### RG-005 : Gestion Quantités Partielles URGENT_B ⚠️
**Règle Critique :** Si aucun carton avec capacité disponible → accepter quantité partielle
**Action :** Passer à la finalisation **SANS** créer nouveau carton

### RG-006 : Discrétisation Knapsack
**Règle Technique :** Discrétiser capacités continues avec précision 0.01
**Implémentation :** `capacite_discretisee = ARRONDI(capacite_continue × 100)`

### RG-007 : Validation Contraintes Finales
**Contrôles :** Occupation ≤ 100%, articles critiques à 100%, cohérence données

---

## 🏗️ 5. ARCHITECTURE TECHNIQUE DÉTAILLÉE

### Services Core à Implémenter

```mermaid
classDiagram
    class ServiceOptimisationColis {
        <<Orchestrateur Principal>>
        +optimiserColis(OptimisationContext) PackingResult
        +executerPhase1Critiques(articles) List~Carton~
        +executerPhase2UrgentB(cartons, articles) List~Carton~
        +executerPhase3Safe(cartons, articles) List~Carton~
        +executerPhase4Validation(cartons) PackingResult
    }

    class ServiceCalculOccupation {
        <<Calculs Fondamentaux>>
        +calculerOccupationRequise(List~Article~) double
        +calculerNombreCartonsNecessaires(double) int
        +verifierCapaciteCarton(Carton, Article) boolean
        +distribuerArticlesParCartons(articles, nb_cartons) Map~Integer,List~Article~~
    }

    class ServiceClassificationArticles {
        <<Classification & Filtrage>>
        +filtrerParGrade(List~Article~, List~GradeCriticite~) List~Article~
        +trierParPriorite(List~Article~) List~Article~
        +identifierStrategieOptimisation(List~Article~) StrategieType
    }

    class ServiceProjectionStock {
        <<Calculs SAFE>>
        +calculerObjectifStockOptimal(Article, int) int
        +calculerQuantiteOptimale(Article, int) int
        +evaluerInteretValorisationStock(Article, int) double
    }

    class ServiceValidationContraintes {
        <<Validation & Contrôles>>
        +validerOccupationCarton(Carton) boolean
        +validerCoherenceDonnees(PackingResult) boolean
        +genererRapportValidation(PackingResult) ValidationReport
    }

    ServiceOptimisationColis --> ServiceCalculOccupation
    ServiceOptimisationColis --> ServiceClassificationArticles
    ServiceOptimisationColis --> ServiceProjectionStock
    ServiceOptimisationColis --> ServiceValidationContraintes
```

### Modèles de Données Principaux

```java
// Contexte d'entrée
public class OptimisationContext {
    List<Article> articles_input;           // Articles à traiter
    CartonConstraints contraintes_carton;   // Contraintes physiques
    Map<ArticleType, Double> coefficients_occupation; // Règles métier
    int search_depth;                       // Horizon projections (SAFE)
}

// Résultat final
public class PackingResult {
    List<Carton> cartons_finaux;           // Cartons optimisés
    MetriquesOptimisation metriques;       // KPIs performance
    List<ArticlePartiel> quantites_partielles; // URGENT_B partiels
    boolean validation_success;            // État validation
    long temps_execution_ms;               // Performance
}

// Article à traiter
public class Article {
    String id;
    ArticleType type;                      // TYPE_1, TYPE_2, TYPE_3
    GradeCriticite grade;                  // CRITIQUE_A → SAFE
    int quantite;
    double coefficient_occupation;          // Selon type
    int[] stock_projections;               // Pour SAFE uniquement
    int stock_minimum, stock_maximum;      // Pour SAFE uniquement
}

// Carton résultat
public class Carton {
    String id;
    double occupation_actuelle;            // 0.0 à 1.0
    List<Article> articles_contenus;       // Articles placés
    double capacite_maximale = 1.0;        // Constante
    boolean est_finalise;                  // État
}
```

---

## 🔄 6. ALGORITHMES DÉTAILLÉS

### Algorithme Principal (Orchestrateur)

```java
ALGORITHME OptimiserColis(context)
DEBUT
    // === PHASE 1: TRAITEMENT CRITIQUES (Court-Circuit) ===
    articles_critiques ← FiltrerParGrade(context.articles, [CRITIQUE_A, CRITIQUE_B, URGENT_A])
    cartons_resultats ← TraiterArticlesCritiques(articles_critiques)

    // === PHASE 2: COMPLÉTION URGENT_B ===
    articles_urgent_b ← FiltrerParGrade(context.articles, [URGENT_B])
    SI articles_urgent_b NON VIDE ALORS
        cartons_resultats ← CompleterAvecUrgentB(cartons_resultats, articles_urgent_b)
    FIN_SI

    // === PHASE 3: OPTIMISATION SAFE (Knapsack) ===
    articles_safe ← FiltrerParGrade(context.articles, [SAFE])
    SI articles_safe NON VIDE ALORS
        cartons_resultats ← OptimiserAvecSafe(cartons_resultats, articles_safe)
    FIN_SI

    // === PHASE 4: VALIDATION ET MÉTRIQUES ===
    RETOURNER ValiderEtGenererRapport(cartons_resultats)
FIN
```

### Algorithme Court-Circuit (Phase 1)

```java
ALGORITHME TraiterArticlesCritiques(articles_critiques)
DEBUT
    occupation_totale ← CalculerOccupationRequise(articles_critiques)
    nombre_cartons ← PLAFOND(occupation_totale)  // Garantie mathématique

    cartons ← CreerCartons(nombre_cartons)
    distribution ← DistribuerArticlesParCartons(articles_critiques, nombre_cartons)

    POUR chaque carton DANS cartons FAIRE
        POUR chaque article DANS distribution[carton.index] FAIRE
            PlacerArticle(carton, article)  // Placement garanti
        FIN_POUR
    FIN_POUR

    RETOURNER cartons
FIN
```

### Algorithme Gestion Quantités Partielles (Phase 2)

```mermaid
flowchart TD
    A[⚡ Début Phase URGENT_B] --> B[📋 Pour chaque article URGENT_B]
    B --> C[📊 quantite_restante = article.quantite_totale]

    C --> D[🔄 Pour chaque carton existant]
    D --> E[📐 capacite_libre = 1.0 - carton.occupation]
    E --> F[⚖️ quantite_max = capacite_libre / article.coefficient]

    F --> G{❓ quantite_max >= quantite_restante ?}
    G -->|OUI| H[✅ Placer quantité totale]
    G -->|NON| I[📊 Placer quantité partielle]

    H --> J[🔄 quantite_restante = 0]
    I --> K[🔄 quantite_restante -= quantite_partielle]

    J --> L{🔄 Article suivant ?}
    K --> M{❓ quantite_restante > 0 ?}

    M -->|OUI| N{🔄 Carton suivant disponible ?}
    M -->|NON| L

    N -->|OUI| D
    N -->|NON| O[🚨 DÉCISION CRITIQUE RG-005]

    O --> P[✅ Accepter quantité partielle]
    P --> Q[📋 Enregistrer dans rapport]
    Q --> L

    L -->|OUI| B
    L -->|NON| R[✅ Fin Phase URGENT_B]

    style O fill:#ff6b6b
    style P fill:#4ecdc4
    style Q fill:#45b7d1
```

### Algorithme Knapsack Multi-Contraintes (Phase 3)

```java
ALGORITHME KnapsackMultiContraintes(articles_safe, cartons_existants)
DEBUT
    POUR CHAQUE carton DANS cartons_existants FAIRE
        capacite_restante ← 1.0 - carton.occupation_actuelle
        capacite_discretisee ← ARRONDI(capacite_restante × 100)  // RG-006
        articles_candidats ← FiltrerParCapacite(articles_safe, capacite_restante)

        // === PROGRAMMATION DYNAMIQUE ===
        dp ← InitialiserTableDP(articles_candidats.size, capacite_discretisee)

        POUR i DE 1 À articles_candidats.size FAIRE
            POUR w DE 0 À capacite_discretisee FAIRE
                article ← articles_candidats[i-1]
                cout_occupation_discret ← ARRONDI(article.quantite × article.coefficient × 100)
                valeur_stock ← CalculerValeurValorisationStock(article)  // (min+max)/2 - stock_final

                SI cout_occupation_discret <= w ALORS
                    dp[i][w] ← MAX(dp[i-1][w], dp[i-1][w-cout_occupation_discret] + valeur_stock)
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

---

## 📋 7. VARIABLES ET PARAMÈTRES DÉTAILLÉS

### Phase 1 : Variables Court-Circuit

| Variable | Type | Description | Valeurs | Rôle |
|----------|------|-------------|---------|------|
| `articles_critiques` | List\<Article\> | CRITIQUE_A/B + URGENT_A | 0-N articles | Entrée phase 1 |
| `occupation_totale` | double | Somme occupations critiques | 0.0-N.0 | Calcul besoins |
| `nombre_cartons` | int | PLAFOND(occupation_totale) | 1-N cartons | Cartons à créer |
| `distribution` | Map\<Integer,List\<Article\>\> | Répartition par carton | - | Stratégie placement |

### Phase 2 : Variables Quantités Partielles

| Variable | Type | Description | Valeurs | Rôle |
|----------|------|-------------|---------|------|
| `quantite_restante` | int | Quantité non encore placée | 0-N | Suivi progression |
| `capacite_libre` | double | Espace disponible carton | 0.0-1.0 | Contrainte placement |
| `quantite_max_possible` | int | Max plaçable dans carton | 0-N | Limite calculée |
| `quantite_partielle` | int | Quantité effectivement placée | 0-quantite_restante | Résultat partiel |

### Phase 3 : Variables Knapsack

| Variable | Type | Description | Valeurs | Rôle |
|----------|------|-------------|---------|------|
| `capacite_discretisee` | int | Capacité × 100 (précision 0.01) | 0-100 | Contrainte DP |
| `dp[i][w]` | double | Valeur optimale i articles, capacité w | 0.0-MAX | Mémorisation DP |
| `cout_occupation_discret` | int | (quantité × coeff) × 100 | 1-100 | Poids discret |
| `valeur_stock` | double | (min+max)/2 - stock_final | R | Utilité article |
| `solution_optimale` | List\<Article\> | Articles sélectionnés | 0-M articles | Résultat knapsack |

---

## ✅ 8. CRITÈRES D'ACCEPTATION FONCTIONNELS

### CA-001 : Court-Circuit Articles Critiques ✅
```gherkin
Given une liste d'articles contenant des matériels CRITIQUE_A et CRITIQUE_B
  And des coefficients d'occupation TYPE_1=0.2, TYPE_2=0.25, TYPE_3=0.1
When j'exécute l'algorithme d'optimisation des colis
Then tous les articles CRITIQUE_A et CRITIQUE_B sont placés à 100%
  And le nombre de cartons créés = PLAFOND(occupation_totale_critiques)
  And aucun algorithme de knapsack n'est exécuté pour ces articles
  And le temps d'exécution phase 1 < 100ms pour 1000 articles critiques
```

### CA-002 : Calcul Mathématique Exact URGENT_A ✅
```gherkin
Given une liste d'articles contenant 50 unités URGENT_A de TYPE_1 (coeff 0.2)
When j'exécute l'algorithme d'optimisation des colis
Then les 50 unités URGENT_A sont placées à 100%
  And occupation_requise = 50 × 0.2 = 10.0
  And nombre_cartons_créés = PLAFOND(10.0) = 10 cartons
  And le traitement utilise la stratégie court-circuit (pas knapsack)
```

### CA-003 : Optimisation SAFE avec Objectif (min+max)/2 ✅
```gherkin
Given des articles SAFE avec projections de stock
  And stock_min = 10, stock_max = 50 pour un article
  And des cartons avec espace restant disponible
When j'exécute l'algorithme d'optimisation des colis
Then l'objectif de stock calculé = (10 + 50) / 2 = 30
  And l'algorithme de knapsack maximise la valorisation vers cet objectif
  And les articles SAFE sont placés selon la solution optimale du knapsack
```

### CA-004 : Gestion Quantités Partielles URGENT_B (RG-005) 🚨
```gherkin
Given des articles URGENT_B à placer
  And tous les cartons existants ont une occupation ≥ 95%
  And aucun carton n'a la capacité pour la quantité totale URGENT_B
When j'exécute l'algorithme d'optimisation des colis
Then l'algorithme place la quantité partielle possible dans cartons existants
  And aucun nouveau carton n'est créé pour URGENT_B
  And le processus passe directement à la phase SAFE
  And le résultat indique les quantités partielles acceptées dans le rapport
```

### CA-005 : Séquençage 4 Phases Garanti ✅
```gherkin
Given une liste mixte d'articles de tous types de criticité
When j'exécute l'algorithme d'optimisation des colis
Then Phase 1: Articles CRITIQUE_A/B/URGENT_A traités en court-circuit
  And Phase 2: Articles URGENT_B complètent les cartons existants (quantités partielles si nécessaire)
  And Phase 3: Articles SAFE optimisés par knapsack sur espace restant uniquement
  And Phase 4: Validation globale et génération du rapport final avec métriques
```

### CA-006 : Performance et Limites Techniques ⚡
```gherkin
Given une liste d'articles avec quantités très importantes
  And coefficients d'occupation générant des besoins > 1000 cartons
When j'exécute l'algorithme d'optimisation des colis
Then l'algorithme gère les gros volumes sans dégradation de performance
  And le temps d'exécution reste < 10 secondes pour 10000 articles
  And la mémoire utilisée reste raisonnable (< 1GB)

Given un knapsack avec 1000 articles SAFE et capacité restante très fractionnée
When j'exécute l'optimisation knapsack avec discrétisation 0.01
Then le temps d'exécution reste < 5 secondes
  And la table DP n'excède pas 100MB de mémoire
  And la solution trouvée est à moins de 5% de l'optimal théorique
```

### CA-007 : Gestion d'Erreurs Robuste 🛡️
```gherkin
Given des données d'entrée invalides (coefficients négatifs, quantités nulles)
When j'exécute l'algorithme d'optimisation des colis
Then l'algorithme génère des erreurs de validation appropriées
  And aucun traitement n'est effectué sur des données corrompues
  And les messages d'erreur sont explicites et exploitables

Given une liste d'articles avec des coefficients d'occupation > 1.0
When j'exécute l'algorithme d'optimisation des colis
Then une exception "ArticleOccupationInvalidException" est levée
  And le message indique "Article {id} avec coefficient {coeff} > 1.0 impossible"

Given des articles SAFE sans projections de stock valides
When j'exécute l'algorithme d'optimisation des colis
Then les articles SAFE sont ignorés de l'optimisation knapsack
  And un warning est généré dans les logs
  And le rapport final indique les articles ignorés avec justification
```

---

## 🧪 9. STRATÉGIE DE TESTS DÉTAILLÉE

### Tests Unitaires par Service

#### ServiceCalculOccupation
```java
@Test testCalculOccupationSimple() {
    // 10 articles TYPE_1 (coeff 0.2) → occupation = 2.0
    assertEquals(2.0, service.calculerOccupationRequise(articles_TYPE_1_x10));
}

@Test testCalculNombreCartonsNecessaires() {
    // occupation 2.3 → PLAFOND(2.3) = 3 cartons
    assertEquals(3, service.calculerNombreCartonsNecessaires(2.3));
}

@Test testVerificationCapaciteCarton() {
    // carton 80% + article nécessitant 30% → false
    assertFalse(service.verifierCapaciteCarton(carton_80pourcent, article_30pourcent));
}
```

#### ServiceProjectionStock
```java
@Test testCalculObjectifStockOptimal() {
    // stock_min=10, stock_max=50 → objectif = (10+50)/2 = 30
    assertEquals(30, service.calculerObjectifStockOptimal(article_avec_projections));
}

@Test testEvaluationInteretValorisationStock() {
    // Plus l'écart objectif est grand, plus l'intérêt est élevé
    assertTrue(service.evaluerInteretValorisationStock(article_ecart_20) >
               service.evaluerInteretValorisationStock(article_ecart_5));
}
```

### Tests d'Intégration Algorithmiques

```java
@Test testCasUniqueementCritiques() {
    // Entrée: seulement articles CRITIQUE_A/B
    // Résultat attendu: court-circuit pur, pas de knapsack exécuté
    assertThat(result.algorithmes_executes).containsOnly("COURT_CIRCUIT");
}

@Test testCasMixteComplet() {
    // Entrée: articles de toutes criticités
    // Résultat: 4 phases exécutées dans l'ordre
    assertThat(result.phases_executees).containsExactly("CRITIQUES", "URGENT_B", "SAFE", "VALIDATION");
}

@Test testGestionQuantitesPartielles() {
    // Cartons saturés + articles URGENT_B
    // Résultat: quantités partielles acceptées, pas de nouveaux cartons
    assertEquals(0, result.nouveaux_cartons_urgent_b);
    assertTrue(result.quantites_partielles.size() > 0);
}
```

---

## 📊 10. MÉTRIQUES ET MONITORING

### KPI Performance Requis

| Métrique | Valeur Cible | Criticité | Mesure |
|----------|--------------|-----------|---------|
| **Temps Exécution Global** | < 5s pour 1000 articles | HIGH | Chronomètre algorithme complet |
| **Mémoire Peak** | < 512MB | MEDIUM | Monitoring heap JVM |
| **Taux Satisfaction Critiques** | 100% | CRITICAL | articles_places/articles_critiques |
| **Taux Occupation Moyen** | > 85% | HIGH | moyenne(carton.occupation) |
| **Efficacité Knapsack** | > 90% optimal | MEDIUM | solution_trouvée/solution_optimale |

### Tableau de Bord Opérationnel

```java
public class MetriquesOptimisation {
    // Performance
    long temps_execution_total_ms;
    long temps_phase_1_ms, temps_phase_2_ms, temps_phase_3_ms;
    long memoire_peak_bytes;

    // Satisfaction
    double taux_satisfaction_critiques;    // Doit être 100%
    double taux_satisfaction_urgent_b;     // Peut être < 100% (quantités partielles)
    double taux_satisfaction_safe;         // Variable selon optimisation

    // Efficacité
    int nombre_cartons_totaux;
    double taux_occupation_moyen;
    double efficacite_knapsack_pourcent;

    // Qualité
    List<ArticlePartiel> quantites_partielles; // Détail URGENT_B
    List<String> warnings;                      // Articles SAFE ignorés, etc.
    boolean validation_globale_success;
}
```

---

## 🚀 11. DÉFINITION OF DONE

### Checklist Technique ✅
- [ ] **Services** : Tous les services implémentés selon les interfaces définies
- [ ] **Algorithmes** : Court-circuit, quantités partielles, knapsack fonctionnels
- [ ] **Validation** : Tous les critères d'acceptation CA-001 à CA-007 validés
- [ ] **Tests** : Couverture > 90% avec tests unitaires et d'intégration
- [ ] **Performance** : Benchmarks respectés (< 5s, < 512MB)
- [ ] **Gestion Erreurs** : Exceptions appropriées et messages explicites

### Checklist Qualité ✅
- [ ] **Code Review** : Approuvé par tech lead + validation architecture
- [ ] **Documentation** : JavaDoc complète + README technique
- [ ] **Logging** : Niveaux appropriés (INFO, WARN, ERROR) avec contexte
- [ ] **Monitoring** : Métriques exposées pour supervision
- [ ] **Sécurité** : Validation inputs, pas de failles injection

### Checklist Déploiement ✅
- [ ] **Tests Environnement** : Validation en environnement de test
- [ ] **Configuration** : Paramètres externalisés (coefficients, timeouts)
- [ ] **Rollback** : Stratégie de retour arrière documentée
- [ ] **Formation** : Documentation utilisateur + formation équipes

---

## 📚 12. RESSOURCES ET RÉFÉRENCES

### Algorithmes et Concepts
- **Knapsack Problem** : Programmation dynamique multi-contraintes
- **Design Patterns** : Strategy (phases), Factory (cartons), Chain of Responsibility
- **Complexity** : Court-circuit O(n), Knapsack O(n×W×C) où W=capacité, C=cartons

### Documentation Technique
- **Architecture Détaillée** : `Conception-Knapsack-Adaptatif-Modulaire.md`
- **Spécifications Originales** : Ticket métier avec diagrammes fonctionnels
- **API Documentation** : Interfaces Java avec exemples d'usage

---

## 📚 13. TABLEAUX DE RÉFÉRENCE DÉVELOPPEURS/ARCHITECTES

### 🔧 Référence Complète des Méthodes et APIs

| Service | Méthode | Entrées | Sortie | Fonctionnement | Utilisation | Complexité | Phase |
|---------|---------|---------|--------|----------------|-------------|------------|-------|
| **ServiceOptimisationColis** | `optimiserColis(context)` | OptimisationContext | PackingResult | Orchestrateur principal exécutant les 4 phases séquentiellement | Point d'entrée unique, gère le flux complet | O(n + m×C + k×W×C) | Toutes |
| **ServiceOptimisationColis** | `executerPhase1Critiques(articles)` | List\<Article\> critiques | List\<Carton\> | Court-circuit : PLAFOND(occupation) cartons garantis | Traitement articles CRITIQUE_A/B/URGENT_A avec 100% garantie | O(n) | Phase 1 |
| **ServiceOptimisationColis** | `executerPhase2UrgentB(cartons, articles)` | Cartons existants + Articles URGENT_B | List\<Carton\> | Complétion intelligente avec gestion quantités partielles selon RG-005 | Remplissage optimal cartons existants avec URGENT_B | O(m×C) | Phase 2 |
| **ServiceOptimisationColis** | `executerPhase3Safe(cartons, articles)` | Cartons + Articles SAFE | List\<Carton\> | Knapsack multi-contraintes avec objectif (min+max)/2 | Optimisation valorisation stock sur espace restant | O(k×W×C) | Phase 3 |
| **ServiceOptimisationColis** | `executerPhase4Validation(cartons)` | List\<Carton\> | PackingResult | Validation contraintes + génération métriques + rapport final | Contrôle qualité et consolidation résultats | O(C) | Phase 4 |
| **ServiceCalculOccupation** | `calculerOccupationRequise(articles)` | List\<Article\> | double | Σ(quantité × coefficient) pour tous les articles | Calcul besoins en espace total pour court-circuit | O(n) | Phase 1 |
| **ServiceCalculOccupation** | `calculerNombreCartonsNecessaires(occupation)` | double | int | PLAFOND(occupation_totale) avec garantie mathématique | Détermination nombre cartons exact pour critiques | O(1) | Phase 1 |
| **ServiceCalculOccupation** | `verifierCapaciteCarton(carton, article)` | Carton + Article | boolean | Teste si (occupation_actuelle + article.occupation) ≤ 1.0 | Validation placement avant ajout article | O(1) | Phases 2,3 |
| **ServiceCalculOccupation** | `distribuerArticlesParCartons(articles, nb)` | List\<Article\> + int | Map\<Integer,List\<Article\>\> | Répartition équilibrée articles sur N cartons par algorithme round-robin | Distribution optimale articles critiques sur cartons créés | O(n) | Phase 1 |
| **ServiceClassificationArticles** | `filtrerParGrade(articles, grades)` | List\<Article\> + List\<Grade\> | List\<Article\> | Filtrage articles correspondant aux grades spécifiés (critiques, urgent_b, safe) | Séparation articles par criticité pour chaque phase | O(n) | Toutes |
| **ServiceClassificationArticles** | `trierParPriorite(articles)` | List\<Article\> | List\<Article\> | Tri décroissant : CRITIQUE_A > CRITIQUE_B > URGENT_A > URGENT_B > SAFE | Garantie ordre de traitement selon règles métier | O(n log n) | Phases 1,2 |
| **ServiceClassificationArticles** | `identifierStrategieOptimisation(articles)` | List\<Article\> | StrategieType | Analyse composition pour choisir : COURT_CIRCUIT, KNAPSACK, HYBRIDE | Optimisation performance selon type d'entrée | O(n) | Stratégie |
| **ServiceProjectionStock** | `calculerObjectifStockOptimal(article, depth)` | Article + int | int | Calcul (stock_min + stock_max) / 2 sur horizon temporel | Définition cible optimisation pour articles SAFE | O(depth) | Phase 3 |
| **ServiceProjectionStock** | `calculerQuantiteOptimale(article, objectif)` | Article + int | int | objectif_stock - stock_final_projete avec contraintes positives | Détermination quantité recommandée pour knapsack | O(1) | Phase 3 |
| **ServiceProjectionStock** | `evaluerInteretValorisationStock(article, depth)` | Article + int | double | Score basé sur écart objectif et impact stock : plus écart grand = plus intérêt | Priorisation articles SAFE dans fonction objectif knapsack | O(depth) | Phase 3 |
| **ServiceValidationContraintes** | `validerOccupationCarton(carton)` | Carton | boolean | Vérification occupation_actuelle ≤ 1.0 + cohérence articles_contenus | Contrôle intégrité finale chaque carton | O(articles_carton) | Phase 4 |
| **ServiceValidationContraintes** | `validerCoherenceDonnees(result)` | PackingResult | boolean | Validation globale : articles_placés ≤ articles_input, sommes cohérentes | Contrôle intégrité algorithme complet | O(total_articles) | Phase 4 |
| **ServiceValidationContraintes** | `genererRapportValidation(result)` | PackingResult | ValidationReport | Génération métriques, warnings, et rapport détaillé avec KPI | Production rapport final pour monitoring/audit | O(cartons + articles) | Phase 4 |
| **AlgorithmeKnapsack** | `initialiserTableDP(n, W)` | int n articles + int W capacité | double\[\]\[\] | Création matrice DP[n+1][W+1] initialisée à 0.0 | Préparation structure mémorisation programmation dynamique | O(n×W) | Phase 3 |
| **AlgorithmeKnapsack** | `calculerValeurValorisationStock(article)` | Article SAFE | double | (stock_min + stock_max)/2 - stock_final avec bonus écart important | Fonction utilité pour maximisation knapsack | O(1) | Phase 3 |
| **AlgorithmeKnapsack** | `reconstruireSolution(dp, articles, W)` | DP table + articles + capacité | List\<Article\> | Backtracking depuis dp[n][W] pour retrouver articles optimaux sélectionnés | Extraction solution optimale de la table de programmation dynamique | O(n) | Phase 3 |
| **AlgorithmeKnapsack** | `appliquerSolution(carton, solution)` | Carton + List\<Article\> | void | Placement effectif articles sélectionnés dans carton avec mise à jour occupation | Application concrète résultat knapsack sur carton physique | O(solution.size) | Phase 3 |
| **GestionQuantitesPartielles** | `calculerCapaciteLibre(carton)` | Carton | double | 1.0 - carton.occupation_actuelle avec validation ≥ 0 | Évaluation espace disponible pour placement URGENT_B | O(1) | Phase 2 |
| **GestionQuantitesPartielles** | `calculerQuantiteMaxPossible(capacite, article)` | double + Article | int | capacite_libre / article.coefficient avec PLANCHER() | Limite quantité plaçable selon contraintes physiques | O(1) | Phase 2 |
| **GestionQuantitesPartielles** | `placerQuantitePartielle(carton, article, qte)` | Carton + Article + int | boolean | Placement qte ≤ quantite_totale avec mise à jour occupation et contenu | Exécution placement partiel selon RG-005 | O(1) | Phase 2 |
| **GestionQuantitesPartielles** | `enregistrerQuantitePartielle(article, qte_placee)` | Article + int | ArticlePartiel | Création enregistrement quantité non satisfaite pour rapport | Traçabilité décisions quantités partielles pour audit | O(1) | Phase 2 |
| **UtilitairesCommuns** | `discretiserCapacite(capacite_continue)` | double | int | ARRONDI(capacite × 100) selon RG-006 pour DP | Transformation continue → discret pour knapsack résolvable | O(1) | Phase 3 |
| **UtilitairesCommuns** | `validerCoefficientsOccupation(coefficients)` | Map\<Type,Double\> | boolean | Vérification tous coefficients > 0 et ≤ 1.0 | Validation configuration métier avant traitement | O(types) | Init |
| **UtilitairesCommuns** | `calculerTauxOccupationMoyen(cartons)` | List\<Carton\> | double | Moyenne pondérée occupations avec gestion cartons vides | Métrique efficacité utilisation espace pour KPI | O(cartons) | Phase 4 |
| **UtilitairesCommuns** | `genererMetriquesPerformance(debut, fin, memoire)` | long + long + long | MetriquesOptimisation | Calcul temps exécution, mémoire peak, ratios satisfaction | Production indicateurs performance pour monitoring | O(1) | Phase 4 |

### 📊 Référence Complète des Variables et Paramètres

| Variable | Type | Domaine/Valeurs | Description Fonctionnelle | Utilisation Algorithme | Calcul/Assignation | Contraintes | Impact Performance | Phase |
|----------|------|----------------|---------------------------|----------------------|------------------|-------------|-------------------|--------|
| **context.articles_input** | List\<Article\> | 1 à N articles | Liste complète articles à traiter dans le colis | Point d'entrée unique, classifiée par criticité en début | Fourni par couche métier/UI | N > 0, articles valides | O(n) parcours | Entrée |
| **context.contraintes_carton** | CartonConstraints | capacite_max = 1.0 | Contraintes physiques cartons standardisés | Validation placements, calculs occupation | Configuration système | capacite > 0 | Constant | Toutes |
| **context.coefficients_occupation** | Map\<Type,Double\> | TYPE_1=0.2, TYPE_2=0.25, TYPE_3=0.1 | Règles métier occupation par type article | Calculs occupation, contraintes knapsack | Configuration métier RG-002 | 0 < coeff ≤ 1.0 | O(1) lookup | Toutes |
| **context.search_depth** | int | 1-30 jours | Horizon temporel projections stock pour SAFE | Calcul objectifs valorisation stock | Paramètre métier configurable | > 0, ≤ horizon_max | O(depth) calculs | Phase 3 |
| **articles_critiques** | List\<Article\> | 0 à N articles | Articles CRITIQUE_A/B + URGENT_A pour court-circuit | Phase 1 : traitement prioritaire garanti 100% | Filtrage grades critiques | Triés par priorité | O(n) traitement | Phase 1 |
| **articles_urgent_b** | List\<Article\> | 0 à M articles | Articles URGENT_B pour complétion avec quantités partielles | Phase 2 : remplissage cartons existants | Filtrage grade URGENT_B | Quantités partielles acceptées | O(m×C) placement | Phase 2 |
| **articles_safe** | List\<Article\> | 0 à K articles | Articles SAFE pour optimisation knapsack valorisation | Phase 3 : optimisation espace restant | Filtrage grade SAFE | Projections stock valides | O(k×W×C) knapsack | Phase 3 |
| **occupation_totale** | double | 0.0 à N×max(coeff) | Somme totale occupation articles critiques | Calcul nombre cartons nécessaires phase 1 | Σ(quantité × coefficient) | ≥ 0 | O(n) calcul | Phase 1 |
| **nombre_cartons_requis** | int | 1 à N cartons | Cartons nécessaires = PLAFOND(occupation_totale) | Création exacte cartons pour garantie 100% critiques | PLAFOND(occupation_totale) | ≥ 1 | O(1) calcul | Phase 1 |
| **cartons_resultats** | List\<Carton\> | 1 à N cartons | Cartons créés et progressivement remplis | État intermédiaire entre phases, résultat final | Création phase 1, modification phases 2-3 | occupation ≤ 1.0 | O(C) gestion | Toutes |
| **quantite_restante** | int | 0 à quantite_totale | Quantité article URGENT_B non encore placée | Suivi progression placement quantités partielles | quantite_totale - Σ(quantites_placees) | ≥ 0 | O(1) mise à jour | Phase 2 |
| **capacite_libre** | double | 0.0 à 1.0 | Espace disponible dans carton = 1.0 - occupation | Contrainte placement article URGENT_B | 1.0 - carton.occupation_actuelle | ≥ 0 | O(1) calcul | Phase 2 |
| **quantite_max_possible** | int | 0 à quantite_restante | Maximum quantité plaçable selon capacité carton | Limite calcul quantités partielles | PLANCHER(capacite_libre / coefficient) | ≤ quantite_restante | O(1) calcul | Phase 2 |
| **quantite_partielle** | int | 0 à quantite_max_possible | Quantité effectivement placée (peut être < demandée) | Implémentation RG-005 acceptation partielle | MIN(quantite_restante, quantite_max_possible) | ≤ quantite_totale | O(1) placement | Phase 2 |
| **capacite_restante** | double | 0.0 à 1.0 | Espace disponible carton pour knapsack SAFE | Contrainte continue knapsack multi-contraintes | 1.0 - occupation_actuelle par carton | ≥ 0 | O(1) par carton | Phase 3 |
| **capacite_discretisee** | int | 0 à 100 | Capacité × 100 pour programmation dynamique | Transformation continue → discret selon RG-006 | ARRONDI(capacite_restante × 100) | 0 ≤ val ≤ 100 | O(1) conversion | Phase 3 |
| **articles_candidats** | List\<Article\> | 0 à K articles | Articles SAFE pouvant entrer dans carton courant | Filtrage préalable knapsack par contrainte capacité | Filtrage capacité > cout_occupation | Capacité respectée | O(k) filtrage | Phase 3 |
| **dp[i][w]** | double | 0.0 à valeur_max | Valeur optimale knapsack i articles, capacité w | Table mémorisation programmation dynamique | dp[i-1][w] ou dp[i-1][w-cout]+valeur | ≥ 0, croissante | O(n×W) espace | Phase 3 |
| **cout_occupation_discret** | int | 1 à 100 | Occupation article × 100 pour DP discrète | Poids article dans contrainte knapsack discrète | ARRONDI(quantité × coefficient × 100) | 1 ≤ val ≤ capacite_discretisee | O(1) calcul | Phase 3 |
| **valeur_stock** | double | R+ | Utilité valorisation stock pour fonction objectif | Maximisation knapsack : plus valeur haute = plus intérêt | (min+max)/2 - stock_final + bonus_ecart | > 0 si pertinent | O(1) calcul | Phase 3 |
| **solution_optimale** | List\<Article\> | 0 à M articles | Articles sélectionnés par knapsack pour carton | Résultat backtracking table DP | Reconstruction depuis dp[n][W] | Respecte contraintes | O(n) reconstruction | Phase 3 |
| **stock_projections** | int[] | Tableau[search_depth] | Projections stock par jour pour articles SAFE | Base calcul objectifs valorisation | Données métier/prévisions | Valeurs réalistes | O(depth) parcours | Phase 3 |
| **stock_minimum** | int | MIN(projections) | Stock minimum projeté sur horizon | Borne inférieure objectif valorisation | MIN(stock_projections[0..depth]) | ≥ 0 | O(depth) calcul | Phase 3 |
| **stock_maximum** | int | MAX(projections) | Stock maximum projeté sur horizon | Borne supérieure objectif valorisation | MAX(stock_projections[0..depth]) | ≥ stock_minimum | O(depth) calcul | Phase 3 |
| **objectif_stock** | int | (min+max)/2 | Cible optimisation stock selon RG-004 | Point équilibre valorisation stock | (stock_minimum + stock_maximum) / 2 | Entre min et max | O(1) calcul | Phase 3 |
| **stock_final** | int | projections[depth] | Stock projeté fin période | État futur si aucune action | stock_projections[search_depth] | Valeur réaliste | O(1) accès | Phase 3 |
| **quantite_optimale** | int | objectif - stock_final | Quantité recommandée pour atteindre objectif | Besoin calculé pour valorisation | objectif_stock - stock_final | ≥ 0 si déficit | O(1) calcul | Phase 3 |
| **ecart_objectif** | int | |objectif - stock_final| | Distance à l'objectif valorisation | Évaluation intérêt article pour priorisation | ABS(stock_final - objectif_stock) | ≥ 0 | O(1) calcul | Phase 3 |
| **interet_valorisation** | double | 0.0 à score_max | Score intérêt article pour knapsack | Fonction utilité complexe avec bonus écart | Fonction(ecart, tendance, criticite_metier) | ≥ 0, plus haut = plus intéressant | O(1) calcul | Phase 3 |
| **taux_satisfaction** | double | 0.0 à 1.0 | % articles placés par niveau criticité | Métrique qualité résultat | articles_places / articles_totaux par grade | 0 ≤ val ≤ 1.0 | O(1) ratio | Phase 4 |
| **taux_occupation_moyen** | double | 0.0 à 1.0 | Occupation moyenne cartons créés | Métrique efficacité utilisation espace | Σ(carton.occupation) / nombre_cartons | > 0, idéalement > 0.85 | O(C) moyenne | Phase 4 |
| **temps_execution_ms** | long | 0 à timeout_max | Durée exécution algorithme complet | Métrique performance temps réel | System.currentTimeMillis() fin - début | < seuils définis | Temps réel | Phase 4 |
| **memoire_peak_bytes** | long | 0 à heap_max | Pic utilisation mémoire pendant traitement | Métrique performance mémoire | Runtime.getRuntime().totalMemory() | < limites système | Temps réel | Phase 4 |
| **validation_success** | boolean | true/false | État validation globale algorithme | Indicateur qualité résultat final | ET logique toutes validations | true requis | O(1) | Phase 4 |
| **quantites_partielles** | List\<ArticlePartiel\> | 0 à N enregistrements | Détail quantités URGENT_B non satisfaites | Traçabilité décisions RG-005 | Enregistrement lors acceptation partielle | Informatif audit | O(partielles) | Phase 4 |
| **warnings** | List\<String\> | 0 à N messages | Alertes non bloquantes (SAFE ignorés, etc.) | Informations qualité pour monitoring | Ajout lors détection anomalies | Informatif debug | O(warnings) | Phase 4 |

---

**Créé le :** $(date)
**Assigné à :** Équipe Backend
**Reviewer :** Tech Lead + Product Owner
**Sprint :** À définir selon roadmap produit

> ⚠️ **Point d'Attention Critique :** La RG-005 (quantités partielles URGENT_B) est un cas limite métier important qui impacte directement la satisfaction utilisateur. Une attention particulière doit être portée à sa validation.

> 📚 **Ces tableaux de référence constituent la documentation technique définitive pour l'implémentation. Ils doivent être consultés systématiquement lors du développement et des code reviews.**
