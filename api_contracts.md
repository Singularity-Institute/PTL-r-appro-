# Contrats d'Interface des APIs

## API 1 - Récupération Stock en Transit et Pending

### Description
Récupère le nombre d'exemplaires dans les APP pending et en Transit vers un technicien donné

### Endpoint
```
GET /api/v1/stock/on-delivery
```

### Paramètres d'entrée
| Paramètre | Type | Obligatoire | Description | Exemple |
|-----------|------|-------------|-------------|---------|
| `codeTechnicien` | string | Oui | Code identifiant du technicien | "TECH001" |
| `articleType` | string | Oui | Type d'article à rechercher | "CABLE_HDMI" |

### Réponse succès (200)
```json
{
  "codeTechnicien": "TECH001",
  "articleType": "CABLE_HDMI",
  "onDeliveryStock": 15,
  "details": {
    "pendingStock": 8,
    "transitStock": 7
  },
  "timestamp": "2025-09-22T10:30:00Z"
}
```

### Exemple de requête
```
GET /api/v1/stock/on-delivery?codeTechnicien=TECH001&articleType=CABLE_HDMI
```

---

## API 2 - Initialisation Stock Initial Technicien

### Description
Calcule le stock initial J0 d'un technicien en incluant les stocks en transit, pending et stock D

### Endpoint
```
GET /api/v1/stock/initial
```

### Paramètres d'entrée
| Paramètre | Type | Obligatoire | Description | Exemple |
|-----------|------|-------------|-------------|---------|
| `codeTechnicien` | string | Oui | Code identifiant du technicien | "TECH001" |
| `articleType` | string | Oui | Type d'article à rechercher | "CABLE_HDMI" |

### Réponse succès (200)
```json
{
  "codeTechnicien": "TECH001",
  "articleType": "CABLE_HDMI",
  "stockJ0": 25,
  "breakdown": {
    "onDeliveryStock": 15,
    "stockD": 10
  },
  "calculation": "stockJ0 = onDeliveryStock + stockD",
  "timestamp": "2025-09-22T10:30:00Z"
}
```

### Dépendances
- **API 1** : Utilise le résultat de l'API 1 pour obtenir `onDeliveryStock`
- **Stock D** : Ajoute le stock D existant

### Formule
```
Stock J0 = API1(onDeliveryStock) + Stock D
```

### Exemple de requête
```
GET /api/v1/stock/initial?codeTechnicien=TECH001&articleType=CABLE_HDMI
```

---

## API 3 - Liste Articles par Type d'Intervention

### Description
Retourne la liste des articles demandés avec leurs quantités pour un type d'intervention et un technicien

### Endpoint
```
GET /api/v1/intervention/articles
```

### Paramètres d'entrée
| Paramètre | Type | Obligatoire | Description | Valeurs possibles | Exemple |
|-----------|------|-------------|-------------|-------------------|---------|
| `interventionType` | string | Oui | Type d'intervention | EX, SAV, SS | "SAV" |
| `codeTechnicien` | string | Oui | Code identifiant du technicien | - | "TECH001" |

### Réponse succès (200)
```json
{
  "codeTechnicien": "TECH001",
  "interventionType": "SAV",
  "articles": [
    {
      "articleType": "CABLE_HDMI",
      "qteBrute": 5
    },
    {
      "articleType": "BOITIER_CPL",
      "qteBrute": 3
    },
    {
      "articleType": "MODEM_FIBRE",
      "qteBrute": 2
    }
  ],
  "totalArticles": 3,
  "timestamp": "2025-09-22T10:30:00Z"
}
```

### Exemple de requête
```
GET /api/v1/intervention/articles?interventionType=SAV&codeTechnicien=TECH001
```

---

## API 4 - Consommation Prévisionnelle par Distributeur

### Description
Calcule la consommation prévisionnelle d'un technicien pour un distributeur en appliquant le facteur d'imprévus

### Endpoint
```
GET /api/v1/consumption/forecast
```

### Paramètres d'entrée
| Paramètre | Type | Obligatoire | Description | Valeurs possibles | Exemple |
|-----------|------|-------------|-------------|-------------------|---------|
| `codeTechnicien` | string | Oui | Code identifiant du technicien | - | "TECH001" |
| `searchDepth` | integer | Oui | Profondeur de recherche en jours | ≥ 1 | 30 |
| `distributeur` | string | Oui | Code distributeur | ORG, GBH | "ORG" |
| `imprevFacteur` | number | Oui | Facteur multiplicateur pour les imprévus | ≥ 1.0 | 1.2 |

### Réponse succès (200)
```json
{
  "codeTechnicien": "TECH001",
  "distributeur": "ORG",
  "searchDepth": 30,
  "imprevFacteur": 1.2,
  "forecasts": [
    {
      "interventionType": "SAV",
      "articles": [
        {
          "articleType": "CABLE_HDMI",
          "qteBrute": 5,
          "qteBruteFinal1": 6.0
        },
        {
          "articleType": "BOITIER_CPL",
          "qteBrute": 3,
          "qteBruteFinal1": 3.6
        }
      ]
    },
    {
      "interventionType": "EX",
      "articles": [
        {
          "articleType": "MODEM_FIBRE",
          "qteBrute": 4,
          "qteBruteFinal1": 4.8
        }
      ]
    }
  ],
  "calculation": "qteBruteFinal1 = qteBrute * imprevFacteur",
  "timestamp": "2025-09-22T10:30:00Z"
}
```

### Dépendances
- **API 3** : Utilise l'API 3 pour obtenir les quantités brutes par type d'intervention

### Formule
```
QTE_Brute_Final_1 = QTE_Brute (API 3) × Imprev_Facteur
```

### Exemple de requête
```
GET /api/v1/consumption/forecast?codeTechnicien=TECH001&searchDepth=30&distributeur=ORG&imprevFacteur=1.2
```

---

## Architecture de Traitement Global

### Logique de traitement pour tous les distributeurs

```
Pour chaque Distributeur (ORG, GBH) {
    Pour chaque Technicien {
        Pour chaque Type_Intervention (SS, EX, SAV) {
            API4(technicien, searchDepth, distributeur, imprevFacteur)
            ↓
            API3(technicien, interventionType)
            ↓
            Calcul: qteBruteFinal1 = qteBrute × imprevFacteur
        }
    }
}
```

---

## Réponses d'Erreur Communes

### 400 - Bad Request
```json
{
  "error": "Paramètre manquant ou invalide",
  "code": "INVALID_PARAMETER"
}
```

### 404 - Not Found
```json
{
  "error": "Technicien non trouvé",
  "code": "TECHNICIAN_NOT_FOUND"
}
```

### 500 - Internal Server Error
```json
{
  "error": "Erreur interne du serveur",
  "code": "INTERNAL_SERVER_ERROR"
}
```

---

## Schémas de Données

### Types d'Intervention
- **EX** : Extension
- **SAV** : Service Après-Vente
- **SS** : Support Spécialisé

### Types de Distributeurs
- **ORG** : Orange
- **GBH** : Groupe BH

### Types d'Articles (exemples)
- `CABLE_HDMI`
- `BOITIER_CPL`
- `MODEM_FIBRE`
- `DECODEUR_TV`
- `ROUTEUR_WIFI`

---

## Relations entre APIs

1. **API 2** dépend de **API 1** + Stock D
2. **API 4** dépend de **API 3**
3. Le traitement global utilise **API 4** qui appelle **API 3** en cascade

### Flux de données
```
Stock D + API1 → API2 (Stock Initial)
API3 → API4 (Consommation Prévisionnelle)
```