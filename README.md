# Suhas Reddy — Claude Code Skills Marketplace

A personal marketplace of Claude Code plugins and skills for production DevOps, IaC, and infrastructure workflows.

## Install this marketplace

```bash
/plugin marketplace add suhasramanand-marketplace https://github.com/suhasramanand/claude-skills-marketplace
```

Or add it manually to `~/.claude/plugins/known_marketplaces.json`:

```json
{
  "suhasramanand-marketplace": {
    "source": {
      "source": "github",
      "repo": "suhasramanand/claude-skills-marketplace"
    }
  }
}
```

## Available Plugins

| Plugin | Category | Description |
|--------|----------|-------------|
| [terraform-best-practices](./plugins/terraform-best-practices) | DevOps / IaC | Production Terraform guidance: state management, modules, security, CI/CD, and anti-patterns |
| [kubernetes-best-practices](./plugins/kubernetes-best-practices) | DevOps / K8s | Production Kubernetes guidance: pod security, RBAC, NetworkPolicy, resource management, observability, and cluster hardening |

## Plugin Structure

```
claude-skills-marketplace/
├── plugin.json                          # Marketplace registry
├── README.md
└── plugins/
    ├── terraform-best-practices/
    │   ├── .claude-plugin/
    │   │   └── plugin.json              # Plugin metadata
    │   └── skills/
    │       └── terraform/
    │           └── SKILL.md             # Auto-activating skill definition
    └── kubernetes-best-practices/
        ├── .claude-plugin/
        │   └── plugin.json
        └── skills/
            └── kubernetes/
                └── SKILL.md
```

## Adding More Skills

Each plugin lives under `plugins/<name>/` and follows the standard Claude Code plugin structure. Skills in `skills/<name>/SKILL.md` auto-activate based on their `description` trigger conditions — no slash command needed.

## License

MIT
