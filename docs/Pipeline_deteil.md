
e detail · MD
Copier

# Détail du Pipeline — 6 Fonctions Lambda
 
## Vue d'ensemble
 
Le pipeline est composé de 6 fonctions Lambda organisées en deux chaînes parallèles (Consultant et Mission) convergeant vers un moteur de matching central.
 
```
[λ1] → [λ2] → [λ3] ──┐
                       ├──► [λ6 Matching] → Résultat
[λ4] → [λ5] ──────────┘
```
 
---
 
## Chaîne Consultant
 
### Lambda 1 — Conversion de Documents (UTF-8)
 
**Déclencheur :** S3 Event sur bucket `dossierscompetences` (`.docx`)
 
**Responsabilité :** Convertir le fichier Word reçu en texte brut exploitable par les étapes suivantes.
 
**Traitements :**
- Extraction du texte brut depuis le `.docx` (structure XML interne)
- Gestion de l'encodage — normalisation en UTF-8
- Suppression des métadonnées Word (styles, révisions, commentaires)
- Transmission du texte brut à Lambda 2
 
**Points d'attention :**
- Les fichiers `.docx` produits par différentes versions de Word ont des structures XML légèrement différentes — la robustesse de l'extraction est critique
- Timeout configuré au-delà du timeout par défaut de 3s pour les fichiers volumineux
- Lambda Layer requis : `docx2txt`, `pandas`, `numpy` (Python 3.7)
 
---
 
### Lambda 2 — Extraction NLP (AWS Comprehend)
 
**Déclencheur :** Succès de Lambda 1
 
**Responsabilité :** Extraire les entités nommées et les compétences clés de chaque dossier consultant via l'API AWS Comprehend.
 
**Traitements :**
- Appel API `detect_entities` et `detect_key_phrases` sur le texte brut
- Parsing de la réponse JSON Comprehend
- Structuration des entités en document MongoDB
- Push dans DocumentDB collection `dcs`
 
**Configuration IAM :**
```
Rôle Lambda-2-Role :
  - s3:GetObject         (bucket dossierscompetences)
  - comprehend:DetectEntities
  - comprehend:DetectKeyPhrases
  - rds-db:connect       (DocumentDB)
```
 
**Points d'attention :**
- Timeout Comprehend variable selon la longueur du texte — timeout Lambda étendu
- Gestion des quotas API Comprehend (rate limiting) avec retry exponentiel
- Les entités retournées incluent des types : PERSON, ORGANIZATION, TITLE, OTHER → filtrées pour ne garder que les compétences techniques
 
---
 
### Lambda 3 — Nettoyage des Profils Consultants
 
**Déclencheur :** Succès de Lambda 2
 
**Responsabilité :** Normaliser, dédupliquer et structurer les mots-clés extraits pour les rendre compatibles avec le moteur de matching.
 
**Traitements :**
- Mise en minuscules de tous les termes
- Suppression des stop words (articles, prépositions, termes génériques)
- Déduplication des synonymes (ex: "ML" et "Machine Learning" → même token)
- Filtrage des termes trop courts (< 2 caractères) ou trop génériques
- Structuration du profil normalisé dans `dc_nettoye`
 
**Structure d'un profil normalisé :**
```json
{
  "consultant_id": "dc_xxx",
  "mots_cles": ["python", "aws", "lambda", "sql", "power bi"],
  "date_traitement": "2023-xx-xx",
  "source_fichier": "dossier_xxx.docx"
}
```
 
---
 
## Chaîne Mission
 
### Lambda 4 — Ingestion des Missions
 
**Déclencheur :** S3 Event sur bucket `opportunites` (`.xlsx`)
 
**Responsabilité :** Lire les fiches de missions hebdomadaires et déclencher l'extraction NLP sur les descriptions.
 
**Traitements :**
- Lecture du fichier Excel (une ligne = une mission)
- Extraction des champs clés : titre, description, compétences requises, niveau d'expérience, localisation
- Appel Comprehend sur les descriptions textuelles de mission
- Transmission à Lambda 5
 
**Format attendu du fichier Excel :**
 
| Colonne | Description |
|---------|-------------|
| `mission_id` | Identifiant unique de la mission |
| `titre` | Intitulé du poste |
| `description` | Description détaillée (texte libre) |
| `competences` | Compétences requises (liste) |
| `experience` | Niveau d'expérience requis |
| `localisation` | Ville ou remote |
 
---
 
### Lambda 5 — Nettoyage des Missions
 
**Déclencheur :** Succès de Lambda 4
 
**Responsabilité :** Normaliser les données de missions et les stocker dans DocumentDB prêtes pour le matching.
 
**Traitements :**
- Même pipeline de normalisation que Lambda 3 (minuscules, stop words, déduplication)
- Enrichissement avec les mots-clés extraits par Comprehend sur la description
- Fusion compétences explicites (champ `competences`) + compétences implicites (description NLP)
- Stockage dans collection `opportunites`
 
**Structure d'une mission normalisée :**
```json
{
  "mission_id": "opp_xxx",
  "titre": "Data Engineer Cloud",
  "mots_cles": ["python", "aws", "s3", "lambda", "etl", "sql", "cloud"],
  "experience_requise": "3-5 ans",
  "localisation": "Paris",
  "date_ingestion": "2023-xx-xx"
}
```
 
---
 
## Moteur de Matching
 
### Lambda 6 — Moteur Jaccard (cœur du pipeline)
 
**Déclencheur :** Planifié (cron hebdomadaire) ou déclenché manuellement
 
**Responsabilité :** Calculer le score de similarité entre chaque consultant disponible et chaque mission ouverte, retourner le Top 3 par mission.
 
**Algorithme — Similarité de Jaccard :**
 
```
Score(C, M) = |mots_clés(C) ∩ mots_clés(M)| / |mots_clés(C) ∪ mots_clés(M)|
```
 
- Score = 0 → aucun mot en commun
- Score = 1 → profil et mission identiques (cas théorique)
- En pratique, les scores varient entre 0.05 et 0.35 selon la spécialisation
 
**Pourquoi Jaccard plutôt qu'un modèle ML ?**
 
Pour un POC → production rapide, Jaccard présente trois avantages décisifs :
1. **Pas de données d'entraînement** — aucun historique de matching labellisé disponible
2. **Explicable** — le score peut être justifié par la liste des mots communs
3. **Rapide à implémenter** — quelques dizaines de lignes de Python contre des semaines de ML
 
**Complexité :** O(n × m) avec n consultants et m missions — acceptable pour des volumes de quelques centaines.
 
**Format de sortie :**
```json
{
  "mission_id": "opp_xxx",
  "titre": "Data Engineer Cloud",
  "top_matches": [
    {"consultant_id": "dc_001", "score": 0.5714, "mots_communs": ["python","aws","lambda"]},
    {"consultant_id": "dc_003", "score": 0.4000, "mots_communs": ["python","aws","sql"]},
    {"consultant_id": "dc_007", "score": 0.2857, "mots_communs": ["python","sql"]}
  ],
  "date_matching": "2023-xx-xx"
}
```
 
