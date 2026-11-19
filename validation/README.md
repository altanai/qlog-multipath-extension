# Multipath `path_id` validation profile

This profile connects the multipath `PathID` field to complete qlog events and traces. Load `path-id.cddl` in the same CDDL compilation unit as the qlog main schema, the QUIC event schema, and the multipath event schema. The overlay depends on the main schema's `Event`, `Trace`, and `CommonFields` definitions; the composed `Event` retains the QUIC event data types supplied by the QUIC event schema.

Use `MultipathEvent` to validate a complete event. It checks the normal qlog event shape and protocol data, and constrains a top-level `path_id` if it is present. Use `PathScopedEvent` for an event that semantically describes one known path; that root requires the top-level field. Use `MultipathTrace` to validate `common_fields.path_id` when the trace uses the qlog common-fields optimization.

The CDDL profile checks field placement and type. A semantic validation pass must still decide whether an event describes one known path, whether the event is connection-wide, and whether one `common_fields.path_id` applies to every event in the trace. Resolve an event's effective Path ID from its own `path_id` first, then from `common_fields.path_id`. If a common field is used, all events in the trace must have that same effective Path ID; an event-level override to another path is invalid. A value found only inside `data` is not the event-level `path_id`.

## Reproducible examples

Run the standard-library-only semantic checks with:

```sh
python3 scripts/check_path_id_validation.py
```

The check covers the inclusive boundaries `0` and `4294967295`, an inherited common-field value, negative and overflowing integers, a missing field when the test marks the path context as known, and a field incorrectly placed only inside `data`. The profile is an overlay: use it with the pinned qlog main, QUIC, and multipath schemas when running a CDDL validator. CDDL alone cannot decide the semantic conditions listed above.
