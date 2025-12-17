# Gestion des Poids pour la Stratégie PRDV avec TimeFold
## Guide de Calibration et d'Utilisation

---

## 📋 Présentation de l'Outil

### Objet

Ce document présente un **outil interactif HTML** conçu pour calibrer et comprendre l'impact des contraintes de planification dans le système TimeFold. Il permet aux décideurs et opérationnels de visualiser en temps réel comment les poids (α, β, γ, δ) influencent les décisions d'optimisation.

### Fichier concerné

**`Gestion des poids pour la stratégie PRDV avec TimeFold.html`**

Outil web autonome (pas de serveur requis), à ouvrir directement dans un navigateur.

---

## 🎯 Pourquoi cet Outil est Essentiel

### Le Problème Business

Sans visualisation, il est difficile de comprendre concrètement ce que signifient des paramètres comme **α=100, γ=200**. Les questions suivantes restent floues :
- *"À partir de quelle distance le système privilégie-t-il la date plutôt que l'optimisation des km ?"*
- *"Combien d'heures de surcharge le système tolère-t-il pour tenir un planning ?"*
- *"Quelle est la valeur monétaire d'un jour de retard selon nos réglages ?"*

### La Solution

L'outil HTML traduit ces paramètres abstraits en **arbitrages concrets et mesurables** :
- **Distance critique (d₀)** : "En dessous de 3.8 km, on optimise les trajets ; au-dessus, on tient la date"
- **Charge critique (c₀)** : "On accepte max 2h10 de surcharge pour éviter de décaler au lendemain"
- **Équivalences** : "1 jour de retard = 3.8 km supplémentaires" ou "1 heure de charge = 2.4 km à 10 km de distance"

---

## 🏗️ Structure de l'Outil

L'outil est divisé en **deux grandes sections** :

### Section 1 : Calculateurs Paramétriques

Explorent comment les poids influencent le comportement du système. Permettent de trouver les valeurs critiques (d₀, c₀) et de visualiser les sensibilités.

**4 calculateurs :**
1. Sensibilités Globales
2. Distance Critique (d₀)
3. Charge Critique (c₀)
4. Fonction d'Équivalence c(d)

### Section 2 : Calculateurs d'Équivalences

Traduisent les arbitrages en équivalences concrètes pour faciliter la calibration métier.

**3 calculateurs :**
1. Distance ↔ Date
2. Date ↔ Charge
3. Distance ↔ Charge

---

## 📊 Description Détaillée des Calculateurs

---

## Section 1 : Calculateurs Paramétriques

---

### 1.1 Sensibilités Globales

#### 🎯 À quoi ça sert

Visualiser **quelle contrainte influence le plus le score global** selon la distance parcourue. À petite distance, la contrainte distance peut dominer. À grande distance, c'est la date qui prime. Ce graphique montre ces transitions.

#### 📐 Formule

$$\left|\frac{\partial S}{\partial x}\right| \text{ pour chaque contrainte } x$$

Où S est le score global et x représente chacune des 4 contraintes (distance, distance cumulée, date, charge).

#### 🎛️ Paramètres ajustables

- **α (Poids Distance)** : 10 à 1000
- **β (Poids Distance Cumulée)** : 10 à 1000
- **γ (Poids Date)** : 10 à 1000
- **δ (Poids Charge)** : 5 à 1000

#### 📈 Visualisation

**Graphique logarithmique** montrant 4 courbes :
- **Distance (α)** : Courbe bleue décroissante
- **Distance Cumulée (β)** : Courbe violette décroissante (pointillés)
- **Date (γ)** : Ligne verte horizontale
- **Charge (δ)** : Ligne rouge horizontale

#### 💡 Résultats affichés

- **Σ (Somme des poids)** : Total α+β+γ+δ
- **Contributions** : Pourcentage de chaque contrainte dans le score total
  - Exemple : α=23.8%, β=23.8%, γ=47.6%, δ=4.8%

#### 🔍 Interprétations Clés

1. **Les courbes distance décroissent en 1/(1+d)²** : Plus on s'éloigne, moins les km supplémentaires comptent
2. **Les courbes date et charge restent plates** : Leur importance ne dépend pas de la distance
3. **Le croisement des courbes** montre où une contrainte devient plus influente qu'une autre
4. **Configuration actuelle** (α=β=100, γ=200, δ=20) : La date contribue à 47.6% du score total

#### 📌 Usage Recommandé

- **Avant calibration** : Comprendre la configuration actuelle et identifier les déséquilibres
- **Pendant ajustement** : Voir en temps réel l'impact des changements de poids
- **Pour validation** : Vérifier que les priorités métier se reflètent dans les contributions

---

### 1.2 Distance Critique (d₀)

#### 🎯 À quoi ça sert

Trouver la **distance pivot** où le système bascule de *"j'optimise les km"* à *"je privilégie la date au planning"*.

- **En dessous de d₀** : Économiser des km vaut la peine de retarder
- **Au-dessus de d₀** : Respecter la date prime sur l'économie de carburant

#### 📐 Formule

$$d_0 = \sqrt{\frac{46\alpha}{\gamma}} - 1$$

#### 🎛️ Paramètres ajustables

- **α (Poids Distance)** : 10 à 1000
- **γ (Poids Date)** : 10 à 1000

#### 📈 Visualisation

Graphique montrant **d₀ en fonction du ratio α/γ** avec un point marquant la configuration actuelle.

#### 💡 Résultat affiché

- **d₀ = X.XX km**
- Interprétation : *"À d < X km : Distance domine | À d > X km : Date domine"*

#### 🔍 Interprétations Clés

| Configuration | d₀ | Comportement |
|---------------|-----|--------------|
| α=100, γ=200 | 3.8 km | Pour les interventions proches (< 3.8 km), le système accepte de retarder pour économiser des km. Pour les lointaines (> 3.8 km), il refuse le retard |
| α=200, γ=200 | 5.8 km | Zone d'optimisation distance s'étend (trajets urbains) |
| α=50, γ=200 | 2.3 km | Le système devient "date first" dès les courtes distances |

#### 💼 Exemples Métier

- **d₀ = 5 km** : On optimise les trajets urbains (< 5 km) mais on respecte le planning pour les trajets périurbains
- **d₀ = 2 km** : Système très strict sur les dates, n'optimise que pour les interventions à proximité immédiate
- **d₀ = 10 km** : Le système privilégie l'optimisation kilométrique sur une large zone

#### ⚙️ Règle de Calibration

- **Augmenter α ou diminuer γ** → d₀ augmente → Plus d'optimisation distance
- **Diminuer α ou augmenter γ** → d₀ diminue → Plus de respect des dates

---

### 1.3 Charge Critique (c₀)

#### 🎯 À quoi ça sert

Déterminer **combien d'heures de charge supplémentaire** le système accepte d'ajouter à une journée plutôt que de reporter l'intervention au lendemain. C'est l'arbitrage entre *"surcharger aujourd'hui"* vs *"planifier demain"*.

#### 📐 Formule

$$c_0 = \frac{10\gamma}{46\delta}$$

#### 🎛️ Paramètres ajustables

- **γ (Poids Date)** : 10 à 1000
- **δ (Poids Charge)** : 5 à 1000

#### 📈 Visualisation

Graphique montrant **c₀ en fonction du ratio γ/δ** avec un point marquant la configuration actuelle.

#### 💡 Résultats affichés

- **c₀ = X.XX heures (XhYY)**
- Interprétation : *"Ajouter X.XX h à une journée = retarder d'1 jour"*

#### 🔍 Interprétations Clés

| Configuration | c₀ | Comportement |
|---------------|-----|--------------|
| γ=200, δ=20 | 2h10 | Le système accepte de surcharger une journée de 2h10 maximum plutôt que de décaler |
| γ=200, δ=50 | 0h52 | Système très protecteur des techniciens, décale facilement |
| γ=400, δ=20 | 4h21 | Priorité absolue au SLA client, accepte de fortes surcharges |

#### 💼 Exemples Métier

- **c₀ = 1h** : *"On protège les techniciens, max 1h de surcharge"* → Bien-être prioritaire
- **c₀ = 4h** : *"On privilégie la réactivité client"* → SLA prioritaire
- **c₀ = 2h** : Configuration équilibrée

#### ⚙️ Règle de Calibration

- **Augmenter γ ou diminuer δ** → c₀ augmente → Tolère plus de charge (priorité date)
- **Diminuer γ ou augmenter δ** → c₀ diminue → Protège les techniciens (bien-être)

#### ⚠️ Point de Vigilance

**Impact RH** : Un c₀ trop élevé (> 3h) peut conduire à des surcharges systématiques et à l'épuisement des techniciens. Surveiller les KPIs de charge journalière.

---

### 1.4 Fonction d'Équivalence c(d)

#### 🎯 À quoi ça sert

Calculer **combien de km le système est prêt à parcourir pour économiser 1h de charge**, selon la distance déjà parcourue. Plus on est loin, plus la charge devient négligeable face à la distance (car les km supplémentaires comptent peu).

#### 📐 Formule

$$c(d) = \frac{\delta(1+d)^2}{10\alpha}$$

Où d est la distance de référence en km.

#### 🎛️ Paramètres ajustables

- **α (Poids Distance)** : 10 à 1000
- **δ (Poids Charge)** : 5 à 1000
- **d (Distance de référence)** : 0 à 100 km

#### 📈 Visualisation

Graphique montrant **c(d) en fonction de la distance** (courbe quadratique) avec un point marquant la distance sélectionnée.

#### 💡 Résultat affiché

- **c(d) = X.XX km/heure**
- Interprétation : *"À d km : 1 heure de charge ≡ X.XX km de distance"*

#### 🔍 Interprétations Clés

La courbe **croît quadratiquement** :

| Distance | c(d) | Signification |
|----------|------|---------------|
| 5 km | 0.7 km/h | À petite distance, la charge compte beaucoup |
| 10 km | 2.4 km/h | Arbitrage équilibré |
| 30 km | 19.2 km/h | La charge devient secondaire |
| 60 km | 74.2 km/h | La charge est négligeable (×100 par rapport à 5 km) |

#### 💼 Exemples Métier

**À 5 km** (zone urbaine) :
- Ajouter **2h de charge** = parcourir **1.4 km supplémentaires**
- → La charge est un critère important en ville

**À 50 km** (zone rurale) :
- Ajouter **2h de charge** = parcourir **52 km supplémentaires** !
- → Le système préfère largement surcharger que faire des km

#### ⚙️ Règle Pratique

- **En zone urbaine (< 10 km)** : La charge compte, on équilibre soigneusement
- **En zone rurale (> 30 km)** : La charge devient négligeable, optimisation distance prime

---

## Section 2 : Calculateurs d'Équivalences

---

### 2.1 Distance ↔ Date

#### 🎯 À quoi ça sert

Connaître **la distance que le système accepte de parcourir en plus pour éviter de retarder d'un jour**.

- Si cette distance est **élevée** (ex: 10 km) : Le système privilégie fortement la date
- Si elle est **faible** (ex: 2 km) : Il accepte plus facilement de décaler pour optimiser les trajets

#### 📐 Formule

$$\text{km}_{\text{equiv}} = \left(\sqrt{\frac{46\alpha}{\gamma}} - 1\right) \times \text{jours}$$

#### 🎛️ Paramètres ajustables

- **α (Poids Distance)** : 10 à 1000
- **γ (Poids Date)** : 10 à 1000

#### 🔄 Conversions bidirectionnelles

**Mode 1 : Jours → Km**
- Entrée : Nombre de jours de retard
- Sortie : Distance équivalente en km

**Mode 2 : Km → Jours**
- Entrée : Distance en km
- Sortie : Jours de retard équivalents

#### 📈 Visualisation

Graphique linéaire montrant la relation **jours ↔ km**.

#### 💡 Résultat principal

**1 jour ≡ X.XX km**

#### 🔍 Interprétations Clés

| Configuration | Équivalence | Comportement |
|---------------|-------------|--------------|
| α=100, γ=200 | 1 jour ≡ 3.8 km | Le système accepte 3.8 km de plus pour tenir la date |
| α=200, γ=200 | 1 jour ≡ 5.8 km | La distance compte davantage, système décale plus facilement |
| α=50, γ=300 | 1 jour ≡ 2.0 km | Le système tient coûte que coûte le planning (SLA strict) |

#### 💼 Scénarios Métier

**Scénario 1 : 1 jour ≡ 2 km** (date first)
- Le système privilégie absolument les dates
- Acceptable pour interventions urgentes ou contrats SLA stricts
- Risque : Coûts kilométriques élevés

**Scénario 2 : 1 jour ≡ 10 km** (coût first)
- Le système optimise agressivement les trajets
- Acceptable pour planification anticipée (AUTOPLANIF)
- Risque : Retards fréquents, insatisfaction client

**Scénario 3 : 1 jour ≡ 4 km** (équilibré)
- Compromis date/coût pour la plupart des contextes PRDV
- Configuration recommandée pour les prises de rendez-vous standard

#### 🎯 Aide à la Décision

Pour calibrer, posez-vous la question :

> *"Combien suis-je prêt à payer en km supplémentaires pour tenir un planning ?"*

- **2 km** = ~3€ de carburant → Priorité date
- **5 km** = ~7.5€ de carburant → Équilibré
- **10 km** = ~15€ de carburant → Priorité coût

---

### 2.2 Date ↔ Charge

#### 🎯 À quoi ça sert

Savoir **combien d'heures de charge le système tolère en plus pour tenir la date** du planning. C'est l'arbitrage entre *"respecter les SLA"* et *"protéger le bien-être des techniciens"*.

#### 📐 Formule

$$\text{heures}_{\text{equiv}} = \frac{10\gamma}{46\delta} \times \text{jours}$$

#### 🎛️ Paramètres ajustables

- **γ (Poids Date)** : 10 à 1000
- **δ (Poids Charge)** : 5 à 1000

#### 🔄 Conversions bidirectionnelles

**Mode 1 : Jours → Heures**
- Entrée : Nombre de jours de retard
- Sortie : Heures de charge équivalentes

**Mode 2 : Heures → Jours**
- Entrée : Heures de charge
- Sortie : Jours de retard équivalents

#### 📈 Visualisation

Graphique linéaire montrant la relation **jours ↔ heures de charge**.

#### 💡 Résultat principal

**1 jour ≡ X.XX heures**

#### 🔍 Interprétations Clés

| Configuration | Équivalence | Comportement |
|---------------|-------------|--------------|
| γ=200, δ=20 | 1 jour ≡ 2h10 | Surcharge max 2h10 pour tenir la date |
| γ=200, δ=50 | 1 jour ≡ 0h52 | Très protecteur des techniciens |
| γ=400, δ=20 | 1 jour ≡ 4h21 | Priorité SLA client, accepte fortes surcharges |

#### 💼 Scénarios Métier

**Scénario 1 : 1 jour ≡ 4h** (SLA prioritaire)
- Le système tolère jusqu'à 4h de surcharge pour ne pas décaler
- Adapté aux interventions urgentes ou clients VIP
- ⚠️ Risque : Burn-out techniciens, non-respect du droit du travail

**Scénario 2 : 1 jour ≡ 1h** (bien-être prioritaire)
- Le système protège strictement les horaires des techniciens
- Adapté pour améliorer QVT et attractivité employeur
- ⚠️ Risque : Plus de retards, insatisfaction client

**Scénario 3 : 1 jour ≡ 2h** (équilibré)
- Compromis acceptable pour la plupart des contextes
- Configuration recommandée pour PRDV standard

#### 🎯 Aide à la Décision

Questions à se poser :

> *"Combien d'heures supplémentaires par jour suis-je prêt à demander pour tenir les plannings ?"*

**Repères légaux et RH :**
- **< 1h** : Très respectueux du bien-être
- **1-2h** : Zone acceptable, à surveiller
- **2-3h** : Limite haute, vigilance requise
- **> 3h** : Risque juridique et RH élevé

#### 📊 Indicateurs à Suivre

Si vous augmentez c₀ (tolérez plus de charge), monitorer :
- **Charge journalière moyenne** (objectif : < 8h)
- **% de jours > 8h** (objectif : < 20%)
- **Écart-type de charge** (équité entre techniciens)
- **Taux de turn-over** (indicateur de satisfaction)

---

### 2.3 Distance ↔ Charge

#### 🎯 À quoi ça sert

Voir **l'arbitrage entre "rouler plus" et "charger plus"** selon la distance. À courte distance, la charge compte. À longue distance, elle devient négligeable. Cela montre quand le système privilégie l'optimisation kilométrique vs l'équilibrage de charge.

#### 📐 Formule

$$\text{km}_{\text{equiv}} = \frac{\delta(1+d)^2}{10\alpha} \times \text{heures}$$

Où d est la distance de référence.

#### 🎛️ Paramètres ajustables

- **α (Poids Distance)** : 10 à 1000
- **δ (Poids Charge)** : 5 à 1000
- **d (Distance de référence)** : 0 à 100 km

#### 🔄 Conversions bidirectionnelles

**Mode 1 : Heures → Km**
- Entrée : Heures de charge
- Sortie : Distance équivalente en km (à distance d)

**Mode 2 : Km → Heures**
- Entrée : Distance en km
- Sortie : Heures de charge équivalentes

#### 📈 Visualisation

Graphique linéaire (pour une distance d fixée) montrant **heures ↔ km**.

#### 💡 Résultat principal

**À d=X km : 1 heure ≡ Y.YY km**

#### 🔍 Interprétations Clés

L'équivalence **augmente quadratiquement** avec la distance :

| Distance | 1h de charge ≡ | Facteur |
|----------|----------------|---------|
| 5 km | 0.7 km | ×1 |
| 10 km | 2.4 km | ×3.4 |
| 30 km | 19.2 km | ×27 |
| 60 km | 74.2 km | ×106 |

#### 💼 Analyse par Zone Géographique

**Zone urbaine (< 10 km)** :
- 1h de charge ≈ 1-2 km
- La charge est un critère important
- L'équilibrage de charge prime sur quelques km

**Zone périurbaine (10-30 km)** :
- 1h de charge ≈ 5-20 km
- Arbitrage équilibré
- La charge compte encore

**Zone rurale (> 30 km)** :
- 1h de charge ≈ 20-100+ km
- La charge devient négligeable
- Optimisation distance prime totalement

#### 🎯 Implications Opérationnelles

**Exemple concret :**

Configuration : α=100, δ=20

**Intervention à 5 km** (ville) :
- Ajouter 3h de charge = 2.1 km supplémentaires
- → Le système hésite, la charge est valorisée

**Intervention à 60 km** (campagne) :
- Ajouter 3h de charge = 223 km supplémentaires !
- → Le système préfère largement surcharger que rouler

**Conclusion pratique :**
En zone rurale, le système tend naturellement à surcharger certains techniciens pour optimiser les trajets. Il faut donc **surveiller δ** pour éviter les déséquilibres.

---

## 🎛️ Comment Utiliser l'Outil pour la Calibration

### Étape 1 : Diagnostic Initial

1. **Ouvrir l'outil HTML** dans un navigateur
2. **Vérifier la configuration actuelle** (valeurs par défaut : α=100, β=100, γ=200, δ=20)
3. **Noter les valeurs de référence** :
   - d₀ (distance critique)
   - c₀ (charge critique)
   - Équivalences principales

### Étape 2 : Définir les Objectifs Business

Répondre aux questions stratégiques :

**Sur les coûts :**
- Quel surcoût kilométrique maximum tolérons-nous pour tenir un planning ?
- À partir de quelle distance acceptons-nous de retarder une intervention ?

**Sur la réactivité client :**
- Quel est notre SLA cible (J, J+1, J+3) ?
- Quelle est la pénalité financière d'un jour de retard ?

**Sur les ressources humaines :**
- Quelle surcharge journalière maximum est acceptable ?
- Combien d'heures supplémentaires tolérons-nous par semaine ?

### Étape 3 : Simuler et Ajuster

**Utiliser les calculateurs pour explorer :**

1. **Sensibilités Globales** : Voir la répartition actuelle des contributions
2. **Distance Critique (d₀)** : Ajuster α et γ pour obtenir un d₀ cohérent avec vos zones d'intervention
3. **Charge Critique (c₀)** : Ajuster γ et δ pour respecter vos contraintes RH
4. **Équivalences** : Valider que les arbitrages correspondent à vos priorités métier

**Exemple de calibration :**

**Objectif :** Optimiser les interventions urbaines (< 5 km) tout en tenant les dates pour les interventions > 10 km

**Actions :**
1. Augmenter α de 100 à 150 → d₀ passe de 3.8 km à 4.7 km
2. Garder γ à 200 pour maintenir la priorité date au-delà
3. Augmenter légèrement δ de 20 à 30 → c₀ passe de 2h10 à 1h27 (protection techniciens)

**Résultat :**
- Zone 0-5 km : Optimisation distance active
- Zone 5-10 km : Transition
- Zone > 10 km : Respect strict des dates
- Surcharge max tolérée : 1h30

### Étape 4 : Valider avec les Parties Prenantes

Présenter les résultats des simulations :

**Pour le Commerce :**
- "Avec cette config, 1 jour de retard ≡ 4.7 km → nous privilégions le SLA client"

**Pour les Ops :**
- "Les interventions < 5 km seront optimisées en trajets, > 5 km respecteront les plannings"

**Pour les RH :**
- "Surcharge max tolérée : 1h27 → on protège le bien-être des techniciens"

**Pour la Finance :**
- "Économie estimée de 8% sur les coûts kilométriques en zone urbaine"

### Étape 5 : Déployer et Monitorer

1. **Appliquer les nouveaux poids** dans la configuration TimeFold
2. **Piloter par A/B testing** (20% du trafic pendant 2 semaines)
3. **Suivre les KPIs** :
   - Coûts kilométriques (€/intervention)
   - Taux de respect des dates (%)
   - Charge journalière moyenne (heures)
4. **Ajuster si nécessaire** en revenant à l'outil HTML

---

## 📊 Configurations Recommandées par Stratégie

### Strategy DIGITAL (TT Planif) - Urgence

**Objectif :** Réactivité maximale, SLA courts

**Poids recommandés :**
- α = 80 (distance moins prioritaire)
- β = 80 (distance cumulée moins prioritaire)
- γ = 300 (date absolue)
- δ = 15 (charge secondaire en situation d'urgence)

**Résultats calculés :**
- d₀ ≈ 2.3 km (optimise uniquement les interventions très proches)
- c₀ ≈ 4.3h (accepte forte surcharge pour tenir les délais)
- 1 jour ≡ 2.3 km

**Interprétation :** Système "date first", tient coûte que coûte les SLA urgents.

---

### Strategy SHOP (PRDV) - Équilibre

**Objectif :** Compromis date/coûts/charge

**Poids recommandés :**
- α = 100 (distance importante)
- β = 100 (cumul important)
- γ = 200 (date prioritaire mais pas absolue)
- δ = 20 (charge surveillée)

**Résultats calculés :**
- d₀ ≈ 3.8 km (optimise trajets urbains)
- c₀ ≈ 2.2h (surcharge modérée tolérée)
- 1 jour ≡ 3.8 km

**Interprétation :** Configuration équilibrée pour prises de RDV standard.

---

### Strategy AUTOPLANIF - Optimisation

**Objectif :** Minimiser les coûts kilométriques

**Poids recommandés :**
- α = 150 (distance très importante)
- β = 150 (cumul très important)
- γ = 100 (date flexible)
- δ = 25 (charge à équilibrer)

**Résultats calculés :**
- d₀ ≈ 7.3 km (optimise largement les trajets)
- c₀ ≈ 0.9h (protège les techniciens)
- 1 jour ≡ 7.3 km

**Interprétation :** Système "coût first", optimise agressivement en acceptant des décalages.

---

## 🎓 Formation et Prise en Main

### Public Cible

- **Décideurs** : Comprendre les arbitrages stratégiques
- **Opérationnels** : Ajuster finement selon les retours terrain
- **Analystes** : Simuler des scénarios et préparer des recommandations

### Durée de Formation

- **Découverte** : 30 min (comprendre les concepts de base)
- **Maîtrise** : 2h (savoir calibrer pour un cas d'usage)
- **Expertise** : 1 journée (optimiser toutes les stratégies)

### Support

L'outil est autonome et ne nécessite aucune compétence technique :
- Interface web simple (sliders, graphiques)
- Résultats en temps réel
- Interprétations intégrées

---

## 📌 Points Clés à Retenir

### Les 3 Valeurs Fondamentales

1. **d₀ (Distance critique)** : Le seuil où le système bascule de "optimise distance" à "tient la date"
2. **c₀ (Charge critique)** : Les heures de surcharge tolérées pour ne pas décaler
3. **Équivalence jour-km** : La valeur monétaire d'un jour de retard en termes de distance

### Les Arbitrages Majeurs

- **α vs γ** : Coût kilométrique vs Réactivité client
- **γ vs δ** : SLA vs Bien-être techniciens
- **α vs δ** : Distance vs Charge (varie selon la zone géographique)

### Les Erreurs à Éviter

1. ❌ **Ignorer δ** : Risque de surcharges systématiques
2. ❌ **d₀ trop élevé** : Le système décale trop facilement, insatisfaction client
3. ❌ **c₀ trop élevé** : Burn-out des techniciens
4. ❌ **Calibrer sans valider avec les parties prenantes** : Décalage entre config et attentes métier

### Les Bonnes Pratiques

1. ✅ Commencer par la configuration SHOP (PRDV) : impact direct sur satisfaction client et coûts
2. ✅ Utiliser l'outil HTML **avant** de modifier les paramètres en base
3. ✅ Valider les simulations avec Commerce, Ops, RH et Finance
4. ✅ Déployer progressivement (A/B testing)
5. ✅ Monitorer les KPIs pendant 2-4 semaines après chaque changement
6. ✅ Documenter les raisons de chaque ajustement

---

## 🚀 Prochaines Étapes

### Court Terme (Cette Semaine)

1. ☐ Organiser une session de découverte de l'outil (1h)
2. ☐ Identifier les configurations actuelles des 4 stratégies
3. ☐ Calculer les valeurs de d₀ et c₀ actuelles

### Moyen Terme (Ce Mois)

1. ☐ Atelier de calibration avec les parties prenantes
2. ☐ Définir les configurations cibles par stratégie
3. ☐ Valider les arbitrages avec la direction

### Long Terme (3 Mois)

1. ☐ Déployer les nouvelles configurations (A/B test)
2. ☐ Former les opérationnels à l'usage de l'outil
3. ☐ Mettre en place un processus de révision trimestrielle

---

## 📞 Contact et Support

Pour toute question sur l'utilisation de l'outil ou pour des demandes d'accompagnement à la calibration, contacter :

**Équipe Projet TimeFold**
- Email : [votre-email@domaine.com]
- Documentation complète : `Présentation_Direction_Stratégie_PRDV_TimeFold.md`

---

*Document établi le [Date] - Version 1.0*
*Usage interne uniquement - Confidentiel*
