# Bunnyshell Environments Skill for Claude Code

**Manage Bunnyshell cloud environments as a Claude Skill**

A [Claude Skill](https://docs.anthropic.com/en/docs/claude-code/skills) that enables Claude to create, deploy, and manage cloud environments using the Bunnyshell platform via the `bns` CLI.

Claude autonomously decides when to use this skill based on your environment management needs, loading only the minimal information required for your specific task.

## Features

- **Full Environment Lifecycle** - Create, deploy, stop, start, clone, and delete environments
- **Component Management** - View logs, execute commands, and redeploy individual components
- **Remote Development** - SSH access, port forwarding, and debug sessions
- **Pipeline Monitoring** - Track deployments and view pipeline logs
- **Configuration Authoring** - Create and modify `bunnyshell.yaml` files with proper schema

## Prerequisites

- **Bunnyshell CLI** - Install via Homebrew: `brew install bunnyshell/tap/bunnyshell-cli`
- **API Token** - Get from https://environments.bunnyshell.com/access-token

Configure the CLI:

```bash
bns configure profiles add --name default --token YOUR_TOKEN --default
```

## Installation

### Option 1: Global Installation

```bash
git clone https://github.com/bunnyshell/bunnyshell-environments-skill.git /tmp/bunnyshell-skill-temp

mkdir -p ~/.claude/skills
cp -r /tmp/bunnyshell-skill-temp ~/.claude/skills/bunnyshell-environments

rm -rf /tmp/bunnyshell-skill-temp
```

### Option 2: Project-Specific Installation

```bash
git clone https://github.com/bunnyshell/bunnyshell-environments-skill.git /tmp/bunnyshell-skill-temp

mkdir -p .claude/skills
cp -r /tmp/bunnyshell-skill-temp .claude/skills/bunnyshell-environments

rm -rf /tmp/bunnyshell-skill-temp
```

## Usage Examples

### Environment Operations

```
"List all my Bunnyshell environments"
"Deploy environment ENV_ID"
"Stop the staging environment to save costs"
"Clone production to create a test environment"
```

### Component Management

```
"Show logs for the api component"
"Execute a shell command in the database container"
"Redeploy the frontend component"
```

### Remote Development

```
"SSH into the backend component"
"Forward port 5432 from the database to my local machine"
"Start a debug session for the api"
```

### Configuration

```
"Create a bunnyshell.yaml for my Node.js app"
"Add a PostgreSQL database to my environment config"
"Export the current environment configuration"
```

## Project Structure

```
bunnyshell-environments/
├── SKILL.md              # Skill instructions for Claude
├── README.md             # This file
└── references/
    ├── cli.md            # Complete CLI command reference
    ├── api.md            # REST API documentation
    ├── yaml-schema.md    # bunnyshell.yaml configuration schema
    ├── components.md     # Component types (Helm, Terraform, etc.)
    └── variables.md      # Variables and interpolation syntax
```

## What is a Skill?

Skills are folders of instructions and resources that Claude can discover and use to perform tasks more accurately. When you ask Claude to manage Bunnyshell environments, it discovers this skill, loads the relevant instructions, and executes the appropriate CLI commands or helps author configuration files.

## Learn More

- [Bunnyshell Documentation](https://documentation.bunnyshell.com/)
- [Claude Code Skills Documentation](https://docs.anthropic.com/en/docs/claude-code/skills)
- [Bunnyshell CLI Reference](https://documentation.bunnyshell.com/docs/bunnyshell-cli)

## License

MIT License
