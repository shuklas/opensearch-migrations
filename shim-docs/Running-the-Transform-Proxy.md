# Running the Transformation Shim as a Solr-to-OpenSearch Transform Proxy

The Transformation Shim is an HTTP proxy that translates Solr API requests into OpenSearch `_search` API calls and converts responses back to Solr format. Your Solr clients continue speaking Solr — the shim handles the translation transparently.

The Docker image comes with Solr-to-OpenSearch transforms pre-bundled at `/transforms/`.

## How It Works

```
Solr Client                    Shim Proxy                     OpenSearch
    |                              |                              |
    |-- GET /solr/col/select ----->|                              |
    |                              |-- POST /col/_search -------->|
    |                              |<-- { hits: { ... } } --------|
    |<-- { response: { docs: [] } }|                              |
```

The shim rewrites:
- `/solr/{collection}/select?q=...` → `POST /{collection}/_search` with a JSON query body
- OpenSearch `hits.hits[]._source` → Solr `response.docs[]` format
- Adds a synthetic `responseHeader` with `status` and `QTime`

## Prerequisites

- Docker
- An OpenSearch cluster accessible from the shim container

## Pull the Image

```bash
docker pull public.ecr.aws/opensearchproject/opensearch-migrations-transformation-shim:latest
```

See [ECR](https://gallery.ecr.aws/opensearchproject/opensearch-migrations-transformation-shim) for a list of all available versions.

## Run the Shim

```bash
docker run -p 8080:8080 \
  public.ecr.aws/opensearchproject/opensearch-migrations-transformation-shim:latest \
  --listenPort 8080 \
  --target opensearch=http://your-opensearch-host:9200 \
  --primary opensearch \
  --transformTarget opensearch \
  --transformer-config '[{"SolrTransformerProvider":{"initializationScriptFile":"/transforms/solr-to-opensearch-request.js","bindingsObject":"{}"}}]' \
  --response-transformer-config '[{"SolrTransformerProvider":{"initializationScriptFile":"/transforms/solr-to-opensearch-response.js","bindingsObject":"{}"}}]'
```

Replace `your-opensearch-host:9200` with the address of your OpenSearch cluster.

### With OpenSearch Authentication

Add `--targetAuth` for authenticated clusters:

**AWS SigV4:**
```bash
docker run -p 8080:8080 \
  -e AWS_ACCESS_KEY_ID \
  -e AWS_SECRET_ACCESS_KEY \
  -e AWS_SESSION_TOKEN \
  public.ecr.aws/opensearchproject/opensearch-migrations-transformation-shim:latest \
  --listenPort 8080 \
  --target opensearch=https://your-opensearch-domain.us-east-1.es.amazonaws.com \
  --primary opensearch \
  --transformTarget opensearch \
  --transformer-config '[{"SolrTransformerProvider":{"initializationScriptFile":"/transforms/solr-to-opensearch-request.js","bindingsObject":"{}"}}]' \
  --response-transformer-config '[{"SolrTransformerProvider":{"initializationScriptFile":"/transforms/solr-to-opensearch-response.js","bindingsObject":"{}"}}]' \
  --targetAuth opensearch=sigv4:es,us-east-1
```

**Basic auth:**
```bash
  --targetAuth opensearch=basic:admin:yourpassword
```

### With TLS (self-signed certs)

Add `--insecureBackend` to trust self-signed certificates on the OpenSearch backend.

## Verify It Works

Send a standard Solr query through the shim:

```bash
curl -s "http://localhost:8080/solr/myindex/select?q=*:*&wt=json" | python3 -m json.tool
```

You should get a Solr-format response with data from OpenSearch:

```json
{
    "responseHeader": {
        "status": 0,
        "QTime": 0
    },
    "response": {
        "numFound": 3,
        "start": 0,
        "docs": [
            { "id": "1", "title": "Introduction to OpenSearch" },
            { "id": "2", "title": "Migrating from Solr" }
        ]
    }
}
```

Check the response headers for diagnostic info:

```bash
curl -sD- "http://localhost:8080/solr/myindex/select?q=*:*&wt=json" 2>&1 | grep -i "x-"
```

```
X-Shim-Primary: opensearch
X-Shim-Targets: opensearch
X-Target-opensearch-StatusCode: 200
X-Target-opensearch-Latency: 45
```

## CLI Reference

| Flag | Required | Description |
|------|----------|-------------|
| `--listenPort <port>` | Yes | Port the shim listens on |
| `--target <name=uri>` | Yes | Named backend target |
| `--primary <name>` | Yes | Target whose response is returned to the client |
| `--transformTarget <name>` | No | Target to apply transforms to |
| `--transformer-config <json>` | No | Request transformer config JSON array |
| `--response-transformer-config <json>` | No | Response transformer config JSON array |
| `--targetAuth <spec>` | No | Auth: `sigv4:service,region`, `basic:user:pass`, `header:value`, `none` |
| `--insecureBackend` | No | Trust all backend TLS certificates |
| `--timeout <ms>` | No | Target timeout in milliseconds (default: 30000) |

## What's Supported

The bundled Solr-to-OpenSearch transforms handle:
- `/solr/{collection}/select` queries with `q=*:*` and basic query strings
- Response format conversion (hits → docs)
- Synthetic `responseHeader` generation

Non-select endpoints (update, admin, etc.) are passed through without transformation.

For the full list of supported Solr query features, see [[Solr Query Translation Shim]].

## Related Pages

- [[Solr Query Translation Shim]] — Supported query types, validation modes, and known limitations
- [[Solr Migration Overview]] — End-to-end Solr migration guide
