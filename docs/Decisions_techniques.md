# Décisions Techniques — Pipeline AWS NLP Matching
 
Ce document retrace les principaux choix techniques effectués lors de la conception du pipeline, avec leur justification et les alternatives envisagées.
 
---
 
## 1. Architecture Event-Driven vs Orchestrateur Central
 
**Choix retenu :** Architecture event-driven (S3 triggers → Lambda chain)
 
**Alternatives considérées :**
- Apache Airflow / AWS Step Functions pour orchestrer les étapes
- Un script Python monolithique tournant sur EC2
 
**Justification :**
L'architecture event-driven est la plus adaptée à ce cas d'usage pour trois raisons. Premièrement, les déclenchements sont naturels : un consultant dépose son dossier, le pipeline se lance automatiquement — aucune planification à gérer. Deuxièmement, la scalabilité est gratuite : Lambda scale horizontalement sans configuration. Troisièmement, le coût est minimal sur de petits volumes (facturation à l'exécution uniquement).
 
Airflow aurait introduit une infrastructure à maintenir (EC2, scheduler) pour un gain opérationnel nul sur ce périmètre.
 
---
 
## 2. AWS Comprehend vs Modèle NLP Custom
 
**Choix retenu :** AWS Comprehend (service managé)
 
**Alternatives considérées :**
- spaCy / NLTK pour une extraction locale
- Hugging Face transformers pour une extraction sémantique
 
**Justification :**
Le périmètre du projet était un POC à mettre en production rapidement. AWS Comprehend répond à ce besoin : aucun modèle à entraîner, aucune infrastructure GPU à maintenir, API simple à intégrer depuis Lambda. Les résultats d'extraction d'entités sont suffisants pour alimenter l'algorithme Jaccard.
 
La limite principale de Comprehend : l'extraction reste syntaxique, pas sémantique. "Machine Learning" et "ML" sont deux entités distinctes — gérée en aval par Lambda 3 avec une table de synonymes.
 
---
 
## 3. DocumentDB vs RDS vs DynamoDB
 
**Choix retenu :** AWS DocumentDB (compatible MongoDB)
 
**Alternatives considérées :**
- AWS RDS (PostgreSQL) pour un modèle relationnel strict
- AWS DynamoDB pour du NoSQL serverless
 
**Justification :**
Les profils consultants sont semi-structurés et hétérogènes : un consultant senior peut avoir 40 compétences listées sous forme de paragraphes, un junior en avoir 8 sous forme de liste. Imposer un schéma relationnel rigide aurait nécessité une phase de normalisation complexe et fragile.
 
DocumentDB permet de stocker chaque profil tel quel, en JSON, et d'évoluer le schéma sans migration. La compatibilité MongoDB facilite les requêtes par le moteur de matching.
 
DynamoDB a été écarté car les patterns d'accès du matching (lecture de toute la collection pour calculer les scores) ne correspondent pas à ses points forts (accès par clé primaire).
 
---
 
## 4. Similarité de Jaccard vs Embedding Sémantique
 
**Choix retenu :** Similarité de Jaccard (intersection/union d'ensembles)
 
**Alternatives considérées :**
- Cosine similarity sur embeddings word2vec / sentence-transformers
- TF-IDF pour pondérer les termes rares
 
**Justification :**
Trois contraintes ont dicté ce choix : absence de données d'entraînement historiques, besoin d'explicabilité (le client devait comprendre pourquoi un consultant était recommandé), et délai de mise en production court.
 
Jaccard répond aux trois : implémentation en quelques lignes, résultat justifiable par la liste des mots communs, aucun entraînement requis.
 
La limite principale : insensibilité aux synonymes non normalisés. Partiellement compensée par Lambda 3 (table de synonymes) mais un embedding sémantique type sentence-transformers serait plus robuste sur un volume plus important.
 
---
 
## 5. 6 Lambdas Séparées vs Fonctions Monolithiques
 
**Choix retenu :** 6 fonctions Lambda avec responsabilité unique chacune
 
**Alternative considérée :** 2-3 Lambdas plus larges couvrant plusieurs étapes
 
**Justification :**
Le timeout maximum d'une Lambda est de 15 minutes. Le traitement complet d'un dossier (conversion + Comprehend + nettoyage) peut approcher cette limite sur des fichiers volumineux avec des appels Comprehend lents. Le découpage fin garantit que chaque étape reste dans les limites.
 
Avantage opérationnel : si Lambda 2 (Comprehend) échoue sur un timeout, les fichiers déjà convertis par Lambda 1 ne sont pas re-traités inutilement — chaque étape est indépendante et relançable.
 
---
 
## 6. Lambda Layers vs Packaging par Fonction
 
**Choix retenu :** Lambda Layers partagés entre toutes les fonctions
 
**Alternative considérée :** Packager les dépendances dans chaque ZIP de déploiement
 
**Justification :**
Les bibliothèques `pandas`, `numpy` et `docx2txt` pèsent plusieurs dizaines de Mo. Les inclure dans chaque package de déploiement aurait alourdi chaque Lambda et rendu les mises à jour de dépendances laborieuses (modifier 6 packages plutôt qu'un seul Layer).
 
Les Lambda Layers permettent de partager ces bibliothèques entre toutes les fonctions et de les mettre à jour indépendamment du code métier.
 
