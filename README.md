# standardisation-base-donnees-pharmacie-sql
# 🧹 Standardisation d’une Base de Données Sale — Pharmacie

> **Projet SQL | Data Analyst Freelance**

-----

## 🔍 Le Problème Métier (Situation)

Une pharmacie locale gérait ses ventes sur un vieux logiciel de caisse sans aucune règle de saisie imposée aux employés. Après 2 ans d’utilisation, la base de données était **totalement inexploitable** :

> *“On ne pouvait même pas savoir quel médicament se vendait le mieux. Le même produit apparaissait sous 5 noms différents dans nos rapports.”*

- ❌ **Noms incohérents** : “doliprane”, “Doliprane”, “DOLIPRANE”, “  doliprane  “ → impossible de grouper
- ❌ **3 formats de dates mélangés** : DD/MM/YYYY, MM-DD-YYYY, YYYY-MM-DD → aucun tri possible
- ❌ **Doublons** : des ventes saisies deux fois par erreur → chiffre d’affaires gonflé artificiellement
- ❌ **Prix manquants (NULL)** : 20% des lignes bloquaient tous les calculs de marge

-----

## ✅ Ma Solution SQL (Action)

J’ai conçu un pipeline de nettoyage complet en SQL pur, structuré en 3 phases :

**Phase 1 — Diagnostic** : photographier et mesurer chaque problème avant de toucher aux données.

**Phase 2 — Nettoyage** : corriger les 4 problèmes sans jamais modifier la table source originale.

**Phase 3 — Vérification** : prouver chiffres à l’appui que le nettoyage est complet, puis lancer les analyses métier.

|#|Transformation                                         |Technique SQL utilisée                       |
|-|-------------------------------------------------------|---------------------------------------------|
|1|Standardisation des noms de médicaments                |`TRIM()` + `UPPER()` + `LOWER()` + `SUBSTR()`|
|2|Conversion de tous les formats de dates en YYYY-MM-DD  |`CASE WHEN` + `LIKE` + `SUBSTR()`            |
|3|Suppression des doublons                               |`GROUP BY` + `MIN(vente_id)`                 |
|4|Remplacement des prix NULL par la moyenne du médicament|`COALESCE()` + `AVG()` + table temporaire    |
|5|Rapport avant/après chiffré                            |`UNION ALL` sur les deux tables              |
|6|Analyses CA par médicament, vendeur, mois              |`SUM` + `strftime` + calcul de ratio         |

-----

## 📊 Résultats Concrets (Résultat)

- ✅ **50 lignes brutes** → **44 lignes propres** (6 doublons supprimés)
- ✅ **0 valeur NULL** restante sur les prix (100% des lignes exploitables)
- ✅ **10 noms standardisés** : une seule forme par médicament
- ✅ **100% des dates** au format YYYY-MM-DD (compatible tout outil BI)
- ✅ Calcul du CA par médicament désormais possible en < 1 seconde

-----

## 🗂️ Structure du Projet

```
standardisation-base-pharmacie-sql/
│
├── nettoyage_pharmacie.sql     ← Script principal (diagnostic + nettoyage + analyses)
└── README.md                   ← Ce fichier
```

-----

## 🧱 Architecture du Pipeline

```
┌─────────────────┐     DIAGNOSTIC      ┌──────────────────────┐
│  ventes_brutes  │ ──────────────────► │  Mesure des problèmes │
│  (50 lignes)    │                     │  (4 requêtes)         │
│  table source   │                     └──────────────────────┘
│  NON MODIFIÉE   │
└────────┬────────┘
         │  NETTOYAGE
         │  TRIM + UPPER + LOWER
         │  CASE WHEN (dates)
         │  GROUP BY (doublons)
         │  COALESCE (NULL)
         ▼
┌─────────────────┐     ANALYSES        ┌──────────────────────┐
│  ventes_propres │ ──────────────────► │  CA par médicament   │
│  (44 lignes)    │                     │  Perf. par vendeur   │
│  table cible    │                     │  Évolution mensuelle │
└─────────────────┘                     └──────────────────────┘
         │
         ▼
┌─────────────────┐
│  prix_moyens    │ ← Table temporaire pour remplacer les NULL
│  (TEMP TABLE)   │
└─────────────────┘
```

-----

## 💡 Les 4 Techniques Clés

**1. Standardiser les noms avec TRIM + UPPER/LOWER**

```sql
-- Résultat : "  DOLIPRANE  " → "Doliprane"
UPPER(SUBSTR(TRIM(medicament), 1, 1)) ||
LOWER(SUBSTR(TRIM(medicament), 2))
-- TRIM()   : supprime les espaces avant/après
-- UPPER()  : 1ère lettre en majuscule
-- LOWER()  : reste en minuscule
-- SUBSTR() : découpe la chaîne caractère par caractère
-- ||       : opérateur de concaténation en SQL
```

**2. Détecter et convertir les formats de dates**

```sql
-- LIKE '__/__/____' détecte le pattern DD/MM/YYYY
-- On extrait ensuite les parties et on réordonne
CASE
    WHEN date_vente LIKE '__/__/____'   -- DD/MM/YYYY
    THEN SUBSTR(date_vente, 7, 4) || '-' || SUBSTR(date_vente, 4, 2) || '-' || SUBSTR(date_vente, 1, 2)
    WHEN date_vente LIKE '__-__-____'   -- MM-DD-YYYY
    THEN SUBSTR(date_vente, 7, 4) || '-' || SUBSTR(date_vente, 1, 2) || '-' || SUBSTR(date_vente, 4, 2)
    ELSE date_vente                     -- YYYY-MM-DD : déjà correct
END
```

**3. Supprimer les doublons avec GROUP BY**

```sql
-- On garde une seule ligne par combinaison unique
-- MIN(vente_id) : on conserve la première occurrence
GROUP BY medicament, date_vente, quantite, vendeur
-- Deux lignes identiques sur ces critères = doublon supprimé
```

**4. Remplacer les NULL par la moyenne**

```sql
-- COALESCE retourne le premier argument non-NULL
COALESCE(prix_unitaire, prix_moyen)
-- Si prix_unitaire = 5.30  → on garde 5.30
-- Si prix_unitaire = NULL  → on prend la moyenne du médicament
-- Bien meilleur que mettre 0 qui fausserait les calculs 

-----

## 🚧 Défis Rencontrés & Apprentissages

**Défi 1 — Ne jamais modifier la table source**

> Mon premier réflexe était de faire des `UPDATE` directement sur `ventes_brutes`. Mauvaise idée : si une correction est fausse, les données originales sont perdues. J’ai adopté la bonne pratique : créer une table `ventes_propres` séparée et ne jamais toucher à la source. Le client peut auditer et comparer à tout moment.

**Défi 2 — Les faux doublons sur les dates**

> En groupant pour supprimer les doublons, j’avais des lignes identiques qui n’étaient pas détectées parce que “15/01/2024” et “2024-01-15” représentent la même date mais sont deux chaînes différentes. La solution : appliquer la conversion de date DANS le GROUP BY, pas seulement dans le SELECT.

**Défi 3 — COALESCE vs valeur fixe pour les NULL**

> J’ai d’abord remplacé les NULL par 0, ce qui donnait un CA aberrant sur certains médicaments. La bonne approche : calculer la moyenne réelle du médicament sur les lignes connues, stocker ça dans une table temporaire, et joindre au moment de l’insertion. `AVG()` ignore automatiquement les NULL — ce qui simplifie tout.

**Défi 4 — Prouver le nettoyage avec UNION ALL**

> Un nettoyage sans rapport avant/après n’est pas livrable. J’ai utilisé `UNION ALL` pour afficher les métriques clés (nb lignes, nb NULL, nb médicaments distincts) sur les deux tables côte à côte. C’est ce tableau que je montre au client pour valider la mission.

-----

## À Propos

**Data Analyst spécialisé dans l’optimisation et l’automatisation des données métier.**

Je transforme vos données brutes en insights actionnables via SQL, Python et des outils de visualisation.



-----

*Made with SQL & ☕*
