# validate_hierarchy

Post-install / post-upgrade validation for the CAS → PS1 → SEC SCCM hierarchy.

This role runs a PowerShell walk of the whole hierarchy from the SMS Provider and
fails the play if anything critical is wrong. It's meant to run **last** (after
`install_secondary_site`), and it's especially useful to re-run **after an in-console
upgrade** (e.g. to 2503) to confirm every tier actually moved to the new build and
replication is still healthy — the kind of "half-upgraded hierarchy" that is easy to
miss by eye.

## What it checks

1. **SMS Provider connectivity** — the CAS and PS1 providers answer WMI.
2. **Sites & relationships** — `CAS` (Type 4), `PS1` (Type 2, child of CAS), `SEC`
   (Type 1, child of PS1) all exist with the correct type and parent.
3. **Versions** — every site reports the same build (catches a CAS that upgraded while
   PS1/SEC lagged). Optionally pin an exact expected build.
4. **Site server services** — `SMS_EXECUTIVE` and `SMS_SITE_COMPONENT_MANAGER` running
   on each site server (passive node checked as a warning).
5. **Site system roles** — SQL, Service Connection Point (Dmp Connector), Management
   Point, Distribution Point, SMS Provider, and the passive site server are installed
   on the expected hosts across all three sites.
6. **Database replication** — every `SMS_ReplicationLink` is Active.
7. **Component health** — no components in a Critical state on any site.
8. **Update packages** — informational dump of `SMS_CM_UpdatePackages` (name/version/state).

Each item prints `[ OK ]`, `[WARN]`, or `[FAIL]`. Any `[FAIL]` fails the play (unless
`ludus_sccm_validate_fail_on_error: false`).

## Where it runs

On the PS1 primary site server (`ps1-pss`), which already has the console and the
hierarchy `role_vars` in scope. It queries the CAS provider for hierarchy-wide data
(sites, roles, replication) and reaches out to each site server over WMI.

## Key variables

See `defaults/main.yml`. In the standard lab you don't need to set anything — the
expected topology defaults to the same `ludus_sccm_*` variables the install roles use.

| Variable | Purpose | Default |
|---|---|---|
| `ludus_sccm_validate_expected_version` | Require every site to be exactly this build (e.g. `5.00.9135.1000` for 2503). Empty = only require consistency. | `""` |
| `ludus_sccm_validate_require_version_consistency` | Fail if sites are on mixed versions. | `true` |
| `ludus_sccm_validate_replication_must_be_active` | Treat non-Active replication links as a hard failure. | `true` |
| `ludus_sccm_validate_fail_on_error` | Fail the play on any critical error (set `false` to report only). | `true` |
| `ludus_sccm_validate_retries` / `_delay_seconds` | Retry loop so replication/components can settle after an install or upgrade. | `10` / `30` |

## Example (pin the post-upgrade build)

```yaml
- name: mayyhem.ludus_sccm.validate_hierarchy
  depends_on:
    - vm_name: "{{ range_id }}-ps1-pss"
      role: mayyhem.ludus_sccm.install_secondary_site
  role_vars:
    ludus_sccm_validate_expected_version: "5.00.9135.1000"
```
