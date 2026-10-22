# SAP BTP AI Skills Hub

A personal library of AI skills for SAP BTP developers. These skills extend Claude Code and other AI agents with SAP-specific knowledge — so you spend less time looking up docs and more time building.

## Available Skills

| Skill | Description |
|-------|-------------|
| [`btp-xs-security`](./skills/btp-xs-security/SKILL.md) | Write, review, and troubleshoot `xs-security.json` (XSUAA authorization descriptor) for SAP BTP apps |
| [`btp-mta-descriptor`](./skills/btp-mta-descriptor/SKILL.md) | Write, review, and troubleshoot `mta.yaml` MTA deployment descriptors for SAP BTP CAP applications |
| [`cap-cds-model`](./skills/cap-cds-model/SKILL.md) | Write CAP CDS schema, service definitions, authorization rules, and Fiori annotations |
| [`btp-destination`](./skills/btp-destination/SKILL.md) | Configure BTP Destination Service entries — OAuth2, BasicAuth, Principal Propagation, Cloud Connector |
| [`ai-core-deployment`](./skills/ai-core-deployment/SKILL.md) | Configure SAP AI Core deployments, Generative AI Hub access, serving templates, and resource groups |
| [`cap-nodejs-project`](./skills/cap-nodejs-project/SKILL.md) | Generate a complete, deployable CAP Node.js project for BTP CF — production-tested end-to-end |
| [`cap-java-project`](./skills/cap-java-project/SKILL.md) | Generate a complete, deployable CAP Java (Spring Boot) project for BTP CF with Maven and MTA |

## Installation

Install a specific skill using the `npx skills` CLI:

```bash
npx skills add jitendrasingh1305/sap-btp-ai-skills-hub --skill btp-xs-security
npx skills add jitendrasingh1305/sap-btp-ai-skills-hub --skill btp-mta-descriptor
npx skills add jitendrasingh1305/sap-btp-ai-skills-hub --skill cap-cds-model
npx skills add jitendrasingh1305/sap-btp-ai-skills-hub --skill btp-destination
npx skills add jitendrasingh1305/sap-btp-ai-skills-hub --skill ai-core-deployment
npx skills add jitendrasingh1305/sap-btp-ai-skills-hub --skill cap-nodejs-project
npx skills add jitendrasingh1305/sap-btp-ai-skills-hub --skill cap-java-project
```

Install all skills in this repo:

```bash
npx skills add jitendrasingh1305/sap-btp-ai-skills-hub
```

## Usage

Once installed, the skill activates automatically in Claude Code when you mention relevant keywords — no slash command needed. For example:

- *"Generate an xs-security.json for my CAP app with Viewer and Admin roles"*
- *"Why am I getting insufficient_scope error?"*
- *"Convert my app to multi-tenant XSUAA config"*

## Contributing

Skills are plain Markdown files. To add a new skill:

1. Create a folder under `skills/<skill-slug>/`
2. Add a `SKILL.md` with YAML frontmatter (`name`, `description`) and instructions
3. Submit a PR

## License

Apache 2.0 — see [LICENSE](./LICENSE)
