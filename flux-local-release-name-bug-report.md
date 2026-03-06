# flux-local ignores HelmRelease.spec.releaseName when rendering Helm charts

## Summary

`flux-local build hr` appears to ignore `HelmRelease.spec.releaseName` and instead renders Helm templates using `HelmRelease.metadata.name` as `.Release.Name`.

This produces different manifest names from the live Flux `helm-controller` reconciliation.

## Expected behavior

When a `HelmRelease` sets `spec.releaseName`, `flux-local build hr` should render templates with that value as Helm's release name, matching `helm-controller`.

## Actual behavior

Given:

- `metadata.name: kube-state-metrics`
- `spec.releaseName: kube-state-metrics-release`

`flux-local build hr` renders a `Deployment` named:

- `kube-state-metrics`

But the live cluster, reconciled by Flux, creates:

- `kube-state-metrics-release`

## Why this looks wrong

The chart uses Helm's standard fullname helper based on `.Release.Name`:

```gotemplate
{{- define "kube-state-metrics.fullname" -}}
{{- if .Values.fullnameOverride -}}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- $name := default .Chart.Name .Values.nameOverride -}}
{{- if contains $name .Release.Name -}}
{{- .Release.Name | trunc 63 | trimSuffix "-" -}}
{{- else -}}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" -}}
{{- end -}}
{{- end -}}
{{- end -}}
```

With `spec.releaseName: kube-state-metrics-release`, the rendered deployment name should therefore be `kube-state-metrics-release`.

## Example HelmRelease

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: kube-state-metrics
  namespace: flux-system
spec:
  targetNamespace: datakit
  chart:
    spec:
      chart: apps/id1/kube-state-metrics/chart
      sourceRef:
        kind: GitRepository
        name: flux-system
        namespace: flux-system
  releaseName: kube-state-metrics-release
```

## Command used

```bash
flux-local --log-level ERROR build hr "kube-state-metrics" -n flux-system --path="./clusters/id1" \
  | yq eval 'del(.metadata.annotations."config.kubernetes.io/index", .metadata.annotations."internal.config.kubernetes.io/index")' -
```

## Observed cluster state

```bash
k get deploy -n datakit kube-state-metrics-release
NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
kube-state-metrics-release   1/1     1            1           52d
```

## Repository evidence

- The `HelmRelease` object name is `kube-state-metrics`.
- The same `HelmRelease` explicitly sets `spec.releaseName: kube-state-metrics-release`.
- The overlay only changes the chart path and does not patch `releaseName`.

## Expected fix

`flux-local build hr` should pass `HelmRelease.spec.releaseName` through to Helm rendering so `.Release.Name` matches the value used by Flux `helm-controller`.

## Minimal repro idea

1. Create a `HelmRelease` with `metadata.name: foo`.
2. Set `spec.releaseName: bar`.
3. Use a chart template that renders `.Release.Name`.
4. Compare:
   - `flux-local build hr foo ...`
   - live reconciliation by Flux `helm-controller`

If `flux-local` renders `foo` while the cluster renders `bar`, the tool is not honoring `spec.releaseName`.
