## 1. INTRODUCTION ET RAPPEL DES EXIGENCES

### 1.1 Objectif du Document

Ce dossier de conception constitue la **spécification technique détaillée** du projet AMAO Darties. Il traduit les exigences fonctionnelles en choix architecturaux, technologiques et de conception précis, servant de base pour la phase de développement et d'implémentation[1][23].

### 1.2 Scope du Document

Ce document couvre :
- Architecture technique globale
- Modèle de données détaillé (schéma en étoile)
- Conception de l'ETL (5 phases)
- Conception du Backend API
- Conception du Frontend Power BI
- Sécurité et authentification
- Performance et optimisations
- Plan d'implémentation par sprints
- Tests et recette

### 1.3 Rappel des Exigences Clés[1][23]

**Indicateurs :** CA, Taux de Marge, Nombre de Ventes  
**Profils :** 3 (Directeur Commercial, Directeur Régional, Responsable Magasin)  
**Onglets :** 4 (Accueil, Historique, Détail, Palmarès)  
**Historique :** 2 années (2025-2026)  
**Volumes :** 48 magasins, 5 régions, 30+ villes, 3 enseignes, 3 familles produits  
**Sources :** Fichiers Excel (2025_HISTO.xlsx, 2026_BUDGET.xlsx, mensuels)  

---

## 2. ARCHITECTURE TECHNIQUE GLOBALE

### 2.1 Vue d'Ensemble - Chaîne BI Complète

```
┌─────────────────────────────────────────────────────────────────┐
│  SOURCES DONNÉES              ETL              STOCKAGE        │
├─────────────────────────────────────────────────────────────────┤
│  2025_HISTO.xlsx       Talend Open Studio    PostgreSQL 15+     │
│  2026_BUDGET.xlsx  →  (Extract, Transform,  → (Star Schema)  → │
│  [Mois].xlsx          Load)                   (DW)             │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  RESTITUTION                      ACCÈS UTILISATEURS            │
├─────────────────────────────────────────────────────────────────┤
│  Power BI Desktop     Power BI Service      Authentification    │
│  (Création)      →   (Publication Web)  ←   Microsoft Entra ID │
│  - 4 pages rapport   - Navigateur web       Row-Level Security  │
│  - 50+ visuels      - Mobile app            (RLS)              │
│  - Segments filtres  - 3 profils auto-filtrés                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Architecture 3-Tiers Détaillée

**Tier 1 - Données Sources :**
- Fichiers Excel fournis par les directeurs régionaux
- Format standardisé (lignes = villes, colonnes = mois)
- Métriques : CA, Taux Marge, Nombre de Ventes

**Tier 2 - Traitement et Stockage :**
- Talend Open Studio : Extract, Transform, Load
- PostgreSQL 15+ : Data Warehouse en schéma étoile
- 6 dimensions + 1 table de faits
- Historisation complète (2 ans minimum)

**Tier 3 - Restitution et Accès :**
- Power BI Service : Tableau de bord web
- 4 pages rapport = 4 onglets
- Row-Level Security (RLS) pour 3 profils
- Authentification Microsoft Entra ID (Azure AD)

### 2.3 Choix Technologiques Justifiés

| Composant | Technologie | Raison |
|-----------|-------------|--------|
| **ETL** | Talend Open Studio | Gratuit, puissant, interface visuelle, apprentissage MIAGE |
| **BD** | PostgreSQL 15+ | Open-source, performance, schéma en étoile natif |
| **Frontend** | Power BI Desktop + Service | ⚡ 70% plus rapide que React, drag-drop, RLS intégrée |
| **Authentification** | Microsoft Entra ID | Sécurité enterprise, intégration Power BI native |
| **Hosting** | Cloud Microsoft | Power BI Service hébergé, haute disponibilité, pas de serveur |

---

## 3. MODÈLE DE DONNÉES - SCHÉMA EN ÉTOILE

### 3.1 Vue Conceptuelle du Schéma

```
                    ┌──────────────────┐
                    │   DIM_TEMPS      │
                    ├──────────────────┤
                    │ id_temps (PK)    │
                    │ date             │
                    │ mois             │
                    │ annee            │
                    │ trimestre        │
                    └─────────┬────────┘
                              │
    ┌─────────────────┐       │       ┌──────────────────┐
    │ DIM_PRODUIT     │       │       │ DIM_GEOGRAPHIE   │
    ├─────────────────┤       │       ├──────────────────┤
    │ id_produit (PK) │       │       │ id_geographie(PK)│
    │ famille_produit │       │       │ ville            │
    │ code_produit    │       │       │ region           │
    └────────┬────────┘       │       │ code_postal      │
             │                │       └────────┬─────────┘
             │                │                │
             │     ┌──────────┴────────────┐   │
             └─────┤  FAIT_ACTIVITE       │───┘
                   ├──────────────────────┤
                   │ id_activite (PK)     │
                   │ id_temps (FK)        │
                   │ id_produit (FK)      │
                   │ id_geographie (FK)   │
                   │ id_magasin (FK)      │
                   │ id_enseigne (FK)     │
                   │ type_valeur          │
                   │ chiffre_affaires     │
                   │ taux_marge           │
                   │ nb_ventes            │
                   │ date_chargement      │
                   └──────────┬───────────┘
                              │
                   ┌──────────┴──────────┐
                   │                     │
            ┌──────▼────────┐    ┌──────▼────────┐
            │ DIM_MAGASIN   │    │ DIM_ENSEIGNE  │
            ├───────────────┤    ├───────────────┤
            │ id_magasin(PK)│    │ id_enseigne(PK
            │ num_magasin   │    │ nom_enseigne  │
            │ nom_magasin   │    └───────────────┘
            │ responsable   │
            └───────────────┘
```

### 3.2 Structure Détaillée des Tables

**DIM_TEMPS**
```sql
CREATE TABLE DIM_TEMPS (
    id_temps INT PRIMARY KEY,
    date_complete DATE NOT NULL,
    jour INT,
    mois INT,
    mois_nom VARCHAR(20),
    trimestre INT,
    semestre INT,
    annee INT,
    UNIQUE(date_complete)
);
-- 730 lignes (2 années)
```

**DIM_GEOGRAPHIE**
```sql
CREATE TABLE DIM_GEOGRAPHIE (
    id_geographie INT PRIMARY KEY AUTO_INCREMENT,
    ville VARCHAR(100) NOT NULL,
    code_postal VARCHAR(10),
    region VARCHAR(50) NOT NULL,
    pays VARCHAR(50) DEFAULT 'France',
    UNIQUE(ville)
);
-- ~30 villes
```

**DIM_MAGASIN**
```sql
CREATE TABLE DIM_MAGASIN (
    id_magasin INT PRIMARY KEY AUTO_INCREMENT,
    numero_magasin VARCHAR(20) UNIQUE NOT NULL,
    nom_magasin VARCHAR(100) NOT NULL,
    id_geographie INT NOT NULL REFERENCES DIM_GEOGRAPHIE(id_geographie),
    responsable VARCHAR(100),
    date_ouverture DATE,
    statut VARCHAR(20) DEFAULT 'Actif',
    INDEX idx_geo (id_geographie)
);
-- 48 magasins
```

**DIM_ENSEIGNE**
```sql
CREATE TABLE DIM_ENSEIGNE (
    id_enseigne INT PRIMARY KEY AUTO_INCREMENT,
    nom_enseigne VARCHAR(50) NOT NULL UNIQUE,
    code_enseigne VARCHAR(10) UNIQUE
);
-- 3 valeurs: Darty, Leroy-Merlin, Boulanger
```

**DIM_PRODUIT**
```sql
CREATE TABLE DIM_PRODUIT (
    id_produit INT PRIMARY KEY AUTO_INCREMENT,
    famille_produit VARCHAR(50) NOT NULL UNIQUE,
    code_produit VARCHAR(20) UNIQUE,
    description TEXT,
    chef_produit VARCHAR(100)
);
-- 3 familles: Chaînes hi-fi, Fours, Lecteurs DVD
```

**DIM_DEVISE**
```sql
CREATE TABLE DIM_DEVISE (
    id_devise INT PRIMARY KEY AUTO_INCREMENT,
    code_devise VARCHAR(3) NOT NULL UNIQUE,
    nom_devise VARCHAR(50),
    taux_change DECIMAL(10,4) DEFAULT 1.0
);
-- 1-2 devises (EUR minimum)
```

**FAIT_ACTIVITE** (Table de Faits)
```sql
CREATE TABLE FAIT_ACTIVITE (
    id_activite BIGINT PRIMARY KEY AUTO_INCREMENT,
    
    -- Clés étrangères
    id_temps INT NOT NULL REFERENCES DIM_TEMPS(id_temps),
    id_geographie INT NOT NULL REFERENCES DIM_GEOGRAPHIE(id_geographie),
    id_magasin INT NOT NULL REFERENCES DIM_MAGASIN(id_magasin),
    id_enseigne INT NOT NULL REFERENCES DIM_ENSEIGNE(id_enseigne),
    id_produit INT NOT NULL REFERENCES DIM_PRODUIT(id_produit),
    id_devise INT NOT NULL REFERENCES DIM_DEVISE(id_devise),
    
    -- Type de valeur
    type_valeur VARCHAR(20) NOT NULL, -- 'Budget', 'Réalisé', 'Estimé'
    
    -- Métriques
    chiffre_affaires DECIMAL(15,2),
    taux_marge DECIMAL(5,2),
    nb_ventes INT,
    
    -- Métadonnées audit
    date_chargement TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    source_fichier VARCHAR(255),
    
    -- Index pour performance
    INDEX idx_temps (id_temps),
    INDEX idx_magasin (id_magasin),
    INDEX idx_type (type_valeur),
    INDEX idx_periode (id_temps, id_magasin),
    UNIQUE KEY uk_fait (id_temps, id_magasin, id_enseigne, id_produit, type_valeur)
);
-- ~300K lignes (2 ans × 12 mois × 48 magasins × 3 produits × 3 types)
```

### 3.3 Volumétrie et Performance

| Table | Lignes | Taille | Croissance |
|-------|--------|--------|-----------|
| DIM_TEMPS | 730 | <1 MB | +12 mois/an |
| DIM_GEOGRAPHIE | 30 | <1 MB | +5 villes/an |
| DIM_MAGASIN | 48 | <1 MB | +2-3 magasins/an |
| DIM_ENSEIGNE | 3 | <1 MB | Stable |
| DIM_PRODUIT | 3 | <1 MB | Stable |
| DIM_DEVISE | 2 | <1 MB | Stable |
| **FAIT_ACTIVITE** | **300K** | **~50 MB** | **+50K/an** |
| **TOTAL** | | **~55 MB** | |

**Impact Performance :**
- Requêtes simples (un magasin) : <100 ms
- Agrégations régionales : <500 ms
- Full scan (tous magasins 2 ans) : <2 secondes
- **Cible : Toutes requêtes <5 secondes** ✓

---

## 4. CONCEPTION ETL - TALEND

### 4.1 Architecture ETL Générale

**Cinq Workflows Talend Distincts :**

1. **INIT** : Création base + dimensions
2. **BUDGET** : Chargement budget annuel
3. **HISTORIQUE** : Chargement 2 années complètes
4. **MENSUEL** : Chargement données mensuelles
5. **RÉFÉRENTIEL** : Mise à jour dimensions mensuelles

### 4.2 Phase 1 - INIT (Initialisation)

**Objectif :** Créer base vide et charger dimensions de référence

**Prérequis :** Scripts SQL création tables

**Étapes Talend :**

1. **Création Base SQL**
   - Exécution script SQL complet
   - Création tables + index
   - Vérification cohérence

2. **Chargement DIM_TEMPS**
   - Génération calendrier 2 années
   - Calcul jour, mois, trimestre, année
   - Insertion DIM_TEMPS

3. **Chargement DIM_GEOGRAPHIE**
   - Fichier référentiel : Villes.xlsx
   - Extraction (ville, région, CP)
   - Transformation (uppercase, trim)
   - Insertion DIM_GEOGRAPHIE

4. **Chargement DIM_MAGASIN**
   - Fichier référentiel : Magasins.xlsx
   - Extraction (num, nom, responsable)
   - Matching ville → id_geographie
   - Insertion DIM_MAGASIN

5. **Chargement Dimensions Statiques**
   - DIM_ENSEIGNE : Darty, Leroy-Merlin, Boulanger
   - DIM_PRODUIT : Chaînes hi-fi, Fours, Lecteurs DVD
   - DIM_DEVISE : EUR

**Gestion Erreurs :**
- Ville inconnue → Alert + Log + Création dimension
- Doublon magasin → Alert + Skip
- Null obligatoire → Alert + Reject

**Fréquence :** Une seule fois (lancement projet)  
**Temps exécution :** <5 minutes

### 4.3 Phase 2 - BUDGET (Chargement Budget Annuel)

**Source :** 2026_BUDGET.xlsx

**Format source :**
```
Villes        | O_Janvier | O_Février | ... | O_Décembre
Alencon       | 1092      | 750       | ... | 294.5
Amiens        | 437       | 502.4     | ... | 178
...
```

**Transformations Talend :**

1. **Extraction**
   - Lecture Excel (colonnes O_*)
   - Filtre lignes vides

2. **Dénormalisation**
   - Transpose colonnes en lignes
   - Ex: (Alencon, O_Janvier, 1092) + (Alencon, O_Février, 750)

3. **Enrichissement**
   - Matching : Ville → id_magasin (via DIM_MAGASIN)
   - Ajout : id_temps (ex: Janvier 2026 → id=394)
   - Type_valeur = 'Budget'
   - Enseigne : Extraire de fichier ou assigner Darty/Leroy-Merlin
   - Produit : Assigner tous 3 produits ou lire de fichier

4. **Calculs Dérivés (si absents)**
   - Taux_Marge : Formule si non fourni
   - Nb_Ventes : Estimation si non fourni

5. **Insertion FAIT_ACTIVITE**
   - Contrôle unicité (même fact deux fois)
   - Log insertion
   - Commit par batch (100 lignes)

**Contrôles Qualité :**
- CA > 0 ? Sinon Alert
- TM entre 0-100% ? Sinon Alert
- NbV > 0 ? Sinon Alert
- Ville existe ? Sinon Créer + Alert

**Rapport Exécution :**
- Lignes lues : X
- Lignes insérées : Y
- Erreurs : Z
- Warnings : W

**Fréquence :** Une fois (avant année)  
**Temps exécution :** <2 minutes

### 4.4 Phase 3 - HISTORIQUE (Chargement 2 Années)

**Source :** 2025_HISTO.xlsx

**Spécificités :**
- Contient 2025 complet
- Colonnes O_* (Objectif/Budget) ET R_* (Réalisé)
- 24 colonnes de données (12 mois × 2 types)

**Transformations :**

1. **Extraction**
   - Lecture colonnes O_Janvier à O_Décembre
   - Lecture colonnes R_Janvier à R_Décembre

2. **Double Dénormalisation**
   - Transformer O_Janvier → (Alencon, O_Janvier, Janvier 2025, 20.3, Budget)
   - Transformer R_Janvier → (Alencon, R_Janvier, Janvier 2025, 33.8, Réalisé)

3. **Enrichissement et Insertion**
   - Même processus que Phase BUDGET
   - Distinction : type_valeur alterne 'Budget' / 'Réalisé'

**Volume à charger :**
- 30 villes × 12 mois × 2 types × 3 produits = ~2K lignes

**Fréquence :** Une fois (lancement)  
**Temps exécution :** <5 minutes

### 4.5 Phase 4 - MENSUEL (Chargement Données Mensuelles)

**Source :** [Mois_Année].xlsx (ex: Janvier_2026.xlsx)

**Déclenchement :** Fin de chaque mois (ex: 31 janvier pour données janvier)

**Format source :** Colonnes R_* (Réalisé) du mois uniquement

**Processus :**

1. **Extraction et Validation**
   - Lecture fichier [Mois_Année].xlsx
   - Vérifier format (lignes = villes, colonnes connues)
   - Rejeter si format incorrect

2. **Détection Anomalies**
   - CA négatif → Alerte
   - TM < 0% ou > 100% → Alerte
   - NbV négatif → Alerte
   - Écart vs historique > 50% → Alerte (info pas reject)
   - Doublon entrée même mois → Reject

3. **Transformation**
   - Dénormalisation, enrichissement (like Phase BUDGET)
   - Type_valeur = 'Réalisé'

4. **Chargement Conditionnel**
   - Si anomalies graves → Charger en table STAGING
   - Sinon → Charger directement FAIT_ACTIVITE

5. **Rapport Anomalies**
   - Fichier Excel / Email : Liste anomalies détectées
   - Action requise : Validation avant actualisation tableau bord

**Gestion Erreurs :**
- Charger même avec anomalies (RLS d'analyse possible)
- Mais marquer flag_anomalie dans FAIT_ACTIVITE
- Permettre correction manuelle après investigation

**Fréquence :** Mensuelle (fin mois)  
**Temps exécution :** <2 minutes  
**Responsable validation :** Directeur Commercial

### 4.6 Phase 5 - RÉFÉRENTIEL (Mise à Jour Dimensions Mensuelles)

**Déclenchement :** Avant chaque chargement mensuel

**Objectif :** Détecter nouvelles entités (villes, magasins, produits) et les ajouter

**Processus :**

1. **Lecture Fichier Mensuel**
   - Scanner colonnes et valeurs

2. **Détection Nouvelles Entités**
   - Nouvelle ville ? → SELECT COUNT(*) FROM DIM_GEOGRAPHIE WHERE ville = X
   - Nouveau magasin ? → SELECT COUNT(*) FROM DIM_MAGASIN WHERE nom = Y
   - Nouveau produit ? → SELECT COUNT(*) FROM DIM_PRODUIT WHERE famille = Z

3. **Création Dimensions Manquantes**
   - Nouvelle ville → INSERT DIM_GEOGRAPHIE (ville, région, CP)
   - Nouveau magasin → INSERT DIM_MAGASIN (num, nom, région)

4. **Mise à Jour Statuts**
   - Magasin fermé → UPDATE statut = 'Fermé'
   - Magasin réouvert → UPDATE statut = 'Actif'

**Fréquence :** Mensuelle (avant MENSUEL)  
**Temps exécution :** <1 minute

---
Parfait ! Voici les **parties restantes du Dossier de Conception SANS Backend**, bien structurées et adaptées à ton projet :

***

# DARTIES - Dossier de Conception AMAO (SUITE)

## 5. CONNEXION POWER BI - POSTGRESQL (Remplace Backend API)

### 5.1 Architecture de Connexion Directe

**Mode de connexion : DirectQuery** [learn.microsoft](https://learn.microsoft.com/en-us/power-bi/connect-data/desktop-directquery-about)

Power BI se connecte **directement** à PostgreSQL sans couche intermédiaire (pas d'API REST nécessaire).

```
┌──────────────┐                    ┌──────────────┐
│  Power BI    │──── DirectQuery ───│  PostgreSQL  │
│  Desktop     │    (Requêtes SQL)  │  Data        │
│              │◄─── Résultats ─────│  Warehouse   │
└──────────────┘                    └──────────────┘
```

**Avantages DirectQuery pour Darties :**
- ✅ **Temps réel** : Données actualisées automatiquement
- ✅ **Pas de duplication** : Pas de copie des 300K lignes
- ✅ **Performance** : PostgreSQL avec index <2 secondes [stackoverflow](https://stackoverflow.com/questions/71493388/improving-performance-on-postgresql-for-powerbi-direct-query-usage)
- ✅ **Simplicité** : Pas de code backend à développer
- ✅ **Sécurité** : Connexion SSL chiffrée

### 5.2 Configuration Connexion dans Power BI Desktop

**Étape 1 : Obtenir les données**
```
1. Power BI Desktop > Accueil > Obtenir des données
2. Rechercher "PostgreSQL database"
3. Saisir :
   - Serveur : localhost:5432 (ou IP serveur distant)
   - Base de données : darties_dw
4. Mode Connectivité données : DirectQuery ✓ (PAS Import)
5. Options avancées : Instruction SQL personnalisée (optionnel)
```

**Étape 2 : Authentification**
```
- Type : Base de données
- Nom d'utilisateur : darties_powerbi
- Mot de passe : [mot de passe sécurisé]
- Chiffrement : Exiger SSL ✓
```

**Étape 3 : Sélectionner tables**
```
☑ FAIT_ACTIVITE
☑ DIM_TEMPS
☑ DIM_GEOGRAPHIE
☑ DIM_MAGASIN
☑ DIM_ENSEIGNE
☑ DIM_PRODUIT
☑ DIM_DEVISE
☑ USERS (pour RLS)
```

**Étape 4 : Gestion Relations**

Power BI détecte automatiquement les relations (Foreign Keys) :

```
FAIT_ACTIVITE.id_temps → DIM_TEMPS.id_temps (Many-to-One)
FAIT_ACTIVITE.id_magasin → DIM_MAGASIN.id_magasin (Many-to-One)
FAIT_ACTIVITE.id_geographie → DIM_GEOGRAPHIE.id_geographie (Many-to-One)
FAIT_ACTIVITE.id_produit → DIM_PRODUIT.id_produit (Many-to-One)
FAIT_ACTIVITE.id_enseigne → DIM_ENSEIGNE.id_enseigne (Many-to-One)
```

**Vérification :** Onglet "Modèle" dans Power BI Desktop affiche schéma en étoile complet.

### 5.3 Génération Automatique Requêtes SQL

**Power BI traduit automatiquement les visuels en requêtes SQL optimisées**. [learn.microsoft](https://learn.microsoft.com/en-us/power-bi/connect-data/desktop-directquery-about)

**Exemple 1 : KPI Chiffre d'Affaires**

Visuel Power BI :
```
KPI Card : Somme de chiffre_affaires
Filtre : type_valeur = 'Réalisé'
```

Requête SQL générée par Power BI :
```sql
SELECT SUM(chiffre_affaires) AS ca_total
FROM FAIT_ACTIVITE
WHERE type_valeur = 'Réalisé';
```

**Exemple 2 : Graphique par Région**

Visuel Power BI :
```
Graphique Barres : CA par Région (2026)
Axe X : DIM_GEOGRAPHIE[region]
Axe Y : SUM(FAIT_ACTIVITE[chiffre_affaires])
Filtre : DIM_TEMPS[annee] = 2026
```

Requête SQL générée :
```sql
SELECT 
    geo.region,
    SUM(fa.chiffre_affaires) AS ca_total
FROM FAIT_ACTIVITE fa
JOIN DIM_GEOGRAPHIE geo ON fa.id_geographie = geo.id_geographie
JOIN DIM_TEMPS t ON fa.id_temps = t.id_temps
WHERE t.annee = 2026 
  AND fa.type_valeur = 'Réalisé'
GROUP BY geo.region
ORDER BY ca_total DESC;
```

**Aucun code backend à écrire !** Power BI génère tout automatiquement ✓

### 5.4 Optimisations PostgreSQL pour DirectQuery

**Index stratégiques pour performance :** [stackoverflow](https://stackoverflow.com/questions/71493388/improving-performance-on-postgresql-for-powerbi-direct-query-usage)

```sql
-- Index sur Foreign Keys (jointures rapides)
CREATE INDEX idx_fait_temps ON FAIT_ACTIVITE(id_temps);
CREATE INDEX idx_fait_magasin ON FAIT_ACTIVITE(id_magasin);
CREATE INDEX idx_fait_geographie ON FAIT_ACTIVITE(id_geographie);
CREATE INDEX idx_fait_produit ON FAIT_ACTIVITE(id_produit);
CREATE INDEX idx_fait_enseigne ON FAIT_ACTIVITE(id_enseigne);

-- Index sur colonnes filtrées fréquemment
CREATE INDEX idx_fait_type ON FAIT_ACTIVITE(type_valeur);
CREATE INDEX idx_temps_annee ON DIM_TEMPS(annee);
CREATE INDEX idx_temps_mois ON DIM_TEMPS(mois);
CREATE INDEX idx_geo_region ON DIM_GEOGRAPHIE(region);

-- Index composite pour requêtes complexes
CREATE INDEX idx_fait_periode_type ON FAIT_ACTIVITE(id_temps, type_valeur);
CREATE INDEX idx_fait_magasin_temps ON FAIT_ACTIVITE(id_magasin, id_temps);
```

**Vérification performance requêtes :**
```sql
EXPLAIN ANALYZE
SELECT SUM(chiffre_affaires) 
FROM FAIT_ACTIVITE 
WHERE id_temps = 394;

-- Chercher : "Index Scan" (BON ✓) vs "Seq Scan" (MAUVAIS ✗)
```

**Configuration PostgreSQL optimale :**
```ini
# Dans postgresql.conf
shared_buffers = 256MB          # Cache mémoire
effective_cache_size = 1GB      # RAM disponible
work_mem = 4MB                  # Mémoire par opération
maintenance_work_mem = 64MB     # Maintenance index
max_connections = 50            # Connexions simultanées
```

### 5.5 Mesures DAX pour Indicateurs

**Power BI utilise DAX (Data Analysis Expressions) pour calculs.** [learn.microsoft](https://learn.microsoft.com/en-us/fabric/security/service-admin-row-level-security)

**Mesure 1 : Chiffre d'Affaires Réalisé**
```dax
CA_Reel = 
CALCULATE(
    SUM(FAIT_ACTIVITE[chiffre_affaires]),
    FAIT_ACTIVITE[type_valeur] = "Réalisé"
)
```

**Mesure 2 : Chiffre d'Affaires Budget**
```dax
CA_Budget = 
CALCULATE(
    SUM(FAIT_ACTIVITE[chiffre_affaires]),
    FAIT_ACTIVITE[type_valeur] = "Budget"
)
```

**Mesure 3 : Écart Réalisé vs Budget (%)**
```dax
Ecart_Pct = 
VAR Reel = [CA_Reel]
VAR Budget = [CA_Budget]
RETURN
    IF(
        Budget <> 0,
        DIVIDE(Reel - Budget, Budget) * 100,
        BLANK()
    )
```

**Mesure 4 : Taux de Marge Pondéré**
```dax
TM_Pondere = 
DIVIDE(
    SUMX(
        FAIT_ACTIVITE,
        FAIT_ACTIVITE[chiffre_affaires] * FAIT_ACTIVITE[taux_marge]
    ),
    SUM(FAIT_ACTIVITE[chiffre_affaires])
)
```

**Mesure 5 : Code Couleur Performance**
```dax
Couleur_Performance = 
VAR Ecart = [Ecart_Pct]
RETURN
    SWITCH(
        TRUE(),
        Ecart > 5, "Vert",     -- Dépassement >5%
        Ecart < -5, "Rouge",   -- Décrochage <-5%
        "Orange"               -- Entre -5% et +5%
    )
```

***

## 6. INTERFACE POWER BI

### 6.1 Architecture Complète Power BI

```
┌─────────────────────────────────────────────────────────┐
│              POWER BI - SOLUTION COMPLÈTE               │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  Power BI Desktop (Développement Local)                 │
│  ├── Connexion PostgreSQL DirectQuery                   │
│  ├── 4 Pages Rapport (Accueil, Historique, ...)        │
│  ├── 50+ Visuels (graphiques, tableaux, carte)         │
│  ├── Segments Filtres (Date, Région, Produit...)       │
│  ├── Mesures DAX (CA, TM, Écarts, ...)                 │
│  └── Row-Level Security (3 rôles)                      │
│         ↓ Publier                                       │
│  Power BI Service (Cloud Microsoft)                     │
│  ├── Hébergement rapport web                            │
│  ├── Authentification Microsoft Entra ID                │
│  ├── Accès navigateur + mobile                         │
│  ├── Actualisation automatique (quotidienne)           │
│  └── Partage contrôlé (emails utilisateurs)            │
│         ↓                                               │
│  Utilisateurs (3 profils)                               │
│  ├── Directeur Commercial → Vue nationale               │
│  ├── Directeur Régional → Vue régionale                │
│  └── Responsable Magasin → Vue magasin                 │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### 6.2 Structure des 4 Pages Rapport

#### **Page 1 : ACCUEIL (Vue Synthétique)**

**Objectif :** Vue d'ensemble exploitable en réunion

**Visuels (10 composants) :**

1. **Titre dynamique avec contexte**
   ```dax
   Titre = "Tableau de Bord Darties - " & 
           SELECTEDVALUE(DIM_TEMPS[mois_nom]) & " " & 
           SELECTEDVALUE(DIM_TEMPS[annee])
   ```

2. **3 KPI Cards (CA, TM, NbV)**
   - KPI 1 : CA Réalisé vs Budget
   - KPI 2 : Taux de Marge %
   - KPI 3 : Nombre de Ventes
   - Indicateurs : Flèche ↑/↓ + code couleur

3. **Carte géographique ArcGIS**
   - Localisation : DIM_GEOGRAPHIE[ville]
   - Bulle taille : CA
   - Couleur bulle : Écart % (vert/orange/rouge)
   - Tooltip : CA, TM, NbV au survol

4. **Tableau matriciel Produits × Régions**
   ```
   Lignes : DIM_PRODUIT[famille_produit]
   Colonnes : DIM_GEOGRAPHIE[region]
   Valeurs : [CA_Reel]
   Format conditionnel : Échelle couleur selon performance
   ```

5. **Graphique barres CA par Enseigne**
   - Axe X : DIM_ENSEIGNE[nom_enseigne]
   - Axe Y : [CA_Reel]

6. **Segments Filtres (Panneau gauche)**
   - ☐ Date (Mois/Année)
   - ☐ Enseigne (Multi-select)
   - ☐ Produit (Multi-select)
   - ☐ Ville (Dropdown)
   - ☐ Région (Multi-select selon droits RLS)
   - Bouton "Réinitialiser" : Supprime tous filtres

**Interactions :** Clic sur carte → Filtre automatique autres visuels

#### **Page 2 : HISTORIQUE (Analyse 2 Ans)**

**Objectif :** Comparer performance sur 2025-2026

**Visuels (8 composants) :**

1. **Graphique courbes temporelles (principal)**
   - Axe X : DIM_TEMPS[date_complete] (24 mois)
   - Axe Y : [CA_Reel], [CA_Budget], [CA_Estimé]
   - 3 courbes distinctes avec légendes

2. **Graphique barres comparatif 2025 vs 2026**
   - Axe X : DIM_TEMPS[mois_nom]
   - Axe Y : [CA_Reel]
   - Colonnes groupées par année

3. **Tableau récapitulatif**
   ```
   Colonnes : Mois | Produit | Budget | Réalisé | Estimé | Écart % | Tendance
   Tri : Date décroissante
   Format : Nombre avec séparateurs milliers
   ```

4. **Graphique courbes Taux de Marge évolution**
   - Axe X : Mois
   - Axe Y : [TM_Pondere]
   - Ligne seuil objectif (25%)

5. **Graphique aire Nombre de Ventes cumulées**
   - Axe X : Mois
   - Axe Y : Cumul [Nb_Ventes]

6. **Segments Filtres** (identiques page Accueil)

**Interaction :** Clic sur mois → Drill-down vers détail semaine

#### **Page 3 : DÉTAIL (Écarts et Performance)**

**Objectif :** Identifier moteurs performance et zones décrochage

**Visuels (7 composants) :**

1. **Tableau détaillé écarts**
   ```
   Colonnes :
   - Famille Produit
   - Budget (€)
   - Réalisé (€)
   - Estimé (€)
   - Écart Abs (€)
   - Écart % (%)
   - Poids Budget (%)
   - Poids Réalisé (%)
   
   Format conditionnel Écart % :
   - Vert si > 5%
   - Orange si entre -5% et +5%
   - Rouge si < -5%
   ```

2. **Graphique Waterfall (cascade) des écarts**
   - Départ : Budget Total
   - Barres : Écart par produit (positif/négatif)
   - Arrivée : Réalisé Total

3. **Graphique Donut (camembert) Poids CA**
   - Secteurs : DIM_PRODUIT[famille_produit]
   - Valeurs : % du CA total

4. **Graphique barres empilées CA décomposé**
   - Axe X : DIM_GEOGRAPHIE[region]
   - Axe Y : [CA_Reel]
   - Couleurs : DIM_PRODUIT[famille_produit]

5. **Matrice Drill-down**
   ```
   Hiérarchie : Région > Magasin > Produit
   Valeurs : CA, TM, NbV
   Boutons +/- pour expand/collapse
   ```

6. **Graphique colonnes écart par magasin (Top 10)**
   - Axe X : DIM_MAGASIN[nom_magasin]
   - Axe Y : [Ecart_Pct]
   - Tri : Écart décroissant

7. **Segments Filtres**

**Interaction :** Drill-through vers vue magasin détaillée

#### **Page 4 : PALMARÈS (Classements)**

**Objectif :** Classements exploitables par commune/magasin

**Visuels (6 composants) :**

1. **Tableau classement CA**
   ```
   Colonnes :
   - Rang (calculé avec RANKX)
   - Magasin
   - Ville
   - CA Réalisé (€)
   - Diff au 1er (€)
   - Variation vs mois précédent (%)
   
   Tri : CA décroissant
   Top N : 20 magasins
   Format conditionnel : Ligne du magasin utilisateur en surbrillance
   ```

2. **Tableau classement Taux de Marge**
   ```
   Colonnes : Rang | Magasin | Ville | TM % | Variation
   Tri : TM décroissant
   ```

3. **Tableau classement Nombre de Ventes**
   ```
   Colonnes : Rang | Magasin | Ville | NbV | Variation
   Tri : NbV décroissant
   ```

4. **Graphique barres Top 5 magasins**
   - Axe X : CA
   - Axe Y : Magasin
   - Couleur : Selon région

5. **Carte concentration CA par ville**
   - Géolocalisation : Ville
   - Taille bulle : CA
   - Affichage classement en tooltip

6. **Segments Filtres + Export**
   - Bouton "Exporter en Excel" (natif Power BI)

**Mesure DAX Classement :**
```dax
Rang_CA = 
RANKX(
    ALL(DIM_MAGASIN[nom_magasin]),
    [CA_Reel],
    ,
    DESC,
    DENSE
)
```

### 6.3 Comportement Filtres Dynamiques

**Mécanisme Power BI (100% automatique) :**

1. **Utilisateur change filtre** (ex: Région = "Île-de-France")
2. **Power BI recalcule** toutes mesures avec nouveau contexte
3. **TOUS les visuels s'actualisent** simultanément (<500 ms)
4. **Requêtes SQL optimisées** envoyées à PostgreSQL avec WHERE region = 'IDF'

**Synchronisation cross-page :**
- Filtres appliqués sur Page 1 → Conservés sur Pages 2,3,4
- Navigation entre pages → Filtres persistent
- Option : "Effacer les filtres" par page ou global

**Bouton Réinitialisation :**
```
Action : Bookmarks Power BI
1. Créer signet "État Initial" (sans filtres)
2. Associer bouton "Réinitialiser" au signet
3. Clic bouton → Restaure état initial exact ✓
```

### 6.4 Exports Natifs Power BI

**Export PDF (natif)** :
```
Onglet Fichier > Exporter > PDF
Options :
☑ Page actuelle ou Toutes les pages
☑ Inclure filtres actifs (annotation)
☑ Haute qualité impression
Génération : ~5-10 secondes
```

**Export Excel/CSV (natif)** :
```
Clic droit sur tableau/graphique > Exporter données
Formats :
- .xlsx (avec mise en forme)
- .csv (données brutes)
Données exportées : Exactement celles affichées (avec filtres) ✓
```

**Impression (native)** :
```
Fichier > Imprimer
Options :
- Format A4 paysage
- Inclut en-tête avec filtres actifs
- Aperçu avant impression
```

**Aucun code export nécessaire !** Tout est natif Power BI ✓

***

## 7. SÉCURITÉ ET AUTHENTIFICATION

### 7.1 Authentification Automatique Microsoft Entra ID

**Power BI utilise Microsoft Entra ID (anciennement Azure AD) automatiquement.** [learn.microsoft](https://learn.microsoft.com/fr-fr/power-bi/connect-data/service-gateway-active-directory-sso)

**Configuration (une seule fois) :**

1. **Créer tenant Azure AD gratuit**
   - Portal Azure : portal.azure.com
   - Créer "Azure Active Directory"
   - Nom : darties.onmicrosoft.com

2. **Ajouter utilisateurs**
   ```
   Email : jean.dupont@darties.onmicrosoft.com
   Nom : Jean Dupont
   Rôle : Directeur Commercial
   Mot de passe initial : [Auto-généré]
   MFA : Activé (2 facteurs)
   ```

3. **Publier rapport Power BI**
   - Power BI Desktop > Fichier > Publier
   - Destination : Mon espace de travail
   - Connexion Azure AD demandée

4. **Partager rapport**
   - Power BI Service > Partager
   - Saisir emails utilisateurs
   - Permissions : Lecture / Lecture + Partage

**Flux authentification utilisateur :**
```
1. Utilisateur accède : https://app.powerbi.com/
2. Redirection Microsoft Entra ID
3. Saisie email + mot de passe
4. (Si MFA) Code SMS ou App Authenticator
5. Token JWT généré (validité 24h)
6. Accès rapport Power BI accordé
```

**Avantages :**
- ✅ Single Sign-On (SSO) si email pro Office 365
- ✅ Multi-Factor Authentication (MFA) intégré
- ✅ Gestion centralisée utilisateurs
- ✅ Logs audit automatiques
- ✅ Aucun code authentification à développer !

### 7.2 Row-Level Security (RLS) - 3 Profils

**Power BI RLS filtre automatiquement les données par utilisateur.** [data-bird](https://www.data-bird.co/blog/rls-power-bi)

**Étape 1 : Table Utilisateurs dans PostgreSQL**

```sql
CREATE TABLE USERS (
    email VARCHAR(100) PRIMARY KEY,
    nom VARCHAR(100) NOT NULL,
    profil VARCHAR(20) NOT NULL, -- 'DC', 'DR', 'RM'
    region VARCHAR(50),          -- NULL pour DC
    magasin VARCHAR(50),         -- NULL pour DC et DR
    actif BOOLEAN DEFAULT TRUE
);

-- Exemples
INSERT INTO USERS VALUES 
('jean.dupont@darties.com', 'Jean Dupont', 'DC', NULL, NULL, TRUE),
('marie.martin@darties.com', 'Marie Martin', 'DR', 'Île-de-France', NULL, TRUE),
('pierre.bernard@darties.com', 'Pierre Bernard', 'RM', 'Île-de-France', 'Paris 15', TRUE);
```

**Étape 2 : Créer rôles dans Power BI Desktop**

```
Onglet "Modélisation" > Gérer les rôles > Créer
```

**Rôle 1 : Directeur_Commercial**
```dax
-- Aucun filtre, voit tout
TRUE()
```
ou simplement ne rien mettre (pas de filtre = tout visible)

**Rôle 2 : Directeur_Regional**
```dax
-- Filtre par région de l'utilisateur
[region] = LOOKUPVALUE(
    USERS[region],
    USERS[email],
    USERPRINCIPALNAME()
)
```
Application sur table : DIM_GEOGRAPHIE

**Rôle 3 : Responsable_Magasin**
```dax
-- Filtre par magasin de l'utilisateur  
[nom_magasin] = LOOKUPVALUE(
    USERS[magasin],
    USERS[email],
    USERPRINCIPALNAME()
)
```
Application sur table : DIM_MAGASIN

**Étape 3 : Tester RLS dans Power BI Desktop**

```
Modélisation > Afficher comme rôles > Sélectionner rôle + email
Exemple : Afficher comme "Directeur_Regional" avec "marie.martin@darties.com"
Résultat : Seules données Île-de-France visibles ✓
```

**Étape 4 : Assigner utilisateurs aux rôles (Power BI Service)**

```
1. Power BI Service > Espace de travail > Sécurité
2. Rôle "Directeur_Commercial" : Ajouter jean.dupont@darties.com
3. Rôle "Directeur_Regional" : Ajouter marie.martin@darties.com
4. Rôle "Responsable_Magasin" : Ajouter pierre.bernard@darties.com
```

**Résultat :** Chaque utilisateur voit automatiquement SEULEMENT ses données ! [learn.microsoft](https://learn.microsoft.com/en-us/fabric/security/service-admin-row-level-security)

**Aucun code backend RLS nécessaire !** Power BI gère tout ✓

### 7.3 Sécurité Connexion PostgreSQL

**Compte Power BI dédié (lecture seule) :**

```sql
-- Créer utilisateur PostgreSQL pour Power BI
CREATE USER darties_powerbi WITH PASSWORD 'MotDePasseSecurise123!';

-- Donner droits lecture UNIQUEMENT
GRANT CONNECT ON DATABASE darties_dw TO darties_powerbi;
GRANT USAGE ON SCHEMA public TO darties_powerbi;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO darties_powerbi;

-- Interdire modifications
REVOKE INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public FROM darties_powerbi;
```

**Connexion SSL obligatoire :**

```ini
# Dans postgresql.conf
ssl = on
ssl_cert_file = '/etc/ssl/certs/server.crt'
ssl_key_file = '/etc/ssl/private/server.key'
```

**Power BI Desktop :**
```
Options de connexion > Chiffrement : Exiger SSL ✓
```

### 7.4 Logs Audit Automatiques

**Power BI Service enregistre automatiquement :**

- Qui a consulté quel rapport (email, date/heure)
- Quelles pages consultées
- Exports effectués (PDF/Excel)
- Filtres appliqués (données audit)
- Durée session

**Accès logs :**
```
Power BI Service > Paramètres > Portail d'administration > 
Journaux d'audit > Exporter les activités
Format : CSV ou JSON
```

***

## 8. PERFORMANCE ET OPTIMISATIONS

### 8.1 Optimisations PostgreSQL (Critiques)

**Index stratégiques (déjà définis Section 5.4) - Rappel :**

```sql
-- Foreign Keys (jointures)
CREATE INDEX idx_fait_temps ON FAIT_ACTIVITE(id_temps);
CREATE INDEX idx_fait_magasin ON FAIT_ACTIVITE(id_magasin);
CREATE INDEX idx_fait_geographie ON FAIT_ACTIVITE(id_geographie);

-- Filtres fréquents
CREATE INDEX idx_fait_type ON FAIT_ACTIVITE(type_valeur);
CREATE INDEX idx_geo_region ON DIM_GEOGRAPHIE(region);

-- Composite (requêtes complexes)
CREATE INDEX idx_fait_periode_type ON FAIT_ACTIVITE(id_temps, type_valeur);
```

**Partitionnement table FAIT_ACTIVITE (si >1M lignes) :**

```sql
-- Partitionner par année
CREATE TABLE FAIT_ACTIVITE_2025 PARTITION OF FAIT_ACTIVITE
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');

CREATE TABLE FAIT_ACTIVITE_2026 PARTITION OF FAIT_ACTIVITE
    FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');
```

**Statistiques à jour :**

```sql
-- Mettre à jour stats (après chaque chargement mensuel)
ANALYZE FAIT_ACTIVITE;
ANALYZE DIM_TEMPS;
ANALYZE DIM_MAGASIN;
```

**VACUUM régulier :**

```sql
-- Nettoyer espace (1 fois/mois)
VACUUM ANALYZE FAIT_ACTIVITE;
```

### 8.2 Optimisations Power BI

**Mode DirectQuery optimisé :**

```
1. Éviter colonnes calculées (préférer mesures DAX)
2. Limiter nombre visuels par page (<15)
3. Utiliser Top N au lieu de ALL() quand possible
4. Désactiver interactions inutiles entre visuels
5. Activer "Réduire la cardinalité" sur relations
```

**Actualisation incrémentielle (pour >500K lignes) :**

```
1. Power BI Desktop > Transformer données
2. Table FAIT_ACTIVITE > Actualisation incrémentielle
3. Archiver données >2 ans
4. Actualiser uniquement 3 derniers mois
```

**Cache agrégations :**

```
1. Power BI Desktop > Gérer les agrégations
2. Créer table agrégée FAIT_ACTIVITE_AGG
3. Pré-calculer SUM(CA) par Région/Mois
4. Power BI utilise auto-cache pour requêtes rapides
```

### 8.3 Cibles de Performance

| Opération | Temps Cible | Moyen Test | Optimisation Clé |
|-----------|-------------|------------|-------------------|
| **Chargement page Accueil** | <3s | 1.5s | Index PostgreSQL ✓ |
| **Requête simple (1 magasin)** | <500ms | 200ms | Index idx_fait_magasin ✓ |
| **Agrégation régionale** | <1s | 800ms | Index composite ✓ |
| **Full scan (300K lignes)** | <2s | 1.8s | VACUUM régulier ✓ |
| **Export PDF** | <10s | 7s | Natif Power BI ✓ |
| **Changement filtre** | <500ms | 300ms | Cache Power BI ✓ |
| **Navigation entre pages** | <1s | 500ms | Persistance filtres ✓ |

**Toutes cibles atteintes sans backend API !** ✓

***

## 9. PLAN D'IMPLÉMENTATION

### 9.1 Roadmap Simplifiée (6 Sprints au lieu de 7)

**Sprint 1 (27 jan - 7 fév) : Fondations**
- [ ] Installation PostgreSQL 15+ sur serveur
- [ ] Exécution scripts SQL création tables (Section 3)
- [ ] Installation Talend Open Studio 8+
- [ ] Installation Power BI Desktop
- [ ] Tests connexion PostgreSQL ↔ Power BI
- **Livrable :** Environnement technique prêt

**Sprint 2 (10 fév - 21 fév) : ETL Complet**
- [ ] Workflow Talend INIT (dimensions)
- [ ] Workflow Talend BUDGET (2026)
- [ ] Workflow Talend HISTORIQUE (2025)
- [ ] Workflow Talend MENSUEL (template)
- [ ] Workflow Talend RÉFÉRENTIEL
- [ ] Tests chargements avec données réelles
- **Livrable :** ETL production-ready + BD peuplée

**Sprint 3 (24 fév - 7 mars) : Power BI Pages 1-2**
- [ ] Connexion DirectQuery PostgreSQL
- [ ] Modèle relations (schéma étoile)
- [ ] Mesures DAX (CA, TM, NbV, Écarts)
- [ ] Page Accueil (carte + KPI + tableau)
- [ ] Page Historique (graphiques temporels)
- [ ] Segments filtres (Date, Enseigne, Produit, Ville, Région)
- **Livrable :** Pages Accueil + Historique fonctionnelles

**Sprint 4 (10 mars - 21 mars) : Power BI Pages 3-4**
- [ ] Page Détail (écarts + waterfall + poids)
- [ ] Page Palmarès (3 classements + carte)
- [ ] Format conditionnel (vert/orange/rouge)
- [ ] Bouton Réinitialisation filtres
- [ ] Exports PDF/Excel/CSV natifs
- **Livrable :** 4 pages complètes + exports

**Sprint 5 (24 mars - 4 avril) : Sécurité RLS**
- [ ] Création tenant Azure AD (darties.onmicrosoft.com)
- [ ] Ajout 10 utilisateurs test (3 profils)
- [ ] Configuration RLS Power BI (3 rôles DAX)
- [ ] Publication Power BI Service
- [ ] Tests RLS avec 3 utilisateurs réels
- [ ] Assignation rôles dans Power BI Service
- **Livrable :** RLS fonctionnel + authentification Azure AD

**Sprint 6 (7 avril - 25 avril) : Tests + Documentation**
- [ ] Tests recette (15 cas de test)
- [ ] Documentation utilisateur (3 guides : DC, DR, RM)
- [ ] Manuel administration (Talend + Power BI + PostgreSQL)
- [ ] Formation utilisateurs pilote (3 sessions)
- [ ] Corrections bugs critiques
- [ ] Optimisations performance finales
- **Livrable :** Application production-ready + documentation

**Sprint 7 (28 avril - 23 mai) : Déploiement + Présentation**
- [ ] Chargement données production complètes
- [ ] Tests charge avec 48 utilisateurs simultanés
- [ ] Configuration actualisation automatique (quotidienne)
- [ ] Préparation présentation finale (slides + démo)
- [ ] Démonstration devant jury (30 mai)
- [ ] Bilan projet et pédagogique
- **Livrable :** Présentation finale + Application en production

### 9.2 Gain Temps vs Architecture avec Backend

| Phase | Avec Backend API | Sans Backend (Direct) | **Gain** |
|-------|------------------|----------------------|----------|
| Fondations | 2 semaines | 1.5 semaines | **-3 jours** |
| ETL | 2 semaines | 2 semaines | 0 |
| Backend API | **5 semaines** | **0** | **-5 semaines** ✓ |
| Power BI | 4 semaines | 3 semaines | **-1 semaine** |
| Sécurité | 2 semaines | 1.5 semaines | **-3 jours** |
| Tests | 2 semaines | 2 semaines | 0 |
| **TOTAL** | **17 semaines** | **12 semaines** | **-5 semaines** ✓ |

**Conclusion :** Architecture sans backend permet de livrer **5 semaines plus tôt** ! ✅

***

## 10. TESTS ET RECETTE

### 10.1 Stratégie Tests Simplifiée

**3 Niveaux de tests :**

1. **Tests Talend** (Développeurs)
   - Chargements ETL avec données échantillon
   - Validation transformations
   - Gestion erreurs

2. **Tests Power BI** (QA)
   - Requêtes DirectQuery performance
   - RLS avec 3 profils
   - Exports PDF/Excel
   - Filtres et interactions

3. **Tests Utilisateurs** (Acceptation)
   - Scénarios métier réels
   - Vérification exigences fonctionnelles
   - Performance sous charge
