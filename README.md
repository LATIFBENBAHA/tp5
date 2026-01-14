# TP5: Budgétisation basée sur l'activité (T2A & BBZ)

## Contexte
Ce jeu de données représente l'activité réalisée en N-1 par l'hôpital.
L'objectif du TP est de construire le budget N+1 en s'appuyant sur :
1. L'analyse de l'activité historique (Volumes & Recettes T2A).
2. La projection des volumes pour N+1 (Prévision).
3. Le calcul des charges nécessaires (BBZ - Budget Base Zéro) via les profils de ressources.

## Description des fichiers

### 1. patient_ghm.csv (Le Réel)
Liste exhaustive des séjours N-1 enrichie avec le GHM (Groupe Homogène de Malades).
- `ghm_code`: Code du groupe (ex: GHM-CH-01).
- `tarif_eur`: Recette réelle facturée (incluant sévérité).
- `los_days`: Durée de séjour réelle.
- `or_hours`: Durée de bloc opératoire consommée (heures).

### 2. ghm_catalog.csv (Le Référentiel Financier)
Liste des GHM, tarifs de base socles, et pôles responsables.
Utilisez ce fichier pour comprendre la hiérarchie et les tarifs théoriques.

### 3. ghm_resource_profile.csv (Le Référentiel Coûts/Ressources)
Profil standard de consommation de ressources pour 1 séjour de chaque GHM.
Utilisé pour le calcul de charges BBZ (ex: 1 séjour GHM-CH-01 consomme 7 jours de lit et 4h de bloc).
Colonnes clés :
- `ward_days`: Journées d'hospitalisation standard (pour calibrer les effectifs soignants).
- `or_hours`: Heures de bloc standard.

### 4. activity_by_pole_ghm_Nminus1.csv (L'Agrégat)
Synthèse pré-calculée de l'activité N-1 par Pôle et GHM.
Base de travail pour vos projections N+1.

## Hypothèses de Génération
- Les GHM ont été attribués selon une logique médicale simplifiée (Chirurgie vs Médecine, Score ASA, Urgences).
- Les tarifs incluent des majorations pour sévérité (ASA élevé, Implants).
- Les dates aberrantes ont été corrigées (Entrée = Sortie si négatif).
- Les pôles regroupent les services logiques (ex: SRV-CHIR-* -> Pôle Chirurgie).

4. **Organisation de l'Atelier (4 Équipes)** :
   
   Répartissez-vous en 4 équipes. Chaque équipe doit simuler 3 scénarios spécifiques à son périmètre et proposer un arbitrage :

   - **Équipe A : Pôle Chirurgie**
     - **S1 (Innovation Robotique)** : Recrutement d'un chirurgien expert (+50 séjours `GHM-CH-04`). Impact : hausse recettes mais surcoût matériel.
     - **S2 (Virage Ambulatoire)** : Transfert volontariste de 30% des `GHM-CH-02` (Standard) vers `GHM-CH-03` (Ambulatoire).
     - **S3 (Pénurie RH)** : Pénurie d'IBODE contraignant à fermer 1 salle de bloc (-15% de capacité globale). Quels GHM prioriser ?

   - **Équipe B : Pôle Médecine**
     - **S1 (Crise Hivernale)** : Épidémie sévère (+20% de `GHM-MED-01` sur 3 mois). Impact sur le besoin en lits ?
     - **S2 (Efficience DMS)** : Objectif Direction de réduire la DMS de 10% sur les séjours lourds (`GHM-MED-02`) pour libérer de la capacité.
     - **S3 (Hôpital de Jour)** : Transformation de 10% de `GHM-MED-01` en hospitalisation de jour (tarif et coûts réduits).

   - **Équipe C : Pôle Urgences & Anesthésie**
     - **S1 (Filtrage Maison Médicale)** : Baisse de 15% des `GHM-URG-02` (consultations simples) grâce à un partenariat ville.
     - **S2 (Afflux Massif)** : Hausse non programmée de +10% des passages. Impact sur le besoin d'ouverture de lits portes (GHM-URG-01).
     - **S3 (Optimisation Bloc)** : Réduction du temps de surveillance SSPI (-10% heures) grâce à un nouveau protocole anesthésique.

   - **Équipe D : Direction & Stratégie**
     - **S1 (Investissement)** : L'équipe A demande un robot (Coût fixe +500k€). Est-ce rentable avec leur S1 ?
     - **S2 (Austérité ARS)** : L'ARS demande 2% d'économie globale sur les charges de structure. Répartir l'effort entre les pôles.
     - **S3 (Fusion-Acquisition)** : Scénario de fusion avec une clinique voisine apportant +300 `GHM-CH-03` et +100 `GHM-MED-01`.

   **Mise en commun** : Chaque équipe présente son "Scénario Cible" (celui qu'elle préconise) et la Direction arbitre pour construire le budget final consolidé.

