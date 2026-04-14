# Migration Assistant for OpenSearch

Migration Assistant is a tool for migrating data from Elasticsearch, OpenSearch, and Apache Solr clusters to OpenSearch. It provides a Kubernetes-native, workflow-driven approach to orchestrate migrations using declarative YAML configuration.

> **Important**: The configuration schema changes between Migration Assistant versions. Do not copy YAML examples from documentation verbatim. After deploying, run `workflow configure sample` on the Migration Console to get the accurate schema for your installed version.

## Already deployed?

Reconnect to your Migration Console:

```bash
# Set kubectl context to your EKS cluster
aws eks update-kubeconfig --region <REGION> --name migration-eks-cluster-<STAGE>-<REGION>

# Connect to the Migration Console pod
kubectl exec -it migration-console-0 -n ma -- /bin/bash
```

Or resume / start a Kiro AI assistant session:

```bash
# Bootstrap a new Kiro agent session (installs Kiro CLI if needed)
curl -sL -H 'Accept: application/vnd.github.raw' \
  'https://api.github.com/repos/opensearch-project/opensearch-migrations/contents/bootstrap-kiro-agent.sh?ref=kiro-agent' | bash
```

Once on the Migration Console, check your version and schema:

```bash
console --version                # Check your installed version
workflow configure sample        # See the configuration schema for your version
```

Skip ahead to:
1. [[Workflow CLI Overview]] - Understand the concepts
2. [[Workflow CLI Getting Started]] - Configure and run your migration

## Key capabilities

| Capability | Description |
|------------|-------------|
| **Metadata migration** | Migrate index templates, component templates, index settings, and aliases |
| **Document backfill** | Migrate existing documents using snapshot-based reindexing (RFS) |
| **Version compatibility** | Support for a range of Elasticsearch and OpenSearch source versions to OpenSearch targets |
| **Amazon OpenSearch Serverless** | Supported as a migration target for document backfill and index metadata |
| **Solr query translation** | Transparent HTTP proxy (Transformation Shim) that converts Solr API requests to OpenSearch, enabling incremental cutover without application changes |
| **Solr document backfill** | Migrate documents from Solr backups using SolrReader with automatic schema translation |

## Architecture overview

Migration Assistant runs on Kubernetes and uses Argo Workflows for orchestration. The diagram below shows the architecture on AWS EKS, but Migration Assistant works equivalently on any Kubernetes distribution including GKE, AKS, OpenShift, and self-managed Kubernetes clusters.

![Migration Assistant Architecture](diagrams/eks-architecture.svg)

For a detailed walkthrough of the migration process, see the [[Architecture]] page.

### Core components

- **Migration Console**: A Kubernetes pod that provides the interface for configuring, submitting, and monitoring migrations
- **Workflow CLI**: A command-line tool within the Migration Console for defining migrations in YAML and submitting them as workflows
- **Argo Workflows**: Kubernetes-native workflow engine that orchestrates migration tasks
- **RFS (Reindex-from-Snapshot)**: High-performance document migration using Lucene segment files

## Migration Console orientation

When you connect to the Migration Console pod, you'll find:

- **`console`** - Main CLI for migration operations and status
- **`workflow`** - Configure and submit migration workflows
- **`kubectl`** - Pre-configured for the migration namespace
- **`aws` CLI** - Available on EKS deployments for AWS operations

The pod has network access to both your source and target clusters. Your workflow configurations are stored in the pod's filesystem and submitted to Argo Workflows for execution.

Connect to the Migration Console using:
```bash
kubectl exec -it deployment/migration-console -n <namespace> -- bash
```

## Getting started

### 1. Understand what a migration involves

Read [[What Is a Migration]] to understand the three aspects of a migration (metadata, backfill, capture and replay), how Reindex-from-Snapshot works, and how to choose your migration scenario.

### 2. Check your version

Run `console --version` and `workflow configure sample` to understand your installed version's capabilities and configuration schema.

### 3. Understand the architecture

Review the [[Architecture]] page to understand the end-to-end migration process and components.

### 4. Check compatibility

Review the [[Migration Paths]] page to verify your source and target versions are supported.

### 5. Deploy Migration Assistant

Choose your deployment option:
- [[Deploying to Kubernetes]] - Generic Kubernetes deployment
- [[Deploying to EKS]] - AWS EKS-specific deployment with IAM integration

### 6. Run your migration

Configure and execute your migration:
- [[Workflow CLI Overview]] - Concepts and architecture
- [[Workflow CLI Getting Started]] - Step-by-step tutorial
- [[Backfill Workflow]] - Document and metadata migration

> **Migrating from Apache Solr?** See [[Solr Migration Overview]] for the Solr-specific migration approach, which uses a different architecture than Elasticsearch migrations.

### Query OpenSearch with your existing Solr queries

> **Preview**: Solr query translation is currently available as a preview feature. This release comes with pre-bundled transforms for common Solr query patterns and does not support custom transformations. Full support, including custom transforms and additional query coverage, will be available in a future release.

If you're migrating from Solr, you don't need to rewrite your application's queries. The [[Solr Query Translation Shim]] is an HTTP proxy that accepts Solr API requests (`/solr/{collection}/select?q=...`) and translates them to OpenSearch `_search` API calls on the fly. Responses are converted back to Solr format, so your existing clients work without code changes.

Point your Solr clients at the shim and start querying your OpenSearch cluster immediately — no application changes required. See [[Running the Transform Proxy]] for a step-by-step guide on pulling the image and running the shim with Docker.

## What's included

Migration Assistant focuses on snapshot-based migrations:

| Feature | Included |
|---------|----------|
| Index templates | ✓ |
| Component templates | ✓ |
| Index settings | ✓ |
| Index mappings | ✓ |
| Aliases | ✓ |
| Documents | ✓ |
| ISM policies | ✗ |
| Security configuration | ✗ |
| Kibana/Dashboards objects | ✗ |

## Getting help

- [[Troubleshooting]] - Common issues and solutions
- [GitHub Issues](https://github.com/opensearch-project/opensearch-migrations/issues) - Report bugs or request features
- [OpenSearch Forum](https://forum.opensearch.org/) - Community discussions
