# External evidence for experiment review

Load this only when a decision-grade claim depends on evidence outside the repository.
Sources include object stores; data platforms and catalogues; experiment trackers and model
registries; and databases, jobs, logs, or APIs.

Use least-privilege, read-only access. Pin the artifact with its URI, table/model/run
identity, workspace or region, and strongest available version, checksum, snapshot, or
observation time. Check metadata, schema, counts, and stable identifiers before retrieving
or recomputing more than needed.

Re-derive the cohort, estimand, comparator, metric, uncertainty, and gate result from that
source, tying any export back to it. Never mutate external state or expose sensitive data. If
access or identity is unavailable, mark the claim `unverified` and name the read-only check
needed.
