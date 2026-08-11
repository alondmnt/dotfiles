# External evidence for PR review

Load this only when a load-bearing PR claim depends on evidence outside the repository. The
goal is to verify that claim or boundary contract, not perform a full scientific review.
Sources include object stores; data platforms and catalogues; experiment trackers and model
registries; and databases, jobs, logs, or APIs.

Use least-privilege, read-only access. Pin the source with its URI, table/model/run identity,
workspace or region, and strongest available version, checksum, snapshot, or observation
time. Start with the smallest decisive check: metadata, schema, counts, stable identifiers,
completion state, or representative records. Treat a local export as authoritative only when
tied to that source.

Never mutate external state or expose credentials or sensitive records. If access or
identity is unavailable, mark the claim `unverified` and name the read-only check needed.
