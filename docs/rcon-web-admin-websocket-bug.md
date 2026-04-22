# RCON Web Admin Chart Bug: Missing Websocket URL Env Vars Without Ingress

## Problem Description

When deploying `rcon-web-admin` chart with **LoadBalancer** service (ingress disabled), the container fails to connect to the Minecraft server via WebSocket because the required environment variables are not set.

### Affected Versions

- `rcon-web-admin` 1.2.0 (and likely earlier versions)

### Symptom

The web UI displays "WebSocket connection failed" or similar error. The browser console shows attempts to connect to an undefined host (e.g., `ws://:4327`).

---

## Root Cause Analysis

### Bug Location

**File**: `charts/rcon-web-admin/templates/deployment.yaml`

**Problematic Code** (lines 75-78):

```yaml
{{- if .Values.ingress.enabled }}
{{ $wsUrl := print .Values.ingress.host (trimSuffix "/" .Values.ingress.path) "/websocket" }}
- name: RWA_WEBSOCKET_URL_SSL
  value: {{ .Values.rconWeb.websocketUrlSsl | default "ws://localhost:4327" | quote }}
- name: RWA_WEBSOCKET_URL
  value: {{ .Values.rconWeb.websocketUrl | default "wss://localhost:4327" | quote }}
{{- end }}
```

### Why This Is Wrong

1. **Conditional Logic Error**: The websocket URL env vars are wrapped in `{{- if .Values.ingress.enabled }}`, meaning they are **only** set when ingress is enabled.

2. **Inverted Defaults**: When ingress IS enabled, the defaults are `ws://localhost:4327` (HTTP) and `wss://localhost:4327` (HTTPS), which is backwards.

3. **Missing Fallback**: When ingress is disabled (common for LoadBalancer deployments), these env vars are never set at all, causing the application to use internal defaults that may not work.

4. **Unused Variable**: The `$wsUrl` variable is calculated but never used.

---

## The Fix

### Proposed Changes to `deployment.yaml`

Replace the problematic section with:

```yaml
{{- if .Values.ingress.enabled }}
{{ $wsUrl := print .Values.ingress.host (trimSuffix "/" .Values.ingress.path) "/websocket" }}
- name: RWA_WEBSOCKET_URL_SSL
  value: {{ .Values.rconWeb.websocketUrlSsl | default (printf "wss://%s%s/websocket" (trimPrefix "http://" (trimPrefix "https://" .Values.ingress.host)) (trimSuffix "/" .Values.ingress.path)) | quote }}
- name: RWA_WEBSOCKET_URL
  value: {{ .Values.rconWeb.websocketUrl | default (printf "ws://%s%s/websocket" (trimPrefix "http://" (trimPrefix "https://" .Values.ingress.host)) (trimSuffix "/" .Values.ingress.path)) | quote }}
{{- else }}
- name: RWA_WEBSOCKET_URL_SSL
  value: {{ .Values.rconWeb.websocketUrlSsl | default "wss://localhost:4327" | quote }}
- name: RWA_WEBSOCKET_URL
  value: {{ .Values.rconWeb.websocketUrl | default "ws://localhost:4327" | quote }}
{{- end }}
```

### Alternative Simpler Fix

For most LoadBalancer deployments, `ws://localhost:4327` works because the browser connects to the LoadBalancer IP, which forwards to the same pod's port 4327:

```yaml
- name: RWA_WEBSOCKET_URL_SSL
  value: {{ .Values.rconWeb.websocketUrlSsl | default "wss://localhost:4327" | quote }}
- name: RWA_WEBSOCKET_URL
  value: {{ .Values.rconWeb.websocketUrl | default "ws://localhost:4327" | quote }}
```

**Note**: Remove the `{{- if .Values.ingress.enabled }}` wrapper entirely - the values are always needed, regardless of ingress configuration.

---

## CI Test to Prevent Regression

### Current CI Gap

The existing workflow (`lint-test.yaml`) installs charts with `ct install`, which uses MetalLB to provide a LoadBalancer. However, there is **no test** that verifies the websocket environment variables are properly set.

### Proposed CI Test

Add a test job that verifies the deployment has correct websocket env vars:

```yaml
  test-websocket-env:
    runs-on: ubuntu-latest
    needs: lint-test
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Helm
        uses: azure/setup-helm@v4
        with:
          version: v3.17.2

      - name: Create kind cluster
        uses: helm/kind-action@v1.12.0

      - name: Install MetalLB
        run: kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.10/config/manifests/metallb-native.yaml

      - name: Create test values file (LoadBalancer, no ingress)
        run: |
          cat > /tmp/test-values.yaml << 'EOF'
          service:
            type: LoadBalancer
            httpPort: 8080

          rconWeb:
            password: testpassword
            rconHost: ""
            rconPort: 0

          ingress:
            enabled: false
          EOF

      - name: Template chart and check environment variables
        run: |
          helm template rcon-web ./charts/rcon-web-admin -f /tmp/test-values.yaml > /tmp/rendered.yaml

          # Check RWA_WEBSOCKET_URL is set
          if ! grep -q "name: RWA_WEBSOCKET_URL" /tmp/rendered.yaml; then
            echo "ERROR: RWA_WEBSOCKET_URL env var is missing!"
            exit 1
          fi

          # Check RWA_WEBSOCKET_URL_SSL is set
          if ! grep -q "name: RWA_WEBSOCKET_URL_SSL" /tmp/rendered.yaml; then
            echo "ERROR: RWA_WEBSOCKET_URL_SSL env var is missing!"
            exit 1
          fi

          # Verify default value works for LoadBalancer
          if grep -q 'value: "ws://localhost:4327"' /tmp/rendered.yaml; then
            echo "SUCCESS: Websocket URL has correct default for LoadBalancer"
          else
            echo "WARNING: Websocket URL default may not work for LoadBalancer"
          fi

          echo "All websocket env var checks passed!"

      - name: Install chart and verify pods have env vars
        run: |
          kubectl config use-context kind-kind

          # Install chart
          helm install rcon-web ./charts/rcon-web-admin \
            -f /tmp/test-values.yaml \
            --wait --timeout 5m

          # Wait for pod
          kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=rcon-web-admin --timeout=60s

          # Check pod env vars
          WEBSOCKET_URL=$(kubectl get pod -l app.kubernetes.io/name=rcon-web-admin -o jsonpath='{.items[0].spec.containers[0].env[?(@.name=="RWA_WEBSOCKET_URL")].value}')
          WEBSOCKET_URL_SSL=$(kubectl get pod -l app.kubernetes.io/name=rcon-web-admin -o jsonpath='{.items[0].spec.containers[0].env[?(@.name=="RWA_WEBSOCKET_URL_SSL")].value}')

          if [[ -z "$WEBSOCKET_URL" ]]; then
            echo "ERROR: RWA_WEBSOCKET_URL is not set in running pod!"
            exit 1
          fi

          if [[ -z "$WEBSOCKET_URL_SSL" ]]; then
            echo "ERROR: RWA_WEBSOCKET_URL_SSL is not set in running pod!"
            exit 1
          fi

          echo "Pod has correct websocket env vars:"
          echo "  RWA_WEBSOCKET_URL=$WEBSOCKET_URL"
          echo "  RWA_WEBSOCKET_URL_SSL=$WEBSOCKET_URL_SSL"
```

### Alternative: Chart-Testing Compatible Test

Add a CI test values file in `charts/rcon-web-admin/ci/websocket-lb-values.yaml`:

```yaml
# Test that websocket env vars work without ingress (LoadBalancer mode)
service:
  type: LoadBalancer
  httpPort: 8080

rconWeb:
  password: testpassword
  rconHost: ""
  rconPort: 0

ingress:
  enabled: false
```

Then add to workflow:

```yaml
- name: Test websocket env vars (no ingress)
  run: |
    helm template test ./charts/rcon-web-admin \
      -f ./charts/rcon-web-admin/ci/websocket-lb-values.yaml \
      --no-hooks \
      > /tmp/rendered.yaml

    # Verify env vars are present
    grep -q "name: RWA_WEBSOCKET_URL" /tmp/rendered.yaml || \
      (echo "FAIL: RWA_WEBSOCKET_URL not found" && exit 1)

    grep -q "name: RWA_WEBSOCKET_URL_SSL" /tmp/rendered.yaml || \
      (echo "FAIL: RWA_WEBSOCKET_URL_SSL not found" && exit 1)

    echo "PASS: Websocket env vars present without ingress"
```

---

## Validation Checklist

Before merging the fix, verify:

- [ ] Chart templates correctly with `ingress.enabled: false`
- [ ] Chart templates correctly with `ingress.enabled: true`
- [ ] CI test fails with current broken code
- [ ] CI test passes with fix applied
- [ ] Manual testing with LoadBalancer service works

---

## References

- Upstream Issue: (link to issue if created)
- Related PR: (link to PR if created)
- Original Discovery: gopher-hub deployment debugging session
