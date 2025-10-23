# Processus de Réception APP Multi-Colis (Nouvelle Approche)

## Vue d'ensemble

Cette approche simplifie le travail du technicien en déportant la logique de détection d'écarts globaux vers le système informatique. Le technicien se concentre sur la validation colis par colis, et le système consolide automatiquement les écarts pour détecter les compensations.

---

## 1. Réception Colis par Colis (Côté Technicien)

### 1.1 Processus pour chaque colis

Pour chaque colis de l'expédition, le technicien :

1. **Ouvre le colis** et compte les équipements reçus par type (DO, centrales, DFO, etc.)

2. **Compare avec le bon de livraison (BL)** pour ce colis spécifique

3. **Déclare le statut du colis** :
   - **Sans écart** : les quantités correspondent au BL → valider le colis directement
   - **Avec écart** : une ou plusieurs quantités diffèrent du BL → cocher "Présence d'écart" et valider

4. **Répète** pour tous les colis de l'APP

### 1.2 Points importants

- Le technicien **ne fait pas** de contrôle global multi-colis à ce stade
- Il se contente de signaler les écarts colis par colis
- Il n'a **pas besoin d'identifier** quel colis est "fautif" ou de réaffecter des pièces entre colis
- Chaque colis est validé indépendamment avec ou sans écart

---

## 2. Consolidation Automatique (Côté Système)

### 2.1 Déclenchement

Lorsque le technicien tente de **valider globalement l'APP** (après avoir validé tous les colis), le système lance automatiquement la consolidation des écarts.

### 2.2 Logique de consolidation

Le système :

1. **Récupère tous les colis** de l'APP avec leur statut (avec/sans écart)

2. **Calcule les écarts par type de matériel** :
   - Pour chaque type (DO, centrales, DFO, etc.)
   - Somme les écarts de tous les colis : `Écart global = Σ(Reçu - Attendu) pour tous les colis`

3. **Analyse le résultat** :
   - **Écart global = 0 pour tous les types** → Les écarts se compensent parfaitement
   - **Écart global ≠ 0 pour au moins un type** → Il reste un écart réel

### 2.3 Scénarios possibles

#### Scénario A : Écarts qui se compensent (Écart global = 0)

**Exemple** :
- Colis A : -2 DO, +1 centrale
- Colis B : +2 DO, -1 centrale
- **Écart global** : 0 DO, 0 centrale

**Action du système** :
- Affiche une **pop-up informative** :
  ```
  ⚠️ Écarts détectés sur les colis individuels

  Cependant, les écarts se compensent globalement :
  - DO : Écart global = 0 (+2 / -2)
  - Centrales : Écart global = 0 (+1 / -1)

  La réception est conforme au total.

  [Confirmer la réception] [Réessayer le comptage]
  ```

- **Si "Confirmer"** : valider l'APP avec traçabilité des écarts colis par colis (pour audit)
- **Si "Réessayer"** : permettre au technicien de recompter/corriger les colis

#### Scénario B : Écart global réel (Écart global ≠ 0)

**Exemple** :
- Colis A : -3 DO
- Colis B : +2 DO
- **Écart global** : -1 DO

**Action du système** :
- Affiche une **pop-up d'alerte** :
  ```
  ❌ Écart global détecté

  Différence par rapport au bon de livraison :
  - DO : -1 (manquant)

  Un processus d'écart doit être créé.

  [Confirmer et créer l'écart] [Réessayer le comptage]
  ```

- **Si "Confirmer"** :
  - Créer un dossier d'écart global pour l'APP
  - Tracer l'écart résiduel par type
  - Lancer le workflow de traitement d'écart (déclaration fournisseur, etc.)

- **Si "Réessayer"** : permettre au technicien de recompter/corriger les colis

---

## 3. Avantages de Cette Approche

### 3.1 Simplicité pour le technicien
- Pas de calcul mental complexe sur plusieurs colis
- Validation colis par colis, simple et rapide
- Pas de gestion manuelle de réaffectation de pièces

### 3.2 Fiabilité
- Le système détecte automatiquement les compensations d'écarts
- Réduction des erreurs humaines de calcul
- Traçabilité complète des écarts (même compensés)

### 3.3 Flexibilité
- Possibilité de réessayer en cas de doute
- Conservation de l'historique des écarts colis par colis pour audit

---

## 4. Workflow Complet - Schéma Récapitulatif

```
Réception APP Multi-Colis
        |
        v
[Pour chaque colis]
    - Ouvrir et compter
    - Comparer au BL
    - Valider avec/sans écart
        |
        v
[Tous les colis validés ?]
    Non → Continuer
    Oui → Validation globale APP
        |
        v
[Système : Consolidation automatique]
    - Calcul écarts par type
    - Σ(Reçu - Attendu) pour tous colis
        |
        v
    [Écart global = 0 ?]
        |
        +-- Oui → Pop-up informative
        |         "Écarts se compensent"
        |         [Confirmer] / [Réessayer]
        |
        +-- Non → Pop-up d'alerte
                  "Écart global détecté : X unités"
                  [Créer écart] / [Réessayer]
```

---

## 5. Points à Clarifier

### 5.1 Gestion du "Réessayer"
- Quels colis le technicien peut-il modifier ?
- Faut-il invalider tous les colis ou seulement ceux avec écart ?

### 5.2 Traçabilité
- Doit-on conserver l'historique des écarts colis par colis même s'ils se compensent ?
- Quel niveau de détail dans les logs ?

### 5.3 Écarts partiels
- Si DO se compensent mais pas les centrales, comment gérer ?
- Créer un écart uniquement pour les types non compensés ?

### 5.4 Réaffectation manuelle
- Faut-il permettre au technicien de déplacer des pièces entre colis avant validation globale ?
- Ou cette fonctionnalité n'est-elle plus nécessaire avec la consolidation automatique ?

### 5.5 Validation partielle
- Peut-on valider un APP même avec des colis en attente ?
- Ou tous les colis doivent-ils être validés avant la consolidation ?
