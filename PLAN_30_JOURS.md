# PLAN 30 JOURS — JugeAI (version détaillée)

**Équipe GDS** : Soumaya (S) & Manal (M) — J1 = lundi, **J30 = mardi (hackathon)**, à adapter à la date réelle.

**Règles** : travail en parallèle (jamais d'attente) · une branche Git par tâche (`feat/ocr`, `feat/rag`…), relue par l'autre via pull request · teach-back le vendredi (10 min : « voilà ce que j'ai construit et pourquoi ») · échange de rôles en semaine 3 · jeu de test dès J2, baseline J14, comparaison J21 · lire le règlement du hackathon dès J1.

**Rythme du soir (2h)** : 20 min de théorie, 1h40 de pratique. Commit + 3 lignes dans `JOURNAL.md` avant de s'arrêter.

**Structure du repo** :

```
judge-AI/
├── data/        raw/  processed/  lois/  dossiers_test/
├── docs/        01_cadrage.md  04_glossaire_juridique.md
├── src/         ingestion/  normalization/  rag/  generation/  rules/  ui/
├── tests/
├── eval.py
├── JOURNAL.md
└── README.md
```

---

### 📅 JOUR 1 — LUNDI (Setup & compréhension)

| Qui | Action | Apprentissage |
|-----|--------|---------------|
| **Soumaya** | 
1) Créer le repo GitHub `judge-AI`, inviter Manal. 
2) Créer la structure de dossiers ci-dessus (fichiers `.gitkeep` dans les dossiers vides). 
3) `python -m venv .venv`, `requirements.txt` de base (`pdfplumber`, `pymupdf`, `pytesseract`, `chromadb`, `sentence-transformers`, `pydantic`, `streamlit`, `pytest`). 
4) Installer Ollama, puis `ollama pull qwen2.5:7b` et `ollama pull mistral`. 5) Tester `ollama run qwen2.5:7b "Qu'est-ce qu'un bail ?"` | Git (clone, branch, commit, push), `.gitignore` (venv, données sensibles), structure de projet Python |
| **Manal** | 
1) Lire la note de cadrage en entier et noter : les 5 livrables, le pipeline, le cas MVP (litige locatif). 
2) Lire le règlement du hackathon (prototype préparé à l'avance ? critères du jury ? format ?). 
3) Théorie RAG : vidéo de 10 min + schéma « question → recherche de passages → LLM → réponse ». 
4) Installer Tesseract avec les langues `ara` et `fra` (`tesseract --list-langs` doit afficher les deux), `pdfplumber`, `PyMuPDF` | Pourquoi un RAG plutôt qu'un LLM seul, différence OCR vs extraction directe d'un PDF texte |

**À faire ensemble (20 min fin de séance)** : *« Si vous deviez expliquer JugeAI à un juge en 3 phrases, que dites-vous ? »* — Écrire dans `docs/01_cadrage.md`. Puis vérifier que les deux machines cloneront et lanceront le même projet.

**⚠️ Pièges** : Tesseract sans le pack `ara` (l'OCR arabe échoue en silence) ; ne jamais commiter de vrais documents judiciaires.

> **🎯 Livrables J1** : Repo initialisé + structure + `requirements.txt` + `docs/01_cadrage.md` + un LLM local qui répond sur les deux machines

---

### 📅 JOUR 2 — MARDI (Corpus juridique & jeu de test)

| Qui | Action | Apprentissage |
|-----|--------|---------------|
| **Soumaya** | 
1) Récupérer le texte de la Loi 67-12 et des articles du DOC sur les baux/obligations (sgg.gov.ma). 
2) Nettoyer (en-têtes, numéros de page). 
3) Écrire `src/ingestion/split_articles.py` : regex qui découpe en un objet par article `{loi, article, titre, texte, langue}`. 
4) Sauvegarder en JSON dans `data/lois/`. 
5) Contrôle visuel de 10 articles tirés au hasard | Regex Python (`re`), structure d'un texte de loi, JSON |
| **Manal** | 
1) Créer 3-4 dossiers factices de litiges locatifs (bail, mise en demeure, 2-3 reçus) en FR, en AR, et un mixte. 
2) Planter des anomalies connues : mise en demeure datée avant le bail, reçu daté après la résiliation, loyer différent entre bail et reçu. 
3) Produire une version **scannée** (impression + photo ou export image dégradé). 
4) Écrire pour chaque dossier un `verite.json` : dates, montants, parties, liste des anomalies | Construire un jeu d'évaluation avec « vérité terrain » |

**À faire ensemble** : 15 termes juridiques FR/AR (bail, bailleur, preneur, caution, mise en demeure, loyer, charges, préavis, éviction, résiliation, quittance…) dans `docs/04_glossaire_juridique.md`.

**⚠️ Pièges** : mettre des anomalies trop évidentes (le jeu de test doit contenir des cas difficiles) ; oublier de noter la vérité terrain au moment de créer le dossier.

> **🎯 Livrables J2** : `data/lois/*.json` + `data/dossiers_test/` (3-4 dossiers + `verite.json`) + glossaire (15 termes)

---

### 📅 JOUR 3 — MERCREDI (Embeddings & OCR : comparer avant de choisir)

| Qui | Action | Apprentissage |
|-----|--------|---------------|
| **Soumaya** |
1) Écrire 10 requêtes FR/AR avec l'article attendu (ex. « durée du préavis » → article X). 
2) Charger `BAAI/bge-m3` et `intfloat/multilingual-e5-base` avec `sentence-transformers` (pour e5, préfixer `query: ` et `passage: `). 
3) Encoder les articles de `data/lois/`, calculer la similarité cosinus.
4) Mesurer **recall@1 et recall@3** pour chaque modèle.
5) Noter aussi le temps d'encodage | Embeddings multilingues, similarité cosinus, recall@k |
| **Manal** |
1) Convertir un PDF scanné en images (`pdf2image`, 300 dpi).
2) OCR brut : `pytesseract.image_to_string(img, lang="ara+fra")`.
3) Prétraitement OpenCV : niveaux de gris, redressement (deskew), binarisation (Otsu), suppression du bruit. 
4) Comparer OCR brut vs prétraité vs `pdfplumber` (sur un PDF texte) avec le taux d'erreur de caractères (`jiwer`, CER) sur 3 pages dont vous connaissez le texte exact | Traitement d'image, CER, `--psm` de Tesseract (essayer 3, 4 et 6) |

**À faire ensemble** : 15 min pour présenter chacune ses chiffres.

**⚠️ Pièges** : tester sur trop peu de requêtes ; oublier les préfixes e5 (les résultats s'effondrent).

> **🎯 Livrables J3** : Tableau comparatif embeddings (recall, vitesse) + tableau OCR (CER par méthode) dans `docs/`

---

### 📅 JOUR 4 — JEUDI (Vector DB & choix du LLM)

| Qui | Action | Apprentissage |
|-----|--------|---------------|
| **Soumaya** | 
1) Créer `src/rag/retrieval.py` avec `chromadb.PersistentClient(path="chroma_db")`. 
2) Collection `lois` avec le modèle d'embedding gagnant de J3. 
3) Indexer chaque article avec métadonnées `{loi, article, langue}`. 
4) Fonction `search_laws(query, k=3, loi=None)` avec filtre par métadonnée. 
5) Tester 5 requêtes | Base vectorielle, filtrage par métadonnées, persistance |
| **Manal** | 
1) Préparer 10 questions juridiques FR/AR avec réponse attendue. 
2) Pour 3 modèles Ollama (`qwen2.5:7b`, `mistral`, `llama3.1:8b`), mesurer tokens/s (`ollama run --verbose`) et noter la qualité sur 5 critères : français, arabe, exactitude, hallucinations, respect des consignes. 
3) Noter RAM utilisée sur vos laptops. 
4) Conclure : modèle retenu + modèle de secours | Choisir un modèle selon contraintes matérielles, repérer les hallucinations |

**⚠️ Pièges** : choisir le modèle le plus gros sans tester la vitesse (une démo de 3 minutes ne peut pas attendre 90 s par réponse).

> **🎯 Livrables J4** : Modèle choisi + `retrieval.py` v0 + tableau benchmark LLM

---

### 📅 JOUR 5 — VENDREDI (Squelette bout en bout)

| Qui | Action | Apprentissage |
|-----|--------|---------------|
| **Soumaya** | 
1) Finaliser `retrieval.py` : `search_laws()` renvoie texte + métadonnées. 
2) Écrire `src/generation/ask.py` : prend une question, récupère 3 articles, construit un prompt « Réponds uniquement avec ces extraits, cite l'article », appelle Ollama | Intégration retrieval + LLM, prompt simple |
| **Manal** | 
1) Écrire `src/ingestion/ingest.py` : PDF → si texte natif, `pdfplumber` ; sinon OCR. 
2) Sortie : liste `[{page: 1, texte: "..."}]`. 
3) Sauvegarder en `data/processed/<dossier>.json` | Pipeline d'ingestion avec choix automatique texte/OCR, structure « une page = une entrée » |

**À faire ensemble** : brancher les deux dans `main.py` : `python main.py dossier.pdf "Quel est le loyer ?"` doit afficher une réponse avec sources. Teach-back + sync.

**⚠️ Pièges** : perdre le numéro de page dès l'ingestion (impossible à retrouver ensuite).

> **🎯 Livrables J5** : **Squelette bout en bout (ligne de commande)**, même imparfait

---

### 📅 JOUR 6-7 — WEEK-END (Optionnel & revue)

| Qui | Action |
|-----|--------|
| **Soumaya** | *Samedi (optionnel, 1-2h)* : tutoriel Pydantic (`BaseModel`, validation, `model_json_schema()`) et sortie JSON avec Ollama. *Dimanche (1h)* : relire le code de Manal sur GitHub, laisser 3 commentaires de revue |
| **Manal** | *Samedi (optionnel, 1-2h)* : lire sur BM25 et la recherche hybride (dense + mots-clés). *Dimanche (1h)* : relire le code de Soumaya, laisser 3 commentaires de revue |

**À faire ensemble (dimanche, 20 min)** : fusionner les 6 commentaires en liste d'améliorations, planifier la semaine 2.

> **🎯 Livrables J6-7** : Liste de 6 améliorations + plan de la semaine 2

---

### 📅 JOUR 8 — LUNDI (Chunking & extraction)

| Qui | Action | Apprentissage |
|-----|--------|---------------|
| **Soumaya** | 
1) `src/normalization/chunking.py` : découper chaque page en chunks de ~500 caractères avec chevauchement de ~100, sans couper une phrase. 
2) Chaque chunk garde `{dossier, page, chunk_id}`. 
3) `src/normalization/arabic.py` : retirer les diacritiques (tashkeel), unifier les alef (أ إ آ → ا), convertir les chiffres indo-arabes (٠-٩) en chiffres latins. 
4) Tester sur un texte arabe de vos dossiers | Stratégies de chunking (taille, overlap), normalisation de l'arabe |
| **Manal** | 
1) `src/normalization/extraction.py` : extraire dates, montants (MAD/DH) et parties. 
2) Dates : regex pour les formats `jj/mm/aaaa`, « 5 mars 2025 », et dates arabes, puis `dateparser.parse(texte, languages=["fr","ar"])`. 
3) Sortie Gold : `{type, valeur, page, extrait}`. 
4) Vérifier sur les `verite.json` : combien de dates retrouvées ? | Extraction d'information, formats de dates FR/AR |

**⚠️ Pièges** : ambiguïté `03/04/2025` (jour/mois) : fixer la convention jj/mm ; chunks trop petits qui perdent le contexte.

> **🎯 Livrables J8** : `chunking.py` + `arabic.py` + `extraction.py` + rapport « X dates sur Y retrouvées »

---

### 📅 JOUR 9 — MARDI (Anomalies de dates & indexation)

| Qui | Action | Apprentissage |
|-----|--------|---------------|
| **Soumaya** | 
1) Créer une collection Chroma par dossier (`dossier_<id>`). 
2) Indexer les chunks avec leurs métadonnées (page, type de pièce). 
3) Fonction `search_dossier(dossier_id, query, k=5)` qui renvoie texte + page. 
4) Vérifier : « Quel est le loyer ? » doit retrouver la bonne page | Traçabilité page → chunk, isolation des dossiers |
| **Manal** | 
1) `src/rules/anomalies.py` : prend le JSON Gold d'un dossier. 
2) Écrire 6-8 règles : mise en demeure avant le bail, reçu après résiliation, loyer du reçu ≠ loyer du bail, date dans le futur, deux dates contradictoires pour le même événement, pièce attendue manquante.
 3) Chaque alerte : `{regle, gravite, pieces, pages, message}`. 
 4) Comparer à `verite.json` | Pourquoi des règles valent mieux qu'un LLM pour les contradictions chiffrées, faux positifs / faux négatifs |

**⚠️ Pièges** : chaque règle doit avoir au moins un test (même un dict à la main).

> **🎯 Livrables J9** : Dossier indexé avec pages + `anomalies.py` qui détecte les anomalies plantées

---

### 📅 JOUR 10 — MERCREDI (RAG complet & schémas)

| Qui | Action | Apprentissage |
|-----|--------|---------------|
| **Soumaya** | 
1) `src/rag/rag.py` : étapes `retrieve` (3-5 passages du dossier + 3 articles de loi) → `build_prompt` → `generate`. 
2) Prompt système : rôle (« assistant d'aide au magistrat »), règles (« ne cite que les extraits fournis », « si absent, réponds *information non trouvée* », « ne donne jamais de verdict »), format de sortie. 
3) Tester 5 questions | Prompt engineering : rôle + tâche + contexte + contraintes + format |
| **Manal** |
1) `src/generation/schemas.py` : modèles Pydantic `Synthese`, `Evenement` (date, description, page), `Contradiction` (point, version_A, version_B, pages), `ReferenceLegale` (loi, article, extrait), `Alerte`. 
2) Un modèle global `RapportJuge` qui les regroupe. 
3) Exemples JSON valides | Pydantic, modélisation des livrables |

**À faire ensemble** : pair 30 min pour brancher `RapportJuge` comme format de sortie du RAG.

> **🎯 Livrables J10** : `rag.py` + `schemas.py` branchés + premier rapport complet (même brouillon)

---

### 📅 JOUR 11 — JEUDI (Recherche hybride & sortie structurée)

| Qui | Action | Apprentissage |
|-----|--------|---------------|
| **Soumaya** | 
1) Ajouter `rank_bm25` : index BM25 sur les chunks. 
2) Fusion des résultats dense + BM25 par **Reciprocal Rank Fusion**. 
3) Comparer recall@3 : dense seul vs hybride, sur 15 requêtes (dont numéros d'articles, montants, noms propres). 
4) Garder le meilleur | Limites du dense seul (chiffres, noms), fusion de classements |
| **Manal** | 
1) Appeler Ollama avec `format=RapportJuge.model_json_schema()` pour forcer un JSON valide. 
2) Valider avec Pydantic ; en cas d'erreur, retenter jusqu'à 2 fois en renvoyant l'erreur au modèle. 
3) Mesurer le taux de JSON valides sur 10 appels | Génération structurée, boucle de validation et retry |

> **🎯 Livrables J11** : Retrieval hybride mesuré + JSON valide pour chaque livrable (taux noté)

---

### 📅 JOUR 12 — VENDREDI (Références légales & contradictions)

| Qui | Action | Apprentissage |
|-----|--------|---------------|
| **Soumaya** | 
1) Forcer le format de citation `[Loi 67-12, art. X]`. 
2) `src/rules/verify_citations.py` : extraire chaque citation de la sortie et vérifier qu'elle existe dans `data/lois/`. 
3) Si inexistante : la retirer et créer une alerte « citation non vérifiée ». 
4) Tester en forçant le LLM à inventer un article | Anti-hallucination par vérification, RAG ancré dans le corpus |
| **Manal** | 
1) Générer la **chronologie** à partir des données Gold (dates triées, avec page source) — le LLM ne fait que la rédaction. 
2) Générer le **tableau contradictoire** : pour chaque champ clé (loyer, date de paiement, durée), comparer bailleur vs locataire. 
3) Valider sur les dossiers de test | Séparer ce qui est calculé (règles) de ce qui est rédigé (LLM) |

**À faire ensemble** : teach-back + run des 5 livrables sur un dossier.

> **🎯 Livrables J12** : Les 5 livrables en JSON, avec citations vérifiées

---

### 📅 JOUR 13-14 — WEEK-END (Baseline)

| Qui | Action |
|-----|--------|
| **Soumaya** | 
*Samedi (optionnel)* : lire sur le reranking (cross-encoder, `bge-reranker`). 
*Dimanche* : script `run_all.py` qui lance le pipeline sur tous les dossiers de test et sauvegarde les sorties |
| **Manal** | 
*Samedi (optionnel)* : lire sur l'évaluation de RAG (fidélité aux sources, pertinence). 
*Dimanche* : comparer les sorties à `verite.json` : anomalies détectées (rappel / précision), citations valides (%), recall@3 du retrieval, temps total |

> **🎯 Livrables J13-14** : **Baseline chiffrée** dans `docs/eval_baseline.md` (un tableau, 4-5 métriques)

---

### 📅 JOUR 15 — LUNDI (Échange de rôles)

| Qui | Action | Apprentissage |
|-----|--------|---------------|
| **Soumaya** | Reprend l'**ingestion/OCR** de Manal : 
1) lire et comprendre `ingest.py`. 
2) Tester sur scans arabes et documents mixtes. 
3) Améliorer (prétraitement, `--psm`). 
4) Mesurer le CER avant/après | OCR arabe, documents hybrides ar+fr |
| **Manal** | Reprend le **retrieval** de Soumaya : 
1) lire `retrieval.py` et le chunking. 
2) Ajouter la métrique recall@k à l'évaluation. 
3) Tester un reranker en bonus. 
4) Mesurer le gain | Retrieval, reranking, évaluation |

**À faire ensemble** : 15 min : chacune explique ce qu'elle a changé dans le code de l'autre.

> **🎯 Livrables J15** : Chacune a modifié (PR) et compris le code de l'autre

---

### 📅 JOUR 16 — MARDI (Évaluation & tests)

| Qui | Action | Apprentissage |
|-----|--------|---------------|
| **Soumaya** | `eval.py` à la racine : lance le pipeline sur `data/dossiers_test/`, compare à `verite.json`, affiche un tableau (anomalies détectées, citations valides, recall@3, temps) et l'enregistre en JSON daté | Évaluation automatisée, reproductibilité |
| **Manal** | `tests/` avec `pytest` : 
1) extraction de dates (5 cas FR, 5 AR). 
2) anomalies (un test par règle). 
3) vérificateur de citations (1 vraie, 1 inventée). 
4) normalisation arabe. Objectif : `pytest` passe en moins de 30 s | Tests unitaires, cas limites |

> **🎯 Livrables J16** : `eval.py` + `tests/` qui passent

---

### 📅 JOUR 17 — MERCREDI (Prompts & interface)

| Qui | Action | Apprentissage |
|-----|--------|---------------|
| **Soumaya** | 
1) Ouvrir `docs/erreurs.md` : pour chaque erreur de la baseline, noter cause (retrieval ? prompt ? OCR ?), correction, effet mesuré. 
2) Modifier **un seul** prompt à la fois, relancer `eval.py`. 
3) Garder la modification seulement si les métriques s'améliorent | Amélioration guidée par les mesures |
| **Manal** | 
1) `src/ui/app.py` avec Streamlit : `st.file_uploader`, bouton « Analyser », `st.spinner`. 
2) `st.tabs` : Synthèse, Chronologie, Contradictions, Références légales, Alertes. 
3) Bandeau permanent : *« Cet outil est une aide à la décision, il ne remplace pas le jugement du magistrat. »* | Streamlit, composants UI, éthique IA |

> **🎯 Livrables J17** : Prompts v2 (gain mesuré) + UI v1 avec avertissement

---

### 📅 JOUR 18 — JEUDI (Sources cliquables & performance)

| Qui | Action | Apprentissage |
|-----|--------|---------------|
| **Soumaya** | 
1) Chaque affirmation affichée porte `(pièce, page)`. 
2) Au clic (`st.expander`), afficher l'extrait source. 
3) Vérifier : un juge peut-il vérifier chaque info en moins de 10 secondes ? | Traçabilité côté utilisateur |
| **Manal** | 
1) Mesurer le temps de chaque étape (OCR, chunking, embeddings, génération) avec `time.perf_counter` et des logs. 
2) Cache des embeddings (par hash du fichier). 
3) Réduire le plus lent. Objectif : moins de 2 min par dossier | Profilage, caching, logs |

> **🎯 Livrables J18** : Sources visibles dans l'UI + benchmark de performance

---

### 📅 JOUR 19 — VENDREDI (Test sur dossiers inédits)

| Qui | Action | Apprentissage |
|-----|--------|---------------|
| **Soumaya** | Test « aveugle » : demander à une amie de créer un dossier que vous n'avez jamais vu, le passer dans l'UI, noter chaque écart. Corriger les bugs | Debug, QA, biais du jeu de test |
| **Manal** | Même test avec un dossier 100 % arabe. Noter OCR, dates, citations | Validation multilingue |

**À faire ensemble** : teach-back + tri des bugs (bloquant / gênant / cosmétique).

> **🎯 Livrables J19** : Rapport de bugs, les bloquants fermés

---

### 📅 JOUR 20-21 — WEEK-END (Mesure du gain)

| Qui | Action |
|-----|--------|
| **Soumaya** | *Samedi (optionnel)* : Docker (`Dockerfile` simple). *Dimanche* : relancer `eval.py` |
| **Manal** | *Samedi (optionnel)* : améliorer l'UX. *Dimanche* : comparer à la baseline de J14, rédiger la section « Résultats » du README (tableau avant/après + 1 exemple de sortie) |

> **🎯 Livrables J20-21** : **Gain mesuré vs J14** + README (Résultats)

---

### 📅 JOUR 22 — LUNDI (Cas limites & confidentialité)

| Qui | Action | Apprentissage |
|-----|--------|---------------|
| **Soumaya** | 
1) Scan illisible : seuil de confiance OCR, message « page illisible » au lieu d'inventer. 
2) Pièce manquante (pas de reçus, pas de bail) : alerte explicite. 
3) PDF vide ou protégé : message d'erreur propre. 
4) Un test par cas | Robustesse, gestion d'erreurs, échec explicite |
| **Manal** | 
1) `src/normalization/anonymize.py` : masquer CIN (format : 1-2 lettres + 5-6 chiffres), téléphones, IBAN/RIB, noms avec une liste de remplacement. 
2) Mode 100 % local : aucun appel réseau pendant l'analyse. 
3) Note de confidentialité dans le README (Loi 09-08, CNDP) | Protection des données personnelles |

> **🎯 Livrables J22** : Gestion d'erreurs testée + anonymisation + note de confidentialité

---

### 📅 JOUR 23 — MARDI (Échelle & packaging)

| Qui | Action | Apprentissage |
|-----|--------|---------------|
| **Soumaya** | 
1) Assembler un dossier de 40-100 pages (concaténer vos dossiers de test). 
2) Mesurer temps, RAM, qualité. 
3) Optimiser si > 2 min (batching des embeddings, moins de chunks) | Benchmarking à l'échelle |
| **Manal** | 
1) `pip freeze` propre dans `requirements.txt`. 
2) Script `install.sh` / `install.bat`. 
3) Tester sur une machine propre : clone → install → lancement. Noter chaque friction | Packaging, reproductibilité |

> **🎯 Livrables J23** : Installation reproductible + test d'échelle

---

### 📅 JOUR 24 — MERCREDI (Garde-fou & documentation)

| Qui | Action | Apprentissage |
|-----|--------|---------------|
| **Soumaya** | 
1) Test automatique « sans verdict » : 10 questions piège (« Qui a raison ? », « Quel jugement rendre ? »). 
2) Le système doit refuser et rediriger vers les faits et les textes. 
3) Intégrer ce test à `eval.py` | Éthique IA, tests de sécurité |
| **Manal** | 1) README : objectif, installation, utilisation, méthodologie, résultats, limites. 2) Schéma Mermaid de l'architecture (Bronze → Silver → Gold → RAG → livrables). 3) Section « Limites » : manuscrit, langues rares, hallucinations résiduelles | Documentation technique, honnêteté sur les limites |

> **🎯 Livrables J24** : Test « sans verdict » intégré + README complet avec schéma

---

### 📅 JOUR 25 — JEUDI (Gel & plan B)

| Qui | Action |
|-----|--------|
| **Les deux** | 
1) **Gel des fonctionnalités** : plus aucune nouvelle fonction, seulement des corrections. 
2) Tests complets (`pytest`, `eval.py`, démo) sur les deux machines. 
3) Plan B : vidéo de la démo + captures d'écran + sorties JSON pré-calculées. 
4) Repo propre, `requirements.txt` à jour. 
5) Relire les consignes de soumission |

> **🎯 Livrables J25** : Version stable + plan B prêt

---

### 📅 JOUR 26 — VENDREDI (Slides & démo)

| Qui | Action | Apprentissage |
|-----|--------|---------------|
| **Soumaya** | Slides (10-12) : Problème → Solution → Pipeline (schéma) → Démo → **Métriques avant/après** → Limites → Éthique. Une idée par slide, peu de texte | Storytelling technique |
| **Manal** | Script de démo minuté (3 min) : upload → 5 livrables → clic sur une source → alerte détectée. Plus une note de présentation (1 page) | Pitch, démo maîtrisée |

**À faire ensemble** : teach-back final : chacune présente la moitié de l'autre.

> **🎯 Livrables J26** : Deck + script de démo + note de présentation

---

### 📅 JOUR 27-28 — WEEK-END (Répétition & repos)

| Qui | Action |
|-----|--------|
| **Les deux** | *Samedi* : répétition complète devant un tiers (ami ou prof), chronométrée ; noter les questions posées et les moments flous. *Dimanche* : repos |

> **🎯 Livrables J27-28** : Notes de feedback

---

### 📅 JOUR 29 — LUNDI (Dernière préparation)

| Qui | Action |
|-----|--------|
| **Les deux** | 
1) Préparer les réponses aux questions techniques : « pourquoi pas ChatGPT ? », « comment évitez-vous les hallucinations ? », « que se passe-t-il avec un scan illisible ? », « et la confidentialité ? ». 
2) Tout pushé sur GitHub + sauvegarde ZIP + clé USB. 
3) Dernière répétition. 
4) Dormir tôt |

> **🎯 Livrables J29** : Q/R préparées + repo final sauvegardé

---

### 📅 JOUR 30 — MARDI (Hackathon 🎯)

| Qui | Action |
|-----|--------|
| **Les deux** | Arriver en avance, tester l'installation sur la machine de présentation, présenter, faire la démo en direct (plan B prêt) |

> **🎯 Livrables J30** : Démo live + présentation orale

---

## 📚 RÉCAPITULATIF DES APPRENTISSAGES

| Domaine | À apprendre | Quand |
|---------|-------------|-------|
| **Collaboration** | Git, branches, pull requests, revue de code | J1+ |
| **Extraction** | pdfplumber, PyMuPDF, Tesseract (ara+fra), OpenCV, CER | J1-J5, J15 |
| **NLP FR/AR** | regex, normalisation arabe, `dateparser` | J2, J8 |
| **Recherche** | bge-m3, multilingual-e5, ChromaDB, BM25, RRF, reranking | J3-J11 |
| **LLM local** | Ollama, benchmark, prompt engineering | J4, J10, J17 |
| **Sortie structurée** | Pydantic, JSON schema, retry | J10-J11 |
| **Fiabilité** | citations vérifiées, règles déterministes, évaluation | J9, J12, J16 |
| **Qualité** | pytest, logs, profilage | J16, J18 |
| **Interface** | Streamlit | J17-J18 |
| **Éthique / droit** | Loi 67-12, DOC, Loi 09-08, garde-fou « sans verdict » | J2, J22, J24 |
| **Livraison** | packaging, README, plan B, pitch | J23-J30 |

---

## 🔄 RÉUNION QUOTIDIENNE (10 min, fin de séance)

Chaque soir, répondre par écrit dans `JOURNAL.md` partagé :

1. **Qu'ai-je fait aujourd'hui ?**
2. **Qu'ai-je appris ?**
3. **Qu'est-ce qui bloque / que fais-je demain ?**

Syncs de 15 min : lundi, mercredi, vendredi.

---

## ⚠️ GESTION DES BLOCS (si une tâche déraille)

- **Indispensable** : squelette (J5), corpus indexé, extraction + anomalies de dates, 5 livrables en JSON, citations vérifiées, UI simple, jeu de test + métriques.
- **À couper en premier** : reranking, Docker, recherche hybride, anonymisation avancée.
- **Hors périmètre volontairement** : autres typologies d'affaires (travail, commercial) et export PDF — un cas d'usage locatif solide vaut mieux.
- **Si bloquée > 2 jours** : demander de l'aide (professeur, pairs, en ligne).