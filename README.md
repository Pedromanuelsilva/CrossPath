# CrossPath

CrossPath is a Tika-compatible proxy service for an internal enterprise search platform built around Apache ManifoldCF and OpenSearch.

This repository is only for the proxy layer, not the full search platform. The broader system uses ManifoldCF to crawl Windows file shares, preserve Active Directory-backed ACL visibility, extract document content, and send indexed content to OpenSearch for search portal use.

Within that larger architecture, CrossPath sits between ManifoldCF's `tikaservice` transformation connector and the actual extraction backends. It exposes Tika-compatible endpoints so ManifoldCF can treat it like a normal Tika service, while CrossPath applies routing and fallback rules behind the scenes.

## Role In The Larger System

At a high level, the full platform looks like this:

`Windows SMB shares -> ManifoldCF -> OpenSearch -> search portal`

Within ManifoldCF's extraction path, CrossPath sits between the `tikaservice` transformation connector and the extraction backends:

`ManifoldCF tikaservice -> CrossPath -> Docling / Tika`

CrossPath is responsible for:

- exposing Tika-compatible endpoints: `PUT /meta`, `PUT /tika`, `PUT /detect/stream`
- routing documents to the best extraction backend based on file type and policy
- using Docling for formats where richer structured extraction is preferred
- falling back to Apache Tika for legacy, unsupported, or failed Docling cases
- returning Tika-compatible responses back to ManifoldCF

## Scope

In scope for this repository:

- the Python proxy service
- compatibility with ManifoldCF `tikaservice`
- routing, buffering, fallback, normalization, and observability
- orchestration of Docling Serve and Tika Server as external services

Out of scope for this repository:

- crawling Windows shares
- Active Directory authority handling
- OpenSearch indexing logic
- search portal / UI
- end-user authorization model outside the metadata and ACL information already handled by the surrounding platform

## Design Direction

CrossPath is intended to remain a small compatibility and orchestration layer rather than a heavy parser itself.

Current direction:

- runtime: Python 3.13+
- framework: FastAPI
- deployment: containerized
- extractors: Docling Serve and Apache Tika as external services

Longer term, the proxy may evolve into a smarter document normalization layer for known formats such as spreadsheets and structured exports, while still returning plain text and metadata in a Tika-compatible way for ManifoldCF.
