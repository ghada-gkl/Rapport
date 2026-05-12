# Chapitre 4 — Compréhension des Données

---

## 4.1 Collecte de Données

### 4.1.1 Sources de Données

La constitution du corpus documentaire sur lequel repose notre système RAG a nécessité une réflexion approfondie sur les sources à mobiliser. Deux grandes catégories de données ont été identifiées et exploitées.

La première source est constituée de la **documentation officielle de formation SAP**, soit plus de 297 supports de cours au format PDF couvrant l'ensemble des modules fonctionnels et techniques de la plateforme : gestion des bases de données (BC100 – Introduction à ABAP, ADM100 – Administration Basis, AC020 – Comptabilité analytique), sécurité, architecture NetWeaver, connectivité RFC, gestion des jobs en arrière-plan, et bien d'autres. Ces documents, issus du programme de formation accrédité SAP, représentent la référence métier la plus complète disponible pour un technicien SAP.

La seconde source est le **portail d'aide en ligne SAP** (help.sap.com), dont des sections thématiques ont été extraites via un processus de scraping structuré. Ce portail fournit une documentation orientée configuration, codes d'erreur et notes de correction (SAP Notes), complémentaire aux livres de formation.

Le choix délibéré de croiser ces deux sources répond à un besoin réel : les livres de formation expliquent les concepts et les procédures, tandis que la documentation d'aide traite les cas d'erreurs et les paramètres système. Un moteur de recherche ne s'appuyant que sur l'une des deux sources aurait des angles morts évidents.

---

### 4.1.2 Description des Fichiers

L'extraction et la transformation des données se sont déroulées en plusieurs étapes, produisant une famille de fichiers intermédiaires et finaux dont la structure reflète la progression du pipeline.

| Fichier | Taille | Contenu |
|---|---|---|
| `output/<COURS>/structured_content.json` | Variable | Contenu hiérarchique extrait par livre (sections, procédures, tableaux) |
| `normalized/unified_content.json` | ~147 Mo | 142 798 chunks normalisés — toutes sources confondues |
| `normalized/by_source/technical_books.json` | ~138 Mo | 78 645 chunks issus des livres de formation |
| `normalized/by_source/sap_help.json` | ~55 Mo | 17 785 chunks issus du portail d'aide |
| `normalized/by_source/figures.json` | ~42 Mo | 46 368 chunks de légendes de figures |
| `chunked_for_embedding.json` | ~222 Mo | Chunks finaux prêts pour l'embedding (parents + enfants) |
| `metadata.json` | ~202 Mo | Métadonnées alignées par index avec les vecteurs |
| `embeddings.npy` | ~486 Mo | Vecteurs d'embeddings — forme (1 278 438, 1024), float32 |

Chaque livre de formation dispose de son propre répertoire dans `output/`, contenant le contenu structuré, un manifeste de figures, les images extraites au format PNG, et un rapport d'extraction (pages traitées, sections identifiées, figures détectées).

---

### 4.1.3 Description des Données — Variables Principales

Une fois normalisées et découpées, les données sont stockées dans une table PostgreSQL nommée `sap_chunks`, dont le schéma a été conçu pour supporter à la fois la recherche sémantique par vecteur et le filtrage structuré par métadonnées.

```sql
CREATE TABLE sap_chunks (
    chunk_id         UUID PRIMARY KEY,
    text             TEXT NOT NULL,           -- contenu textuel du chunk
    title            TEXT,                    -- titre de la section parente
    source_type      VARCHAR,                 -- 'technical_book' | 'sap_help' | 'figure'
    source_book      TEXT,                    -- nom du livre ou de la page source
    module           VARCHAR,                 -- module SAP : BASIS, ABAP, MM, SD, FI…
    section_type     VARCHAR,                 -- 'section' | 'procedure' | 'note' | 'table'
    breadcrumb       TEXT[],                  -- chemin hiérarchique (ex: ["Chap3","Sect2"])
    mentioned_tcodes TEXT[],                  -- codes transaction détectés (SM37, ST22…)
    mentioned_sap_notes TEXT[],              -- numéros de notes SAP (5-7 chiffres)
    is_parent        BOOLEAN,                 -- chunk parent non-embedé
    parent_chunk_id  UUID,                    -- référence au chunk parent
    embedding        vector(1024)             -- vecteur BGE-large-en-v1.5
);
```

Les **variables principales** qui structurent la sémantique de chaque chunk sont :

- **`text`** : le contenu brut de la documentation, après nettoyage des artefacts d'extraction. C'est la variable centrale — celle que l'utilisateur finit par lire dans la réponse du chatbot.
- **`embedding`** : vecteur de 1 024 dimensions produit par le modèle `BAAI/bge-large-en-v1.5`. C'est la représentation numérique dense du sens du texte, utilisée pour la recherche par similarité cosinus.
- **`mentioned_tcodes`** : tableau des codes transaction SAP explicitement mentionnés dans le chunk (SM37, ST22, SE38…). Cette variable est critique pour le routage des requêtes : une question portant sur SM37 sera filtrée directement sur les chunks qui mentionnent ce code.
- **`module`** : appartenance fonctionnelle du chunk (BASIS, ABAP, MM, SD, FI, SECURITY, HANA, FIORI). Permet de restreindre la recherche à un périmètre fonctionnel précis.
- **`source_type`** : distingue les chunks de livres de formation (`technical_book`), d'aide en ligne (`sap_help`) et de figures (`figure`). Ce champ oriente le moteur vers la bonne nature de documentation selon le type de question.
- **`breadcrumb`** : chemin hiérarchique reconstruit à partir de la structure du document d'origine. Il permet de contextualiser un chunk dans son environnement documentaire sans avoir à relire le document entier.

---

## 4.2 Exploration des Données

### 4.2.1 Analyses Réalisées

L'exploration des données a été conduite à deux niveaux : une analyse de la distribution volumétrique du corpus, et une analyse de la richesse sémantique des métadonnées.

**Distribution par source :**

| Source | Chunks | Part |
|---|---|---|
| Livres de formation | 78 645 | 55 % |
| Figures & légendes | 46 368 | 33 % |
| SAP Help Portal | 17 785 | 12 % |
| **Total** | **142 798** | **100 %** |

**Distribution par module :**

| Module | Chunks |
|---|---|
| ABAP | 12 699 |
| BASIS | 4 251 |
| Fiori | 6 263 |
| HANA | 3 994 |
| MM / SD / FI | ~8 000 |

**Distribution par type de section :**
Les chunks se répartissent entre sections conceptuelles (majoritaires), procédures pas à pas, notes d'avertissement et tableaux de paramètres. Cette diversité est importante : une question « comment faire » appelle une procédure, tandis qu'une question « qu'est-ce que » appelle une section conceptuelle.

**Analyse des T-codes référencés :**
Plus de 80 codes transaction SAP distincts ont été détectés dans le corpus, avec une concentration notable autour des transactions de monitoring (SM37, ST22, SM21, SM50) et de développement (SE38, SE11, SE80). La distribution est très asymétrique : quelques T-codes très documentés concentrent l'essentiel des chunks associés.

---

### 4.2.2 Observations

Plusieurs observations importantes ont émergé de cette exploration.

**Déséquilibre entre sources** : les livres de formation dominent le corpus (55 %). Cela reflète la réalité de la documentation SAP : les livres sont exhaustifs, structurés et longs, tandis que les pages d'aide sont plus ciblées. Ce déséquilibre est intentionnel — pour un système de type assistant technique, le contenu pédagogique des livres est plus utile que les pages d'aide ponctuelles.

**Richesse des figures** : avec 46 368 chunks dédiés aux légendes et descriptions de figures, le corpus couvre une dimension souvent négligée dans les systèmes RAG. Les schémas d'architecture SAP, les captures d'écran de transactions et les diagrammes de flux contiennent des informations difficiles à trouver dans le texte seul.

**Couverture inégale des modules** : ABAP et Fiori sont sur-représentés par rapport à des modules comme la comptabilité (FI) ou la gestion des matériaux (MM). Cette inégalité est directement liée aux livres disponibles lors de la collecte — elle constitue une limite à documenter clairement.

**Longueur variable des chunks** : après découpage, la taille des chunks varie entre quelques dizaines de mots (notes courtes, en-têtes de tableau) et 512 tokens (limite maximale). Cette variabilité impacte la qualité de l'embedding — les chunks très courts sont moins bien représentés dans l'espace vectoriel.

---

## 4.3 Qualité des Données

### 4.3.1 Problèmes Détectés

L'extraction de texte depuis des PDFs techniques n'est jamais propre. Plusieurs catégories de bruit ont été identifiées et mesurées.

**Artefacts d'encodage PDF** : le format PDF stocke les glyphes de certaines polices via des identifiants internes appelés codes CID (Character Identifier). Lorsque la police n'est pas embarquée dans le fichier, `pdfplumber` ne peut pas décoder ces glyphes et produit des séquences du type `(cid:123)`. Ces artefacts ont été filtrés par expression régulière mais leur présence reste sporadique dans les chunks les moins bien encodés.

**Résidus HTML** : les chunks issus du portail SAP Help contiennent par endroits des balises HTML mal nettoyées (`<b>`, `<i>`, `&nbsp;`, `&amp;`). Ces résidus perturbent la tokenisation et dégradent la qualité de l'embedding.

**Faux positifs dans la détection de T-codes** : le système extrait les T-codes par expression régulière sur le motif `[A-Z]{2,4}\d{1,3}`. Cette heuristique, bien que fonctionnelle, génère des faux positifs sur des abréviations comme `/ABAP/`, `/API/BUSINESS` ou des numéros de section du type `BC100`. Une liste blanche de 80+ T-codes connus a été appliquée en post-traitement pour filtrer ces faux positifs.

**Gap vocabulaire du modèle d'embedding** : `BAAI/bge-large-en-v1.5` est un modèle généraliste entraîné sur du texte académique et web. Il ne connaît pas nativement les codes transaction SAP ni leur signification. Une requête utilisateur sur « SM37 » et un chunk documentant « le moniteur de jobs en arrière-plan » ne seraient pas rapprochés sans intervention. Ce problème a été traité par **expansion de synonymes** : les descriptions textuelles des T-codes les plus courants ont été pré-concaténées aux chunks les mentionnant.

**Séquences de points (TOC)** : les tables des matières dans les PDFs SAP génèrent des lignes de type `Chapitre 3 .......... 42` qui, une fois extraites, polluent les chunks avec des suites de points sans valeur sémantique. Ces séquences ont été filtrées par regex.

---

### 4.3.2 Approche Multi-Agent pour la Collecte et Gestion des Certificats SSL

La collecte des données à l'échelle de 297 documents PDF et de plusieurs centaines de pages web ne pouvait pas être réalisée manuellement. Une architecture multi-agent a été mise en place pour paralléliser et fiabiliser ce processus.

**Extraction PDF par agents parallèles** : le script `Sap_batch_extractor.py` orchestre un ensemble d'agents d'extraction, chacun responsable d'un sous-ensemble de livres. Chaque agent opère de façon autonome : il charge le PDF, détecte la hiérarchie des sections à partir de l'analyse des polices (taille, graisse, famille), extrait le texte de chaque section, isole les figures, et produit un fichier `structured_content.json` pour son livre. Le traitement de 165 cours en parallèle réduit le temps total d'extraction de plusieurs heures à quelques dizaines de minutes.

**Gestion des certificats SSL pour la base de données** : l'ingestion des vecteurs dans la base PostgreSQL distante (hébergée sur Neon ou Supabase, selon les phases du projet) expose le pipeline à des interruptions de connexion liées aux certificats SSL. Deux mécanismes de résilience ont été implémentés :

1. **TCP keepalives** : la connexion PostgreSQL est configurée avec `keepalives=1`, `keepalives_idle=30` secondes et `keepalives_interval=10` secondes. Ces paramètres maintiennent la connexion active pendant les longues phases d'ingestion par batch et évitent les timeouts silencieux côté serveur.

2. **Boucle de retry sur drop SSL** : lorsqu'une exception contient la chaîne `"SSL"` (drop de connexion, rehandshake échoué), le pipeline attend 2 secondes et retente automatiquement la connexion jusqu'à 3 fois avant d'abandonner le batch. Cette stratégie a permis d'ingérer les 1,27 million de vecteurs sans interruption manuelle, malgré les instabilités de réseau observées sur les connexions cloud longue durée.

```python
# Exemple de gestion du retry SSL dans ingest_to_db.py
for attempt in range(3):
    try:
        conn = psycopg2.connect(DB_URL, keepalives=1, keepalives_idle=30)
        # ... ingestion du batch
        break
    except Exception as exc:
        if "SSL" in str(exc):
            print(f"[WARN] SSL drop, retrying... ({attempt+1}/3)")
            time.sleep(2)
        else:
            raise
```

**Indexation HNSW en post-ingestion** : une fois tous les vecteurs insérés, un index HNSW (Hierarchical Navigable Small World) est construit sur la colonne `embedding` avec les paramètres `m=16, ef_construction=64`. Cet index transforme la recherche par similarité cosinus d'une complexité linéaire O(N) en une complexité approximative O(log N), ce qui rend le système viable à l'échelle de plus d'un million de vecteurs.

---

### 4.3.3 Actions Prévues

Plusieurs améliorations sont envisagées pour renforcer la qualité des données dans les prochaines itérations du projet.

- **Enrichissement du corpus SAP Help** : la part actuelle du portail d'aide (12 % des chunks) est sous-représentée. Une collecte plus systématique, couvrant notamment les notes de correction (SAP Notes) et les messages d'erreur ABAP, améliorerait la couverture des cas de dépannage.
- **Rééquilibrage des modules** : la surreprésentation d'ABAP et Fiori sera corrigée par l'ajout de livres dédiés aux modules FI, CO et MM lors de la prochaine campagne de collecte.
- **Validation humaine d'un sous-ensemble** : un échantillon de 500 chunks représentatifs fera l'objet d'une relecture manuelle pour mesurer précisément le taux de bruit résiduel et valider les heuristiques de nettoyage.
- **Embeddings domaine-spécifiques** : le remplacement de `bge-large-en-v1.5` par un modèle fine-tuné sur la documentation SAP (en utilisant les paires question/chunk comme données d'entraînement) est en cours d'évaluation.

---

## 4.4 Analyse des Relations

### 4.4.1 Corrélations Importantes

L'analyse des relations entre les variables du corpus révèle plusieurs corrélations structurantes pour le système de récupération.

**Corrélation T-code ↔ Module** : les codes transaction SAP sont quasi-systématiquement associés à un unique module fonctionnel. SM37 appartient exclusivement à BASIS, SE38 à ABAP, SU01 à SECURITY. Cette corrélation quasi-déterministe permet d'inférer le module à partir du T-code détecté dans la requête, ce qui enrichit automatiquement les filtres de récupération sans que l'utilisateur ait besoin de spécifier le périmètre.

**Corrélation source_type ↔ intent** : les chunks de type `sap_help` sont statistiquement plus pertinents pour les questions de type « dépannage » ou « configuration » (recherche d'un code d'erreur, paramètre système), tandis que les chunks de type `technical_book` répondent mieux aux questions de type « explication » ou « procédure ». Cette corrélation a été exploitée dans le mécanisme de `source_hint` du module NLP : l'intent détecté oriente le filtre `source_type` de la requête vectorielle.

**Corrélation longueur du chunk ↔ qualité du score de reranking** : les chunks trop courts (moins de 40 mots) obtiennent systématiquement des scores plus faibles au cross-encoder de reranking, indépendamment de leur pertinence réelle. Ce phénomène s'explique par le fait que le cross-encoder `ms-marco-MiniLM-L-6-v2` a été entraîné sur des passages de longueur « naturelle » — il pénalise implicitement les fragments très courts. Cette observation a conduit à l'introduction d'un filtre de longueur minimale lors de la segmentation.

**Corrélation breadcrumb_depth ↔ spécificité** : les chunks dont le `breadcrumb` présente une profondeur de 3 niveaux ou plus (ex : `["Chapitre 5", "Section 5.2", "Procédure 5.2.1"]`) sont plus spécifiques et répondent mieux aux questions précises. Les chunks de premier niveau (introduction de chapitre) sont plus utiles pour les questions conceptuelles. Cette relation a orienté la stratégie de chunking hiérarchique adoptée.

---

### 4.4.2 Interprétation

Ces corrélations ne sont pas de simples curiosités statistiques — elles ont directement informé les choix d'architecture du pipeline RAG.

La forte corrélation T-code/module a justifié la construction du dictionnaire `_TCODE_TO_MODULE` dans le module NLP : plutôt que de demander au LLM d'inférer le module, une simple table de correspondance déterministe suffit dans plus de 90 % des cas, avec un gain de latence significatif.

La corrélation source_type/intent a conduit à l'introduction du champ `source_hint` dans la structure de sortie du parseur de requête. Ce champ filtre les chunks récupérés vers la nature de documentation la plus adaptée, réduisant le bruit dans le contexte fourni au LLM.

La relation entre longueur de chunk et score de reranking a renforcé la décision de fixer la taille minimale d'un chunk à 25 caractères dans le module de synthèse (`_extract_key_sentences`), garantissant que les fragments trop courts sont écartés avant même d'atteindre le reranker.

---

## 4.5 Identification des Variables Pertinentes

À l'issue de cette phase d'exploration et d'analyse, six variables se dégagent comme déterminantes pour la qualité de récupération du système.

| Variable | Rôle | Impact |
|---|---|---|
| `embedding` (1024-dim) | Représentation sémantique dense | Qualité de la recherche vectorielle |
| `mentioned_tcodes` | Entités SAP nommées | Précision du filtrage par transaction |
| `module` | Périmètre fonctionnel | Réduction du bruit de récupération |
| `source_type` | Nature de la documentation | Adéquation type-question/type-source |
| `section_type` | Nature du contenu | Distinction procédure vs. concept |
| `breadcrumb` | Profondeur hiérarchique | Spécificité du chunk |

Ces six variables forment le socle du moteur de filtrage hybride : les deux premières opèrent dans l'espace vectoriel continu, les quatre suivantes dans l'espace structuré des métadonnées. Leur combinaison permet de restreindre l'espace de recherche avant même de calculer les similarités cosinus, ce qui améliore à la fois la précision et la performance.

---

## 4.6 Hypothèses Issues des Données

L'analyse du corpus conduit à formuler plusieurs hypothèses vérifiables qui ont guidé les choix d'implémentation du système.

**H1 — La combinaison recherche dense + BM25 surpasse la recherche dense seule** : les T-codes SAP sont des termes rares et spécifiques que les embeddings généralistes ne représentent pas bien dans l'espace vectoriel. La recherche par mots-clés (BM25) devrait significativement améliorer le rappel sur les requêtes contenant des T-codes explicites. Cette hypothèse est soutenue par la littérature sur les systèmes RAG hybrides et a motivé l'implémentation de la fusion RRF (Reciprocal Rank Fusion) avec une pondération asymétrique en faveur de BM25 (60 % BM25, 40 % vectoriel).

**H2 — Le reranking par cross-encoder améliore la précision sur les questions complexes** : la recherche vectorielle optimise la similarité sémantique globale, pas la pertinence question/réponse. Un cross-encoder qui lit simultanément la question et le chunk devrait mieux capturer la pertinence fine. Le seuil `MIN_RERANK_SCORE = 2.0` a été fixé empiriquement pour équilibrer précision et rappel.

**H3 — L'expansion des synonymes T-code réduit les échecs de récupération** : sans expansion, une requête « comment surveiller les jobs en arrière-plan » ne retrouvera pas un chunk qui parle exclusivement de « SM37 ». L'ajout de la description textuelle du T-code au chunk au moment de l'indexation devrait combler ce gap lexical.

**H4 — La pré-synthèse par extraction de phrases améliore la cohérence des réponses** : fournir au LLM de génération un contexte déjà filtré (les 3 phrases les plus pertinentes par chunk plutôt que l'intégralité du chunk) devrait produire des réponses plus ciblées et moins bruitées, en réduisant la dilution sémantique du contexte.

---

## 4.7 Conclusion

Ce chapitre a permis de poser les bases empiriques du projet. Le corpus constitué de 142 798 chunks issus de 297 livres de formation SAP et du portail d'aide représente une base documentaire solide, à condition de maîtriser ses biais : déséquilibre entre modules, gap vocabulaire du modèle d'embedding, et bruit résiduel de l'extraction PDF.

L'analyse des relations entre variables a révélé des corrélations déterministes (T-code ↔ module) et probabilistes (source_type ↔ intent) qui ont directement informé l'architecture du pipeline de récupération. Les six variables identifiées comme pertinentes forment le squelette du moteur hybride dense/sparse qui sera décrit dans le chapitre suivant.

Les hypothèses formulées — combinaison hybride, reranking cross-encoder, expansion de synonymes, pré-synthèse — ne sont pas de simples intuitions : elles découlent directement des caractéristiques observées des données et constituent le fil directeur des évaluations quantitatives présentées dans le chapitre d'expérimentation.

La qualité d'un système RAG ne se joue pas uniquement dans l'architecture de récupération. Elle commence ici — dans la rigueur avec laquelle on comprend, nettoie et structure les données avant même d'écrire la première ligne du pipeline.
