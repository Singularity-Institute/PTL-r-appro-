# Conception Algorithmique : Knapsack Adaptatif Modulaire pour Optimisation de Colis

## Table des Matières
1. [Vue d'ensemble du Système](#vue-densemble-du-système)
2. [Architecture Modulaire](#architecture-modulaire)
3. [Modélisation des Données](#modélisation-des-données)
4. [Services et APIs](#services-et-apis)
5. [Algorithmes Principaux](#algorithmes-principaux)
6. [Diagrammes de Séquence](#diagrammes-de-séquence)
7. [Diagramme d'États](#diagramme-détats)
8. [Stratégies d'Extension](#stratégies-dextension)
9. [Cas d'Usage et Exemples](#cas-dusage-et-exemples)

## Vue d'ensemble du Système

### Problématique
Optimiser la composition de colis constitués de cartons en respectant :
- La hiérarchie de criticité des matériels (CRITIQUE_A > CRITIQUE_B > URGENT_A > URGENT_B > SAFE)
- Les contraintes de capacité par type d'article via coefficients d'occupation
- Les objectifs de stock projeté pour les articles SAFE

### Objectifs Principaux
1. **Garantie d'inclusion** : Tous les articles critiques (A/B) et urgents A doivent être inclus
2. **Optimisation progressive** : Utilisation optimale de l'espace disponible
3. **Flexibilité** : Architecture modulaire supportant l'extension
4. **Performance** : Algorithmes efficaces pour volumes importants

## Architecture Modulaire

### Patterns de Conception Utilisés

#### 1. Strategy Pattern
```java
public interface PackingStrategy {
    PackingResult execute(List<Article> articles, List<Carton> cartons, PackingContext context);
}

public class CriticalPackingStrategy implements PackingStrategy { ... }
public class UrgentBPackingStrategy implements PackingStrategy { ... }
public class SafeKnapsackStrategy implements PackingStrategy { ... }
public class SpecialUrgentBOnlyStrategy implements PackingStrategy { ... }
```

#### 2. Factory Pattern
```java
public interface CartonFactory {
    Carton createCarton(CartonType type, Map<ArticleType, Double> coefficientsOccupation);
}

public class StandardCartonFactory implements CartonFactory { ... }
public class CustomCartonFactory implements CartonFactory { ... }
```

#### 3. Chain of Responsibility Pattern
```java
public abstract class PackingPhaseHandler {
    protected PackingPhaseHandler nextHandler;
    public abstract PackingResult handlePhase(PackingContext context);
}

public class CriticalPhaseHandler extends PackingPhaseHandler { ... }
public class UrgentBPhaseHandler extends PackingPhaseHandler { ... }
public class SafePhaseHandler extends PackingPhaseHandler { ... }
```

## Modélisation des Données

### Classes Principales

```java
public class Article {
    private String id;
    private String nom;
    private ArticleType type;
    private GradeCriticite grade;
    private int quantiteDemandee;
    private double valeur;
    private Map<Integer, Integer> stockProjections; // Jour -> Stock projeté

    // Getters, setters, méthodes utilitaires
}

public class Carton {
    private String id;
    private CartonType type;
    private Map<ArticleType, Double> coefficientsOccupation;
    private Map<ArticleType, Integer> contenus;
    private double tauxOccupationCourant;
    private boolean isSealed;

    public boolean peutAjouter(Article article, int quantite);
    public double calculerTauxOccupationSiAjout(Article article, int quantite);
    public void ajouterArticle(Article article, int quantite);
}

public class Colis {
    private String id;
    private List<Carton> cartons;
    private double poidsTotal;
    private double volumeTotal;
    private Map<GradeCriticite, Integer> repartitionCriticite;

    public void ajouterCarton(Carton carton);
    public boolean respecteContraintes();
}

public enum GradeCriticite {
    CRITIQUE_A(1, "Critique Grade A"),
    CRITIQUE_B(2, "Critique Grade B"),
    URGENT_A(3, "Urgent Grade A"),
    URGENT_B(4, "Urgent Grade B"),
    SAFE(5, "Safe");

    private final int priorite;
    private final String libelle;
}

public enum ArticleType {
    TYPE_1("Type 1", 0.2),
    TYPE_2("Type 2", 0.25),
    TYPE_3("Type 3", 0.1);

    private final String nom;
    private final double coefficientOccupationDefaut;
}

public class PackingContext {
    private List<Article> articlesInput;
    private Map<ArticleType, Double> coefficientsOccupation;
    private int searchDepth;
    private CartonFactory cartonFactory;
    private List<PackingStrategy> strategies;
    private PackingConfiguration configuration;
}

public class PackingResult {
    private List<Colis> colisGeneres;
    private List<Carton> cartonsUtilises;
    private Map<String, Object> metriques;
    private List<String> alertes;
    private boolean succesComplet;
}
```

## Services et APIs

### 1. OccupationCalculatorService

```java
@Service
public interface OccupationCalculatorService {

    /**
     * Calcule le taux d'occupation global pour une liste d'articles
     */
    double calculerTauxOccupationGlobal(
        List<Article> articles,
        Map<ArticleType, Double> coefficients
    );

    /**
     * Détermine le nombre de cartons nécessaires
     */
    int calculerNombreCartonsNecessaires(
        List<Article> articles,
        Map<ArticleType, Double> coefficients
    );

    /**
     * Calcule le taux d'occupation restant après inclusion d'articles
     */
    double calculerTauxRestant(
        double tauxTotal,
        int nombreCartons
    );

    /**
     * Vérifie si un ajout est possible dans un carton
     */
    boolean peutAjouter(
        Carton carton,
        Article article,
        int quantite
    );
}

@Service
public class OccupationCalculatorServiceImpl implements OccupationCalculatorService {

    @Override
    public double calculerTauxOccupationGlobal(List<Article> articles, Map<ArticleType, Double> coefficients) {
        return articles.stream()
            .mapToDouble(article -> {
                double coeff = coefficients.get(article.getType());
                return article.getQuantiteDemandee() * coeff;
            })
            .sum();
    }

    @Override
    public int calculerNombreCartonsNecessaires(List<Article> articles, Map<ArticleType, Double> coefficients) {
        double tauxTotal = calculerTauxOccupationGlobal(articles, coefficients);
        return (int) Math.ceil(tauxTotal);
    }
}
```

### 2. ArticleClassificationService

```java
@Service
public interface ArticleClassificationService {

    /**
     * Filtre les articles par grade de criticité
     */
    List<Article> filtrerParGrade(
        List<Article> articles,
        List<GradeCriticite> grades
    );

    /**
     * Trie les articles par criticité puis par valeur
     */
    List<Article> trierParCriticiteEtValeur(List<Article> articles);

    /**
     * Identifie les articles pour valorisation stock
     */
    List<Article> identifierArticlesValorisationStock(
        List<Article> articles,
        int searchDepth
    );

    /**
     * Vérifie si la liste contient uniquement des URGENT_B
     */
    boolean estUniquementUrgentB(List<Article> articles);
}
```

### 3. StockProjectionService

```java
@Service
public interface StockProjectionService {

    /**
     * Calcule l'objectif de stock optimal pour un article SAFE
     */
    int calculerObjectifStockOptimal(
        Article article,
        int searchDepth
    );

    /**
     * Détermine la quantité à ajouter pour atteindre l'objectif
     */
    int calculerQuantiteOptimale(
        Article article,
        int objectifStock
    );

    /**
     * Évalue l'intérêt d'un article pour valorisation stock
     */
    double evaluerInteretValorisationStock(
        Article article,
        int searchDepth
    );
}

@Service
public class StockProjectionServiceImpl implements StockProjectionService {

    @Override
    public int calculerObjectifStockOptimal(Article article, int searchDepth) {
        Map<Integer, Integer> projections = article.getStockProjections();
        Integer stockMin = Collections.min(projections.values());
        Integer stockMax = Collections.max(projections.values());
        return (stockMin + stockMax) / 2;
    }
}
```

### 4. KnapsackOptimizationService

```java
@Service
public interface KnapsackOptimizationService {

    /**
     * Applique l'algorithme knapsack classique avec contraintes d'occupation
     */
    KnapsackResult optimiserAvecContraintes(
        List<Article> articles,
        List<Carton> cartons,
        OptimizationObjective objective
    );

    /**
     * Optimise pour objectif de valorisation stock
     */
    KnapsackResult optimiserPourValorisationStock(
        List<Article> articlesSafe,
        List<Carton> cartons,
        int searchDepth
    );

    /**
     * Algorithme knapsack multi-contraintes
     */
    KnapsackResult knapsackMultiContraintes(
        List<Article> items,
        Map<String, Double> contraintes,
        ObjectiveFunction fonction
    );
}

public class KnapsackResult {
    private List<Article> articlesSelectionnes;
    private double valeurTotale;
    private Map<String, Double> contraintesUtilisees;
    private double efficacite;
}
```

### 5. PackingStrategyService

```java
@Service
public interface PackingStrategyService {

    /**
     * Exécute la stratégie d'emballage appropriée
     */
    PackingResult executerStrategie(
        PackingContext context,
        PackingPhase phase
    );

    /**
     * Sélectionne la stratégie optimale selon le contexte
     */
    PackingStrategy selectionnerStrategie(
        List<Article> articles,
        PackingPhase phase
    );

    /**
     * Enregistre une nouvelle stratégie
     */
    void enregistrerStrategie(
        String nom,
        PackingStrategy strategie
    );
}

public enum PackingPhase {
    PHASE_CRITIQUE,     // Critiques A/B + Urgent A
    PHASE_URGENT_B,     // Urgent B en complément
    PHASE_SAFE,         // Articles Safe avec knapsack
    PHASE_URGENT_B_ONLY // Stratégie spéciale si uniquement Urgent B
}
```

## Algorithmes Principaux

### Algorithme Principal : PackingMasterAlgorithm

```java
public class PackingMasterAlgorithm {

    public PackingResult optimiserColis(PackingContext context) {
        // Phase 1: Initialisation et classification
        List<Article> articlesCritiques = classificationService
            .filtrerParGrade(context.getArticlesInput(),
                Arrays.asList(GradeCriticite.CRITIQUE_A, CRITIQUE_B, URGENT_A));

        List<Article> articlesUrgentB = classificationService
            .filtrerParGrade(context.getArticlesInput(),
                Arrays.asList(GradeCriticite.URGENT_B));

        List<Article> articlesSafe = classificationService
            .filtrerParGrade(context.getArticlesInput(),
                Arrays.asList(GradeCriticite.SAFE));

        PackingResult result = new PackingResult();

        // Phase 2: Traitement articles critiques (court-circuit knapsack)
        if (!articlesCritiques.isEmpty()) {
            result = traiterArticlesCritiques(articlesCritiques, context);
        }

        // Phase 3: Complétion avec Urgent B
        if (!articlesUrgentB.isEmpty() && result.hasSpaceAvailable()) {
            result = completerAvecUrgentB(articlesUrgentB, result, context);
        }

        // Phase 4: Optimisation avec articles Safe ou stratégie spéciale
        if (classificationService.estUniquementUrgentB(context.getArticlesInput())) {
            result = appliquerStrategieUrgentBSeul(articlesUrgentB, context);
        } else if (!articlesSafe.isEmpty() && result.hasSpaceAvailable()) {
            result = optimiserAvecArticlesSafe(articlesSafe, result, context);
        }

        // Phase 5: Finalisation et validation
        result = finaliserEtValider(result);

        return result;
    }
}
```

### Algorithme Phase Critique

```java
private PackingResult traiterArticlesCritiques(List<Article> articlesCritiques, PackingContext context) {
    // Calcul du nombre de cartons nécessaires
    double tauxOccupationTotal = occupationCalculatorService
        .calculerTauxOccupationGlobal(articlesCritiques, context.getCoefficientsOccupation());

    int nombreCartonsNecessaires = (int) Math.ceil(tauxOccupationTotal);

    // Création des cartons
    List<Carton> cartons = new ArrayList<>();
    for (int i = 0; i < nombreCartonsNecessaires; i++) {
        cartons.add(context.getCartonFactory().createCarton(
            CartonType.STANDARD,
            context.getCoefficientsOccupation()
        ));
    }

    // Distribution des articles critiques
    PackingResult result = distribuerArticlesCritiques(articlesCritiques, cartons);

    // Calcul de l'espace restant
    double tauxRestant = nombreCartonsNecessaires - tauxOccupationTotal;
    result.setTauxOccupationRestant(tauxRestant);

    return result;
}
```

### Algorithme Knapsack pour Articles Safe

```java
private PackingResult optimiserAvecArticlesSafe(List<Article> articlesSafe, PackingResult resultCourant, PackingContext context) {
    // Identification des articles candidats pour valorisation stock
    List<Article> candidatsValorisationStock = stockProjectionService
        .identifierArticlesValorisationStock(articlesSafe, context.getSearchDepth());

    if (candidatsValorisationStock.isEmpty()) {
        return resultCourant;
    }

    // Préparation fonction objectif personnalisée
    ObjectiveFunction objectiveFonction = (article) -> {
        int objectifStock = stockProjectionService
            .calculerObjectifStockOptimal(article, context.getSearchDepth());
        int stockActuel = article.getStockProjections().get(context.getSearchDepth());

        // Valeur = intérêt pour atteindre (min+max)/2
        double ecartOptimal = Math.abs(objectifStock - stockActuel);
        return article.getValeur() / (1 + ecartOptimal); // Plus l'écart est faible, plus la valeur est élevée
    };

    // Application knapsack multi-contraintes
    Map<String, Double> contraintes = calculerContraintesRestantes(resultCourant.getCartons());

    KnapsackResult knapsackResult = knapsackOptimizationService
        .knapsackMultiContraintes(candidatsValorisationStock, contraintes, objectiveFonction);

    // Intégration du résultat
    return integrerResultatKnapsack(resultCourant, knapsackResult);
}
```

### Algorithme Knapsack Multi-Contraintes

```java
public KnapsackResult knapsackMultiContraintes(List<Article> items, Map<String, Double> contraintes, ObjectiveFunction fonction) {
    int n = items.size();
    Map<String, Integer> contraintesDiscretes = discretiserContraintes(contraintes);

    // Table de programmation dynamique multi-dimensionnelle
    // dp[i][c1][c2][...] = valeur optimale avec les i premiers objets et capacités c1, c2, ...
    MultiDimensionalArray dp = new MultiDimensionalArray(n + 1, contraintesDiscretes);

    // Remplissage de la table DP
    for (int i = 1; i <= n; i++) {
        Article article = items.get(i - 1);
        double valeur = fonction.calculerValeur(article);

        dp.iterateAllStates((state) -> {
            // Option 1: Ne pas prendre l'article
            double valeurSansPrendre = dp.get(i - 1, state);

            // Option 2: Prendre l'article si possible
            double valeurAvecPrendre = 0;
            if (peutPrendreArticle(article, state, contraintes)) {
                Map<String, Integer> nouvelEtat = calculerNouvelEtat(article, state);
                valeurAvecPrendre = valeur + dp.get(i - 1, nouvelEtat);
            }

            dp.set(i, state, Math.max(valeurSansPrendre, valeurAvecPrendre));
        });
    }

    // Reconstruction de la solution
    return reconstruireSolution(dp, items, contraintes, fonction);
}
```

## Diagrammes de Séquence

### Diagramme de Séquence Principal

```mermaid
sequenceDiagram
    participant Client
    participant PMaster as PackingMasterAlgorithm
    participant ClassifS as ArticleClassificationService
    participant OccupS as OccupationCalculatorService
    participant StratS as PackingStrategyService
    participant KnapS as KnapsackOptimizationService
    participant StockS as StockProjectionService

    Client->>PMaster: optimiserColis(PackingContext)

    PMaster->>ClassifS: filtrerParGrade(articles, [CRITIQUE_A, CRITIQUE_B, URGENT_A])
    ClassifS-->>PMaster: articlesCritiques[]

    PMaster->>ClassifS: filtrerParGrade(articles, [URGENT_B])
    ClassifS-->>PMaster: articlesUrgentB[]

    PMaster->>ClassifS: filtrerParGrade(articles, [SAFE])
    ClassifS-->>PMaster: articlesSafe[]

    alt Articles critiques présents
        PMaster->>OccupS: calculerTauxOccupationGlobal(articlesCritiques, coefficients)
        OccupS-->>PMaster: tauxOccupationTotal

        PMaster->>OccupS: calculerNombreCartonsNecessaires(articlesCritiques, coefficients)
        OccupS-->>PMaster: nombreCartons

        PMaster->>PMaster: traiterArticlesCritiques()
        Note over PMaster: Création cartons + distribution articles critiques

        alt Urgent B disponibles ET espace restant
            PMaster->>StratS: executerStrategie(PHASE_URGENT_B)
            StratS->>OccupS: peutAjouter(carton, article, quantite)
            OccupS-->>StratS: boolean
            StratS-->>PMaster: PackingResult

            alt Articles Safe disponibles ET espace restant
                PMaster->>StockS: identifierArticlesValorisationStock(articlesSafe, searchDepth)
                StockS-->>PMaster: candidatsValorisationStock[]

                PMaster->>KnapS: knapsackMultiContraintes(candidats, contraintes, objectiveFunction)
                KnapS->>KnapS: programmationDynamiqueMultiDim()
                KnapS->>KnapS: reconstruireSolution()
                KnapS-->>PMaster: KnapsackResult

                PMaster->>PMaster: integrerResultatKnapsack()
            end
        end
    else Uniquement Urgent B
        PMaster->>StratS: executerStrategie(PHASE_URGENT_B_ONLY)
        StratS->>KnapS: optimiserAvecContraintes(articlesUrgentB, cartons, objective)
        KnapS-->>StratS: KnapsackResult
        StratS-->>PMaster: PackingResult
    end

    PMaster->>PMaster: finaliserEtValider()
    PMaster-->>Client: PackingResult
```

### Diagramme de Séquence Knapsack Multi-Contraintes

```mermaid
sequenceDiagram
    participant KnapS as KnapsackOptimizationService
    participant DP as MultiDimensionalArray
    participant ObjF as ObjectiveFunction
    participant Validator as ContraintValidator

    KnapS->>KnapS: discretiserContraintes(contraintes)
    KnapS->>DP: new MultiDimensionalArray(n+1, contraintesDiscretes)

    loop Pour chaque article i
        loop Pour chaque état possible
            KnapS->>DP: get(i-1, etat) // Option ne pas prendre
            Note over KnapS: valeurSansPrendre

            KnapS->>Validator: peutPrendreArticle(article, etat, contraintes)
            Validator-->>KnapS: boolean

            alt Article peut être pris
                KnapS->>ObjF: calculerValeur(article)
                ObjF-->>KnapS: valeurArticle

                KnapS->>KnapS: calculerNouvelEtat(article, etat)
                KnapS->>DP: get(i-1, nouvelEtat)
                Note over KnapS: valeurAvecPrendre = valeurArticle + valeurPrecedente

                KnapS->>DP: set(i, etat, max(valeurSansPrendre, valeurAvecPrendre))
            else
                KnapS->>DP: set(i, etat, valeurSansPrendre)
            end
        end
    end

    KnapS->>KnapS: reconstruireSolution(dp, articles, contraintes)
    Note over KnapS: Backtracking pour trouver articles sélectionnés
```

## Diagramme d'États

```mermaid
stateDiagram-v2
    [*] --> Initialisation

    Initialisation --> Classification : Context validé

    Classification --> EvaluationComposition : Articles classifiés

    EvaluationComposition --> PhaseUniquementUrgentB : Uniquement URGENT_B
    EvaluationComposition --> PhaseCritique : Articles critiques présents

    PhaseCritique --> CalculOccupation : Articles CRITIQUE_A/B + URGENT_A
    CalculOccupation --> CreationCartons : Taux calculé
    CreationCartons --> DistributionCritiques : Cartons créés
    DistributionCritiques --> EvaluationEspaceRestant : Articles critiques placés

    EvaluationEspaceRestant --> PhaseUrgentB : Espace disponible + URGENT_B
    EvaluationEspaceRestant --> PhaseSafe : Espace disponible + pas URGENT_B
    EvaluationEspaceRestant --> Finalisation : Pas d'espace disponible

    PhaseUrgentB --> CompletionCartons : Tentative complétion
    CompletionCartons --> VerificationCapacite : Pour chaque carton
    VerificationCapacite --> AjoutUrgentB : Capacité suffisante
    VerificationCapacite --> CartonSuivant : Capacité insuffisante
    AjoutUrgentB --> VerificationComplete : Article ajouté
    VerificationComplete --> CartonSuivant : Quantity restante > 0
    VerificationComplete --> UrgentBComplete : Quantity restante = 0
    CartonSuivant --> CreationNouveauCarton : Plus de cartons + quantité restante
    CartonSuivant --> CompletionCartons : Carton suivant disponible
    CreationNouveauCarton --> DistributionCritiques : Nouveau carton créé
    UrgentBComplete --> EvaluationArticlesSafe : URGENT_B traité

    EvaluationArticlesSafe --> PhaseSafe : Articles SAFE disponibles
    EvaluationArticlesSafe --> Finalisation : Pas d'articles SAFE

    PhaseSafe --> IdentificationCandidats : Articles SAFE à valoriser
    IdentificationCandidats --> CalculObjectifStock : Candidats identifiés
    CalculObjectifStock --> PreparationKnapsack : Objectifs calculés
    PreparationKnapsack --> ExecutionKnapsack : Fonction objectif préparée
    ExecutionKnapsack --> ProgrammationDynamique : Contraintes définies
    ProgrammationDynamique --> ReconstructionSolution : Table DP remplie
    ReconstructionSolution --> IntegrationResultat : Solution trouvée
    IntegrationResultat --> Finalisation : Résultat intégré

    PhaseUniquementUrgentB --> StrategieSpeciale : Stratégie dédiée
    StrategieSpeciale --> CreationCartonsUrgentB : Cartons pour URGENT_B
    CreationCartonsUrgentB --> OptimisationEspaceRestant : Cartons créés
    OptimisationEspaceRestant --> KnapsackClassique : Espace restant disponible
    KnapsackClassique --> Finalisation : Optimisation terminée

    Finalisation --> ValidationContraintes : Résultat consolidé
    ValidationContraintes --> ValidationCarton : Pour chaque carton
    ValidationCarton --> ValidationContraintes : Carton suivant
    ValidationCarton --> GenerationMetriques : Tous cartons validés
    GenerationMetriques --> [*] : PackingResult finalisé

    note right of PhaseCritique : Court-circuit knapsack pour articles critiques
    note right of ExecutionKnapsack : Knapsack multi-contraintes avec objectif (min+max)/2
    note right of StrategieSpeciale : Création + optimisation knapsack classique
```

## Stratégies d'Extension

### 1. Extension Types de Cartons

```java
public abstract class CartonTypeTemplate {
    protected Map<ArticleType, Double> coefficientsOccupationBase;

    public abstract boolean peutContenir(ArticleType type);
    public abstract double calculerCoefficientOccupation(ArticleType type);
    public abstract Map<String, Object> getContraintesSpecifiques();

    // Template method
    public final Carton creerCarton(String id) {
        Carton carton = new Carton(id, this.getCartonType());
        carton.setCoefficientsOccupation(this.getCoefficientsOccupation());
        carton.setContraintesSpecifiques(this.getContraintesSpecifiques());
        return carton;
    }
}

// Exemple d'extension
public class CartonRefrigereTemplate extends CartonTypeTemplate {
    @Override
    public boolean peutContenir(ArticleType type) {
        return type.isCompatibleAvecRefrigation();
    }

    @Override
    public double calculerCoefficientOccupation(ArticleType type) {
        // Coefficient réduit pour articles réfrigérés (plus d'espace nécessaire)
        return coefficientsOccupationBase.get(type) * 0.8;
    }
}
```

### 2. Extension Algorithmes d'Optimisation

```java
public interface OptimizationAlgorithm {
    String getName();
    boolean isApplicable(OptimizationContext context);
    OptimizationResult optimize(List<Article> articles, List<Carton> cartons, OptimizationContext context);
    Map<String, Object> getParameters();
    void setParameters(Map<String, Object> parameters);
}

@Component
public class OptimizationAlgorithmRegistry {
    private Map<String, OptimizationAlgorithm> algorithms = new ConcurrentHashMap<>();

    public void registerAlgorithm(String name, OptimizationAlgorithm algorithm) {
        algorithms.put(name, algorithm);
    }

    public OptimizationAlgorithm selectBestAlgorithm(OptimizationContext context) {
        return algorithms.values().stream()
            .filter(algo -> algo.isApplicable(context))
            .max(Comparator.comparing(algo -> evaluateAlgorithmFitness(algo, context)))
            .orElse(getDefaultAlgorithm());
    }
}

// Exemples d'algorithmes extensibles
public class GeneticKnapsackAlgorithm implements OptimizationAlgorithm {
    // Algorithme génétique pour cas complexes
}

public class SimulatedAnnealingAlgorithm implements OptimizationAlgorithm {
    // Recuit simulé pour optimisation fine
}

public class QuantumInspiredAlgorithm implements OptimizationAlgorithm {
    // Algorithme quantique pour exploration solution space
}
```

### 3. Extension Stratégies de Packing

```java
public abstract class PackingStrategy {
    protected String name;
    protected Map<String, Object> parameters;

    public abstract boolean isApplicable(PackingContext context);
    public abstract PackingResult execute(PackingContext context);
    public abstract double estimateEfficiency(PackingContext context);

    // Hooks pour extension
    protected void beforeExecution(PackingContext context) {}
    protected void afterExecution(PackingResult result) {}
    protected void onError(Exception e, PackingContext context) {}
}

// Exemple de nouvelle stratégie
public class HybridPackingStrategy extends PackingStrategy {

    @Override
    public boolean isApplicable(PackingContext context) {
        return context.getArticles().size() > 1000 &&
               context.hasComplexConstraints();
    }

    @Override
    public PackingResult execute(PackingContext context) {
        // Phase 1: Pre-processing avec clustering
        Map<String, List<Article>> clusters = clusteringService.clusterArticles(context.getArticles());

        // Phase 2: Optimisation par cluster
        List<PackingResult> resultatsParCluster = new ArrayList<>();
        for (Map.Entry<String, List<Article>> cluster : clusters.entrySet()) {
            PackingContext clusterContext = context.createSubContext(cluster.getValue());
            PackingResult resultatCluster = optimizeCluster(clusterContext);
            resultatsParCluster.add(resultatCluster);
        }

        // Phase 3: Consolidation globale
        return consolidationService.consolidateResults(resultatsParCluster);
    }
}
```

### 4. Plugin Architecture

```java
public interface PackingPlugin {
    String getPluginId();
    String getVersion();
    List<String> getDependencies();
    void initialize(PluginContext context);
    void shutdown();
    boolean isEnabled();
}

public abstract class PackingExtensionPlugin implements PackingPlugin {

    // Extension points
    public List<CartonTypeTemplate> provideCartonTypes() { return Collections.emptyList(); }
    public List<OptimizationAlgorithm> provideOptimizationAlgorithms() { return Collections.emptyList(); }
    public List<PackingStrategy> providePackingStrategies() { return Collections.emptyList(); }
    public List<ObjectiveFunction> provideObjectiveFunctions() { return Collections.emptyList(); }
    public List<ConstraintValidator> provideConstraintValidators() { return Collections.emptyList(); }
}

@Component
public class PluginManager {
    private final Map<String, PackingPlugin> loadedPlugins = new ConcurrentHashMap<>();
    private final ApplicationEventPublisher eventPublisher;

    public void loadPlugin(PackingPlugin plugin) {
        try {
            validatePlugin(plugin);
            plugin.initialize(createPluginContext());
            loadedPlugins.put(plugin.getPluginId(), plugin);
            eventPublisher.publishEvent(new PluginLoadedEvent(plugin));
        } catch (Exception e) {
            log.error("Failed to load plugin: {}", plugin.getPluginId(), e);
        }
    }

    public void unloadPlugin(String pluginId) {
        PackingPlugin plugin = loadedPlugins.remove(pluginId);
        if (plugin != null) {
            try {
                plugin.shutdown();
                eventPublisher.publishEvent(new PluginUnloadedEvent(plugin));
            } catch (Exception e) {
                log.error("Error during plugin shutdown: {}", pluginId, e);
            }
        }
    }
}
```

## Cas d'Usage et Exemples

### Exemple 1: Configuration Standard

```java
public void exempleConfigurationStandard() {
    // Configuration coefficients d'occupation
    Map<ArticleType, Double> coefficients = Map.of(
        ArticleType.TYPE_1, 0.2,
        ArticleType.TYPE_2, 0.25,
        ArticleType.TYPE_3, 0.1
    );

    // Articles d'exemple
    List<Article> articles = Arrays.asList(
        new Article("A1", ArticleType.TYPE_1, GradeCriticite.CRITIQUE_A, 10),
        new Article("A2", ArticleType.TYPE_2, GradeCriticite.CRITIQUE_B, 5),
        new Article("A3", ArticleType.TYPE_3, GradeCriticite.URGENT_A, 3),
        new Article("A4", ArticleType.TYPE_1, GradeCriticite.URGENT_B, 15),
        new Article("A5", ArticleType.TYPE_2, GradeCriticite.SAFE, 8)
    );

    // Context de packing
    PackingContext context = PackingContext.builder()
        .articles(articles)
        .coefficientsOccupation(coefficients)
        .searchDepth(10)
        .cartonFactory(new StandardCartonFactory())
        .build();

    // Exécution
    PackingResult result = packingMasterAlgorithm.optimiserColis(context);

    // Résultat attendu: 4 cartons (taux = 3.55 → arrondi sup = 4)
    // Carton 1-4: Articles critiques/urgents A
    // Espace restant: 0.45 → complétion avec urgent B puis safe
}
```

### Exemple 2: Cas Uniquement Urgent B

```java
public void exempleUniquementUrgentB() {
    List<Article> articlesUrgentBSeul = Arrays.asList(
        new Article("UB1", ArticleType.TYPE_1, GradeCriticite.URGENT_B, 20),
        new Article("UB2", ArticleType.TYPE_2, GradeCriticite.URGENT_B, 12),
        new Article("UB3", ArticleType.TYPE_3, GradeCriticite.URGENT_B, 8)
    );

    PackingContext context = PackingContext.builder()
        .articles(articlesUrgentBSeul)
        .coefficientsOccupation(coefficients)
        .searchDepth(10)
        .build();

    PackingResult result = packingMasterAlgorithm.optimiserColis(context);

    // Stratégie spéciale activée:
    // 1. Création cartons nécessaires pour urgent B
    // 2. Application knapsack classique sur espace restant
}
```

### Exemple 3: Extension avec Plugin

```java
public class AdvancedOptimizationPlugin extends PackingExtensionPlugin {

    @Override
    public String getPluginId() {
        return "advanced-optimization-v2";
    }

    @Override
    public List<OptimizationAlgorithm> provideOptimizationAlgorithms() {
        return Arrays.asList(
            new GeneticKnapsackAlgorithm(),
            new SimulatedAnnealingAlgorithm(),
            new ParticleSwarmOptimization()
        );
    }

    @Override
    public List<PackingStrategy> providePackingStrategies() {
        return Arrays.asList(
            new HybridPackingStrategy(),
            new AdaptivePackingStrategy(),
            new PredictivePackingStrategy()
        );
    }

    @Override
    public void initialize(PluginContext context) {
        // Configuration des algorithmes avancés
        context.getAlgorithmRegistry().registerAlgorithm("genetic", new GeneticKnapsackAlgorithm());
        context.getStrategyRegistry().registerStrategy("hybrid", new HybridPackingStrategy());
    }
}

// Utilisation du plugin
@PostConstruct
public void loadAdvancedFeatures() {
    pluginManager.loadPlugin(new AdvancedOptimizationPlugin());
}
```

## Métriques et Monitoring

### KPIs de Performance

```java
public class PackingMetrics {
    private double tauxRemplissageMoyen;
    private double efficaciteOptimisation;
    private int nombreCartonsUtilises;
    private int nombreCartonsOptimal;
    private Map<GradeCriticite, Integer> repartitionCriticite;
    private long tempsExecution;
    private double coutTotal;

    public double calculerIndicateurGlobal() {
        return (tauxRemplissageMoyen * 0.4) +
               (efficaciteOptimisation * 0.3) +
               (ratioOptimalite() * 0.3);
    }

    private double ratioOptimalite() {
        return (double) nombreCartonsOptimal / nombreCartonsUtilises;
    }
}

@Component
public class MetricsCollector {

    @EventListener
    public void onPackingCompleted(PackingCompletedEvent event) {
        PackingMetrics metrics = calculateMetrics(event.getResult());
        metricsService.record(metrics);

        if (metrics.getTauxRemplissageMoyen() < THRESHOLD_EFFICACITE) {
            alertService.sendAlert("Faible taux de remplissage détecté: " + metrics.getTauxRemplissageMoyen());
        }
    }
}
```

Cette conception modulaire et adaptative permet:
1. **Extensibilité** : Ajout facile de nouveaux types de cartons, algorithmes, stratégies
2. **Maintenabilité** : Architecture claire avec séparation des responsabilités
3. **Performance** : Algorithmes optimisés avec métriques de monitoring
4. **Flexibilité** : Configuration paramétrable selon les besoins métier
5. **Évolutivité** : Architecture plugin pour extensions futures