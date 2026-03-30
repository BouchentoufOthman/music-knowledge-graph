"""# Music Knowledge Graph

> **Knowledge Engineering & Semantic Web — Final Project**

A complete end-to-end semantic web pipeline over the Music domain:
data collection → RDF knowledge base → Wikidata alignment → SPARQL expansion →
SWRL reasoning → knowledge graph embeddings → RAG question answering.

---

## Pipeline Overview

```
MusicBrainz API
      │
      ▼
[Step 0] Crawl + Clean + NER          → seed_artists.csv, seed_albums.csv
      │
      ▼
[Step 1] Build RDF Knowledge Base     → music_kb.ttl          (498 triples)
      │
      ▼
[Step 2] Entity Linking (Wikidata)    → music_kb_linked.ttl   (848 triples)
      │
      ▼
[Step 3] Predicate Alignment          → music_kb_aligned.ttl  (876 triples)
      │
      ▼
[Step 4] SPARQL Expansion             → music_kg_kge.ttl      (51,806 triples)
      │
      ▼
[Step 5] SWRL Reasoning (OWLReady2)   → inferred class memberships
      │
      ▼
[Step 6] KGE — TransE + RotatE        → train.txt / valid.txt / test.txt
      │
      ▼
[Step 7] RAG — Ollama llama3          → NL → SPARQL → answer
```

---

## Results at a Glance

| Metric | Value |
|--------|-------|
| Seed artists crawled | 20 (160 albums) |
| Final KB size | 51,806 triples · 42,032 entities · 439 relations |
| Entity alignments (owl:sameAs) | 67 (64 Wikidata + 3 DBpedia) |
| Predicate alignments | 14 (6 equivalent · 5 subProperty · 3 related) |
| TransE Hits@1 / Hits@10 | 0.097% / 1.94% |
| RotatE Hits@1 / Hits@10 | **1.93% / 3.45%** (20× better at rank 1) |
| RotatE Harmonic Mean Rank | 40.4 (vs 122.8 for TransE) |
| Size sensitivity Hits@10 | 20K: 3.7% → 50K: 18.4% → Full: **21.7%** |
| RAG success rate | 1/5 with schema injection vs 0/5 baseline |

---

## Repository Structure

```
music-knowledge-graph/
├── notebooks/                  # All Jupyter notebooks (run in order)
│   ├── step0_crawler_ner.ipynb
│   ├── step1_build_knowledge_base.ipynb
│   ├── step2_entity_linking.ipynb
│   ├── step3_predicate_alignment.ipynb
│   ├── step4_kb_expansion.ipynb
│   ├── step5_swrl_reasoning.ipynb
│   ├── step6_kge_pykeen.ipynb
│   └── step7_rag_ollama.ipynb
├── kg_artifacts/               # RDF files
│   ├── music_kb.ttl            # Seed KB (498 triples)
│   ├── music_kb_linked.ttl     # After entity linking
│   ├── music_kb_aligned.ttl    # After predicate alignment
│   ├── entity_mapping_table.csv
│   └── predicate_alignment_table.csv
├── data/
│   ├── samples/                # Small CSV samples (included)
│   │   ├── seed_artists.csv
│   │   ├── seed_albums.csv
│   │   └── ner_entities.csv
│   ├── train.txt               # KGE training split
│   ├── valid.txt               # KGE validation split
│   └── test.txt                # KGE test split
├── reports/
│   ├── final_report.pdf
│   ├── kge_results.csv
│   ├── rag_evaluation.csv
│   ├── size_sensitivity.png
│   └── tsne_embeddings.png
├── requirements.txt
├── .gitignore
└── README.md
```

---

## Installation

### 1. Clone the repository
```bash
git clone https://github.com/{GITHUB_USERNAME}/{REPO_NAME}.git
cd {REPO_NAME}
```

### 2. Create and activate a conda environment
```bash
conda create -n musickg python=3.11
conda activate musickg
```

### 3. Install Python dependencies
```bash
pip install -r requirements.txt
python -m spacy download en_core_web_sm
```

### 4. Install Ollama (for RAG — Step 7 only)
Download from [https://ollama.ai](https://ollama.ai), then:
```bash
ollama pull llama3
```

---

## How to Run

Open JupyterLab from the project root and run notebooks **in order**:

```bash
jupyter lab
```

| Notebook | What it does | Runtime |
|----------|-------------|--------|
| `step0_crawler_ner.ipynb` | Crawl MusicBrainz, clean, run NER | ~5 min (API rate limit) |
| `step1_build_knowledge_base.ipynb` | Build seed RDF KB | <1 min |
| `step2_entity_linking.ipynb` | Align entities to Wikidata/DBpedia | ~3 min |
| `step3_predicate_alignment.ipynb` | Align predicates via SPARQL | ~5 min |
| `step4_kb_expansion.ipynb` | Expand KB to 50K+ triples | ~30 min (SPARQL) |
| `step5_swrl_reasoning.ipynb` | SWRL rules — no Java needed | <1 min |
| `step6_kge_pykeen.ipynb` | Train TransE + RotatE | ~20 min (GPU) / ~60 min (CPU) |
| `step7_rag_ollama.ipynb` | RAG demo — requires Ollama running | interactive |

### Hardware requirements
- RAM: 8 GB minimum, 16 GB recommended (for Step 4 expansion)
- GPU: optional but strongly recommended for Step 6 (CUDA)
- Disk: ~500 MB for full expanded KB

### Running the RAG demo
1. Start Ollama in a separate terminal: `ollama serve`
2. Run all cells in `step7_rag_ollama.ipynb`
3. The last cell opens an interactive CLI — type any music question

Example questions:
```
Which record label is Adele signed to?
What genre does Kendrick Lamar play?
Which artists collaborated with Beyonce?
```

---

## Key Design Decisions

**RDF Modelling:** Two-namespace design (`mus:` for entities, `prop:` for predicates).
All literals are typed with XSD datatypes. `prop:collaboratedWith` is declared
`owl:SymmetricProperty`; `prop:releasedAlbum` and `prop:producedBy` are `owl:inverseOf`.

**Entity Linking:** Three-factor confidence score (label similarity 50%,
type keyword match 30%, rank position 20%). Threshold 0.55.
All 67 seed entities linked at ≥ 0.97 confidence.

**Predicate Alignment:** Two SPARQL strategies:
- Label keyword search: `FILTER(CONTAINS(LCASE(?propLabel), "keyword"))`
- Instance validation: `wd:Subject ?prop wd:Object`

**SWRL Reasoning:** Pellet-free Python forward-chaining (no Java dependency).
Produces semantically equivalent results to a full SWRL reasoner.

**KGE:** RotatE outperforms TransE on Hits@1 (1.93% vs 0.097%) because the KB
contains symmetric (`collaboratedWith`) and inverse (`releasedAlbum`/`producedBy`)
relation patterns that RotatE is specifically designed to handle.

**RAG:** Schema injection (auto-generated from RDF graph) is the critical component.
Without it the LLM generates wrong namespace prefixes and hallucinated property URIs.

---

## Technologies

| Component | Technology |
|-----------|------------|
| RDF / SPARQL | RDFLib 6, SPARQLWrapper |
| Open KB alignment | Wikidata API, DBpedia Spotlight |
| Ontology / Reasoning | OWLReady2 |
| Knowledge Graph Embeddings | PyKEEN 1.10, PyTorch 2.x |
| Embedding visualisation | scikit-learn t-SNE, matplotlib |
| LLM (RAG) | Ollama llama3 (local) |
| NER | spaCy en_core_web_sm |
| Data source | MusicBrainz API (CC0) |

---

## License

Code: MIT License  
Data: MusicBrainz data is CC0. Wikidata data is CC0.
"""
