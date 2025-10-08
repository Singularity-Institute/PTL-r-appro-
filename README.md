# Analyse des incohérences - Équipements CRO

**Date d'analyse**: 2025-10-08
**Fichier source**: `_select_ie_ARTICLE_ID_ie_location_ie_EQUIPMENT_SERIAL_NUMBER_ie__202510081054.csv`

---

## Contexte

**Problématique**: Équipements du constructeur CRO installés (statut PI/NI) mais marqués disponibles (statut D) dans la base logistique.

**Requête SQL**:
```sql
SELECT ie.ARTICLE_ID, ie.location, ie.EQUIPMENT_SERIAL_NUMBER,
       ie.EQUIPMENT_INS_DATE, ie.EQUIPMENT_MODIF_DATE, ie.barcode,
       s.user_code, s.status, s.state, s.contract_id, s.last_update
FROM teamtool_tech_mm.intervention_equipment ie
JOIN teamtool_logistique.stock s ON s.serial_number = ie.EQUIPMENT_SERIAL_NUMBER
JOIN teamtool_tech_mm.intervention i ON i.INTERVENTION_ID = ie.INTERVENTION_ID
WHERE i.CODE_intervention_EVENT_STATUS = 'TER'
  AND s.manufacturer = 'CRO'
  AND ie.MAT_CETAT IN ('PI', 'NI')
  AND s.status = "D"
```

**Légende**:
- PI/NI = Installé (dans intervention_equipment)
- D = Disponible (dans stock logistique)
- CRO = Constructeur

---

## Statistiques générales

| Métrique | Valeur |
|----------|--------|
| **Total d'incohérences** | **322 équipements** |
| Équipements uniques | 322 serial numbers |
| Période d'installation | 2021-05 à 2024-08 |
| Articles distincts | 15 types |
| Techniciens distincts (user_code) | ~40 users |
| Contract_id vides | **322 (100%)** ⚠️ |

---

## Pattern MAJEUR identifié: Dates dans le futur

### Analyse temporelle

| Critère | Nombre | Pourcentage |
|---------|--------|-------------|
| last_update APRÈS installation | 309 | **95%** |
| last_update AVANT installation | 12 | 3% |
| last_update = date installation | 1 | <1% |
| **last_update dans le futur (≥2025)** | **211** | **65%** ⚠️ |
| EQUIPMENT_MODIF_DATE dans le futur | 21 | 6% |

### Distribution par état (state)

| État | Nombre | % Total | Dates futures | % Dates futures |
|------|--------|---------|---------------|-----------------|
| **OO** | 251 | 78% | 195 | **77%** ⚠️ |
| **ON** | 67 | 21% | 13 | 19% |
| C | 3 | 1% | - | - |
| HN | 1 | <1% | - | - |

**Observation critique**: Les équipements en état "OO" ont massivement des dates futures (77%), ce qui suggère un problème de calcul de date lié à cet état spécifique.

---

## Distribution par client

### Vue d'ensemble

| Client | Nombre | % Total | État OO | % OO | État ON | % ON |
|--------|--------|---------|---------|------|---------|------|
| **ORANGE** | 206 | 63% | 181 | **87%** ⚠️ | 25 | 12% |
| **GROUPAMA** | 114 | 35% | 70 | 61% | 40 | 35% |
| Non identifié | 2 | <1% | - | - | - | - |

**Constat**: ORANGE est disproportionnellement affecté avec 87% de ses équipements en état OO.

---

## Top 5 techniciens (user_code) concernés

| Rang | User_code | Nombre d'équipements | % du total |
|------|-----------|----------------------|------------|
| 1 | ELI.008 | 11 | 3.4% |
| 2 | FBRET1 | 9 | 2.8% |
| 3 | T0024 | 8 | 2.5% |
| 4 | ELI.009 | 8 | 2.5% |
| 5 | T9987 | 7 | 2.2% |

**Autres user_code notables**: OKS.475 (7), ELI.021 (7), OKS.769 (6), ELI.067 (6), ELI.060 (6), tsbv2.crc.01 (5)

---

## Top 5 articles (ARTICLE_ID) concernés

| Rang | Article ID | Description (from barcode) | Nombre | % du total |
|------|------------|----------------------------|--------|------------|
| 1 | 6331 | ORG-PIR-ISM | 52 | 16.1% |
| 2 | 6335 | ORG-CONTROL PANEL 4G | 51 | 15.8% |
| 3 | 6337 | ORG-MAG-SHOCK-ISM | 43 | 13.4% |
| 4 | 6319 | GRP-MAG-SHOCK-ISM | 32 | 9.9% |
| 5 | 6339 | ORG-RMTW-ISM | 21 | 6.5% |

**Autres**: 6324 (21), 6330 (19), 6332 (13), 6329 (12), 6325 (12)

**Observation**: Les articles ORANGE (ORG-*) dominent largement le top 3.

---

## Exemples concrets d'incohérences

### Exemple 1: Date future extrême
- **Serial**: 3527780
- **Installé**: 2021-05-17
- **last_update**: **2025-09-15** (4 ans dans le futur!)
- **État**: OO
- **User**: ELI.004

### Exemple 2: Mise à jour avant installation
- **Serial**: 3532720
- **Installé**: 2022-03-03
- **last_update**: 2021-11-16 (4 mois AVANT installation)
- **État**: ON
- **User**: tsbv2.crc.01

### Exemple 3: Installation récente mais incohérence
- **Serial**: 12035419
- **Installé**: 2024-08-09
- **last_update**: 2024-08-06 (3 jours avant installation)
- **État**: ON
- **User**: tsbv2.crc.01

---

## Sources du problème identifiées

### 🔴 1. Dates de stock dans le futur (CRITIQUE)

**Constat**: 65% des équipements ont last_update ≥ 2025, avec une forte corrélation sur l'état OO (77%).

**Hypothèses**:
- Le champ `last_update` ne contient pas la date de mise à jour du stock mais une autre date (garantie, maintenance planifiée, date de retour prévu, etc.)
- Bug dans le calcul de date pour l'état "OO"
- Problème de format de date ou de timezone lors de la synchronisation
- Saisie manuelle incorrecte

**Impact**: Impossible de déterminer la dernière vraie mise à jour du statut stock.

### 🔴 2. Absence totale de contract_id (CRITIQUE)

**Constat**: 100% des équipements n'ont pas de contract_id assigné.

**Hypothèses**:
- Workflow d'installation incomplet: équipements installés sans contrat
- Problème de jointure entre intervention et contrat
- Matériel installé en mode "test" ou "temporaire" sans contrat formel

**Impact**:
- Empêche probablement la mise à jour automatique du statut via trigger/workflow
- Pas de lien entre l'installation et le contrat client
- Facturation potentiellement impactée

### 🟠 3. État "OO" corrélé aux dates futures

**Constat**: 77% des équipements en état "OO" ont des dates futures vs 19% pour état "ON".

**Hypothèses**:
- L'état "OO" (Out of Order?) déclenche un calcul automatique de date incorrect
- Peut-être une "date de remise en service planifiée" au lieu de last_update
- Processus de gestion d'équipement défectueux différent

**Impact**: Impossible de savoir quand ces équipements ont réellement été marqués "disponibles".

### 🟠 4. 95% des last_update postérieures à l'installation

**Constat**: 309/322 équipements ont last_update > EQUIPMENT_INS_DATE.

**Analyse**:
- Normal si mise à jour périodique du stock APRÈS installation
- MAIS combiné avec dates futures = problème de cohérence
- Suggère que le stock est mis à jour après installation mais avec des dates incorrectes

### 🟠 5. Client ORANGE disproportionnellement affecté

**Constat**: 87% des équipements ORANGE sont en état OO vs 61% pour GROUPAMA.

**Hypothèses**:
- Workflow spécifique ORANGE différent de GROUPAMA
- Type de matériel ORANGE plus sujet à pannes (articles capteurs/détecteurs)
- Process de gestion du matériel défectueux différent selon client

---

## Recommandations d'actions

### 🚨 Actions immédiates

1. **Audit du champ `last_update` dans la base logistique**
   - Vérifier la définition exacte de ce champ
   - Identifier si c'est vraiment "dernière mise à jour" ou une autre date
   - Requête: `SELECT * FROM teamtool_logistique.stock WHERE last_update >= '2025-01-01' AND manufacturer='CRO'`

2. **Corriger les 211 dates futures**
   - Script de correction pour ramener à la date du jour ou date d'installation
   - Tracer l'origine de ces dates (logs, triggers, saisie manuelle)

3. **Enquête sur contract_id vides**
   - Vérifier pourquoi 100% des équipements n'ont pas de contrat
   - Identifier le processus normal vs cas aberrants
   - Requête: Vérifier les interventions terminées sans contract_id

### 🔍 Actions d'investigation

4. **Analyser le workflow de l'état "OO"**
   - Auditer le code/trigger qui gère cet état
   - Vérifier les règles de calcul de date associées
   - Documenter la différence avec état "ON"

5. **Audit spécifique client ORANGE**
   - Pourquoi 87% en état OO vs 61% GROUPAMA?
   - Y a-t-il un processus différent selon le client?
   - Articles ORANGE plus fragiles ou processus de gestion différent?

6. **Vérifier la synchronisation intervention ↔ stock**
   - Tracer le workflow: installation → mise à jour stock
   - Identifier les points de rupture de synchronisation
   - Vérifier les triggers automatiques

### 🛠️ Actions préventives

7. **Mise en place de contraintes**
   - Contrainte: `last_update <= CURRENT_DATE`
   - Alerte si installation sans contract_id
   - Validation des états cohérents

8. **Audit périodique**
   - Script mensuel pour détecter ces incohérences
   - Dashboard des équipements "installés mais disponibles"
   - Alerte si nouvelles incohérences

9. **Formation techniciens**
   - Sensibiliser les top 5 user_code concernés (ELI.008, FBRET1, etc.)
   - Procédure claire de mise à jour statut après installation
   - Importance du renseignement du contract_id

---

## Requêtes SQL utiles pour investigation

### Trouver tous les équipements avec dates futures
```sql
SELECT *
FROM teamtool_logistique.stock
WHERE manufacturer = 'CRO'
  AND status = 'D'
  AND last_update >= '2025-01-01'
ORDER BY last_update DESC;
```

### Identifier les interventions terminées sans contrat
```sql
SELECT i.INTERVENTION_ID, i.CODE_intervention_EVENT_STATUS,
       COUNT(ie.EQUIPMENT_SERIAL_NUMBER) as nb_equipements
FROM teamtool_tech_mm.intervention i
JOIN teamtool_tech_mm.intervention_equipment ie ON i.INTERVENTION_ID = ie.INTERVENTION_ID
LEFT JOIN teamtool_tech_mm.intervention_contract ic ON i.INTERVENTION_ID = ic.INTERVENTION_ID
WHERE i.CODE_intervention_EVENT_STATUS = 'TER'
  AND ic.contract_id IS NULL
GROUP BY i.INTERVENTION_ID;
```

### Tracer l'historique d'un équipement spécifique
```sql
SELECT *
FROM teamtool_logistique.stock_history
WHERE serial_number = '3527780'
ORDER BY change_date DESC;
```

### Comparer les états par client
```sql
SELECT
    CASE
        WHEN ie.barcode LIKE '%ORANGE%' THEN 'ORANGE'
        WHEN ie.barcode LIKE '%GROUPAMA%' THEN 'GROUPAMA'
        ELSE 'Autre'
    END as client,
    s.state,
    COUNT(*) as nb_equipements
FROM teamtool_tech_mm.intervention_equipment ie
JOIN teamtool_logistique.stock s ON s.serial_number = ie.EQUIPMENT_SERIAL_NUMBER
JOIN teamtool_tech_mm.intervention i ON i.INTERVENTION_ID = ie.INTERVENTION_ID
WHERE i.CODE_intervention_EVENT_STATUS = 'TER'
  AND s.manufacturer = 'CRO'
  AND ie.MAT_CETAT IN ('PI', 'NI')
  AND s.status = "D"
GROUP BY client, s.state;
```

---

## Conclusion

L'analyse révèle **deux problèmes critiques**:

1. **65% des équipements ont des dates de mise à jour dans le futur**, fortement corrélé avec l'état "OO" (77%)
2. **100% n'ont pas de contract_id**, ce qui suggère un problème de workflow

Ces incohérences empêchent une gestion fiable du stock et nécessitent:
- Une correction immédiate des données (211 dates futures)
- Une investigation approfondie du workflow et des triggers
- Une mise en place de contrôles préventifs

**Priorité**: Auditer le champ `last_update` et le processus de gestion de l'état "OO" qui concentre 78% des incohérences.

---

**Analysé le**: 2025-10-08
**Par**: Claude Code
**Nombre d'enregistrements analysés**: 322
