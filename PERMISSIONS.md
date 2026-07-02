# Permissions — OneLake → Azure AI Search (managed identity)

`onelake_content_understanding_indexer.ipynb` authenticates entirely with Entra ID —
no API keys. Two identities must be granted the right roles or the pipeline fails.

## 1. You (the person running the notebook)

Your interactive Entra identity (`az login` / VS Code sign-in) authorizes the REST +
query calls to Azure AI Search.

| Role | Scope | Why |
|---|---|---|
| **Search Service Contributor** | Search service | Create/update data source, index, skillset, indexer |
| **Search Index Data Contributor** | Search service | Write documents and run the query cell |

Also required: on the search service, **Settings → Keys → API access control** must be
**"Role-based access control"** (or "Both"). If it is key-only, bearer tokens are rejected.

## 2. The Search service's system-assigned managed identity

This is the identity the indexer/skillset use to reach every downstream resource. Enable it
under **Search service → Settings → Identity → System assigned = On**, then grant:

| Role | Scope (target resource) | Why |
|---|---|---|
| **Contributor** | Fabric workspace | Read files from OneLake (`onelake` data source) |
| **Cognitive Services User** | Foundry resource (`AIServicesByIdentity`) | Run the Content Understanding skill |
| **Cognitive Services OpenAI User** | Azure OpenAI resource | Embedding skill **and** the index vectorizer (query-time vectorization) |

Notes:
- The Fabric workspace grant is done **inside Fabric** (workspace → *Manage access* → add the
  search service managed identity as a member), not through Azure RBAC.
- OneLake also needs the tenant setting **"Allow apps running outside of Fabric to access data
  via OneLake"** = enabled.

## 3. Data-plane requirements (not RBAC, but produce the same errors)

- **AOAI**: the embedding deployment (e.g. `text-embedding-3-small`) must exist on the exact
  resource referenced by `resourceUri`. A missing deployment returns a **404**.
- **`resourceUri` / `subdomainUrl` must be bare origins** — no path/query. The notebook
  normalizes `AOAI_ENDPOINT` and `FOUNDRY_ENDPOINT` to their origin for this reason.
- **Foundry region**: must support Content Understanding for the modality (documents) you use.

## Symptom → cause quick map

| Error | Missing piece |
|---|---|
| `401`/`403` on `put`/`get` | Your roles (#1), or RBAC not enabled on the search service |
| `SubdomainUrl ... base subdomain` (400) | `FOUNDRY_ENDPOINT` had a path (normalized via `FOUNDRY_SUBDOMAIN`) |
| Vectorization `404` | Bad `resourceUri` path, or wrong/absent AOAI deployment |
| Vectorization `401`/`403` | Search MI lacks **Cognitive Services OpenAI User** |
| Content Understanding `401`/`403` | Search MI lacks **Cognitive Services User** on Foundry |
| OneLake read failure | Search MI not a Fabric workspace member, or tenant setting off |
