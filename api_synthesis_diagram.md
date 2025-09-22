# Synthèse des APIs - Diagramme

## Vue d'ensemble des APIs

```mermaid
graph TD
    %% API 1
    A1[API 1: Récupération Stock en Transit]
    A1_IN[Input:<br/>• CodeTechnicien<br/>• Article_Type]
    A1_OUT[Output:<br/>OnDeliveryStock]

    %% API 2
    A2[API 2: Initialisation Stock Initial]
    A2_IN[Input:<br/>• CodeTechnicien<br/>• Article_Type]
    A2_OUT[Output:<br/>Stock J0]
    STOCK_D[Stock "D"]

    %% API 3
    A3[API 3: Liste Articles par Intervention]
    A3_IN[Input:<br/>• Intervention_Type<br/>• CodeTechnicien]
    A3_OUT[Output:<br/>Article_Type : QTE_Brute]

    %% API 4
    A4[API 4: Consommation Prévisionnelle]
    A4_IN[Input:<br/>• CodeTechnicien<br/>• Search_depth<br/>• Distributeur<br/>• Imprev_Facteur]
    A4_OUT[Output:<br/>Article_Type : QTE_Brute_Final_1]

    %% Relations
    A1_IN --> A1
    A1 --> A1_OUT

    A2_IN --> A2
    A2 --> A2_OUT
    A1_OUT --> A2
    STOCK_D --> A2

    A3_IN --> A3
    A3 --> A3_OUT

    A4_IN --> A4
    A4 --> A4_OUT
    A3_OUT --> A4

    %% Styling
    classDef apiBox fill:#e1f5fe,stroke:#0277bd,stroke-width:2px
    classDef inputBox fill:#f3e5f5,stroke:#7b1fa2,stroke-width:1px
    classDef outputBox fill:#e8f5e8,stroke:#388e3c,stroke-width:1px
    classDef dataBox fill:#fff3e0,stroke:#f57c00,stroke-width:1px

    class A1,A2,A3,A4 apiBox
    class A1_IN,A2_IN,A3_IN,A4_IN inputBox
    class A1_OUT,A2_OUT,A3_OUT,A4_OUT outputBox
    class STOCK_D dataBox
```

## Flux d'appels entre APIs

```mermaid
sequenceDiagram
    participant USER as Utilisateur
    participant API2 as API 2
    participant API1 as API 1
    participant STOCKD as Stock "D"
    participant API4 as API 4
    participant API3 as API 3

    Note over USER: Initialisation Stock Initial
    USER->>API2: CodeTechnicien, Article_Type
    API2->>API1: CodeTechnicien, Article_Type
    API1-->>API2: OnDeliveryStock
    API2->>STOCKD: Récupération Stock "D"
    STOCKD-->>API2: Stock "D"
    API2-->>USER: Stock J0 = API1 + Stock "D"

    Note over USER: Consommation Prévisionnelle
    USER->>API4: CodeTechnicien, Search_depth, Distributeur, Imprev_Facteur
    API4->>API3: Intervention_Type, CodeTechnicien
    API3-->>API4: Article_Type : QTE_Brute
    API4-->>USER: Article_Type : QTE_Brute_Final_1 = API3 × Imprev_Facteur
```

## Architecture de traitement par Distributeur

```mermaid
flowchart TD
    START([Début du traitement])

    DIST_LOOP{Pour chaque Distributeur<br/>ORG, GBH}
    TECH_LOOP{Pour chaque Technicien}
    INTER_LOOP{Pour chaque Type_Intervention<br/>SS, EX, SAV}

    API4_CALL[Appel API4<br/>CodeTechnicien, Search_depth<br/>Distributeur, Imprev_Facteur]

    API3_CALL[Appel API3<br/>CodeTechnicien, Intervention_Type]

    CALC[Calcul:<br/>QTE_Brute_Final_1 = <br/>QTE_Brute × Imprev_Facteur]

    COLLECT[Collecte des résultats]

    END([Fin du traitement])

    START --> DIST_LOOP
    DIST_LOOP --> TECH_LOOP
    TECH_LOOP --> INTER_LOOP
    INTER_LOOP --> API4_CALL
    API4_CALL --> API3_CALL
    API3_CALL --> CALC
    CALC --> COLLECT

    COLLECT --> INTER_LOOP
    INTER_LOOP -->|Tous les types traités| TECH_LOOP
    TECH_LOOP -->|Tous les techniciens traités| DIST_LOOP
    DIST_LOOP -->|Tous les distributeurs traités| END

    %% Styling
    classDef loopBox fill:#fff3e0,stroke:#f57c00,stroke-width:2px
    classDef apiBox fill:#e1f5fe,stroke:#0277bd,stroke-width:2px
    classDef calcBox fill:#e8f5e8,stroke:#388e3c,stroke-width:2px
    classDef startEndBox fill:#fce4ec,stroke:#c2185b,stroke-width:2px

    class DIST_LOOP,TECH_LOOP,INTER_LOOP loopBox
    class API4_CALL,API3_CALL apiBox
    class CALC,COLLECT calcBox
    class START,END startEndBox
```

## Détail des APIs

| API | Fonction | Dépendances | Formule |
|-----|----------|-------------|---------|
| **API 1** | Récupération stock en transit et pending | - | `OnDeliveryStock = Stock_APP_Pending + Stock_Transit` |
| **API 2** | Stock initial technicien | API 1 + Stock "D" | `Stock_J0 = API1_Result + Stock_D` |
| **API 3** | Articles demandés par intervention | - | `Liste: Article_Type : QTE_Brute` |
| **API 4** | Consommation prévisionnelle | API 3 | `QTE_Brute_Final_1 = API3_Result × Imprev_Facteur` |