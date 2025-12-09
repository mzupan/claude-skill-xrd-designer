# XRD Designer Skill

[![GitHub](https://img.shields.io/badge/GitHub-mzupan%2Fclaude--skill--xrd--designer-blue)](https://github.com/mzupan/claude-skill-xrd-designer)

A Claude Code skill for designing and generating Crossplane XRDs (CompositeResourceDefinitions) with function-pythonic compositions.

## Installation

### Option 1: Personal Installation (Recommended)

Clone directly to your Claude Code skills directory:

```bash
# Create skills directory if it doesn't exist
mkdir -p ~/.claude/skills

# Clone the skill
git clone https://github.com/mzupan/claude-skill-xrd-designer.git ~/.claude/skills/xrd-designer
```

Your directory structure should look like:

```
~/.claude/skills/xrd-designer/
├── SKILL.md        # Main skill definition (required)
├── TEMPLATES.md    # Copy-paste templates
└── README.md       # This file
```

### Option 2: Project Installation (Team Sharing)

Add as a git submodule to your project:

```bash
# Add as submodule
git submodule add https://github.com/mzupan/claude-skill-xrd-designer.git .claude/skills/xrd-designer

# Or clone directly
mkdir -p .claude/skills
git clone https://github.com/mzupan/claude-skill-xrd-designer.git .claude/skills/xrd-designer
```

### Updating

Pull the latest changes:

```bash
cd ~/.claude/skills/xrd-designer
git pull origin main
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

**AWS ElastiCache with connection secrets:**
```
Design an XRD that provisions an AWS ElastiCache Redis cluster and writes
the connection endpoint and auth token to a Kubernetes Secret for apps to consume
```

**RDS Database with ConfigMap:**
```
Create an XRD for provisioning AWS RDS PostgreSQL instances that outputs
the connection string, host, port, and database name to a ConfigMap
```

**S3 Bucket with IAM credentials:**
```
I need an XRD that creates an S3 bucket with a dedicated IAM user and
stores the access keys in a Kubernetes Secret
```

**Multi-cloud object storage:**
```
Design an XRD that abstracts object storage - provisions S3 on AWS or
Cloud Storage on GCP based on a provider field, outputs bucket name and
endpoint to a ConfigMap
```

**Service mesh integration:**
```
Create an XRD that syncs service entries to Istio by calling the Kiali API,
tracking sync status and last sync time
```

**Elasticsearch index with ILM:**
```
Design an XRD for managing Elasticsearch indices that creates the index,
applies index lifecycle management policies, and tracks document count in status
```

**Secret rotation:**
```
Create an XRD that generates database credentials, stores them in a Secret,
and automatically rotates them every 30 days by calling an external vault API
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

## Contributing

Contributions welcome! Please open an issue or PR at [github.com/mzupan/claude-skill-xrd-designer](https://github.com/mzupan/claude-skill-xrd-designer).

## License

MIT - Use freely in your projects.
