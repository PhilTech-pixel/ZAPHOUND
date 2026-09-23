# ZAPHOUND©

A subdomain enumeration and takeover detection pipeline. Chains multiple recon tools together, de-dupes the results, resolves CNAMEs, and runs them through several takeover-detection engines — so you get one clean output directory instead of juggling six tools by hand.

> ⚠️ **For authorized security testing only.** Only run this against domains you own or have explicit permission to test (bug bounty scope, pentest engagement, CTF, etc.). Unauthorized scanning of infrastructure you don't have permission to test may be illegal.

## What it does

1. **Enumerates** subdomains using `subfinder`, `subfaster`, `amass`, `assetfinder`, `findomain`, crt.sh certificate transparency logs, and `waybackurls`
2. **Merges and de-dupes** all results into a single clean list
3. **Probes for live hosts** with `httpx` (skips dead/unresolving domains before wasting time on them)
4. **Resolves CNAME records** for every subdomain and saves them to a readable file for manual review
5. **Runs takeover detection** with `subjack`, `subzy`, and `nuclei` (`-tags takeover`)
6. Prints a summary and points you to the files worth reviewing

## Requirements

Install and make sure these are on your `$PATH`:

| Tool | Purpose | Install |
|---|---|---|
| [subfinder](https://github.com/projectdiscovery/subfinder) | Passive subdomain enum | `go install github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest` |
| subfaster | Fast subdomain enum | *(your local build/binary)* |
| [amass](https://github.com/owasp-amass/amass) | Active/passive subdomain enum | `go install github.com/owasp-amass/amass/v4/...@master` |
| [assetfinder](https://github.com/tomnomnom/assetfinder) | Subdomain discovery | `go install github.com/tomnomnom/assetfinder@latest` |
| [findomain](https://github.com/Findomain/Findomain) | Fast subdomain enum | See repo releases |
| [waybackurls](https://github.com/tomnomnom/waybackurls) | Wayback Machine subdomain harvesting | `go install github.com/tomnomnom/waybackurls@latest` |
| [httpx](https://github.com/projectdiscovery/httpx) | Live host probing | `go install github.com/projectdiscovery/httpx/cmd/httpx@latest` |
| [subjack](https://github.com/haccer/subjack) | Takeover detection (fingerprint-based) | `go install github.com/haccer/subjack@latest` |
| [subzy](https://github.com/PentestPad/subzy) | Takeover detection (fingerprint-based) | `go install github.com/PentestPad/subzy@latest` |
| [nuclei](https://github.com/projectdiscovery/nuclei) | Takeover detection (template-based) | `go install github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest` |
| `dig`, `curl` | DNS/HTTP basics | Usually preinstalled on Linux/Kali |

Missing tools are skipped gracefully — the script will still run and just note what it couldn't do.

### API keys

`subfinder -all` pulls much better results with free API keys configured for GitHub, VirusTotal, Censys, CertSpotter, and URLScan. Add them to:

```
~/.config/subfinder/provider-config.yaml
```

## Usage

```bash
chmod +x ghostdomain.sh
./ghostdomain.sh -d target.com
```

Optional custom output directory:

```bash
./ghostdomain.sh -d target.com -o ./results/target
```

By default, output goes to a timestamped folder: `./recon_target.com_YYYYMMDD_HHMMSS/`

## Output

```
recon_target.com_.../
├── subfinder.txt
├── subfaster.txt
├── amass.txt
├── assetfinder.txt
├── findomain.txt
├── crt_sh.txt
├── waybackurls.txt
├── subdomains.txt          # merged, de-duped, final list
├── live_hosts.txt          # subdomains confirmed live via httpx
├── cnames.txt              # every subdomain's CNAME target — review by hand
├── subjack_results.txt
├── subzy_results.json
└── nuclei_takeover.txt
```

## Verifying results (important)

**A tool flagging "possible takeover" is a lead, not confirmation.** Automated fingerprint matching produces false positives — a distribution or bucket can be fully alive and owned, and still trip a signature because a specific CNAME isn't attached to it.

Before reporting anything, manually verify:

```bash
# S3-style — look for a bucket-not-found error
curl -s http://<subdomain>/ 

# CloudFront-style — bypass cert check, look at the actual response body
curl -sk https://<subdomain>/

# Check what's really behind the CNAME target directly
curl -sk https://<cname-target>/ -H "Host: <subdomain>"
```

What confirms a **real** dangling takeover:
- `NoSuchBucket`, `NoSuchDistribution`, "no such app" style errors
- Generic service-branded "not configured" pages (not the target company's own branding)

What does **not** confirm a takeover, even if a scanner flags it:
- A branded error page belonging to the actual target company (means the resource is alive and owned — see [note] below)
- A working TLS handshake with a valid response
- AWS-generated random-ID resources (ELB names, API Gateway `execute-api` IDs) — these can't be re-claimed even if dangling, since you can't choose that identifier when creating a new resource

## License

MIT — use responsibly, and only where you're authorized to test.
