# {{feature_name}} — decisions

<!--
Why things are the way they are. Append-only-ish: add new entries, don't rewrite old ones unless they're proven wrong (then mark them so).
Each entry: short title, the call, the reason, the date.
-->

## Example: chose X over Y
**Date:** {{date}}
**Call:** Use X.
**Why:** Y had a problem with Z; X handles it natively.
**Tradeoff:** X is heavier at startup but the request path stays simple.
