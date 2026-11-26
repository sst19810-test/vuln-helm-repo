# Vulnerable Helm Test Fixture

Synthetic Helm chart repository for testing Helm detection and IaC security scanners such as Trivy, Checkov, and KICS.

> **Testing fixture only.** Use strictly for local or CI security scanner testing. Never deploy these manifests to a production or shared Kubernetes cluster.

The insecure defaults are intentional. The password is a fabricated test value, not a working credential. Keep these misconfigurations intact when testing scanner behavior.

## Repository Layout

```text
charts/vuln-app/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── clusterrolebinding.yaml
    └── deployment.yaml
```

## Expected Vulnerability Positions

Line numbers below are exact, one-based source-file positions. Rendered YAML line numbers may differ. For the missing resource configuration, the range identifies the container block where requests and limits are absent.

| ID | Misconfiguration | File | Exact Line(s) |
| :--- | :--- | :--- | :--- |
| V-01 | Unpinned `:latest` container image tag | [charts/vuln-app/values.yaml](charts/vuln-app/values.yaml#L3) | 3 (`tag: latest`) |
| V-02 | Hardcoded credential in values file | [charts/vuln-app/values.yaml](charts/vuln-app/values.yaml#L5) | 5 (`dbPassword: ...`) |
| V-03 | Host network namespace shared | [charts/vuln-app/templates/deployment.yaml](charts/vuln-app/templates/deployment.yaml#L16) | 16 (`hostNetwork: true`) |
| V-04 | Host PID namespace shared | [charts/vuln-app/templates/deployment.yaml](charts/vuln-app/templates/deployment.yaml#L17) | 17 (`hostPID: true`) |
| V-05 | Missing CPU and memory resource limits/requests | [charts/vuln-app/templates/deployment.yaml](charts/vuln-app/templates/deployment.yaml#L18-L31) | 18–31 (`containers[0]`, no `resources` block) |
| V-06 | Plaintext secret passed via environment variable | [charts/vuln-app/templates/deployment.yaml](charts/vuln-app/templates/deployment.yaml#L22-L23) | 22–23 (`DB_PASSWORD` and its literal value template) |
| V-07 | Privileged container enabled | [charts/vuln-app/templates/deployment.yaml](charts/vuln-app/templates/deployment.yaml#L25) | 25 (`privileged: true`) |
| V-08 | Container runs as root user (`UID 0`) | [charts/vuln-app/templates/deployment.yaml](charts/vuln-app/templates/deployment.yaml#L26) | 26 (`runAsUser: 0`) |
| V-09 | Privilege escalation allowed | [charts/vuln-app/templates/deployment.yaml](charts/vuln-app/templates/deployment.yaml#L27) | 27 (`allowPrivilegeEscalation: true`) |
| V-10 | Root filesystem is not read-only | [charts/vuln-app/templates/deployment.yaml](charts/vuln-app/templates/deployment.yaml#L28) | 28 (`readOnlyRootFilesystem: false`) |
| V-11 | Dangerous `SYS_ADMIN` capability added | [charts/vuln-app/templates/deployment.yaml](charts/vuln-app/templates/deployment.yaml#L29-L31) | 29–31 (`capabilities.add` containing `SYS_ADMIN`) |
| V-12 | Over-permissive `cluster-admin` role binding | [charts/vuln-app/templates/clusterrolebinding.yaml](charts/vuln-app/templates/clusterrolebinding.yaml#L11) | 11 (`name: cluster-admin`) |

These are fixture IDs, not scanner-specific rule IDs. Findings and severity depend on the scanner version and enabled rules; scanners may also report additional issues.

## Local Validation

Run from the repository root with Helm installed:

```bash
helm lint charts/vuln-app
helm template test-release charts/vuln-app
```

Linting checks chart validity; passing lint does not mean the configuration is secure. Rendering is local and does not install resources into a cluster.

## Local Scan Check

With Trivy installed:

```bash
trivy config charts/vuln-app
```

For scanners that consume Kubernetes YAML, render to a temporary file first:

```bash
fixture_yaml="$(mktemp -t vuln-helm-fixture.XXXXXX)"
helm template test-release charts/vuln-app > "$fixture_yaml"
trivy config "$fixture_yaml"
rm -f "$fixture_yaml"
```

All scanning can be performed locally or in CI without deploying the chart.
