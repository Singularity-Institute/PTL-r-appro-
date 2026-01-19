# Spécification Technico-Fonctionnelle - Transfert de Stocks entre Dépôts (TRF)

## Contexte et Objectif

**En tant que** Responsable Logistique
**Je souhaite** que les utilisateurs logistique puissent créer et traiter un transfert de stocks entre dépôt
**Afin de** réaliser un nouveau type de mouvement de matériels connus (avec numéros de série identifiés)

### Particularités
- Transfert de matériels qui ne sont pas nécessairement dans un état permettant l'envoi via APP entre dépôts (neuf, ON ou OO)
- Matériels peuvent être dans des états différents de "OO" et "ON"
- **Pas d'utilisation de Chronopost** : transferts via véhicule ESAT
- Pas d'étape de validation de colis (passage direct à la validation)
- Pas de poids de colis ni de transporteur

---

## 1. Nouvel Onglet "Transfert de stock"

Création d'un nouvel onglet dans le Menu "Demandes" dédié aux transferts de stocks.

---

## 2. Process TRF - États de la Demande

Lié à l'événement [TOP-6509](https://protectline.atlassian.net/browse/TOP-6509)

### Statuts de la Demande

**EXPÉDITION**
- **"F"** : À faire (création)
- **"C"** : En cours (validé)

**RÉCEPTION**
- **"T"** : Traité (réceptionné dans TT)

### Numérotation
- Création d'un numéro de demande "TRF"
- Question en suspens : même compteur que les APP ou numérotation séparée ?

---

## 3. Expédition - Création de la Demande

### Conditions Initiales des Matériels

Avant insertion dans la demande, les matériels doivent être :

**Dépôt**
- Dans le dépôt de l'utilisateur

**Statut**
- "RE" uniquement

**États Interdits**
- H - Hors service occasion
- HC - Hors service cause client
- HN - Hors service neuf
- AR - À rebuter
- D - Détérioré

**États Autorisés**
- AC - À reconditionner
- AT - À tester
- C - À contrôler
- DF - État par défaut
- NI - Non identifiable
- ON - Opérationnel Neuf
- OO - Opérationnel Occasion

### A/ Ajout des Matériels Scannés

Au moment du scan, les matériels passent :
- **Statut** : à définir (similaire APP entre dépôt au statut "F") → probablement "T"
- **État** : pas de changement
- **Dépôt** : à définir
- **Type d'événement** : "TRF" (si changement de statut uniquement)

---

## 4. Traitement de la Demande d'Expédition

### B1/ Validation de la Demande (Bouton "VALIDER")

**Pop-up de Récapitulatif**
- Article_Type x Quantité
- Expéditeur
- Destinataire

**Au passage de "F" à "C"** (historisation du mouvement) :
- **Statut matériels** : "T"
- **État** : pas de changement
- **Dépôt** : à définir (similaire APP entre dépôt au statut "C")
- **Type d'événement** : "TRF"
- Création du numéro de demande "TRF" (si pas créé initialement)
- **Génération du Bon de Livraison** :
  - Affichage en front
  - Téléchargement dans le navigateur
  - Accessible depuis la Recherche de Demandes
- Demande retrouvable dans la liste des demandes

### B2/ Annulation de la Demande (Bouton "ANNULER")

**Au passage de "F" à "A"** (historisation) :
- Suppression de tous les matériels de la demande
- Si pas de mouvement de stock, rien à changer
- Demande retrouvable dans la Recherche de Demandes au statut "A"
- Pop-up de confirmation : "Voulez-vous vraiment annuler cette demande de transfert"
- Bouton toujours disponible

### B3/ Vidage de la Demande (Bouton "VIDER")

**Statut reste "F"** (historisation via logs) :
- Suppression de tous les matériels de la demande
- Si pas de mouvement de stock, rien à changer
- Possibilité de recommencer le scan
- Pop-up de confirmation : "Voulez-vous vraiment vider cette liste"
- Disponible uniquement s'il y a au moins un matériel valide scanné

---

## 5. Réception - Traitement de la Demande

### Au passage de "C" à "T" (Process de réception)

**Matériels Réceptionnés** (historisation) :
- **Statut** : "RE"
- **État** : pas de changement
- **Dépôt** : dépôt destinataire
- **Type d'événement** : "TRF"

**Nota** : Si matériel reçu non attendu → utiliser aussi "TRF"

### Matériels Non Réceptionnés

Selon le mode de réception :

**Sans scan**
- Pas de retour dans le dépôt d'origine
- Vérification des quantités

**Avec scan**
- Matériels non scannés après validation passent :
  - **Statut** : "RE"
  - **État** : pas de changement
  - **Dépôt** : dépôt d'expédition
  - **Type de mouvement** : "NRP" (cf. [TOP-6363](https://protectline.atlassian.net/browse/TOP-6363))

---

## 6. Formulaire de Création du TRF

### Choix des Dépôts

**Dépôt d'Expédition**

*Si Profil RLO* - Choix manuel :
- ULO : Avermes Yzeure
- ULOM : Montluçon
- PTL : Protectline

*Si Profil Utilisateur Logistique* - Ajout automatique du dépôt de rattachement (via partner ID ou profil) :
- ULO : Avermes Yzeure
- ULOM : Montluçon
- PTL : Protectline

**Dépôt de Destination**
- Pour RLO et Utilisateur Logistique : proposition des deux autres dépôts disponibles différents du dépôt d'expédition

**Adresses**
- Les adresses n'ont pas besoin d'être ajoutées en Front (validé avec Abdel)

**Zone Commentaire**
- Disponible comme pour un APP
- Commentaire présent dans le bon de livraison et le récapitulatif

### Gestion du Scan

**Bouton "Scanner"**
- Disponible uniquement si les deux dépôts sont sélectionnés, sinon grisé
- Module de visualisation du scan
- Bouton "poubelle" dans la colonne "Action" pour supprimer unitairement un matériel scanné

### Gestion de la Demande

**Bouton "VALIDER"**
- Disponible si :
  - Les deux dépôts sont sélectionnés
  - Il y a au moins un matériel valide et scanné
- Pop-up de confirmation : "Voulez-vous vraiment valider ce transfert"

**Bouton "VIDER"**
- Nettoie le contenu sans annuler la demande
- Disponible si : il y a au moins un matériel valide et scanné
- Pop-up de confirmation : "Voulez-vous vraiment vider cette liste"

**Bouton "ANNULER"**
- Annule la demande et rollback les traitements
- Demande passe au statut "A"
- Pop-up de confirmation : "Voulez-vous vraiment annuler cette demande de transfert"
- Toujours disponible (pas de contrainte)

---

## 7. Workflow de Création d'une Demande de Transfert

1. Choix des Dépôts
2. Clic sur "Scanner"
3. Scan d'un matériel (répété pour n matériels)
4. Clic sur "Ajouter" (fermeture de la pop-up)
5. Clic sur "Valider"
6. Confirmation "Oui"
7. Affichage du Bon de Livraison

---

## 8. Bon de Livraison - Complétion des Champs

Pas de changement du document, uniquement récapitulatif de l'alimentation :

| Champ | Valeur |
|-------|--------|
| **Commande** | N° de la DI TRF |
| **Date** | Alimentation habituelle |
| **Client** | Vide |
| **Poids** | Vide |
| **Demande (type)** | Transfert |
| **Expéditeur** | Dépôt Expéditeur (pas d'adresse) |
| **Destinataire** | Dépôt Destinataire (pas d'adresse) |
| **Transporteur** | Vide |
| **Liste des types de matériels** | Alimentation habituelle |
| **Liste des SN par références** | Alimentation habituelle |

---

## 9. Contrôles et Messages d'Information

Mise en place de contrôles sur :
- Statut
- État
- Présence dans la demande
- Dépôt d'expédition

---

## 10. Incohérences et Clarifications Nécessaires

### 🔴 CRITIQUES - Incohérences Bloquantes

#### 10.1. Statut des Matériels lors du Scan - INCOHÉRENCE MAJEURE

**Problème identifié** :
- Section A/ : "au statut : à définir [...] → probablement 'T'" suivi de "Nota : pas de changement je pense"
- Section B1/ : lors de la validation F→C, "Statut matériels : 'T'"

**Incohérence** :
- Les matériels changent-ils de statut lors du scan ou lors de la validation ?
- Si changement lors du scan : pourquoi re-changer lors de la validation ?
- Si pas de changement lors du scan : comment sont-ils "réservés" pour la demande ?

**Impact** :
- Impossible de développer sans clarifier ce point
- Risque de désynchronisation des stocks
- Problème de traçabilité des matériels

**Clarifications requises** :
1. Définir le statut exact lors du scan dans la demande "F"
2. Définir si un nouveau statut intermédiaire est nécessaire (ex: "TRF_EN_COURS")
3. Clarifier le cycle complet : RE → ? → T → RE

**Proposition technique** :
- Option A : Matériels restent "RE" jusqu'à validation, puis passent "T" (risque : matériel scanné par erreur dans 2 demandes)
- Option B : Matériels passent "T_PREPARATION" au scan, puis "T" à la validation (plus sûr)
- Option C : Ajout d'un champ "demande_id" sur le matériel pour le "bloquer" sans changer le statut

---

#### 10.2. Dépôt Intermédiaire - INFORMATION MANQUANTE

**Problème identifié** :
Mention répétée de "au dépôt : à définir" sans précision :
- Lors du scan (section A/)
- Lors de la validation F→C (section B1/)

**Questions fonctionnelles** :
1. Les matériels restent-ils dans le dépôt d'expédition jusqu'à validation ?
2. Faut-il créer un dépôt virtuel "EN_TRANSFERT" ?
3. Comment traiter les matériels d'une demande annulée ?

**Impact** :
- Incohérence potentielle dans les inventaires
- Impossible de générer des états de stock fiables
- Blocage pour les développements de reporting

**Clarifications requises** :
- Définir le dépôt des matériels à chaque étape du cycle de vie
- Spécifier si un dépôt technique/virtuel est nécessaire

**Proposition technique** :
| Étape | Statut Demande | Statut Matériel | Dépôt Matériel |
|-------|----------------|-----------------|----------------|
| Scan | F | RE ou T_PREP | Dépôt Expédition |
| Validation | C | T | EN_TRANSIT (virtuel) |
| Réception | T | RE | Dépôt Destinataire |

---

#### 10.3. Gestion des Écarts de Réception - LOGIQUE CONTRADICTOIRE

**Problème identifié** :
Section 5 - Matériels Non Réceptionnés :

**Mode "Sans scan"** :
- "Pas de retour dans le dépôt d'origine"
- Mais alors où sont physiquement et logiquement les matériels non reçus ?

**Mode "Avec scan"** :
- Matériels non scannés retournent au "dépôt d'expédition"
- **INCOHÉRENCE** : Les matériels sont physiquement dans le camion/chez le destinataire, pas à l'expédition

**Questions critiques** :
1. Comment traiter les matériels perdus/endommagés en transit ?
2. Faut-il créer un état "PERDU_EN_TRANSFERT" ?
3. Qui déclenche la recherche de matériels manquants ?
4. Quel délai avant de déclarer un matériel perdu ?

**Impact Business** :
- Perte de traçabilité des matériels
- Risque d'incohérence inventaire
- Pas de process de gestion des litiges

**Clarifications requises** :
1. Définir le process complet de gestion des écarts (manquants, surplus, endommagés)
2. Spécifier les rôles et responsabilités (expéditeur vs destinataire)
3. Définir les actions correctives et les délais
4. Prévoir un workflow de réclamation/enquête

---

### 🟠 IMPORTANTES - Ambiguïtés Fonctionnelles

#### 10.4. Création du Numéro de Demande - TIMING FLOU

**Problème identifié** :
- Section 2 : "Création d'un numéro de demande TRF" sans précision du moment
- Section B1/ : "Création du numéro de demande TRF (si pas créé initialement)"

**Questions** :
1. Le numéro est-il créé dès l'ouverture du formulaire ?
2. Au premier scan d'un matériel ?
3. Seulement à la validation ?

**Impact** :
- Gestion des compteurs/séquences
- Référencement des logs
- Traçabilité des actions utilisateur

**Clarifications requises** :
- Définir le moment exact de création du numéro
- Spécifier si une demande "F" annulée doit avoir un numéro ou non
- Clarifier la règle de numérotation (partagée avec APP ou séparée)

**Recommandation technique** :
- Création du numéro dès la validation (F→C) uniquement
- Avantages : pas de "trous" dans la numérotation, pas de numéros pour demandes avortées

---

#### 10.5. Historisation du Vidage - STRATÉGIE FLOUE

**Problème identifié** :
Section B3/ : "historisation ?? 'Réinitialisation' les logs suffiront pour traquer cette réinitialisation ?"

**Questions** :
1. Faut-il créer une entrée dans l'historique des demandes ?
2. Les logs applicatifs suffisent-ils pour l'audit ?
3. Quel niveau de détail conserver (liste des matériels vidés) ?

**Impact** :
- Conformité audit/traçabilité
- Capacité à reconstituer l'historique
- Debug en cas de problème

**Clarifications requises** :
- Définir la stratégie d'historisation (table historique vs logs)
- Spécifier les informations à conserver
- Durée de rétention des informations

---

#### 10.6. Rollback lors de l'Annulation - CONDITION AMBIGUË

**Problème identifié** :
Section B2/ : "rollbacker les traitements sur les matériels (si les matériels passent à 'T' à ce stade d'insertion)"

**Ambiguïté** :
- Condition "si" implique que parfois il n'y a pas de rollback
- Mais quand n'y a-t-il pas de rollback ?
- Lié à l'incohérence 10.1 sur le statut lors du scan

**Clarifications requises** :
1. Dans quels cas le rollback est-il nécessaire ?
2. Quel est l'état cible des matériels après annulation ?
3. Comment gérer une annulation après validation (C→A) ?

---

### 🟡 MODÉRÉES - Détails à Préciser

#### 10.7. Contrôles et Messages d'Information - SPÉCIFICATION INCOMPLÈTE

**Problème identifié** :
Section 9 très vague : "Mise en place de contrôles sur : Statut, État, Présence dans la demande, Dépôt d'expédition"

**Manques** :
- Aucune règle de validation détaillée
- Aucun message d'erreur spécifié
- Pas de comportement en cas d'échec de contrôle

**Clarifications requises** :
1. **Contrôle Statut** : Message si matériel pas en "RE" ?
2. **Contrôle État** : Message exact pour chaque état interdit ?
3. **Contrôle Présence** : Que faire si matériel déjà dans une autre demande TRF "F" ou "C" ?
4. **Contrôle Dépôt** : Message si matériel dans mauvais dépôt ?
5. **Contrôle Doublon** : Peut-on scanner 2 fois le même SN dans une demande ?

**Proposition** :
Créer un tableau exhaustif :

| Contrôle | Condition | Message Utilisateur | Action Système |
|----------|-----------|---------------------|----------------|
| Statut | Matériel ≠ "RE" | "Le matériel [SN] n'est pas au statut RE (statut actuel : [X])" | Refus du scan |
| État | Matériel en état HS/AR/D | "Le matériel [SN] est dans un état incompatible : [État]" | Refus du scan |
| Dépôt | Matériel hors dépôt expédition | "Le matériel [SN] n'est pas dans le dépôt d'expédition [Dépôt]" | Refus du scan |
| Doublon | SN déjà scanné | "Le matériel [SN] est déjà dans la liste" | Refus du scan |
| Autre demande | Matériel dans demande TRF active | "Le matériel [SN] est déjà dans la demande TRF [N°]" | Refus du scan |

---

#### 10.8. Gestion de l'État des Matériels à la Réception - LIMITATION FONCTIONNELLE

**Problème identifié** :
Les matériels conservent leur état tout au long du transfert ("État : pas de changement")

**Scénario problématique** :
1. Matériel scanné en état "AT" (À tester) à l'expédition
2. Durant le transport, le matériel est endommagé
3. À la réception, le destinataire constate l'endommagement
4. **MAIS** : pas de mécanisme pour changer l'état à "D" (Détérioré)

**Questions fonctionnelles** :
1. Comment gérer les matériels endommagés pendant le transfert ?
2. Qui est responsable : expéditeur ou destinataire ?
3. Faut-il prévoir un process de litige/contestation ?
4. Peut-on refuser une réception partielle ?

**Impact** :
- Impossible de tracer les dommages survenus en transit
- Pas de responsabilité claire
- Risque de conflit entre dépôts

**Clarifications requises** :
1. Ajouter une étape de contrôle qualité à la réception avec possibilité de modifier l'état ?
2. Prévoir un statut "LITIGE" pour les matériels contestés ?
3. Définir un workflow de résolution de litige ?

---

#### 10.9. Adresses dans le Bon de Livraison - INFORMATION CONTRADICTOIRE

**Problème identifié** :
- Section 6 : "Les adresses n'ont pas besoin d'être ajoutées en Front (validé avec Abdel)"
- Section 8 : Dans le BL, "Expéditeur : Dépôt Expéditeur (pas d'adresse)"

**Ambiguïté** :
- Les adresses sont-elles affichées ou non dans le BL ?
- Si non affichées : comment le chauffeur ESAT sait-il où livrer ?
- Mention "(pas besoin de l'adresse)" = pas affichée ou pas saisie en front ?

**Clarifications requises** :
1. Préciser si les adresses doivent apparaître dans le BL PDF généré
2. Si oui, d'où proviennent-elles (base de données des dépôts) ?
3. Format d'affichage des adresses

**Recommandation** :
- Pas de saisie manuelle en front (OK)
- Mais récupération automatique des adresses depuis le référentiel des dépôts
- Affichage dans le BL pour le chauffeur

---

#### 10.10. Type d'Événement "TRF" - RÈGLE FLOUE

**Problème identifié** :
Section A/ : "Type d'événement 'TRF' (si changement de statut uniquement)"
Suivi de : "Nota : pas de changement je pense"

**Contradiction** :
- Si pas de changement de statut → pas d'événement TRF créé ?
- Mais comment tracer que le matériel a été ajouté à une demande ?

**Clarifications requises** :
1. Faut-il créer un événement "TRF" systématiquement à chaque étape ?
2. Ou uniquement lors des changements de statut ?
3. Comment tracer l'ajout/retrait d'un matériel dans une demande "F" ?

---

#### 10.11. Suppression Unitaire de Matériels - COMPORTEMENT NON SPÉCIFIÉ

**Problème identifié** :
Section 6 : "Bouton 'poubelle' dans la colonne 'Action' pour supprimer unitairement un matériel scanné"

**Questions** :
1. Que se passe-t-il au niveau du statut/dépôt du matériel supprimé ?
2. Le matériel retourne-t-il immédiatement à "RE" (si changé lors du scan) ?
3. Y a-t-il une confirmation avant suppression ?
4. Peut-on supprimer un matériel d'une demande validée "C" ?

**Clarifications requises** :
- Spécifier le comportement exact de cette action
- Définir les règles métier (autorisé uniquement si demande "F" ?)

---

#### 10.12. Matériels Reçus Non Attendus - TRAITEMENT INCOMPLET

**Problème identifié** :
Section 5 : "Nota : Si matériel reçu non attendu → utiliser aussi 'TRF'"

**Questions critiques** :
1. Comment un matériel non attendu peut-il arriver dans un transfert planifié ?
2. Quel statut/dépôt pour ce matériel ?
3. Faut-il créer une alerte pour l'expéditeur ?
4. Peut-on refuser le matériel ?
5. Impact sur la demande (reste "C" ou passe "T") ?

**Scénarios à clarifier** :
- Erreur d'emballage à l'expédition (mauvais SN scanné physiquement)
- Matériel bonus (pourquoi ?)
- Fraude/vol ?

---

### 📋 Tableau Récapitulatif des Clarifications

| ID | Sujet | Priorité | Bloquant Dev ? | Décideur |
|----|-------|----------|----------------|----------|
| 10.1 | Statut matériels au scan | CRITIQUE | ✅ OUI | Responsable Logistique + Tech Lead |
| 10.2 | Dépôt intermédiaire | CRITIQUE | ✅ OUI | Responsable Logistique |
| 10.3 | Écarts de réception | CRITIQUE | ✅ OUI | Responsable Logistique |
| 10.4 | Timing création numéro | IMPORTANTE | ⚠️ PARTIEL | Tech Lead |
| 10.5 | Historisation vidage | IMPORTANTE | ❌ NON | Tech Lead + Audit |
| 10.6 | Rollback annulation | IMPORTANTE | ⚠️ PARTIEL | Responsable Logistique |
| 10.7 | Contrôles détaillés | MODÉRÉE | ❌ NON | Responsable Logistique |
| 10.8 | Changement état réception | MODÉRÉE | ❌ NON | Responsable Logistique |
| 10.9 | Adresses BL | MODÉRÉE | ❌ NON | Responsable Logistique |
| 10.10 | Événements TRF | MODÉRÉE | ❌ NON | Tech Lead |
| 10.11 | Suppression unitaire | MODÉRÉE | ❌ NON | Tech Lead |
| 10.12 | Matériels non attendus | MODÉRÉE | ❌ NON | Responsable Logistique |

---

### 🎯 Actions Recommandées Avant Développement

1. **URGENT** : Organiser un atelier de clarification fonctionnelle sur les points 10.1, 10.2, 10.3
2. **IMPORTANT** : Documenter les règles de gestion des écarts et litiges
3. **SOUHAITABLE** : Créer un diagramme de séquence détaillé avec tous les états
4. **SOUHAITABLE** : Rédiger les spécifications des messages d'erreur
5. **RECOMMANDÉ** : Valider l'architecture technique (dépôt virtuel, statuts intermédiaires)

---

## Points en Suspens

1. Numérotation des demandes TRF : même compteur que les APP ou séparé ?
2. Moment de création du numéro : à la création "F" ou à la validation "F" → "C" ?
3. Statut des matériels lors de l'ajout dans la demande (statut "F")
4. Historisation du vidage de la demande : via logs uniquement ?
