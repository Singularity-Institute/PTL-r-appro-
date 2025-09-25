# TT-DEV-001 - Implémentation Algorithme Knapsack Adaptatif Modulaire

## 📋 Contexte et Objectif

**Epic :** Optimisation Automatique des Colis
**Type :** Feature
**Priorité :** High
**Estimation :** 13 points

### Description
Implémenter un système d'optimisation de colis utilisant un algorithme de knapsack adaptatif et modulaire pour gérer différents niveaux de criticité des matériels (CRITIQUE_A/B, URGENT_A, URGENT_B, SAFE) avec des contraintes d'occupation par type d'article.

### Valeur Métier
- Garantir 100% des matériels critiques dans les colis
- Optimiser l'utilisation de l'espace selon la criticité
- Réduire le nombre de colis tout en respectant les priorités
- Améliorer la satisfaction client par une livraison prioritaire

---

## 🏗️ Architecture Technique

### Services à Implémenter
1. **ServiceCalculOccupation** - Calcul des taux d'occupation
2. **ServiceClassificationArticles** - Classification par criticité
3. **ServiceProjectionStock** - Calcul objectifs stock (min+max)/2
4. **ServiceOptimisationColis** - Orchestrateur principal
5. **ServiceValidationContraintes** - Validation finale

### Coefficients d'Occupation par Type
```
TYPE_1 = 0.20 (20%)
TYPE_2 = 0.25 (25%)
TYPE_3 = 0.10 (10%)
CAPACITE_CARTON = 1.0 (100%)
```

---

## 📝 Règles de Gestion

### RG-001 : Classification des Articles par Criticité
**Priorité :** CRITIQUE_A > CRITIQUE_B > URGENT_A > URGENT_B > SAFE
**Application :** Les articles critiques court-circuitent l'algorithme de knapsack

### RG-002 : Calcul d'Occupation des Cartons
**Formule :** `occupation_totale = Σ(quantité_article × coefficient_type)`
**Contrainte :** `occupation_totale ≤ 1.0`

### RG-003 : Stratégie Court-Circuit pour Articles Critiques
**Règle :** Utiliser `PLAFOND(occupation_requise)` cartons pour garantir 100% des critiques
**Application :** Bypass du knapsack pour CRITIQUE_A, CRITIQUE_B, URGENT_A

### RG-004 : Algorithme Knapsack pour Articles SAFE
**Objectif :** Maximiser la valorisation stock avec cible `(stock_min + stock_max) / 2`
**Contrainte :** Respecter les capacités restantes des cartons

### RG-005 : Gestion des Quantités Partielles URGENT_B
**Règle :** Si aucun carton avec capacité disponible, accepter quantité partielle
**Action :** Passer à la finalisation sans créer nouveau carton

### RG-006 : Validation des Contraintes Finales
**Contrôles :** Occupation ≤ 100%, articles critiques à 100%, cohérence données

---

## ✅ Critères d'Acceptation

### CA-001 : Traitement Articles Critiques (Court-Circuit)
```gherkin
Given une liste d'articles contenant des matériels CRITIQUE_A et CRITIQUE_B
  And des coefficients d'occupation TYPE_1=0.2, TYPE_2=0.25, TYPE_3=0.1
When j'exécute l'algorithme d'optimisation des colis
Then tous les articles CRITIQUE_A et CRITIQUE_B sont placés à 100%
  And le nombre de cartons créés = PLAFOND(occupation_totale_critiques)
  And aucun algorithme de knapsack n'est exécuté pour ces articles
```

### CA-002 : Traitement Articles URGENT_A (Court-Circuit)
```gherkin
Given une liste d'articles contenant 50 unités URGENT_A de TYPE_1 (coeff 0.2)
When j'exécute l'algorithme d'optimisation des colis
Then les 50 unités URGENT_A sont placées à 100%
  And occupation_requise = 50 × 0.2 = 10.0
  And nombre_cartons_créés = PLAFOND(10.0) = 10 cartons
  And le traitement utilise la stratégie court-circuit
```

### CA-003 : Optimisation Articles SAFE avec Knapsack
```gherkin
Given des articles SAFE avec projections de stock
  And stock_min = 10, stock_max = 50 pour un article
  And des cartons avec espace restant disponible
When j'exécute l'algorithme d'optimisation des colis
Then l'objectif de stock calculé = (10 + 50) / 2 = 30
  And l'algorithme de knapsack maximise la valorisation vers cet objectif
  And les articles SAFE sont placés selon la solution optimale
```

### CA-004 : Gestion Quantités Partielles URGENT_B
```gherkin
Given des articles URGENT_B à placer
  And tous les cartons existants ont une occupation ≥ 95%
  And aucun carton n'a la capacité pour la quantité totale
When j'exécute l'algorithme d'optimisation des colis
Then l'algorithme place la quantité partielle possible
  And aucun nouveau carton n'est créé
  And le processus passe directement à la finalisation
  And le résultat indique les quantités partielles acceptées
```

### CA-005 : Validation Contraintes d'Occupation
```gherkin
Given un carton avec occupation calculée
When j'applique la validation des contraintes
Then l'occupation totale ≤ 1.0 (100%)
  And occupation_carton = Σ(quantité_article × coefficient_type)
  And les débordements génèrent une erreur de validation
```

### CA-006 : Séquençage Multi-Phases
```gherkin
Given une liste mixte d'articles de tous types de criticité
When j'exécute l'algorithme d'optimisation des colis
Then Phase 1: Articles CRITIQUE_A/B/URGENT_A traités en court-circuit
  And Phase 2: Articles URGENT_B complètent les cartons existants
  And Phase 3: Articles SAFE optimisés par knapsack sur espace restant
  And Phase 4: Validation et génération du rapport final
```

### CA-007 : Gestion des Cas Limites
```gherkin
Given une liste d'articles avec quantités très importantes
  And coefficients d'occupation générant des besoins > 1000 cartons
When j'exécute l'algorithme d'optimisation des colis
Then l'algorithme gère les gros volumes sans dégradation de performance
  And le temps d'exécution reste < 10 secondes pour 10000 articles
  And la mémoire utilisée reste raisonnable (< 1GB)
```

### CA-008 : Cas Articles Uniquement SAFE
```gherkin
Given une liste d'articles contenant uniquement des matériels SAFE
When j'exécute l'algorithme d'optimisation des colis
Then la stratégie knapsack classique est appliquée directement
  And l'objectif = maximiser la valorisation stock (min+max)/2
  And le nombre de cartons est optimisé selon la solution knapsack
```

### CA-009 : Métriques et Rapports
```gherkin
Given l'exécution complète de l'algorithme d'optimisation
When je consulte le rapport final
Then j'obtiens le nombre total de cartons créés
  And le taux d'occupation moyen des cartons
  And le pourcentage de satisfaction par type de criticité
  And les quantités partielles acceptées (si applicable)
  And le temps d'exécution total
```

### CA-010 : Gestion d'Erreurs et Validation
```gherkin
Given des données d'entrée invalides (coefficients négatifs, quantités nulles)
When j'exécute l'algorithme d'optimisation des colis
Then l'algorithme génère des erreurs de validation appropriées
  And aucun traitement n'est effectué sur des données corrompues
  And les messages d'erreur sont explicites et exploitables
```

---

## 🔧 Spécifications Techniques

### Interfaces Principales

#### ServiceOptimisationColis
```java
public interface ServiceOptimisationColis {
    PackingResult optimiserColis(OptimisationContext context);
}

public class OptimisationContext {
    List<Article> articles_input;
    CartonConstraints contraintes_carton;
    Map<ArticleType, Double> coefficients_occupation;
    int search_depth; // Pour projections stock
}

public class PackingResult {
    List<Carton> cartons_finaux;
    MetriquesOptimisation metriques;
    List<ArticlePartiel> quantites_partielles;
    boolean validation_success;
}
```

#### ServiceCalculOccupation
```java
public interface ServiceCalculOccupation {
    double calculerOccupationRequise(List<Article> articles);
    int calculerNombreCartonsNecessaires(double occupation_totale);
    boolean verifierCapaciteCarton(Carton carton, Article article);
}
```

### Algorithmes Clés

#### Algorithme Principal (Pseudocode)
```
ALGORITHME OptimiserColis(context)
DEBUT
    articles_critiques ← FiltrerParGrade(context.articles, [CRITIQUE_A, CRITIQUE_B, URGENT_A])
    articles_urgent_b ← FiltrerParGrade(context.articles, [URGENT_B])
    articles_safe ← FiltrerParGrade(context.articles, [SAFE])

    // Phase 1: Court-circuit pour critiques
    cartons_resultats ← TraiterArticlesCritiques(articles_critiques)

    // Phase 2: Complétion URGENT_B
    SI articles_urgent_b NON VIDE ALORS
        cartons_resultats ← CompleterAvecUrgentB(cartons_resultats, articles_urgent_b)
    FIN_SI

    // Phase 3: Optimisation SAFE par knapsack
    SI articles_safe NON VIDE ALORS
        cartons_resultats ← OptimiserAvecSafe(cartons_resultats, articles_safe)
    FIN_SI

    // Phase 4: Validation finale
    RETOURNER ValiderEtGenererRapport(cartons_resultats)
FIN
```

#### Algorithme Knapsack Multi-Contraintes (Pseudocode)
```
ALGORITHME KnapsackMultiContraintes(articles_safe, cartons_disponibles)
DEBUT
    POUR CHAQUE carton DANS cartons_disponibles FAIRE
        capacite_restante ← 1.0 - carton.occupation_actuelle
        articles_candidats ← FiltrerParCapacite(articles_safe, capacite_restante)

        // Table de programmation dynamique
        dp ← InitialiserTableDP(articles_candidats.size, capacite_restante)

        POUR i DE 1 A articles_candidats.size FAIRE
            POUR w DE 0 A capacite_restante FAIRE
                article ← articles_candidats[i-1]
                cout_occupation ← article.quantite × article.coefficient
                valeur_stock ← CalculerValeurValorisationStock(article)

                SI cout_occupation <= w ALORS
                    dp[i][w] ← MAX(dp[i-1][w], dp[i-1][w-cout_occupation] + valeur_stock)
                SINON
                    dp[i][w] ← dp[i-1][w]
                FIN_SI
            FIN_POUR
        FIN_POUR

        solution ← ReconstruireSolution(dp, articles_candidats)
        AppliquerSolution(carton, solution)
    FIN_POUR

    RETOURNER cartons_disponibles
FIN
```

---

## 🧪 Tests Unitaires Requis

### Tests ServiceCalculOccupation
- **testCalculOccupationSimple()** : 10 TYPE_1 → 2.0
- **testCalculOccupationMixte()** : 5 TYPE_1 + 2 TYPE_2 → 1.5
- **testNombreCartonsNecessaires()** : occupation 2.3 → 3 cartons
- **testVerificationCapacite()** : carton 80% + article 0.3 → false

### Tests ServiceClassificationArticles
- **testFiltrageCritiques()** : CRITIQUE_A/B + URGENT_A extraits
- **testFiltrageUrgentB()** : URGENT_B uniquement
- **testFiltrageSafe()** : SAFE uniquement
- **testTriParPriorite()** : ordre respecté

### Tests ServiceProjectionStock
- **testCalculObjectifStock()** : (min=10, max=50) → 30
- **testCalculQuantiteOptimale()** : objectif=30, stock_final=20 → 10
- **testEvaluationInteretValorisationStock()** : scoring cohérent

### Tests Algorithme Principal
- **testCasUniqueementCritiques()** : court-circuit only
- **testCasUniquementSafe()** : knapsack only
- **testCasMixteComplet()** : toutes les phases
- **testGestionQuantitesPartielles()** : URGENT_B partiels
- **testValidationContraintes()** : occupation ≤ 100%

---

## 📊 Métriques de Performance

### Indicateurs Clés
- **Temps d'exécution** : < 5 secondes pour 1000 articles
- **Utilisation mémoire** : < 512MB en pic
- **Taux d'occupation moyen** : > 85%
- **Satisfaction articles critiques** : 100%
- **Satisfaction articles SAFE** : > 70%

### Benchmarks
- **Volume Standard** : 100-500 articles → < 1 seconde
- **Gros Volume** : 1000-5000 articles → < 10 secondes
- **Volume Extrême** : 10000+ articles → < 30 secondes

---

## 🚀 Définition of Done

- [ ] Tous les services implémentés selon les interfaces
- [ ] Tous les critères d'acceptation validés par les tests
- [ ] Tests unitaires > 90% de couverture
- [ ] Tests d'intégration sur les cas métier complexes
- [ ] Performance validée sur les benchmarks définis
- [ ] Documentation technique complète
- [ ] Code review approuvé par le tech lead
- [ ] Déploiement en environnement de test validé

---

## 📚 Ressources et Références

- **Algorithme Knapsack** : Programmation dynamique multi-contraintes
- **Design Patterns** : Strategy, Factory, Chain of Responsibility
- **Documentation Architecture** : `/docs/architecture-knapsack-modulaire.md`
- **Spécifications Métier** : Conception-Knapsack-Adaptatif-Modulaire.md

---

**Créé le :** $(date)
**Assigné à :** Équipe Backend
**Reviewer :** Tech Lead + Product Owner
**Sprint :** À définir selon roadmap produit