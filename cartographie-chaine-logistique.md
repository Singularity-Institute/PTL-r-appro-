# Cartographie de la Chaîne Logistique

## Vue d'ensemble du processus logistique

```mermaid
flowchart TD
    A[Constructeurs<br/>- CEOU<br/>- PESANI] --> B[Point Excel]
    B --> C[PTL]
    C --> D[E-SMT<br/>MONTLUCON]
    D --> E[Réception<br/>Référentiel]
    E --> F[Clients]
    F --> G[Expéditions]

    %% Annotations
    B -.-> H[ATF<br/>ATF]
    D -.-> I[ATF<br/>ATF]
    E -.-> J[Éclatement<br/>EAD]

    %% Gestion système
    K[Gestion<br/>- USER_GDOS<br/>- STATUT DOCUMENT/STOCKS<br/>- STATUT C] --> D

    %% Points de contrôle
    L[Stock Technique/AMO/Distribution] --> E
    M[API Métier du réceptionné<br/>Tech chez client] --> F

    classDef constructeur fill:#f9f9f9,stroke:#333,stroke-width:2px
    classDef processus fill:#e1f5fe,stroke:#0277bd,stroke-width:2px
    classDef client fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px

    class A constructeur
    class B,C,D,E processus
    class F,G client
```

## Flux détaillé des données

```mermaid
graph LR
    subgraph "Phase 1: Réception"
        A1[Constructeurs] --> A2[Point Excel]
        A2 --> A3[PTL]
    end

    subgraph "Phase 2: Traitement"
        A3 --> B1[E-SMT MONTLUCON]
        B1 --> B2[Réception]
        B2 --> B3[Référentiel]
    end

    subgraph "Phase 3: Distribution"
        B3 --> C1[Clients]
        C1 --> C2[Expéditions]
    end

    subgraph "Gestion & Support"
        D1[Gestion Système]
        D2[Stock Technique]
        D3[API Métier]
    end

    D1 --> B1
    D2 --> B2
    D3 --> C1
```

## Processus de gestion par statut

```mermaid
stateDiagram-v2
    [*] --> Reception
    Reception --> Validation_Stock
    Validation_Stock --> Distribution
    Distribution --> Expedition
    Expedition --> Livraison_Client
    Livraison_Client --> [*]

    Reception --> Anomalie_Detection : Erreur
    Anomalie_Detection --> Reception : Correction

    Validation_Stock --> Stock_Insuffisant : Manque
    Stock_Insuffisant --> Approvisionnement
    Approvisionnement --> Validation_Stock
```

## Types de flux identifiés

| Type | Description | Responsable |
|------|-------------|------------|
| 🔴 **Flux Excel** | Données constructeurs | CEOU, PESANI |
| ⚠️ **Flux Technique** | Traitement E-SMT | MONTLUCON |
| 🔵 **Flux Réception** | Gestion stocks/référentiel | Système interne |
| ⭕ **Flux Client** | Distribution finale | Clients |
| 🟡 **Flux Retour** | Retours clients | Service client |

## Points de contrôle critiques

```mermaid
mindmap
  root((Contrôles))
    Réception
      Stock technique
      Validation données
      Référentiel
    Distribution
      API Métier
      Éclatement EAD
      Expéditions
    Qualité
      Anomalie détection
      Retours clients
      Suivi performance
```

## Annotations techniques

- **ATF (ATF)** : Points de validation automatique
- **EAD** : Éclatement automatique des données
- **API Métier** : Interface technique client
- **USER_GDOS** : Gestion documentaire et stocks
- **STATUT C** : Contrôle statut commandes

## Actions recommandées

1. **Amélioration** : Automatisation du flux Excel → PTL
2. **Monitoring** : Mise en place d'alertes sur les points critiques
3. **Optimisation** : Réduction des étapes manuelles
4. **Traçabilité** : Renforcement du suivi bout en bout

---
*Cartographie générée à partir de l'analyse des processus logistiques - Version 1.0*