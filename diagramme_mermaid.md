# Diagramme Mermaid - Cycle de vie du système de gestion

```mermaid
stateDiagram-v2
    direction TB

    %% États principaux du cycle de vie
    [*] --> Initialisation : Démarrage système

    state Initialisation {
        direction LR
        [*] --> CROW
        [*] --> MEHARI
        [*] --> SIM_ORANGE
        CROW --> PROTECTLINE : Import données
        MEHARI --> PROTECTLINE : Import données
        SIM_ORANGE --> PROTECTLINE : Import données
    }

    Initialisation --> Distribution : Données chargées

    state Distribution {
        direction TB
        PROTECTLINE --> ESAT_MOULINS : APP
        PROTECTLINE --> ESAT_MONTLUCON : APP
        ESAT_MOULINS --> ESAT_MONTLUCON : APP
        ESAT_MONTLUCON --> ESAT_MOULINS : APP
    }

    Distribution --> Preparation : Données distribuées

    state Preparation {
        direction TB
        ESAT_MOULINS --> Point_Relais_Nord : APP
        ESAT_MONTLUCON --> Point_Relais_Sud : APP

        state Point_Relais_Nord {
            direction LR
            [*] --> TECHNICIENS_OKS
            TECHNICIENS_OKS --> Dispo_Nord
            TECHNICIENS_OKS --> A_Retour_Nord
        }

        state Point_Relais_Sud {
            direction LR
            [*] --> TECHNICIENS_PTL_ELI
            TECHNICIENS_PTL_ELI --> Dispo_Sud
            TECHNICIENS_PTL_ELI --> A_Retour_Sud
        }
    }

    Preparation --> Service_Client : Techniciens prêts

    state Service_Client {
        direction TB
        ESAT_MONTLUCON --> CLIENT : Services (SVL/EX)
        CLIENT --> ESAT_MOULINS : Demandes (SVL/DMP/DMT)

        %% Interventions
        TECHNICIENS_OKS --> CLIENT : INTER
        CLIENT --> TECHNICIENS_OKS : INTER
        CLIENT --> TECHNICIENS_PTL_ELI : INTER
        TECHNICIENS_PTL_ELI --> CLIENT : INTER
    }

    Service_Client --> Retour_Equipement : Services terminés

    state Retour_Equipement {
        direction TB
        TECHNICIENS_OKS --> ESAT_MOULINS : Retours (REN/REO)
        TECHNICIENS_PTL_ELI --> ESAT_MOULINS : Retours (REN/REO)
    }

    state Gestion_Urgences {
        PROTECTLINE --> TECHNICIENS_PTL_ELI : URGENCES
    }

    Retour_Equipement --> Distribution : Cycle suivant
    Service_Client --> Gestion_Urgences : Si urgence
    Gestion_Urgences --> Service_Client : Urgence traitée

    %% Fin du cycle
    Retour_Equipement --> [*] : Fin de journée
```
