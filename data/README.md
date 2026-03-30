# Data

## samples/
Small representative samples of the crawled data:
- `seed_artists.csv` — 20 seed artists from MusicBrainz
- `seed_albums.csv` — 160 albums
- `ner_entities.csv` — 86 NER-extracted entities

## Full dataset
The full expanded KB (51,806 triples) is too large for GitHub.
Generate it by running the notebooks in order:
1. `notebooks/step0_crawler_ner.ipynb`
2. `notebooks/step1_build_knowledge_base.ipynb`
3. `notebooks/step2_entity_linking.ipynb`
4. `notebooks/step3_predicate_alignment.ipynb`
5. `notebooks/step4_kb_expansion.ipynb`

This will produce `music_kg_triples.tsv`, `music_kg_kge.ttl`, etc.
