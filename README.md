# AI-Workflows

Self-hostable automation workflows that use LLMs to take on repetitive analysis work, with a focus on cyber security and threat intelligence.

Each workflow sits in its own folder, with an importable export and a README covering how it works, what it needs, and how to set it up.

## Workflows

| Workflow | Platform | Description |
|---|---|---|
| [CTI OSINT: CyberIntel Daily Digest](CTI%20OSINT/) | n8n | Pulls 42 public security news and research feeds each day and merges articles about the same story into one brief. Enriches CVEs (CISA KEV, EPSS, NVD) and IOCs (VirusTotal, AlienVault OTX), and maps TTPs to MITRE ATT&CK. Outputs a Markdown digest, a STIX 2.1 bundle and a JSON run record, and can also sync to MISP and post to Discord. |

## Design principles

The workflows share a few conventions:

- **Free and local first.** They prefer free-tier hosted models and local inference (for example LM Studio) over paid APIs, and fall back between providers when one hits a limit.
- **LLM output is untrusted.** Model responses are validated against schemas and allow/deny lists before they reach any output. Values extracted deterministically (by regex or lookups) take precedence over model guesses.
- **Degrade, don't fail.** A failed optional step (enrichment, an external service) is recorded in the run notes instead of stopping the run.
- **Interoperable outputs.** Results are published in standard formats such as STIX 2.1 and MISP alongside readable reports.

## Getting started

1. Open the folder for the workflow you want and read its README.
2. Set up the services, credentials and environment variables it lists.
3. Import the workflow file into its platform. For n8n: **Workflows → Import from File**.
4. Run it once manually to test, then enable its schedule.

Exported workflows contain no credentials or API keys. Secrets are supplied through environment variables or the platform's credential store.

## Disclaimer

These workflows produce machine-generated analysis from public sources. Verify the results before acting on them.

## License

[GNU General Public License v3.0](LICENSE)
