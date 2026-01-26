# DARTIES - Dossier de Conception AMAO

**Projet :** Système Décisionnel pour la Direction Commerciale  
**Entreprise :** Darties (Darty, Leroy-Merlin, Boulanger)  
**Version :** 1.0 - Conception Technique  
**Date :** 21 janvier 2026  
**Statut :** Dossier de Conception Complet  
**Pages :** 20-25

---

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

## 5. CONCEPTION BACKEND - API REST

### 5.1 Architecture Application

**Technologie :** Java Spring Boot 3.x

**Structure Projet :**
```
darties-backend/
├── src/main/java/com/darties/
│   ├── DartiesApplication.java       # Main Spring Boot
│   ├── controller/
│   │   ├── AuthController.java       # Login/Logout
│   │   ├── IndicateurController.java # CA, TM, NbV
│   │   ├── OngletController.java     # Accueil, Historique, etc
│   │   └── ExportController.java     # PDF, Excel, CSV
│   ├── service/
│   │   ├── AuthService.java          # Authentification
│   │   ├── IndicateurService.java    # Calculs KPI
│   │   ├── FiltreService.java        # Application filtres
│   │   └── ExportService.java        # Génération exports
│   ├── repository/
│   │   ├── IndicateurRepository.java # Requêtes faits
│   │   └── DimensionRepository.java  # Requêtes dimensions
│   ├── model/
│   │   ├── User.java
│   │   ├── Indicateur.java
│   │   └── Filtre.java
│   ├── dto/
│   │   ├── IndicateurDTO.java
│   │   └── UserDTO.java
│   ├── security/
│   │   ├── JwtTokenProvider.java
│   │   └── SecurityConfig.java
│   └── config/
│       └── DataSourceConfig.java
├── pom.xml                           # Maven dépendances
└── application.properties             # Configuration
```

**Dépendances Spring Boot :**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt</artifactId>
</dependency>
<dependency>
    <groupId>com.itextpdf</groupId>
    <artifactId>itextpdf</artifactId>
</dependency>
```

### 5.2 Endpoints REST Principaux

**Base URL :** `https://api.darties.com/api/v1`

**1. AUTHENTIFICATION**

```
POST /auth/login
Body: { "username": "jean.dupont@darties.com", "password": "***" }
Response: { "token": "eyJhbGciOiJIUzI1NiIs...", "profil": "DC", "valideTo": "2026-01-26T14:32:00Z" }
Status: 200 ou 401

GET /auth/profile
Headers: Authorization: Bearer <token>
Response: { "id": 1, "username": "jean.dupont@darties.com", "profil": "DC", "region": null, "magasin": null }
Status: 200
```

**2. INDICATEURS KPI**

```
GET /indicateurs/ca?mois=202601&region=1&magasin=null
Response: { "ca_budget": 50000, "ca_reel": 52000, "ecart_pct": 4.0 }
Status: 200

GET /indicateurs/marge?mois=202601&profil=DC
Response: { "tm_budget": 25.5, "tm_reel": 26.2, "ecart_pct": 2.7 }
Status: 200

GET /indicateurs/ventes?annee=2025,2026&enseigne=Darty
Response: [
  { "mois": "2025-01", "nbv_budget": 1000, "nbv_reel": 950 },
  { "mois": "2025-02", "nbv_budget": 1100, "nbv_reel": 1120 }
]
Status: 200
```

**3. ONGLETS**

```
GET /onglets/accueil?filtres={date:202601,region:1}
Response: {
  "carte": { "points": [...], "indicateur": "ca" },
  "kpi": { "ca": 52000, "tm": 26.2, "nbv": 950 },
  "tableau": [{ "produit": "Hi-fi", "ca": 20000, ... }]
}
Status: 200

GET /onglets/historique?annees=2025,2026&produit=1
Response: {
  "graphique_courbes": [{ "mois": "2025-01", "budget": 50000, "reel": 52000 }],
  "tableau": [...]
}
Status: 200

GET /onglets/detail?mois=202601&region=1
Response: {
  "ecarts": [{ "produit": "Hi-fi", "budget": 20000, "reel": 22000, "ecart_pct": 10 }],
  "poids": [{ "produit": "Hi-fi", "poids_ca": 42 }]
}
Status: 200

GET /onglets/palmares?classement=ca&region=1
Response: [
  { "rang": 1, "magasin": "Paris 15", "ca": 52000, "diff_meilleur": 0 },
  { "rang": 2, "magasin": "Lyon", "ca": 50000, "diff_meilleur": -2000 }
]
Status: 200
```

**4. EXPORTS**

```
GET /exports/pdf?onglet=accueil&filtres={...}
Response: PDF binaire
Content-Type: application/pdf

GET /exports/excel?graphique=historique_ca
Response: Excel binaire
Content-Type: application/vnd.openxmlformats-officedocument.spreadsheetml.sheet

GET /exports/csv?donnees=palmares
Response: CSV texte
Content-Type: text/csv
```

### 5.3 Gestion Authentification et Droits

**Authentification JWT :**

```java
public String generateToken(User user) {
    long now = System.currentTimeMillis();
    long expiryTime = now + (2 * 60 * 60 * 1000); // 2 heures
    
    return Jwts.builder()
        .setSubject(user.getUsername())
        .claim("profil", user.getProfil())
        .claim("region", user.getRegionId())
        .claim("magasin", user.getMagasinId())
        .setIssuedAt(new Date(now))
        .setExpiration(new Date(expiryTime))
        .signWith(SignatureAlgorithm.HS512, secret)
        .compact();
}
```

**Filtrage Données par Profil :**

```java
@Service
public class FiltreService {
    
    public Query appliquerFiltresProfil(User user, Query query) {
        if ("DC".equals(user.getProfil())) {
            // Directeur Commercial : Pas de filtre
            return query;
        } else if ("DR".equals(user.getProfil())) {
            // Directeur Régional : Filtre par région
            return query.where("region_id", "=", user.getRegionId());
        } else if ("RM".equals(user.getProfil())) {
            // Responsable Magasin : Filtre par magasin
            return query.where("magasin_id", "=", user.getMagasinId());
        }
        return query;
    }
}
```

**Table USERS :**

```sql
CREATE TABLE USERS (
    id_user INT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    email VARCHAR(100) UNIQUE,
    profil VARCHAR(20) NOT NULL, -- 'DC', 'DR', 'RM'
    id_region INT REFERENCES DIM_GEOGRAPHIE(id_geographie),
    id_magasin INT REFERENCES DIM_MAGASIN(id_magasin),
    date_creation TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    actif BOOLEAN DEFAULT TRUE
);
```

### 5.4 Calculs Agrégations Clés

**Chiffre d'Affaires Total :**
```sql
SELECT SUM(chiffre_affaires) as ca_total
FROM FAIT_ACTIVITE
WHERE type_valeur = 'Réalisé'
  AND id_temps IN (SELECT id_temps FROM DIM_TEMPS WHERE annee = 2026 AND mois = 01)
  AND id_magasin = ? -- Filtré selon profil
```

**Taux Marge Pondéré :**
```sql
SELECT 
  SUM(chiffre_affaires * taux_marge) / SUM(chiffre_affaires) as tm_pondere
FROM FAIT_ACTIVITE
WHERE type_valeur = 'Réalisé'
  AND id_temps IN (...)
```

**Écart Budget vs Réalisé :**
```sql
SELECT 
  (SUM(CASE WHEN type_valeur = 'Réalisé' THEN chiffre_affaires ELSE 0 END) -
   SUM(CASE WHEN type_valeur = 'Budget' THEN chiffre_affaires ELSE 0 END)) as ecart_abs,
   
  ((SUM(CASE WHEN type_valeur = 'Réalisé' THEN chiffre_affaires ELSE 0 END) -
    SUM(CASE WHEN type_valeur = 'Budget' THEN chiffre_affaires ELSE 0 END)) /
   SUM(CASE WHEN type_valeur = 'Budget' THEN chiffre_affaires ELSE 0 END) * 100) as ecart_pct
FROM FAIT_ACTIVITE
```

---

## 6. CONCEPTION FRONTEND - POWER BI

### 6.1 Choix Power BI au lieu de React

**Justification Décision :**

| Critère | Impact |
|---------|--------|
| **Délai (30 mai 2026)** | Power BI : 3 sem développement ✓ vs React : 8 sem ✗ |
| **Expertise équipe** | Power BI : Apprentissage MIAGE ✓ vs React : Expertise JS requise ✗ |
| **Fonctionnalités requises** | Toutes natives dans Power BI ✓ |
| **Coût développement** | Power BI : Gratuit Desktop ✓ vs React : Serveur Node + frais infra ✗ |
| **RLS (3 profils)** | Power BI : Intégré RLS DAX ✓ vs React : Code custom ✗ |
| **Exports PDF/Excel** | Power BI : Natif ✓ vs React : Bibliothèques tierces ✗ |
| **Maintenance** | Power BI : Microsoft support ✓ vs React : Équipe custom ✗ |

**Résultat :** **Power BI Desktop + Power BI Service choisi** pour maximaliser délai et minimiser risques.

### 6.2 Architecture Power BI

**Composition :**

1. **Power BI Desktop** (Développement local)
   - Connexion à PostgreSQL
   - Création 4 pages rapport
   - Définition DAX pour RLS
   - Tests avec utilisateurs

2. **Power BI Service** (Publication web)
   - Hébergement rapport en cloud
   - Authentification Microsoft Entra ID
   - Application Row-Level Security (RLS)
   - Accès navigateur + mobile

### 6.3 Structure Rapport Power BI

**Onglet 1 : ACCUEIL (Vue Synthétique)**

Visuels :
- Carte géographique (ArcGIS) : 
  - Localisation magasins/régions
  - Indicateur superposé (CA/TM/NbV)
  - Code couleur dynamique
  
- 3 KPI Cards :
  - CA réalisé (gros chiffre)
  - Taux Marge %
  - Nombre Ventes

- Tableau matriciel :
  - Lignes : Produits (Hi-fi, Fours, DVD)
  - Colonnes : Régions (IDF, Nord-Est, etc)
  - Valeurs : CA avec code couleur

- Graphique barres :
  - Comparaison CA par produit

Segments filtres (gauche) :
- Date (mois/année)
- Enseigne (multi-select)
- Produit (multi-select)
- Ville (dropdown)
- Région (multi-select selon droits)
- Bouton réinitialisation

**Onglet 2 : HISTORIQUE (Analyse 2 Ans)**

Visuels :
- Graphique courbes :
  - X : Mois (24 mois)
  - Y : CA / TM / NbV
  - 3 lignes : Budget vs Réalisé vs Estimé

- Graphique barres comparatif :
  - 2025 vs 2026 (par produit)

- Tableau détaillé :
  - Mois | Produit | Budget | Réalisé | Estimé | Écart %

**Onglet 3 : DÉTAIL (Écarts)**

Visuels :
- Tableau écarts :
  - Colonne 1-7 : Produit | Budget | Réalisé | Estimé | Écart Abs | Écart % | Poids %
  - Format conditionnel : Vert (>5%) / Orange (±5%) / Rouge (<-5%)

- Graphique waterfall :
  - Visualisation écarts cumulés par produit

- Graphique donut (poids) :
  - Part CA chaque produit

**Onglet 4 : PALMARÈS (Classements)**

Visuels (3 tableaux côte à côte) :

Tableau 1 - Classement CA :
- Rang | Magasin | CA | Diff au meilleur | Variation vs mois précédent

Tableau 2 - Classement Marge :
- Rang | Magasin | Taux Marge % | Variation

Tableau 3 - Classement Ventes :
- Rang | Magasin | Nb Ventes | Variation

### 6.4 Implémentation RLS (Row-Level Security)

**Étape 1 : Créer Colonne Profil dans Data**

```sql
-- Dans PostgreSQL, créer table d'accès
CREATE TABLE ROLE_ACCESS (
    id INT PRIMARY KEY,
    username VARCHAR(100),
    profil VARCHAR(20), -- 'DC', 'DR', 'RM'
    region VARCHAR(50),
    magasin INT
);
```

**Étape 2 : Créer Mesures DAX dans Power BI**

```dax
-- Mesure pour Directeur Commercial (pas de filtre)
RoleEval_DC = 
VAR UserRole = USERPRINCIPALNAME()
RETURN IF(
    CALCULATETABLE(
        FILTER(ROLE_ACCESS, [username] = UserRole AND [profil] = "DC")
    ) > 0,
    TRUE(),
    FALSE()
)

-- Mesure pour Directeur Régional (filtre région)
RoleEval_DR =
VAR UserRole = USERPRINCIPALNAME()
RETURN IF(
    CALCULATETABLE(
        FILTER(ROLE_ACCESS, [username] = UserRole AND [profil] = "DR")
    ) > 0,
    [Region] = VALUES(ROLE_ACCESS[Region]),
    FALSE()
)

-- Mesure pour Responsable Magasin (filtre magasin)
RoleEval_RM =
VAR UserRole = USERPRINCIPALNAME()
RETURN IF(
    CALCULATETABLE(
        FILTER(ROLE_ACCESS, [username] = UserRole AND [profil] = "RM")
    ) > 0,
    [Magasin] = VALUES(ROLE_ACCESS[Magasin]),
    FALSE()
)
```

**Étape 3 : Appliquer RLS dans Power BI Service**

1. Accéder à **Paramètres > Sécurité**
2. Créer 3 rôles :
   - Rôle "National" (pas de filtre)
   - Rôle "Regional" (filtre par région)
   - Rôle "Magasin" (filtre par magasin)

3. Ajouter règles DAX à chaque rôle
4. Assigner utilisateurs aux rôles dans Power BI Service

**Étape 4 : Authentification Microsoft Entra ID**

Power BI intègre automatiquement Azure AD :
- Utilisateurs authentifiés via email professionnel
- Assignation automatique rôle basée groupe AD
- Token valide 24h (configurable)

### 6.5 Comportement Filtres et Recalcul

**Mécanisme Power BI :**

1. Utilisateur change filtre (ex: Région = "Île-de-France")
2. Événement déclenché dans segment
3. **TOUS les visuels actualisent automatiquement** :
   - Carte met à jour couleurs
   - KPI recalculent
   - Tableaux filtrent
   - Graphiques re-dessinent

4. **Bouton Réinitialisation** :
   - Réinitialise TOUS les segments
   - Recharge PAGE complète
   - Retrouve état initial exact

**Performance :** Actualisation <500 ms (cache Power BI)

---

## 7. SÉCURITÉ ET AUTHENTIFICATION

### 7.1 Architecture Sécurité

**Niveaux de Sécurité :**

1. **Transport** : HTTPS/TLS (certificat SSL)
2. **Authentification** : JWT tokens (2h validity)
3. **Autorisation** : RLS Power BI + Filtrage backend
4. **Audit** : Logs toutes requêtes utilisateur
5. **Données** : Chiffrement colonnes sensibles (optionnel)

### 7.2 Flux Authentification

```
Utilisateur
    ↓ Saisit credentials (email + mot de passe)
    ↓
[Power BI Service] ← Intègre Microsoft Entra ID
    ↓ Authentification réussie
    ↓ Assigne Rôle RLS (DC/DR/RM)
    ↓
[Accès rapport Power BI]
    ↓ Données filtrées par RLS
    ↓
[Visualisations personnalisées par profil]
```

### 7.3 Gestion des Utilisateurs

**Table USERS (PostgreSQL) :**

```sql
CREATE TABLE USERS (
    id_user INT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(100) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    profil VARCHAR(20) NOT NULL,
    id_region INT REFERENCES DIM_GEOGRAPHIE(id_geographie),
    id_magasin INT REFERENCES DIM_MAGASIN(id_magasin),
    actif BOOLEAN DEFAULT TRUE,
    date_creation TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_login TIMESTAMP
);

-- Exemples :
INSERT INTO USERS VALUES (1, 'jean.dupont@darties.com', 'Jean Dupont', 'DC', NULL, NULL, TRUE, ...);
INSERT INTO USERS VALUES (2, 'marie.martin@darties.com', 'Marie Martin', 'DR', 1, NULL, TRUE, ...);
INSERT INTO USERS VALUES (3, 'pierre.bernard@darties.com', 'Pierre Bernard', 'RM', 1, 15, TRUE, ...);
```

### 7.4 Logs d'Audit

**Table AUDIT_LOG :**

```sql
CREATE TABLE AUDIT_LOG (
    id_audit BIGINT PRIMARY KEY AUTO_INCREMENT,
    id_user INT REFERENCES USERS(id_user),
    action VARCHAR(100), -- 'LOGIN', 'VIEW_REPORT', 'EXPORT_PDF', etc
    ressource VARCHAR(255), -- 'ACCUEIL', 'PALMARES', etc
    filtres JSON, -- Filtres appliqués
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    ip_address VARCHAR(15),
    user_agent TEXT
);
```

**Chaque action loggée :**
- Qui (id_user)
- Quoi (action)
- Où (ressource)
- Quand (timestamp)
- D'où (IP)
- Comment (navigateur)

---

## 8. PERFORMANCE ET OPTIMISATIONS

### 8.1 Cibles de Performance

| Métrique | Cible | Justification |
|----------|-------|--------------|
| Chargement page | <3s | UX acceptable |
| Requête simple (1 magasin) | <500ms | Filtres réactifs |
| Agrégation régionale | <1s | Données volumineuses |
| Full scan (toutes données) | <5s | Cas extrême |
| Export PDF | <10s | Génération serveur |

### 8.2 Optimisations Base Données

**Index PostgreSQL :**

```sql
-- Performance filtres et jointures
CREATE INDEX idx_fait_temps ON FAIT_ACTIVITE(id_temps);
CREATE INDEX idx_fait_magasin ON FAIT_ACTIVITE(id_magasin);
CREATE INDEX idx_fait_type ON FAIT_ACTIVITE(type_valeur);

-- Recherche combinée
CREATE INDEX idx_fait_multi ON FAIT_ACTIVITE(id_temps, id_magasin, id_enseigne, id_produit);

-- Unicité
CREATE UNIQUE INDEX idx_fact_unique ON FAIT_ACTIVITE(id_temps, id_magasin, id_enseigne, id_produit, type_valeur);
```

**Requêtes Optimisées :**

```sql
-- ✅ BON : Utilise index
SELECT SUM(chiffre_affaires)
FROM FAIT_ACTIVITE
WHERE id_temps = 394 AND id_magasin = 15;
-- ~100 ms

-- ❌ MAUVAIS : Full scan
SELECT SUM(chiffre_affaires)
FROM FAIT_ACTIVITE
WHERE EXTRACT(MONTH FROM date_chargement) = 1;
-- ~5s (lent !)
```

**Stratégie Cache :**

```java
@Cacheable(value = "ca_cache", key = "#magasainId")
public BigDecimal getChiffresAffaires(int magasinId) {
    return repository.findCASumByMagasin(magasinId);
}

// Cache invalidé à chaque chargement mensuel :
@CacheEvict(value = "ca_cache", allEntries = true)
public void loadMonthlyData() { ... }
```

### 8.3 Optimisations Power BI

- **DirectQuery :** Connexion temps réel PostgreSQL (vs Import lent)
- **Aggregations :** Pré-calculs pour performance
- **Row-level Security :** Impacte minimalement performance
- **Compression :** Fichiers Power BI compressés <20 MB

---

## 9. PLAN D'IMPLÉMENTATION

### 9.1 Roadmap par Sprints

**Sprint 1 (28 jan - 10 fév) : Fondations BD et ETL**
- [ ] Installation PostgreSQL
- [ ] Scripts SQL création BD complète
- [ ] Setup Talend Open Studio
- [ ] Workflow Talend INIT
- [ ] Workflow Talend BUDGET
- **Livrable :** BD fonctionnelle + workflows testés

**Sprint 2 (11 fév - 24 fév) : ETL Complet**
- [ ] Workflow Talend HISTORIQUE (2025)
- [ ] Workflow Talend MENSUEL
- [ ] Workflow Talend RÉFÉRENTIEL
- [ ] Tests qualité données
- [ ] Rapport anomalies automatisé
- **Livrable :** ETL production-ready

**Sprint 3 (25 fév - 10 mars) : Backend API**
- [ ] Setup Spring Boot projet
- [ ] Endpoints authentification + JWT
- [ ] Endpoints indicateurs (CA, TM, NbV)
- [ ] Filtrage données par profil
- [ ] Tests unitaires API
- **Livrable :** API REST testée

**Sprint 4 (11 mars - 24 mars) : Power BI Basique**
- [ ] Setup Power BI Desktop
- [ ] Connexion PostgreSQL
- [ ] Page Accueil (carte + KPI)
- [ ] Page Historique (graphiques)
- [ ] Segments filtres
- **Livrable :** Pages basiques fonctionnelles

**Sprint 5 (25 mars - 7 avril) : Power BI Avancé**
- [ ] Page Détail (écarts + waterfall)
- [ ] Page Palmarès (classements)
- [ ] Implémentation RLS (3 rôles)
- [ ] Exports PDF/Excel/CSV
- [ ] Tests RLS avec 3 utilisateurs
- **Livrable :** Rapport Power BI 100% fonctionnel

**Sprint 6 (8 - 20 avril) : Finalisations + Tests**
- [ ] Tests recette complets
- [ ] Documentation utilisateur (3 guides)
- [ ] Manuel administration
- [ ] Formation utilisateurs pilote
- [ ] Corrections bugs
- **Livrable :** Application production-ready

**Sprint 7 (21 - 30 mai) : Déploiement + Présentation**
- [ ] Déploiement Power BI Service
- [ ] Configuration authentification Azure AD
- [ ] Préparation présentation finale
- [ ] Démonstration en présentiel
- [ ] Bilan projet et pédagogique
- **Livrable :** Application en production + présentation

### 9.2 Jalons Clés

| Date | Milestone | Critères Acceptation |
|------|-----------|---------------------|
| 21 janvier | Dossier Conception validé | ✓ Document 20-25 pages |
| 28 février | ETL en production | ✓ Workflows testés, données propres |
| 31 mars | Application complète | ✓ 4 onglets + RLS + exports |
| 14 avril | Cahier recette signé | ✓ 100 cas de test passés |
| 30 mai | Présentation finale | ✓ Démo + bilan devant jury |

---

## 10. TESTS ET RECETTE

### 10.1 Stratégie Tests

**Niveaux :**

1. **Tests Unitaires** (Développeurs)
   - Backend : JUnit classes service/controller
   - Coverage cible : >80%

2. **Tests Intégration** (QA)
   - Talend : Chargements avec données test
   - API : Postman/Rest Assured
   - Power BI : RLS avec 3 utilisateurs

3. **Tests Acceptation** (Utilisateurs)
   - Scénarios métier réels
   - Vérification exigences fonctionnelles
   - Performance sous charge

### 10.2 Plan de Recette

**Cas de test prioritaires :**

| N° | Cas de Test | Étapes | Résultat Attendu |
|----|-------------|--------|------------------|
| TC01 | Chargement INIT BD | Exécuter workflow INIT | BD créée + dimensions chargées |
| TC02 | Chargement Budget 2026 | Uploader 2026_BUDGET.xlsx | 12K lignes insérées |
| TC03 | Chargement Historique | Uploader 2025_HISTO.xlsx | 24K lignes (Budget+Réalisé) |
| TC04 | Login Directeur Commercial | Saisir credentials + JWT | Connexion réussie, token obtenu |
| TC05 | Accès Accueil (DC) | Login DC, voir Accueil | Vue nationale (toutes régions) |
| TC06 | Accès Accueil (DR) | Login DR IDF, voir Accueil | Vue IDF uniquement (RLS actif) |
| TC07 | Accès Accueil (RM) | Login RM Paris, voir Accueil | Vue magasin Paris uniquement |
| TC08 | Filtre Date | Changer mois → janvier 2026 | Tous visuels actualisent |
| TC09 | Filtre Produit | Multi-select Hi-fi + Fours | Palmarès affiche 2 produits |
| TC10 | Export PDF Accueil | Clic bouton PDF | Fichier PDF généré |
| TC11 | Chargement Janvier 2026 | Uploader Janvier_2026.xlsx | Données insérées, alertes anomalies |
| TC12 | Performance CA | Requête CA (300K lignes) | Réponse <2 secondes |
| TC13 | RLS Filtre Région | DR Nord-Est requête national | Seule Nord-Est visible |
| TC14 | Réinitialisation Filtres | Clic bouton Reset | Vue initiale exacte restaurée |
| TC15 | Cohérence Carte-Tableau | Afficher Accueil | Valeurs carte = valeurs tableau |

**Acceptation :** ≥95% TC passés avant déploiement

---

## 11. GLOSSAIRE TECHNIQUE

- **API REST** : Interface web pour communication client-serveur
- **Azure AD** : Annuaire Microsoft (authentification cloud)
- **BI** : Business Intelligence
- **DAX** : Data Analysis Expression (langage Power BI)
- **DW** : Data Warehouse (entrepôt données)
- **ETL** : Extract, Transform, Load
- **JWT** : JSON Web Token (authentification sans session)
- **RLS** : Row-Level Security (filtrage données par utilisateur)
- **Spring Boot** : Framework Java développement rapide
- **Star Schema** : Modèle données (étoile) pour analytics

---

## 12. RÉFÉRENCES

[1] Darties - Présentation Projet Darties, 09/12/2025

[23] DATA PLUS - Dossier Fonctionnel AMAO, 2026

---

**Document Rédigé Par :** Équipe MIAGE  
**Date :** 21 janvier 2026  
**Version :** 1.0 - Finale  
**Statut :** Prêt Développement  
**Pages :** 24

**Approbation :**

Directeur Commercial : ________________  
Directeur de Projet : ________________  
Date : ________________

*Ce dossier de conception constitue la spécification technique complète et détaillée du projet AMAO Darties. Il servira de base pour la phase de développement démarrant le 28 janvier 2026.*