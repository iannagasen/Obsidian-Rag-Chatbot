# Ingestion Workflow — Setup Steps

Builds the **"Ingest Obsidian Vault"** n8n workflow that reads markdown files, chunks them, embeds via Ollama, and upserts to Qdrant.

Open n8n at `http://localhost:5678` → New Workflow.

---

## Node 1 — Manual Trigger

Drag in a **Manual Trigger** node. No configuration needed.

---

## Node 2 — Read/Write Files from Disk

- **Operation:** `Read File(s) From Disk`
- **File Selector:** `/data/vault/**/*.md`

---

## Node 3 — IF (filter excluded paths)

Filters out system/personal folders before embedding. Use **AND** logic with all four conditions:

| Field | Operation | Value |
|---|---|---|
| `{{ $binary.data.directory + '/' + $binary.data.fileName }}` | does not contain | `.obsidian` |
| `{{ $binary.data.directory + '/' + $binary.data.fileName }}` | does not contain | `.trash` |
| `{{ $binary.data.directory + '/' + $binary.data.fileName }}` | does not contain | `Personal` |
| `{{ $binary.data.directory + '/' + $binary.data.fileName }}` | does not contain | `.agent` |

Connect the **True** branch to the Qdrant node.

---

## Node 4 — Qdrant Vector Store

- **Operation:** `Insert Documents`
- **Collection:** `obsidian_vault`
- **Credential:** New → Qdrant API
  - URL: `http://qdrant:6333`
  - API Key: *(leave blank)*

### Sub-node: Embeddings Ollama

Attach via the `+` circle at the bottom of the Qdrant node.

- **Credential:** New → Ollama API
  - Base URL: `http://ollama:11434`
- **Model:** `nomic-embed-text:latest`

### Sub-node: Default Data Loader

Attach via the `+` circle at the bottom of the Qdrant node.

- **Type of Data:** `Binary`
- **Mode:** `Load All Input Data`
- **Data Format:** `Text`
- **Options → Metadata → Add field:**
  - Name: `source`
  - Value: `={{ $('Read/Write Files from Disk').item.binary.data.directory + '/' + $('Read/Write Files from Disk').item.binary.data.fileName }}`

> `fileName` alone is just the base name (e.g. `note.md`). Concatenating `directory` gives the full path so retrieved chunks can be traced back to their source file.

### Sub-node: Recursive Character Text Splitter

Attach via the `+` circle at the bottom of the Default Data Loader sub-node.

- **Chunk Size:** `1000`
- **Chunk Overlap:** `200`

---

## Run & Verify

1. Save the workflow, then click **Execute Workflow**.
2. Ingestion runs for ~30–60 minutes (2,293 files).
3. Monitor progress in a terminal:

```bash
watch -n 5 "curl -s http://localhost:6333/collections/obsidian_vault | python3 -m json.tool | grep points_count"
```

4. Spot-check that source metadata is populated:

```bash
curl -s -X POST http://localhost:6333/collections/obsidian_vault/points/scroll \
  -H 'Content-Type: application/json' \
  -d '{"limit": 2, "with_payload": true, "with_vector": false}' | python3 -m json.tool
```

Each point should have a non-null `payload.source` field containing the file path. If it's null, fix the metadata expression in the Default Data Loader and re-run.
