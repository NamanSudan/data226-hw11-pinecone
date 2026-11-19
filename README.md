# DATA 226 – Homework 11: Medium search engine with Pinecone on Airflow

This repo runs the week 13 Pinecone job as an Airflow DAG (`Medium_to_Pinecone`).
The DAG downloads about 2,500 Medium article titles, embeds them with
`all-MiniLM-L6-v2` from sentence-transformers, stores the vectors in a Pinecone
serverless index, and runs a test search.

## Files

| File | Purpose |
|---|---|
| `docker-compose.yaml` | Class Airflow 2.10.1 compose file, with `sentence-transformers==3.4.1` and `pinecone==5.4.2` added to `_PIP_ADDITIONAL_REQUIREMENTS` |
| `dags/build_pinecone_search.py` | The DAG |

## DAG tasks

```
download_data ──> preprocess_data ──┐
                                    ├──> generate_embeddings_and_upsert ──> test_search_query
create_pinecone_index ──────────────┘
```

1. `download_data` downloads `medium_data.csv` to `/tmp/medium_data`.
2. `preprocess_data` builds `title + subtitle` metadata and string ids.
3. `create_pinecone_index` drops and recreates the `semantic-search-fast` index
   (384 dimensions, dotproduct, AWS us-east-1 serverless).
4. `generate_embeddings_and_upsert` embeds the titles in batches of 100 and
   upserts them.
5. `test_search_query` searches for "what is ethics in AI" and logs the top 5 matches.

## Run it

```bash
export AIRFLOW_UID=$(id -u)
docker compose down
docker compose up -d     # the first start pip-installs torch and takes a few minutes
```

Open http://localhost:8081 (login `airflow` / `airflow`). Add the Airflow Variable
`pinecone_api_key` under Admin → Variables, with your Pinecone API key as the value. The key is
not stored in this repo. Then unpause and trigger `Medium_to_Pinecone`.

Unpausing also starts the most recent scheduled run. If you trigger a manual run at the same
time, both runs recreate the same index, and one `create_pinecone_index` attempt may fail with a
404 before its automatic retry succeeds.
