# SPÉCIFICATION TECHNIQUE — PROJET SAM AUTO
## Système de Gestion Automatisée des Alertes de Surveillance (SAM)
### Direction Opérations, Crimes Financiers (DOCF) — Desjardins

---

## 1. RÉSUMÉ EXÉCUTIF

| Attribut | Valeur |
|---|---|
| **Nom du projet** | SAM Auto — Automatisation intelligente du traitement des alertes TUA |
| **Portée** | Réseau des Caisses Desjardins |
| **Objectif principal** | Réduire de ~23 % le volume d'alertes SAM nécessitant un traitement manuel, sans dégrader la performance de la règle TUA |
| **Mécanisme** | Modèle d'analytique avancée segmentant les alertes selon la probabilité qu'elles s'avèrent non concluantes |
| **Méthodes d'automatisation** | Fermeture automatique (seuil S1) et mise en hibernation (seuil S2) |
| **Périmètre initial** | Alertes SAM générées par la règle TUA (Transferts entre comptes non liés) |

---

## 2. CONTEXTE ET JUSTIFICATION

### 2.1 Problématique
La Direction Opérations, Crimes Financiers (DOCF) traite un volume croissant d'alertes de surveillance générées par la règle TUA. Une analyse rétrospective révèle qu'une proportion significative de ces alertes s'avère non concluante après investigation manuelle complète, représentant une charge opérationnelle sans valeur ajoutée en matière de détection de crimes financiers.

### 2.2 Opportunité
L'application d'un modèle d'analytique avancée permet de prédire, au moment de la génération de l'alerte, la probabilité qu'elle soit non concluante. Cette prédiction autorise une segmentation en trois strates de risque, chacune associée à un traitement différencié.

### 2.3 Contraintes
- **Conformité** : Toute automatisation doit préserver ou améliorer la performance de détection de la règle TUA.
- **Auditabilité** : Chaque décision automatisée doit être traçable, justifiable et réversible dans un délai réglementaire.
- **Réversibilité** : Le système doit permettre un retour au processus manuel (statu quo) sans interruption de service.

---

## 3. OBJECTIFS ET INDICATEURS DE SUCCÈS (KPI)

### 3.1 Objectifs quantitatifs
| ID | Objectif | Cible | Mesure |
|---|---|---|---|
| OQ-01 | Réduction du volume d'alertes à traiter manuellement | ~23 % | Volume mensuel d'alertes statu quo vs. volume post-déploiement |
| OQ-02 | Réduction par fermeture automatique | ~10 % | Volume d'alertes fermées automatiquement / volume total |
| OQ-03 | Réduction par hibernation | ~13 % | Volume d'alertes mises en hibernation / volume total |
| OQ-04 | Maintien de la performance de la règle TUA | 100 % | Taux de détection des alertes concluantes (pas de fuite de vrais positifs) |

### 3.2 Objectifs qualitatifs
| ID | Objectif | Mesure |
|---|---|---|
| OQ-05 | Amélioration de la productivité des analystes DOCF | Temps moyen de traitement par alerte concluante |
| OQ-06 | Traçabilité complète des décisions automatisées | Taux de couverture des logs d'audit |
| OQ-07 | Réduction du temps de cycle (alerte → résolution) | Délai médian de clôture des alertes |

---

## 4. PÉRIMÈTRE FONCTIONNEL

### 4.1 Dans le périmètre (In-Scope)
- Alertes SAM générées par la règle de surveillance TUA (transferts de fonds entre comptes non liés).
- Développement et entraînement d'un modèle d'analytique avancée (machine learning supervisé).
- Définition et calibration de deux seuils de segmentation : S1 (fermeture auto) et S2 (hibernation).
- Moteur de décision appliquant les actions automatiques aux alertes segmentées.
- Module de gestion du statut « Hibernation » (cycle de 6 mois, conditions de réveil).
- Interface de suivi et de rapport pour les superviseurs DOCF.
- Journal d'audit immuable de toutes les décisions automatisées.

### 4.2 Hors périmètre (Out-of-Scope)
- Modification de la logique de déclenchement de la règle TUA elle-même.
- Automatisation des alertes autres que TUA (phases ultérieures).
- Traitement des alertes concluantes (maintien du processus d'investigation existant).
- Intégration avec des systèmes externes au réseau des Caisses.

---

## 5. EXIGENCES MÉTIERS

### 5.1 Segmentation des alertes
Le modèle doit produire un score de probabilité [0, 1] pour chaque alerte TUA, représentant la probabilité que l'alerte soit non concluante.

**Segmentation :**

| Segment | Condition | Action | Justification |
|---|---|---|---|
| **A — Risque modéré à élevé** | Score < S2 | Statu quo | L'alerte suit le processus d'analyse manuel existant. |
| **B — Risque faible** | S1 ≤ Score < S2 | Mise en hibernation | Forte probabilité de non-conclusion ; différé de 6 mois. |
| **C — Risque quasi nul** | Score ≥ S1 | Fermeture automatique | Très forte probabilité de non-conclusion ; clôture définitive. |

### 5.2 Règles métier — Hibernation
**Déclenchement :** Une alerte passe au statut « En hibernation » dès que son score est dans l'intervalle [S1, S2[.

**Durée :** 6 mois calendaires à compter de la date de mise en hibernation.

**Conditions de réveil (réactivation en statu quo) :**
Au cours de la période de 6 mois, si l'un des événements suivants survient pour le même membre/client ou sur le même compte :
1. Une autre alerte sur le même compte passe au statut « À revoir ».
2. Ouverture d'un dossier lié au membre/client.
3. Augmentation du niveau de risque du membre/client.
4. Création d'une alerte SOI (Signalement d'Opération Inhabituelle) sur le même compte.

**Expiration :** À l'issue des 6 mois sans condition de réveil, l'alerte est automatiquement fermée.

### 5.3 Règles métier — Fermeture automatique
**Déclenchement :** Une alerte est fermée automatiquement dès que son score ≥ S1.

**Caractère définitif :** Une alerte fermée automatiquement ne sera jamais traitée par un analyste. La fermeture est irréversible par les utilisateurs métier.

**Exception technique :** En cas d'erreur système avérée (bug, corruption de données), une procédure de réouverture d'urgence peut être déclenchée par le Comité de Gouvernance sur justification documentée.

### 5.4 Calibration des seuils S1 et S2
Les seuils sont déterminés par :
- **Analyse ROC/PR** sur l'ensemble de test du modèle.
- **Contrainte métier :** Le taux de faux négatifs (alertes concluantes classées dans les segments B ou C) doit être inférieur à un seuil critique défini par la DOCF.
- **Validation par le Comité de Gouvernance** avant mise en production.

---

## 6. EXIGENCES TECHNIQUES

### 6.1 Architecture cible

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SOURCE DE DONNÉES                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐   │
│  │   SAM DB    │  │   Core      │  │   KYC/CDD   │  │   Historique    │   │
│  │  (Alertes)  │  │  Banking    │  │   (Client)  │  │   Alertes       │   │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  │   (Résolutions) │   │
│         └─────────────────┴─────────────────┘        └─────────────────┘   │
│                              │                                              │
│                              ▼                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                    COUCHE D'INTÉGRATION (ETL/ELT)                  │    │
│  │  • Extraction quotidienne des alertes TUA en attente               │    │
│  │  • Enrichissement avec données client, transactionnelles et KYC      │    │
│  │  • Feature engineering selon spécifications du modèle                │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                              │                                              │
│                              ▼                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │              MOTEUR DE SCORING (Modèle d'Analytique Avancée)       │    │
│  │  • Versionnage du modèle (MLflow ou équivalent)                    │    │
│  │  • Inférence batch ou near-real-time                               │    │
│  │  • Sortie : score [0,1] + features utilisées (SHAP/log)            │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                              │                                              │
│                              ▼                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │              MOTEUR DE DÉCISION (Règles métier)                    │    │
│  │  • Application des seuils S1 et S2                                 │    │
│  │  • Application des règles d'hibernation et de fermeture            │    │
│  │  • Gestion des conditions de réveil                                │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                              │                                              │
│              ┌───────────────┼───────────────┐                              │
│              ▼               ▼               ▼                              │
│  ┌─────────────────┐ ┌──────────────┐ ┌─────────────────┐                   │
│  │   STATU QUO     │ │ HIBERNATION  │ │ FERMETURE AUTO  │                   │
│  │ (Workflow DOCF) │ │   Engine     │ │   Engine        │                   │
│  └─────────────────┘ └──────────────┘ └─────────────────┘                   │
│                              │                                              │
│                              ▼                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                    COUCHE DE PERSISTANCE                             │    │
│  │  • Mise à jour du statut dans SAM                                   │    │
│  │  • Journal d'audit immuable                                         │    │
│  │  • Tables de suivi des alertes en hibernation                       │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                              │                                              │
│                              ▼                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                    TABLEAUX DE BORD & RAPPORTS                      │    │
│  │  • Suivi des volumes par segment                                    │    │
│  │  • Taux de réveil des alertes en hibernation                        │    │
│  │  • Performance post-production (Annexe 2)                          │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Exigences du modèle

**Type de modèle :** Modèle supervisé de classification binaire (concluant / non concluant).

**Variables d'entrée (features) :**
- Variables transactionnelles : montant, fréquence, direction (débit/crédit), contrepartie.
- Variables client : ancienneté, segment, profil de risque, historique d'alertes.
- Variables temporelles : heure, jour de la semaine, proximité avec des événements.
- Variables de la règle TUA : déclencheurs spécifiques, seuils dépassés.
- Variables dérivées : ratio par rapport à l'historique, écarts-types, comportement habituel.

**Performance minimale :**
- AUC-ROC ≥ 0,85 sur l'ensemble de test.
- Taux de faux négatifs (alertes concluantes prédites comme non concluantes) < 0,5 %.
- Stabilité du PSI (Population Stability Index) < 0,1 entre entraînement et production.

**Explicabilité :**
- Chaque décision automatisée doit être accompagnée d'un indicateur d'importance des features (SHAP ou équivalent).
- Les features doivent être documentées dans l'Annexe 1 du projet.

### 6.3 Exigences de l'hibernation engine
- **Planificateur (Scheduler)** : Vérification quotidienne des alertes en hibernation arrivant à échéance.
- **Event Listener** : Surveillance en temps réel (ou near-real-time) des événements de réveil.
- **Gestion d'état** : Machine à états (Hibernation → Réveil → Statu quo / Hibernation → Fermeture).

### 6.4 Exigences de sécurité et d'audit
- **Authentification** : Intégration avec l'annuaire d'entreprise (SSO).
- **Autorisation** : RBAC (Role-Based Access Control) — seuls les administrateurs système peuvent modifier les seuils S1/S2.
- **Journal d'audit** :
  - Horodatage (UTC) de chaque décision.
  - ID de l'alerte, score, features, version du modèle, seuils appliqués.
  - Utilisateur ou processus ayant déclenché l'action.
  - Conservation : 7 ans (conformité AMF/ACPR).
- **Immutabilité** : Les logs d'audit ne peuvent ni être modifiés ni supprimés.

---

## 7. EXIGENCES DE DONNÉES

### 7.1 Sources de données
| Source | Description | Fréquence |
|---|---|---|
| SAM (Système d'Alertes et de Mesures) | Alertes TUA générées | Temps réel / Batch quotidien |
| Core Banking | Données transactionnelles, comptes | Batch quotidien |
| KYC/CDD | Profils de risque client, PEP, sanctions | Batch quotidien |
| Historique SAM | Résolutions passées des alertes (entraînement) | Snapshot initial + incrémental |

### 7.2 Qualité des données
- Taux de complétude des features critiques ≥ 99 %.
- Traitement des valeurs manquantes : imputation documentée ou exclusion avec justification.
- Détection de dérive des données (Data Drift) : alerte si la distribution des features en production diverge significativement de l'entraînement.

---

## 8. PLAN DE MISE EN ŒUVRE

### Phase 1 : Fondation (Mois 1–2)
- Constitution de l'équipe projet (Data Scientists, Ingénieurs ML, Analystes DOCF, Conformité).
- Inventaire des données et accès.
- Extraction de l'historique des alertes TUA et de leurs résolutions (minimum 2 ans).
- Ateliers de définition des seuils S1 et S2 avec la DOCF et la conformité.

### Phase 2 : Modélisation (Mois 2–4)
- Feature engineering et sélection.
- Entraînement et validation du modèle (cross-validation temporelle obligatoire).
- Analyse des faux négatifs et ajustement.
- Documentation de l'Annexe 1 (liste des variables).
- Revue par le Comité de Gouvernance.

### Phase 3 : Développement (Mois 3–5)
- Développement du moteur de scoring.
- Développement du moteur de décision et de l'hibernation engine.
- Développement des interfaces de suivi.
- Tests unitaires et d'intégration.

### Phase 4 : Validation (Mois 5–6)
- Tests en environnement de recette avec données anonymisées.
- Simulation rétrospective (backtesting) sur 3 mois d'historique.
- Validation par la conformité et le Comité de Risques.
- Revue de code et audit de sécurité.

### Phase 5 : Déploiement (Mois 6–7)
- Déploiement en production (fin novembre cible).
- Surveillance renforcée des premières semaines (mode « shadow » ou « monitoring » si applicable).
- Formation des analystes DOCF aux nouveaux statuts et tableaux de bord.

### Phase 6 : Post-production (Mois 7+)
- Suivi hebdomadaire des KPI (Annexe 2).
- Réentraînement du modèle trimestriel ou sur détection de drift.
- Optimisation continue des seuils S1 et S2.

---

## 9. GOUVERNANCE

### 9.1 Comité de pilotage
- **Sponsor** : Directeur, DOCF.
- **Responsable métier** : Chef de section, Surveillance transactionnelle.
- **Responsable technique** : Architecte données / ML.
- **Membres** : Conformité, Juridique, RSSI, Représentants du réseau des Caisses.

### 9.2 Comité de gouvernance du modèle
- Validation des seuils S1/S2 avant mise en production.
- Approbation des réentraînements et des modifications de features.
- Examen des cas de faux négatifs détectés post-production.
- Fréquence : trimestrielle (mensuelle les 3 premiers mois post-déploiement).

### 9.3 Gestion des risques
| Risque | Mitigation |
|---|---|
| Fuite de vrais positifs (faux négatifs du modèle) | Seuil conservateur sur S1 ; monitoring hebdomadaire ; procédure de réouverture d'urgence. |
| Dérive du modèle | Monitoring PSI ; réentraînement programmé ; alerte automatique. |
| Résistance des analystes | Communication préalable ; formation ; positionnement comme outil de priorisation. |
| Problème réglementaire | Validation juridique préalable ; documentation exhaustive ; audit trail immuable. |

---

## 10. LIVRABLES

| ID | Livrable | Responsable | Échéance |
|---|---|---|---|
| L-01 | Spécification technique (ce document) | Chef de projet | M0 |
| L-02 | Dictionnaire des données et Annexe 1 (features) | Data Scientist | M3 |
| L-03 | Modèle entraîné et validé | Data Scientist | M4 |
| L-04 | Code source du moteur de décision | Ingénieur ML | M5 |
| L-05 | Documentation d'architecture | Architecte | M5 |
| L-06 | Rapport de tests et backtesting | QA / Conformité | M6 |
| L-07 | Plan de déploiement et rollback | Ingénieur DevOps | M6 |
| L-08 | Formation et guide utilisateur | Responsable métier | M6 |
| L-09 | Tableaux de bord de suivi | BI / Data Engineer | M5 |
| L-10 | Annexe 2 — Rapport de suivi post-production | DOCF | M7+ (récurrent) |

---

## 11. DÉPENDANCES ET HYPOTHÈSES

### Hypothèses
- L'historique des résolutions d'alertes TUA est disponible et fiable sur 2 ans minimum.
- La DOCF peut fournir un analyste dédié à temps partiel pour la validation métier.
- Les seuils S1 et S2 peuvent être ajustés sans déploiement code (paramétrage).

### Dépendances externes
- Disponibilité des données Core Banking dans le datalake.
- Approbation du Comité de Gouvernance avant chaque mise en production.
- Fenêtre de déploiement validée avec les équipes d'infrastructure.

---

## 12. ANNEXES (À COMPLÉTER)

### Annexe 1 — Liste des variables prédictives
*(À produire par l'équipe Data Science durant la Phase 2)*
- Description de chaque feature.
- Source de données.
- Type (numérique, catégoriel, booléen).
- Méthode de traitement des valeurs manquantes.
- Importance relative (post-entraînement).

### Annexe 2 — Résultats de suivi post-production
*(À produire par la DOCF mensuellement post-déploiement)*
- Volume d'alertes par segment.
- Taux de réveil des alertes en hibernation.
- Taux de faux négatifs détectés.
- Performance de la règle TUA (détection des concluantes).
- Recommandations d'ajustement.

---

## 13. SIGNATURES ET APPROBATIONS

| Rôle | Nom | Signature | Date |
|---|---|---|---|
| Sponsor — Directeur DOCF | | | |
| Responsable métier | | | |
| Responsable technique | | | |
| Conformité | | | |
| Juridique | | | |

---

*Document version : 1.0*
*Date de rédaction : Septembre 2026*
*Classification : Interne — Desjardins*
