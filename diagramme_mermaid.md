# Diagramme Mermaid - Cycle de vie du système de gestion

```mermaid
stateDiagram-v2
    [*] --> Initialisation

    state Initialisation {
        CROW --> PROTECTLINE
        MEHARI --> PROTECTLINE
        SIM_ORANGE --> PROTECTLINE
    }

    Initialisation --> Distribution

    state Distribution {
        PROTECTLINE --> ESAT_MOULINS
        PROTECTLINE --> ESAT_MONTLUCON
        ESAT_MOULINS --> ESAT_MONTLUCON
        ESAT_MONTLUCON --> ESAT_MOULINS
    }

    Distribution --> Preparation

    state Preparation {
        ESAT_MOULINS --> Point_Relais_Nord
        ESAT_MONTLUCON --> Point_Relais_Sud
        Point_Relais_Nord --> TECHNICIENS_OKS
        Point_Relais_Sud --> TECHNICIENS_PTL_ELI
        TECHNICIENS_OKS --> Dispo_Nord
        TECHNICIENS_OKS --> A_Retour_Nord
        TECHNICIENS_PTL_ELI --> Dispo_Sud
        TECHNICIENS_PTL_ELI --> A_Retour_Sud
    }

    Preparation --> Service_Client

    state Service_Client {
        ESAT_MONTLUCON --> CLIENT : Services_SVL_EX
        CLIENT --> ESAT_MOULINS : Demandes_SVL_DMP_DMT
        TECHNICIENS_OKS --> CLIENT : INTER
        CLIENT --> TECHNICIENS_OKS : INTER
        CLIENT --> TECHNICIENS_PTL_ELI : INTER
        TECHNICIENS_PTL_ELI --> CLIENT : INTER
    }

    Service_Client --> Retour_Equipement

    state Retour_Equipement {
        TECHNICIENS_OKS --> ESAT_MOULINS : Retours_REN_REO
        TECHNICIENS_PTL_ELI --> ESAT_MOULINS : Retours_REN_REO
    }

    state Gestion_Urgences {
        PROTECTLINE --> TECHNICIENS_PTL_ELI : URGENCES
    }

    Service_Client --> Gestion_Urgences
    Gestion_Urgences --> Service_Client
    Retour_Equipement --> Distribution
    Retour_Equipement --> [*]

    note right of Initialisation : Import des fichiers Excel
    note right of Distribution : Synchronisation des données
    note right of Service_Client : Phase opérationnelle
    note right of Retour_Equipement : Fin de cycle
```
