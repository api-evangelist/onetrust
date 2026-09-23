# Superseded scaffold specs

The seven `onetrust-*-api-openapi.yml` files here (and `onetrust-openapi-scaffold.yml`,
the master they were split from) were hand-authored summaries of the OneTrust
developer documentation, not harvested contracts. Together they described ~15
operations with `additionalProperties: true` generic responses.

On 2026-08-27 the enrichment pipeline located the real, publicly downloadable
OneTrust OpenAPI definitions on the developer portal's ReadMe hub:

    https://developer.onetrust.com/onetrust/openapi/<definition>.json

37 definitions, 631 operations, all HTTP 200, all verbatim. They now live at the
top level of `openapi/`. These scaffolds are retained only as a record of what
the profile used to claim and MUST NOT be re-registered in apis.yml.
