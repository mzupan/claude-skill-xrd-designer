---
name: xrd-designer
description: Design and generate Crossplane XRDs (CompositeResourceDefinitions) with function-pythonic compositions. Use when creating new XRDs, designing Crossplane resources, working with function-pythonic, or building data source type XRDs. Triggers on mentions of XRD, Crossplane, composition, function-pythonic, fortra, or managed resources.
allowed-tools: Read, Write, Glob, Grep, Bash
---

# XRD Designer

Design and generate production-ready Crossplane XRDs with function-pythonic compositions following established patterns.

## Quick Reference

### XRD Types

| Type | API Domain | Use Case |
|------|-----------|----------|
| Multi-Cloud | `platform.example.org` | AWS + GCP support |
| AWS-Only | `aws.platform.example.org` | AWS-specific resources |
| GCP-Only | `gcp.platform.example.org` | GCP-specific resources |
| Service Integration | `platform.example.org` | HTTP-based external service sync |
| Data Source | `platform.example.org` | Read-only lookups (no managed resources) |

### File Structure

```
xrds/<resource-name>/
├── definition.yaml           # XRD definition
├── composition.yaml          # function-pythonic composition
├── composition-aws.yaml      # AWS composition (if multi-cloud)
├── composition-gcp.yaml      # GCP composition (if multi-cloud)
├── README.md                 # Quick-start overview
├── USAGE.md                  # API reference with examples
├── INSTALL.md                # Installation guide
├── TROUBLESHOOTING.md        # Common issues
└── IAM.md                    # Permissions guide
```

## Instructions

### Step 1: Gather Requirements

Ask the user:
1. **What resource/service are you managing?** (e.g., Elasticsearch index, JWT tokens, DNS records)
2. **Is this multi-cloud, single-cloud, or service integration?**
3. **What fields does the user need to configure?** (spec fields)
4. **What status information should be visible?** (status fields)
5. **Does it need external lookups?** (K8s secrets, external APIs)

### Step 2: Choose XRD Pattern

**Service Integration XRD** (HTTP-based sync to external service):
- Use `mode: Pipeline` with function-pythonic
- Implement rate-limiting via status field timestamps
- Include idempotent operations (GET check before POST/PUT)
- Track sync status in status fields

**Data Source XRD** (read-only lookups):
- Use `mode: Pipeline` with function-pythonic
- NO managed resources created
- Returns discovered data in status fields
- Rate-limited checks to prevent API spam

**Standard XRD** (managed cloud resources):
- Use traditional compositions with provider resources
- Separate AWS/GCP compositions for multi-cloud

### Step 3: Generate Definition

Use this template structure:

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: <plural>.<group>
spec:
  group: platform.example.org  # or aws.platform.example.org / gcp.platform.example.org
  names:
    kind: <Kind>
    plural: <plural>
  versions:
  - name: v1alpha1
    served: true
    referenceable: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              # User-configurable fields here
            required:
            - <required-fields>
          status:
            type: object
            properties:
              # Observable status fields here
    additionalPrinterColumns:
    # kubectl columns for visibility
```

### Step 4: Generate Composition

For function-pythonic compositions, follow this pattern:

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: <compositionname>
spec:
  compositeTypeRef:
    apiVersion: platform.example.org/v1alpha1
    kind: <Kind>
  mode: Pipeline
  pipeline:
  - step: <step-name>
    functionRef:
      name: fortra-function-pythonic
    input:
      apiVersion: pythonic.fn.fortra.com/v1alpha1
      kind: Composite
      composite: |
        from crossplane.pythonic import BaseComposite
        import requests
        import json
        from datetime import datetime

        class Composite(BaseComposite):
            CHECK_INTERVAL = 300  # Rate-limit: 5 minutes

            async def compose(self):
                # Implementation here
                pass
```

## function-pythonic Patterns

### Rate-Limiting Pattern

```python
CHECK_INTERVAL = 300  # 5 minutes

async def compose(self):
    now = datetime.utcnow()

    # Get observed status
    observed_status = None
    last_checked_str = ''
    try:
        observed_status = self.request.observed.composite.resource.status
        last_checked_str = str(observed_status.lastChecked) if hasattr(observed_status, 'lastChecked') else ''
    except:
        pass

    # Rate limiting check
    if last_checked_str:
        try:
            last_checked = datetime.fromisoformat(last_checked_str.replace('Z', ''))
            elapsed = (now - last_checked).total_seconds()

            if elapsed < self.CHECK_INTERVAL:
                # CRITICAL: Preserve ALL status fields unchanged
                if observed_status:
                    self.status.field1 = str(observed_status.field1) if hasattr(observed_status, 'field1') else ''
                    self.status.field2 = str(observed_status.field2) if hasattr(observed_status, 'field2') else ''
                    self.status.lastChecked = last_checked_str
                self.ready = True
                return
        except Exception:
            pass

    # Update timestamp and proceed with check
    self.status.lastChecked = now.isoformat() + 'Z'
```

### K8s Secret Lookup Pattern

```python
# Read service account token
try:
    with open('/var/run/secrets/kubernetes.io/serviceaccount/token', 'r') as f:
        k8s_token = f.read().strip()
except Exception as e:
    self.status.error = f'Failed to read SA token: {str(e)}'
    self.ready = False
    return

# Get namespace
try:
    with open('/var/run/secrets/kubernetes.io/serviceaccount/namespace', 'r') as f:
        namespace = f.read().strip()
except:
    namespace = 'default'

# Fetch secret
k8s_api_url = f'https://kubernetes.default.svc.cluster.local/api/v1/namespaces/{namespace}/secrets/{secret_name}'
headers = {
    'Authorization': f'Bearer {k8s_token}',
    'Accept': 'application/json'
}
response = requests.get(k8s_api_url, headers=headers, verify=False, timeout=10)

if response.status_code == 200:
    secret_data = response.json()
    password = base64.b64decode(secret_data['data'][secret_key]).decode('utf-8')
```

### HTTP Service Integration Pattern

```python
# Check if resource exists (GET)
try:
    check_url = f'{endpoint}/{path}'
    response = requests.get(check_url, timeout=10)

    if response.status_code == 200:
        # Resource exists - compare config
        remote_config = response.json()
        needs_update = False
        for key, value in local_config.items():
            if str(remote_config.get(key)) != str(value):
                needs_update = True
                break

        if not needs_update:
            self.status.syncStatus = 'Synced'
            self.ready = True
            return

        # Update existing (PUT)
        response = requests.put(check_url, json=local_config, timeout=30)
    elif response.status_code == 404:
        # Create new (POST)
        response = requests.post(endpoint, json=local_config, timeout=30)

    if response.status_code in [200, 201, 202]:
        self.status.syncStatus = 'Success'
        self.status.lastHttpStatus = str(response.status_code)
    else:
        self.status.syncStatus = 'Error'
        self.status.error = f'HTTP {response.status_code}'

except requests.exceptions.ConnectionError:
    self.status.error = 'Connection refused'
except requests.exceptions.Timeout:
    self.status.error = 'Request timeout'
except Exception as e:
    self.status.error = str(e)

self.ready = True
```

### Spec Field Access Pattern

```python
# Always convert to string with fallback defaults
field1 = str(self.spec.fieldName or 'default_value')
port = int(self.spec.server.port or 9200)
enabled = str(self.spec.enabled or 'true').lower() == 'true'

# Nested fields
host = str(self.spec.server.host or 'localhost')
secret_name = str(self.spec.server.password.secretName or 'default-secret')
```

### Status Field Pattern

```python
# Always set as strings
self.status.exists = 'true' if exists else 'false'
self.status.error = ''  # Clear on success
self.status.lastChecked = datetime.utcnow().isoformat() + 'Z'
self.status.message = f'Resource {name}: synced successfully'

# Set ready state
self.ready = True  # Prevents rapid reconciliation
```

### Error Handling Pattern

```python
try:
    # Operation
    response = requests.get(url, timeout=10)
    if response.status_code == 200:
        # Success handling
        self.status.error = ''
    elif response.status_code == 401:
        self.status.error = 'Authentication failed'
    elif response.status_code == 404:
        self.status.error = 'Resource not found'
    else:
        self.status.error = f'HTTP {response.status_code}'

except requests.exceptions.ConnectionError:
    self.status.error = f'Connection refused to {host}:{port}'
except requests.exceptions.Timeout:
    self.status.error = 'Request timeout'
except json.JSONDecodeError as e:
    self.status.error = f'Invalid JSON: {str(e)}'
except Exception as e:
    self.status.error = f'Error: {str(e)}'

# ALWAYS set ready to prevent reconciliation loops
self.ready = True
```

## Data Source XRD Pattern

Data source XRDs lookup information without creating managed resources:

```yaml
# definition.yaml - Note: NO claim configured
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: customdatadns.platform.example.org
spec:
  group: platform.example.org
  names:
    kind: CustomDataDNS
    plural: customdatadns
  versions:
  - name: v1alpha1
    served: true
    referenceable: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              hostname:
                type: string
                description: DNS hostname to lookup
              recordType:
                type: string
                enum: ["A", "AAAA", "CNAME", "TXT", "MX"]
                default: "A"
            required:
            - hostname
          status:
            type: object
            properties:
              records:
                type: string
                description: Resolved DNS records (JSON array)
              resolved:
                type: string
                description: Whether DNS lookup succeeded
              error:
                type: string
              lastChecked:
                type: string
```

## Naming Conventions

| Element | Convention | Example |
|---------|------------|---------|
| Kind | PascalCase | `CustomElasticsearchIndex` |
| Plural | lowercase | `customelasticsearchindices` |
| Composition name | lowercase | `customelasticsearchindex` |
| Spec fields | camelCase | `indexName`, `serverConfig` |
| Status fields | camelCase | `lastChecked`, `syncStatus` |
| Directory | camelCase | `customElasticsearchIndex/` |

## Status Field Best Practices

Always include these status fields:

| Field | Type | Purpose |
|-------|------|---------|
| `lastChecked` | string (ISO 8601) | Rate-limiting timestamp |
| `error` | string | Error message (empty on success) |
| `message` | string | Human-readable status |

Common additional fields:

| Field | Type | Purpose |
|-------|------|---------|
| `exists` | string ("true"/"false") | Resource existence |
| `syncStatus` | string | Sync state (Success/Error/Pending) |
| `lastHttpStatus` | string | HTTP response code |
| `resourceId` | string | External resource identifier |

## Printer Columns

Always include useful kubectl columns:

```yaml
additionalPrinterColumns:
- name: PRIMARY_FIELD
  type: string
  jsonPath: .spec.primaryField
- name: STATUS
  type: string
  jsonPath: .status.syncStatus
- name: ERROR
  type: string
  jsonPath: .status.error
- name: AGE
  type: date
  jsonPath: .metadata.creationTimestamp
```

## Helm Integration

XRDs should be deployable via Helm with enable/disable toggles:

```yaml
# values.yaml
xrds:
  customNewResource:
    enabled: true
```

Templates are auto-generated by GitHub Actions from `xrds/*/` directories.

## Documentation Requirements

Each XRD must include:

1. **README.md** (~5K): Quick-start, architecture, use cases
2. **USAGE.md** (~10K): API reference, simple examples first
3. **INSTALL.md** (~18K): Complete setup guide
4. **TROUBLESHOOTING.md** (~9K): Common issues with fixes
5. **IAM.md** (~4K): Permissions and security

## Checklist

Before completing an XRD:

- [ ] Definition has proper group/kind/plural
- [ ] All required spec fields documented
- [ ] Status fields include lastChecked, error, message
- [ ] Printer columns show useful info
- [ ] Composition uses rate-limiting pattern
- [ ] Error handling covers all failure modes
- [ ] Status preserved during rate-limit skip
- [ ] `self.ready = True` set on ALL code paths
- [ ] Added to Helm values.yaml
- [ ] Documentation files created
