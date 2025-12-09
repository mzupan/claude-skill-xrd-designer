# XRD Templates

## Service Integration XRD Template

### definition.yaml

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: custom<resources>.platform.example.org
spec:
  group: platform.example.org
  names:
    kind: Custom<Resource>
    plural: custom<resources>
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
              # Primary identifier
              name:
                type: string
                description: Unique name for this resource
              # Configuration as JSON
              config:
                type: string
                description: Resource configuration as JSON string
              # Service endpoint (optional - can be from Helm values)
              endpoint:
                type: string
                default: "http://service.default.svc.cluster.local:8080"
                description: Service endpoint URL
              path:
                type: string
                default: "/v1/resources"
                description: API path
            required:
            - name
            - config
          status:
            type: object
            properties:
              syncStatus:
                type: string
                description: Sync status (Success/Error/Pending)
              lastSyncTime:
                type: string
                description: Last successful sync timestamp (ISO 8601)
              lastHttpStatus:
                type: string
                description: HTTP status code from last request
              resourceId:
                type: string
                description: ID of the resource in external service
              error:
                type: string
                description: Error message if any
              message:
                type: string
                description: Human-readable status message
    additionalPrinterColumns:
    - name: NAME
      type: string
      jsonPath: .spec.name
    - name: SYNC
      type: string
      jsonPath: .status.syncStatus
    - name: HTTP
      type: string
      jsonPath: .status.lastHttpStatus
    - name: ERROR
      type: string
      jsonPath: .status.error
    - name: AGE
      type: date
      jsonPath: .metadata.creationTimestamp
```

### composition.yaml (Service Integration)

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: custom<resources>.platform.example.org
spec:
  compositeTypeRef:
    apiVersion: platform.example.org/v1alpha1
    kind: Custom<Resource>
  mode: Pipeline
  pipeline:
  - step: sync-resource
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
            SYNC_INTERVAL = 300  # Rate-limit: 5 minutes

            async def compose(self):
                now = datetime.utcnow()

                # Get observed status from previous reconciliation
                observed_status = None
                last_sync_str = ''
                try:
                    observed_status = self.request.observed.composite.resource.status
                    last_sync_str = str(observed_status.lastSyncTime) if hasattr(observed_status, 'lastSyncTime') else ''
                except:
                    pass

                # Rate limiting check
                if last_sync_str:
                    try:
                        last_sync = datetime.fromisoformat(last_sync_str.replace('Z', ''))
                        elapsed = (now - last_sync).total_seconds()

                        if elapsed < self.SYNC_INTERVAL:
                            # Preserve ALL status fields unchanged
                            if observed_status:
                                self.status.syncStatus = str(observed_status.syncStatus) if hasattr(observed_status, 'syncStatus') else ''
                                self.status.lastSyncTime = last_sync_str
                                self.status.lastHttpStatus = str(observed_status.lastHttpStatus) if hasattr(observed_status, 'lastHttpStatus') else ''
                                self.status.resourceId = str(observed_status.resourceId) if hasattr(observed_status, 'resourceId') else ''
                                self.status.error = str(observed_status.error) if hasattr(observed_status, 'error') else ''
                                self.status.message = str(observed_status.message) if hasattr(observed_status, 'message') else ''
                            self.ready = True
                            return
                    except Exception:
                        pass

                # Get spec fields
                name = str(self.spec.name or '')
                config_str = str(self.spec.config or '{}')
                endpoint = str(self.spec.endpoint or 'http://service.default.svc.cluster.local:8080')
                path = str(self.spec.path or '/v1/resources')

                # Validate JSON config
                try:
                    config = json.loads(config_str)
                except json.JSONDecodeError as e:
                    self.status.syncStatus = 'Error'
                    self.status.error = f'Invalid JSON config: {str(e)}'
                    self.status.message = 'Configuration must be valid JSON'
                    self.status.lastSyncTime = now.isoformat() + 'Z'
                    self.ready = True
                    return

                # Ensure config has an id
                if 'id' not in config:
                    config['id'] = name

                resource_id = str(config.get('id', name))
                base_url = f'{endpoint}{path}'

                # Check if resource exists
                try:
                    check_url = f'{base_url}/{resource_id}'
                    response = requests.get(check_url, timeout=10)

                    if response.status_code == 200:
                        # Resource exists - compare config
                        remote = response.json()
                        # Handle wrapped responses
                        if isinstance(remote, dict) and len(remote) == 1:
                            key = list(remote.keys())[0]
                            if isinstance(remote[key], dict):
                                remote = remote[key]

                        needs_update = False
                        for key, value in config.items():
                            if str(remote.get(key)) != str(value):
                                needs_update = True
                                break

                        if not needs_update:
                            # No changes needed
                            self.status.syncStatus = 'Success'
                            self.status.lastSyncTime = now.isoformat() + 'Z'
                            self.status.lastHttpStatus = '200'
                            self.status.resourceId = resource_id
                            self.status.error = ''
                            self.status.message = f'Resource {resource_id} in sync'
                            self.ready = True
                            return

                        # Update existing resource
                        response = requests.put(check_url, json=config, timeout=30)

                    elif response.status_code == 404:
                        # Create new resource
                        response = requests.post(base_url, json=config, timeout=30)

                    else:
                        self.status.syncStatus = 'Error'
                        self.status.lastHttpStatus = str(response.status_code)
                        self.status.error = f'Check failed: HTTP {response.status_code}'
                        self.status.message = 'Failed to check resource status'
                        self.status.lastSyncTime = now.isoformat() + 'Z'
                        self.ready = True
                        return

                    # Handle create/update response
                    if response.status_code in [200, 201, 202]:
                        self.status.syncStatus = 'Success'
                        self.status.lastHttpStatus = str(response.status_code)
                        self.status.resourceId = resource_id
                        self.status.error = ''
                        self.status.message = f'Resource {resource_id} synced successfully'
                    else:
                        self.status.syncStatus = 'Error'
                        self.status.lastHttpStatus = str(response.status_code)
                        self.status.error = f'Sync failed: HTTP {response.status_code}'
                        self.status.message = 'Failed to sync resource'

                except requests.exceptions.ConnectionError:
                    self.status.syncStatus = 'Error'
                    self.status.error = f'Connection refused to {endpoint}'
                    self.status.message = 'Cannot connect to service'
                except requests.exceptions.Timeout:
                    self.status.syncStatus = 'Error'
                    self.status.error = 'Request timeout'
                    self.status.message = 'Service not responding'
                except Exception as e:
                    self.status.syncStatus = 'Error'
                    self.status.error = f'Error: {str(e)}'
                    self.status.message = 'Unexpected error during sync'

                self.status.lastSyncTime = now.isoformat() + 'Z'
                self.ready = True
```

## Data Source XRD Template

### definition.yaml

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: customdata<resource>.platform.example.org
spec:
  group: platform.example.org
  names:
    kind: CustomData<Resource>
    plural: customdata<resources>
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
              # Lookup parameters
              query:
                type: string
                description: Query parameter for lookup
              # Additional filters
              filters:
                type: object
                additionalProperties:
                  type: string
                description: Additional filter parameters
            required:
            - query
          status:
            type: object
            properties:
              found:
                type: string
                description: Whether resource was found
              result:
                type: string
                description: Lookup result (JSON)
              count:
                type: string
                description: Number of results
              error:
                type: string
                description: Error message if any
              lastChecked:
                type: string
                description: Last lookup timestamp
    additionalPrinterColumns:
    - name: QUERY
      type: string
      jsonPath: .spec.query
    - name: FOUND
      type: string
      jsonPath: .status.found
    - name: COUNT
      type: string
      jsonPath: .status.count
    - name: ERROR
      type: string
      jsonPath: .status.error
    - name: AGE
      type: date
      jsonPath: .metadata.creationTimestamp
```

### composition.yaml (Data Source)

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: customdata<resource>
spec:
  compositeTypeRef:
    apiVersion: platform.example.org/v1alpha1
    kind: CustomData<Resource>
  mode: Pipeline
  pipeline:
  - step: lookup-resource
    functionRef:
      name: fortra-function-pythonic
    input:
      apiVersion: pythonic.fn.fortra.com/v1alpha1
      kind: Composite
      composite: |
        from crossplane.pythonic import BaseComposite
        import json
        from datetime import datetime

        class Composite(BaseComposite):
            CHECK_INTERVAL = 300

            async def compose(self):
                now = datetime.utcnow()

                # Rate limiting
                observed_status = None
                last_checked_str = ''
                try:
                    observed_status = self.request.observed.composite.resource.status
                    last_checked_str = str(observed_status.lastChecked) if hasattr(observed_status, 'lastChecked') else ''
                except:
                    pass

                if last_checked_str:
                    try:
                        last_checked = datetime.fromisoformat(last_checked_str.replace('Z', ''))
                        if (now - last_checked).total_seconds() < self.CHECK_INTERVAL:
                            if observed_status:
                                self.status.found = str(observed_status.found) if hasattr(observed_status, 'found') else 'false'
                                self.status.result = str(observed_status.result) if hasattr(observed_status, 'result') else ''
                                self.status.count = str(observed_status.count) if hasattr(observed_status, 'count') else '0'
                                self.status.error = str(observed_status.error) if hasattr(observed_status, 'error') else ''
                                self.status.lastChecked = last_checked_str
                            self.ready = True
                            return
                    except:
                        pass

                # Get query parameters
                query = str(self.spec.query or '')

                # Perform lookup (customize this section)
                try:
                    # Example: DNS lookup, API call, K8s resource query, etc.
                    results = []  # Your lookup logic here

                    if results:
                        self.status.found = 'true'
                        self.status.result = json.dumps(results)
                        self.status.count = str(len(results))
                        self.status.error = ''
                    else:
                        self.status.found = 'false'
                        self.status.result = '[]'
                        self.status.count = '0'
                        self.status.error = ''

                except Exception as e:
                    self.status.found = 'false'
                    self.status.error = str(e)

                self.status.lastChecked = now.isoformat() + 'Z'
                self.ready = True
```

## Elasticsearch XRD Template

### definition.yaml (Elasticsearch pattern)

```yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: customelasticsearch<resources>.platform.example.org
spec:
  group: platform.example.org
  names:
    kind: CustomElasticsearch<Resource>
    plural: customelasticsearch<resources>
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
              # Resource identifier
              <resourceId>:
                type: string
                description: Unique identifier for this resource
              # Resource configuration
              <resourceConfig>:
                type: string
                description: Configuration as JSON string
              # Server configuration block
              server:
                type: object
                description: Elasticsearch server configuration
                properties:
                  host:
                    type: string
                    default: elasticsearch-es-http.default.svc.cluster.local
                    description: Elasticsearch host
                  port:
                    type: integer
                    default: 9200
                    description: Elasticsearch port
                  user:
                    type: string
                    default: elastic
                    description: Elasticsearch username
                  password:
                    type: object
                    description: Password secret reference
                    properties:
                      secretName:
                        type: string
                        default: elasticsearch-es-elastic-user
                        description: Name of the secret containing password
                      secretKey:
                        type: string
                        default: elastic
                        description: Key in the secret containing password
                    required:
                    - secretName
                    - secretKey
                required:
                - host
                - port
                - user
                - password
            required:
            - <resourceId>
          status:
            type: object
            properties:
              exists:
                type: string
                description: Whether the resource exists in Elasticsearch
              error:
                type: string
                description: Error message if any
              lastChecked:
                type: string
                description: Last time the resource was checked
              message:
                type: string
                description: Human-readable status message
```

### Common Elasticsearch composition patterns

```python
# Elasticsearch base URL and auth
es_url = f'http://{host}:{port}'
auth = (user, password)

# Check resource exists
response = requests.get(f'{es_url}/_resource/{resource_id}', auth=auth, timeout=10)

# Create resource
response = requests.put(f'{es_url}/_resource/{resource_id}', json=config, auth=auth, timeout=30)

# Delete resource (usually NOT implemented - manual only)
# response = requests.delete(f'{es_url}/_resource/{resource_id}', auth=auth, timeout=10)
```
