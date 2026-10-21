# Grafana downgrade and future upgrade

The temporary downgrade matches the company chart version while integration
with the rest of the setup is verified. Once integration is proven, return to
the upgraded version below.

| Setting | Upgraded version (before this change / future target) | Downgraded version (current target) |
| --- | --- | --- |
| Helm dependency | `grafana` | `grafana` |
| Chart version | `14.0.1` | `10.5.15` |
| Repository | `https://grafana-community.github.io/helm-charts` | `https://grafana.github.io/helm-charts` |
| Default Grafana application version | `13.2.4` | `12.3.1` |

Chart and application versions are distinct. The application versions above
come from the chart metadata; no image tag is overridden in our values.

## Changes for the downgrade

In `charts/grafana/Chart.yaml`, set:

```yaml
dependencies:
  - name: grafana
    version: 10.5.15
    repository: https://grafana.github.io/helm-charts
```

The existing `values.yaml`, credentials Secret template, and PostgreSQL template
work with this chart version without changes. Keep `credentialsSecret`,
`grafana.admin.existingSecret`, and the database password secret reference
pointing to the same Secret. Credentials remain in `values.yaml`, and Helm
creates the Kubernetes Secret; no secret manager is needed.

From the repository root, regenerate and validate dependencies:

```sh
helm dependency update charts/grafana
helm lint charts/grafana
helm template grafana charts/grafana --namespace grafana > /tmp/grafana-rendered.yaml
```

The repository ignores `charts/*/Chart.lock` and `charts/*/charts/`. The lockfile
and downloaded dependency are refreshed locally; the committed dependency pin
in `Chart.yaml` is what Argo CD uses to resolve the chart.

Commit and push the chart and this document to `main`. The existing Argo CD
application uses `charts/grafana`, release name `grafana`, and namespace
`grafana`, with automatic sync enabled. Refresh and check the sync result.

## Existing database before a downgrade

If the upgraded application has already run, back up PostgreSQL and record the
deployed versions before syncing the downgrade. Pause automatic sync while
preparing any restore. A successful Helm render does not establish database
downgrade compatibility: the older application may not support migrations
already applied by the newer application.

For an existing installation, use a database backup from before the upgrade
when reverting, or validate the downgrade against a copy of the database first.
For a disposable Minikube installation, a fresh PostgreSQL database is another
option, but loses saved Grafana state. Deleting the application alone does not
remove the PostgreSQL PVC. Keep the original PVC/backup until validation is done.
Changing the password in `values.yaml` does not change the password of an
already initialized PostgreSQL database.

## Return to the upgraded version

1. Prove the company-version integration works: admin login, configured data
   sources, dashboard queries, alerting if configured, and persistence after
   restarting Grafana.
2. Back up PostgreSQL, retain the credentials, and record the deployed versions.
3. Change the dependency back to chart `14.0.1` and repository
   `https://grafana-community.github.io/helm-charts`.
4. Run the dependency update, lint, and render commands above. Review the
   rendered changes and the upstream chart and Grafana upgrade notes for the
   versions being crossed. No additional values changes are identified for
   this chart's current settings; validate behavior with the integrated setup.
5. Commit and push, check Argo CD sync and pod readiness, then repeat the
   integration and persistence checks. Check Grafana logs for migration errors.

If the upgrade fails after database migrations, reverting `Chart.yaml` alone
may be insufficient. Restore the corresponding pre-upgrade database backup
alongside the older application version.

## Validation of this change

Chart `10.5.15` passes `helm lint` and renders successfully with release name
`grafana`. The rendered manifests contain the credentials Secret, matching
Grafana/PostgreSQL password references, and `grafana-postgresql:5432` as the
database host. Live integration and database downgrade behavior require testing
in Minikube.

## Upstream references

- [Original Grafana Helm repository](https://grafana.github.io/helm-charts/)
- [Grafana chart 10.5.15 source](https://github.com/grafana/helm-charts/tree/grafana-10.5.15/charts/grafana)
- [Community Grafana Helm repository](https://grafana-community.github.io/helm-charts/)
- [Grafana upgrade guide](https://grafana.com/docs/grafana/latest/upgrade-guide/)
