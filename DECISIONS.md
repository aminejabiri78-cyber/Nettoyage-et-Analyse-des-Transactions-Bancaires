# 📋 DECISIONS.md — Journal des Décisions de Nettoyage

**Projet :** FinanceCore — Analyse des Transactions Bancaires  
**Fichier source :** `bank-transaction.csv`  
**Fichier produit :** `financecore_clean.csv`  
**Auteur :** Data Engineer  
**Date :** 2024  

---

## 1️⃣ IMPORTATION & EXPLORATION

### 1.1 — Normalisation des noms de colonnes
**Décision :** `df.columns = df.columns.str.lower().str.strip()`  
**Justification :** Les noms de colonnes bruts peuvent contenir des majuscules ou des espaces superflus selon l'export CSV. La normalisation en minuscules sans espaces garantit un accès uniforme et sans erreur lors de toutes les opérations ultérieures.  
**Impact :** Aucune perte de données ; amélioration de la robustesse du pipeline.

### 1.2 — Analyse exploratoire initiale
**Décision :** Exécution de `df.head()`, `df.dtypes`, `df.shape`, `df.info()`, `df.describe()`  
**Justification :** Étape obligatoire pour comprendre la structure, les types et les dimensions du dataset avant toute transformation. Permet de détecter d'emblée les anomalies de type ou les colonnes problématiques.  
**Impact :** Aucune modification des données ; sert de baseline documentaire.

### 1.3 — Analyse et visualisation des valeurs manquantes
**Décision :** Calcul du taux de valeurs manquantes par colonne (`isnull().sum() / shape[0] * 100`) avec visualisation en barplot.  
**Justification :** Identifier les colonnes à fort taux de manquants permet de choisir une stratégie adaptée (imputation, suppression, ou tolérance). La visualisation facilite la communication avec les parties prenantes.  
**Impact :** Aucune modification des données ; guide les décisions des étapes suivantes.

### 1.4 — Détection des doublons sur `transaction_id`
**Décision :** Identification des doublons via `df.duplicated(subset=["transaction_id"], keep=False)` puis vérification du nombre de dates distinctes par transaction.  
**Justification :** `transaction_id` est supposé être une clé primaire unique. La vérification du nombre de dates distinctes permet de distinguer les vrais doublons (même enregistrement copié) des cas ambigus (même ID, dates différentes — potentielle erreur de saisie).  
**Impact :** Diagnostic préalable à la suppression.

---

## 2️⃣ NETTOYAGE DES DONNÉES

### 2.1 — Suppression des doublons sur `transaction_id`
**Décision :** `df.drop_duplicates(subset="transaction_id", keep="first", inplace=True)`  
**Justification :** Chaque transaction bancaire doit être unique. On conserve la première occurrence (chronologiquement la plus ancienne selon l'ordre d'import), en supposant qu'elle est la plus fiable. La stratégie `keep="first"` est conservative et reproductible.  
**Alternative considérée :** `keep="last"` — rejeté car l'ordre d'insertion n'est pas garanti cohérent.  
**Impact :** Réduction du nombre de lignes ; garantie d'unicité sur la clé primaire.

### 2.2 — Conversion de `date_transaction` en datetime
**Décision :** `pd.to_datetime(df["date_transaction"], errors="coerce", dayfirst=True)`  
**Justification :** La colonne contient des dates au format français (JJ/MM/AAAA). Le paramètre `dayfirst=True` évite l'inversion jour/mois. `errors="coerce"` transforme les dates invalides en `NaT` plutôt que de lever une exception, permettant une imputation ultérieure.  
**Impact :** Activation des opérations temporelles (extraction année, mois, trimestre, etc.).

### 2.3 — Nettoyage et conversion de `montant`
**Décision :** Conversion en `str` → remplacement de `,` par `.` → conversion en `float`  
**Justification :** Les montants utilisent la virgule comme séparateur décimal (convention française). Pandas ne parse pas ce format nativement en numérique. La conversion préalable en chaîne de caractères permet le remplacement avant parsing.  
**Impact :** Correction silencieuse du format ; les valeurs non convertibles deviennent `NaN` via `errors="coerce"`.

### 2.4 — Nettoyage de `solde_avant` (suppression du suffixe " EUR")
**Décision :** Détection des lignes contenant "EUR", suppression du suffixe, puis conversion en `float`.  
**Justification :** Certaines valeurs du champ contiennent l'unité monétaire en suffixe, rendant la colonne de type `object`. La suppression ciblée du suffixe permet une conversion numérique propre sans affecter les valeurs déjà conformes.  
**Impact :** Homogénéisation du type de la colonne.

### 2.5 — Normalisation de `devise`
**Décision :** `df["devise"].str.upper().str.strip()`  
**Justification :** Les codes de devises (EUR, USD, GBP…) doivent être en majuscules selon la norme ISO 4217. Les espaces parasites peuvent provoquer des regroupements incorrects dans les analyses ultérieures.  
**Impact :** Standardisation ; aucune perte de données.

### 2.6 — Normalisation de `segment_client`
**Décision :** `df["segment_client"].str.capitalize()`  
**Justification :** Les valeurs de segment (ex. "premium", "STANDARD") présentent des casses incohérentes. La capitalisation assure une présentation uniforme et évite les doublons logiques (ex. "premium" ≠ "Premium" dans un `groupby`).  
**Impact :** Correction de la cardinalité ; amélioration de la cohérence des regroupements.

### 2.7 — Nettoyage de `agence`
**Décision :** `df["agence"].str.strip()`  
**Justification :** Les espaces en début/fin de chaîne sont une source classique d'erreurs silencieuses lors des jointures et `groupby`. Le strip est une précaution standard.  
**Impact :** Réduction des faux doublons dans les analyses par agence.

### 2.8 — Imputation des valeurs manquantes
**Décision :**  
- `score_credit_client` → **médiane** (robuste aux valeurs extrêmes)  
- `agence` → **mode** (valeur la plus fréquente)  
- `segment_client` → **mode**  
- `date_transaction` → **mode**  

**Justification :**  
La médiane est préférée à la moyenne pour `score_credit_client` car la distribution des scores de crédit est souvent asymétrique et sensible aux outliers. Le mode est approprié pour les variables catégorielles (`agence`, `segment_client`) car il préserve la répartition naturelle. Le mode de date est une approximation conservative pour les dates manquantes.  
**Alternative considérée :** Suppression des lignes manquantes — rejetée car le taux de manquants ne justifie pas la perte de données.  
**Impact :** Complétude du dataset ; biais potentiel minimal.

### 2.9 — Suppression de la colonne `taux_interet`
**Décision :** `df.drop(columns="taux_interet")`  
**Justification :** La colonne présente un taux de valeurs manquantes trop élevé pour être exploitable (identifié lors de l'exploration), ou est redondante avec d'autres indicateurs. Sa suppression allège le dataset et évite des erreurs dans les modèles futurs.  
**Impact :** Réduction dimensionnelle ; aucune valeur informationnelle perdue.

---

## 3️⃣ DÉTECTION & TRAITEMENT DES VALEURS ABERRANTES

### 3.1 — Détection des outliers sur `montant` (méthode IQR)
**Décision :** Utilisation de la règle IQR : `val_min = Q1 - 1.5×IQR` / `val_max = Q3 + 1.5×IQR`  
**Justification :** La méthode IQR est robuste et non paramétrique (ne suppose pas une distribution normale). Elle est largement adoptée en analyse financière pour détecter les transactions anormalement élevées ou négatives.  
**Alternative considérée :** Z-score (µ ± 3σ) — rejeté car sensible aux distributions asymétriques fréquentes dans les montants financiers.  
**Impact :** Identification des transactions potentiellement frauduleuses ou erronées.

### 3.2 — Détection des outliers sur `score_credit_client` (méthode IQR + règle métier)
**Décision :** Double détection : règle IQR **ET** règle métier (score < 0 ou > 850).  
**Justification :** Les scores de crédit ont des bornes métier absolues (0–850 selon les référentiels standard). La règle IQR détecte les valeurs statistiquement aberrantes dans la distribution observée. Combiner les deux approches assure une couverture complète des anomalies.  
**Impact :** Détection plus précise ; les deux règles sont complémentaires.

### 3.3 — Flag `is_anomalie` (sans suppression)
**Décision :** Création d'une colonne binaire `is_anomalie = anomalie_montant | anomalie_score_credit` sans suppression des lignes.  
**Justification :** En contexte bancaire, les transactions aberrantes sont des signaux à investiguer, pas nécessairement à supprimer. Le flag préserve les données pour l'analyse fraud/risque tout en les isolant pour les modèles analytiques classiques.  
**Impact :** Traçabilité complète ; données intactes pour audit.

---

## 4️⃣ FEATURE ENGINEERING

### 4.1 — Extraction de variables temporelles
**Décision :** Création de `annee`, `mois`, `trimestre`, `semaine-jour` depuis `date_transaction`.  
**Justification :** Les patterns temporels sont déterminants dans l'analyse bancaire (saisonnalité des dépenses, pics en fin de mois, comportements weekend vs semaine). Ces variables permettent des analyses de tendances et l'entraînement de modèles de prédiction temporelle.  
**Remarque :** Le nom `semaine-jour` (avec tiret) peut poser problème lors d'appels d'attributs Python ; renommer en `jour_semaine` serait préférable.

### 4.2 — Vérification du montant en EUR (`montant_eur_verifie`)
**Décision :** `df["montant_eur_verifie"] = df["montant"] / df["taux_change_eur"]` puis comparaison avec `montant_eur`.  
**Justification :** Contrôle de qualité sur la cohérence entre le montant brut, le taux de change et la conversion EUR fournie dans les données source. Permet de détecter des erreurs de conversion ou des taux incohérents.  
**Impact :** Validation métier ; les écarts identifiés dans `comparer` nécessitent une investigation.

### 4.3 — Catégorisation du risque crédit
**Décision :** Création de `categorie_risque` via une fonction seuil sur `score_credit_client` :  
- ≥ 700 → `"Low"` | ≥ 580 → `"Medium"` | < 580 → `"High"`  
**Justification :** Segmentation standard en gestion du risque bancaire. Facilite le reporting réglementaire et le ciblage des actions commerciales ou de recouvrement.  
**Remarque :** `"Meduim"` dans le code original est une faute de frappe — corriger en `"Medium"`.

### 4.4 — Calcul du solde net par client
**Décision :** Agrégation `total_credit` et `total_debit` par `client_id`, puis `solde_net = total_credit - total_debit`, jointure sur le dataframe principal.  
**Justification :** Le solde net est un indicateur synthétique du comportement financier global de chaque client. Utile pour la segmentation, le scoring et la détection d'anomalies comportementales.  
**Impact :** Enrichissement des données ; duplication possible si un client a plusieurs lignes (à traiter dans les analyses agrégées).

### 4.5 — Statistiques comportementales par client
**Décision :** Calcul via `groupby("client_id").agg()` de :  
- `nb_transaction` : nombre de transactions  
- `montant_moyen` : montant moyen par transaction  
- `nb_produit` : diversité des produits utilisés  
**Justification :** Ces KPIs permettent de profiler le comportement transactionnel des clients (clients actifs vs passifs, mono-produit vs multi-produit). Base pour la segmentation RFM (Récence, Fréquence, Montant).  
**Impact :** Enrichissement significatif ; attention à la granularité lors des analyses (ces valeurs sont identiques sur toutes les lignes d'un même client).

### 4.6 — Taux de rejet par agence
**Décision :** `df.groupby("agence")["statut"].transform(lambda x: (x == "Rejete").mean() * 100)`  
**Justification :** Le taux de rejet est un indicateur de performance opérationnelle par agence. Utiliser `transform` (plutôt qu'`agg`) permet de conserver la granularité ligne-à-ligne tout en diffusant la statistique agence sur chaque transaction.  
**Impact :** Enrichissement ; métrique prête pour le reporting et la détection d'agences sous-performantes.

---

## ⚠️ POINTS D'ATTENTION & RECOMMANDATIONS

| # | Observation | Recommandation |
|---|-------------|----------------|
| 1 | `"Meduim"` dans `risque()` | Corriger en `"Medium"` |
| 2 | Colonne `semaine-jour` (tiret) | Renommer en `jour_semaine` |
| 3 | Jointure `solde` et `stats` peuvent multiplier les lignes | Vérifier l'unicité post-merge avec `df.shape` |
| 4 | `montant_eur_verifie` vs `montant_eur` — écarts détectés ? | Documenter les lignes avec écarts |
| 5 | Imputation de `date_transaction` par le mode | Envisager une imputation par interpolation si les dates sont continues |
| 6 | `taux_interet` supprimée sans backup | S'assurer que la colonne est bien archivée dans le fichier source |

---

*Document généré automatiquement à partir du pipeline de nettoyage `bank-transaction.csv → financecore_clean.csv`*
