# Example agent prompt: local security assessment with HexStrike in Docker

A template to paste into Codex, Claude Code or another MCP client once the container is running
and connected (see [README.md](README.md)). Replace every `<…>` placeholder, delete what does not
apply, and adapt the REPORTING section to wherever your team tracks findings.

The HexStrike part (container layout, pre-flight check, rules) is what makes the agent use the
container correctly; keep it even if you rewrite the rest.

---

```text
SECURITY ASSESSMENT: <APPLICATION NAME>

GOAL
Find security vulnerabilities in <APPLICATION NAME> by reading its code and testing the running application. Report each one with enough detail for a developer to reproduce and fix it.

SCOPE
Code: <REPO PATH ON THE HOST, e.g. C:\Users\me\Projects\my-app> (and <OTHER REPO PATHS>). Say immediately if other repositories are needed for the best possible analysis.
Running application: <SCHEME>://localhost:<PORT>/ (<what it is, e.g. the web front end>).
Other services of the stack: <PORT: what it is, e.g. 5038: internal API>. If you are unsure which ports belong to the stack, read launch settings, compose files and configuration in the repositories and list them.
<Optional: production exposure, e.g. "The internal API is not exposed in production: only the front end can reach it. Rate issues that need direct access to it as defense in depth, and check whether each one can be triggered through the front end.">
Allowed targets: only the ports above, on localhost / 127.0.0.1. Anything else, including other local services on this machine and every external host, is out of scope.
Test accounts: <where the credentials are, or "email: … / password: …">. If a test needs another user or role (IDOR, privilege escalation), ask me instead of creating accounts.
Do not limit yourself to static analysis. You can use the browser, curl, HexStrike or anything else, as long as it points at an allowed target.

DATA HANDLING
Use placeholders (<TOKEN>, <PASSWORD>, <USER_ID>) for credentials, tokens, API keys and personal data in everything you report. Redact request and response samples, and trim scanner output to the lines that prove the finding.
Prove data-access bugs with the minimum read: a banner, a row count, one redacted field. Never dump tables (no sqlmap --dump / --dump-all, no bulk exports).

REPORTING
<Where findings go: a file, an issue tracker, a shared sheet... and its format.>
For each finding give: title, severity, status, affected component (URL as the application sees it, https://localhost:<PORT>/..., plus file:line when known), description, reproduction steps, evidence, suggested fix.
Status:
- suspected: from code reading, static analysis, or an untriaged scanner result.
- verified: code path traced end to end: input reaches the sink with no guard in between. A scanner hit alone never qualifies; find the code that causes it.
- reproduced: exploited against the running application, with reproduction steps a human can re-run by hand (curl, browser steps, or the exact tool command with its parameters), not just "HexStrike reported it".
- false positive: investigated, not an issue; say why.
Report each root cause once, even if several tools flag it.

TOOLING: HEXSTRIKE AI
HexStrike AI is connected as an MCP server (hexstrike). It wraps security tools (nmap, nuclei, ffuf, sqlmap, dalfox, nikto, httpx, katana, arjun, jwt_analyzer, trivy, checkov, ...) and runs them inside a Kali Linux Docker container on this machine. Use it to widen dynamic coverage. It does not replace reading the code.

Addressing: inside the container, localhost is the container itself. From HexStrike, reach the allowed targets as host.docker.internal: <SCHEME>://host.docker.internal:<PORT>/. host.docker.internal is this machine, and the same scope rules apply to it: only the allowed ports. Browser and curl run on the host and keep using localhost.

Container layout:
- /targets = <HEXSTRIKE_TARGETS host folder>, mounted READ-ONLY. In scope: /targets/<repo> (<and /targets/other-repo>); do not read or scan other folders there. Point file-based tools here (trivy fs /targets/<repo>, checkov -d /targets/<repo>). Never try to write into /targets.
- /workspace = <HEXSTRIKE_WORKSPACE host folder>, read/write. Write raw tool output only here. It may contain unredacted data: never paste from it without redacting.
- Wordlists: /usr/share/wordlists/... and SecLists at /usr/share/seclists/... Prefer small lists, for example /usr/share/seclists/Discovery/Web-Content/common.txt.

Pre-flight check: run once per session, before any scan. Use execute_command with use_cache=false for every command here.
a. Health: call server_health. status must be "healthy". Note which tools in tools_status are false; only installed tools work (outguess is expected to be missing).
b. Each allowed HTTP port: run curl -sk -m 10 -o /dev/null -w "%{http_code}" <SCHEME>://host.docker.internal:<PORT>/
   - 2xx or 3xx: reachable.
   - 400 (the app rejects the host name, typical of IIS Express and ASP.NET Core host filtering): run it again with -H "Host: localhost:<PORT>". If that returns 2xx or 3xx, every HexStrike call against that port must pass the header "Host: localhost:<PORT>" (each tool's custom-header option).
   - 000 or a connection error: try the other scheme (http/https) once. If it is still 000, HexStrike cannot see that port. Tell me (is the app running? is Docker running?) and continue with code review, browser and curl.
   For non-HTTP ports (databases, queues) use nc -zv -w 5 host.docker.internal <PORT>. Do not treat an nmap scan of host.docker.internal as proof: the host gateway can make closed ports look open, so confirm with curl or nc.
c. Repository mount: ls /targets/<repo> must list the repository, and touch /targets/<repo>/.preflight must fail with "Read-only file system". If the listing is empty or the touch succeeds, stop and tell me: the mount is wrong.
d. Workspace: touch /workspace/.preflight && rm /workspace/.preflight must succeed.
e. Wordlist: ls /usr/share/seclists/Discovery/Web-Content/common.txt must exist.
f. Show me the result as a short table (check, result, action taken), including the scheme and Host header decision for each port.

Rules for every HexStrike call:
- Targets: only the allowed targets, addressed as host.docker.internal. Never call the external-facing workflows (bugbounty_osint_gathering, bugbounty_reconnaissance_workflow, correlate_threat_intelligence, subdomain or OSINT tools): they query the internet.
- TLS: local development certificates are issued for localhost and do not match host.docker.internal. Pass the tool's skip-verification flag where needed (for example -k).
- Host header: if the pre-flight required it, pass "Host: localhost:<PORT>" on every call against that port.
- Wordlists: always pass an explicit wordlist path that exists in the container.
- Load: keep concurrency moderate (ffuf/feroxbuster at 10 threads or fewer, nuclei with a rate limit) so the application stays usable for manual testing.
- Destructive payloads: no sqlmap --os-shell, --os-cmd, --file-write or risk 3. No payloads that delete or mass-modify data. Brute force only against test accounts, with small lists, after asking me, and stop if an account locks out.
- Cache: HexStrike caches identical commands for one hour. When you re-run something to confirm a finding or check a fix, use execute_command with use_cache=false, or call clear_cache first.
- Authenticated scans: log in with a test account, take the session cookie or bearer token, and pass it to the tool as a header. Repeat key scans for each role to find authorization flaws. Never write token values into a report.

How to use it (alongside, not instead of, code review):
a. Recon: httpx_probe (with tech detection) on the reachable ports; katana_crawl on the main URL with JS crawling, unauthenticated and authenticated.
b. Attack surface: list endpoints from the code (controllers, route maps, pages, API clients) and from the crawl, and diff the two. Routes found only in code are prime targets. Use ffuf/feroxbuster for hidden paths and arjun for hidden parameters on the main endpoints.
c. Broad scanning: nuclei (exposures, misconfiguration, tokens, default-logins; skip DoS templates), nikto, sslscan/testssl.
d. Targeted testing, driven by what the code shows: sqlmap on endpoints whose queries are built from input, dalfox on reflected or stored output, jwt_analyzer on issued tokens, api_fuzzer / comprehensive_api_audit on APIs, graphql_scanner if GraphQL exists, bugbounty_authentication_bypass_testing and bugbounty_file_upload_testing against the local URL.
e. Supply chain and config: trivy_scan in filesystem mode on /targets/<repo> (vulnerable packages, leaked secrets); checkov_iac_scan on Docker, compose and IaC files.
f. Triage: every scanner hit starts as suspected. Trace it to code or reproduce it by hand before raising its status.

Recording HexStrike findings:
- Reproduction steps: the exact HexStrike tool and parameters, plus a manual equivalent (curl or browser steps against localhost) whenever one exists.
- Evidence: the trimmed, redacted tool output.
- Affected component: the URL as the application sees it (localhost), not host.docker.internal.
- Mention which HexStrike tool found it, so you can measure what the scanner added.
```
