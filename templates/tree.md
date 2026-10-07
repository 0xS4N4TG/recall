# {{project_name}}

Last touched: {{date}}

## Features

<!--
Format per feature:
- **<name>** [status] — one-line summary of current state
  - paths: `glob/**`, `another/glob/**`
  - last: YYYY-MM-DD (session-slug)

Sub-features (from `/recall new <feature> <parent>`) nest one level deeper under
their parent, same 3-line format. Directory layout stays flat
(features/<child>/); nesting lives only in this file.

Status values: active | stable | blocked | archived
Keep each feature to 3 lines. The whole file should stay under ~5 KB even with 50+ features.
-->

- **example-feature** [active] — short summary of where this feature stands right now
  - paths: `src/example/**`
  - last: {{date}} (initial)
  - **example-subfeature** [active] — child of example-feature
    - paths: `src/example/subpart/**`
    - last: {{date}} (initial)
