# expense-config

GitOps config repo for `expense-api`. Kustomize overlays (`dev`, `staging`,
`prod`) built on the base K8s manifests from `expense-project`'s W5D3 work,
plus Argo CD `AppProject`/`Application`/`ApplicationSet` definitions.

See `expense-project`'s `.github/PIPELINE.md` for how this fits into the
overall CI/CD pipeline.
