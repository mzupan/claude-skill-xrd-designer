# XRD Designer Skill

A Claude Code skill for designing and generating Crossplane XRDs (CompositeResourceDefinitions) with function-pythonic compositions.

## Features

### XRD Types Supported

| Type | Description |
|------|-------------|
| **Service Integration** | HTTP-based sync to external services with rate-limiting |
| **Data Source** | Read-only lookups (no managed resources created) |
| **Multi-Cloud** | AWS + GCP compositions from single definition |
| **Single-Cloud** | AWS-only or GCP-only resources |
| **Elasticsearch** | Index, transform, and request management patterns |

### Patterns Included

- **Rate-limiting** - 5-minute intervals via status timestamps to prevent API spam
- **K8s Secret Lookup** - Service account token + Kubernetes API for secret retrieval
- **HTTP Service Sync** - GET/PUT/POST with idempotent operations
- **Status Preservation** - Full status copy during rate-limit windows
- **Error Handling** - Connection, timeout, JSON, HTTP error patterns
- **Elasticsearch Auth** - Secret-based authentication for ES clusters

### Templates Provided

- Service Integration XRD (definition + composition)
- Data Source XRD (definition + composition)
- Elasticsearch XRD patterns
- Complete function-pythonic composition examples

## Installation

### Option 1: Personal Installation (Recommended)

Copy the skill to your Claude Code skills directory:

```bash
# Create skills directory if it doesn't exist
mkdir -p ~/.claude/skills

# Clone or copy the skill
cp -r xrd-designer ~/.claude/skills/
```

Your directory structure should look like:

```
~/.claude/skills/xrd-designer/
├── SKILL.md        # Main skill definition (required)
├── TEMPLATES.md    # Copy-paste templates
└── README.md       # This file
```

### Option 2: Project Installation (Team Sharing)

Add to your project's `.claude/skills/` directory:

```bash
mkdir -p .claude/skills
cp -r xrd-designer .claude/skills/
git add .claude/skills/xrd-designer
git commit -m "Add XRD designer skill"
```

### Verification

Restart Claude Code after installation. The skill will be available automatically.

To verify, ask Claude: *"What XRD types can you help me design?"*

## Usage

### Automatic Activation

The skill auto-activates when you mention:
- XRD, Crossplane, composition
- function-pythonic, fortra
- Managed resources, data source
- Elasticsearch index/transform

### Example Prompts

**Design a new XRD:**
```
Design an XRD for managing Redis cache configurations
```

**Service integration pattern:**
```
Create an XRD that syncs feature flags to an external service via HTTP
```

**Data source lookup:**
```
I need an XRD that looks up AWS EC2 instances by tag without managing them
```

**Elasticsearch resource:**
```
Create an XRD for managing Elasticsearch ingest pipelines
```

### Workflow

1. **Describe your resource** - What are you managing?
2. **Claude asks clarifying questions** - Multi-cloud? Required fields? Status info?
3. **Claude generates files:**
   - `definition.yaml` - XRD schema
   - `composition.yaml` - function-pythonic logic
   - Helm values entry (if applicable)
4. **Review and customize** - Adjust fields, add business logic

## Skill Structure

```
xrd-designer/
├── SKILL.md           # Main skill with:
│                      #   - XRD type reference
│                      #   - Step-by-step instructions
│                      #   - function-pythonic patterns
│                      #   - Rate-limiting code
│                      #   - K8s secret lookup code
│                      #   - HTTP sync patterns
│                      #   - Naming conventions
│                      #   - Checklist
│
├── TEMPLATES.md       # Complete templates for:
│                      #   - Service Integration XRD
│                      #   - Data Source XRD
│                      #   - Elasticsearch XRD
│
└── README.md          # This file
```

## Key Patterns Reference

### Rate-Limiting (Prevents API Spam)

```python
CHECK_INTERVAL = 300  # 5 minutes

if elapsed < self.CHECK_INTERVAL:
    # Preserve ALL status fields unchanged
    self.status.field = observed_status.field
    self.ready = True
    return
```

### K8s Secret Lookup

```python
# Read SA token
with open('/var/run/secrets/kubernetes.io/serviceaccount/token') as f:
    token = f.read().strip()

# Query K8s API
response = requests.get(
    f'https://kubernetes.default.svc.cluster.local/api/v1/namespaces/{ns}/secrets/{name}',
    headers={'Authorization': f'Bearer {token}'},
    verify=False
)
```

### Idempotent HTTP Sync

```python
# Check exists (GET)
response = requests.get(url, timeout=10)

if response.status_code == 200:
    # Compare config, update if different (PUT)
elif response.status_code == 404:
    # Create new (POST)
```

## Customization

### Adding Your Domain

Edit `SKILL.md` and `TEMPLATES.md` to replace:
- `platform.example.org` → `platform.yourdomain.com`
- `Custom<Resource>` → `YourPrefix<Resource>`

### Adding New Patterns

Add new sections to `SKILL.md`:

```markdown
## Your New Pattern

Description of when to use this pattern.

\```python
# Code example
\```
```

## Requirements

- Claude Code with skills support
- Crossplane cluster (for applying generated XRDs)
- function-pythonic installed in cluster

## License

MIT - Use freely in your projects.
