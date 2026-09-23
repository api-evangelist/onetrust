# Superseded collections

Every Postman / OpenCollection file in this directory was generated from the seven hand-authored
scaffold OpenAPIs now archived in `openapi/_superseded/`. They describe ~15 operations with generic
`additionalProperties: true` bodies.

On 2026-08-27 the enrichment pipeline harvested OneTrust's 37 real, publicly downloadable OpenAPI
definitions (631 operations) from the developer portal. These collections do not reflect them and must
not be re-registered in `apis.yml`. Regenerate from `openapi/*.json` before publishing collections
again.

OneTrust publishes no first-party Postman workspace or collection — searched 2026-08-27 against the
Postman public network; the only "OneTrust" workspaces are third-party (Postman's own Atlas catalog
team and an unaffiliated user team). No `Postman` pointer is wired in `apis.yml` for that reason.
