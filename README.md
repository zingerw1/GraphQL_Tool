<div align="center">

```
  ██████╗  ██████╗ ██╗         ███████╗██╗  ██╗██████╗ ██╗      ██████╗ ██╗████████╗
 ██╔════╝ ██╔═══██╗██║         ██╔════╝╚██╗██╔╝██╔══██╗██║     ██╔═══██╗██║╚══██╔══╝
 ██║  ███╗██║   ██║██║         █████╗   ╚███╔╝ ██████╔╝██║     ██║   ██║██║   ██║
 ██║   ██║██║▄▄ ██║██║         ██╔══╝   ██╔██╗ ██╔═══╝ ██║     ██║   ██║██║   ██║
 ╚██████╔╝╚██████╔╝███████╗    ███████╗██╔╝ ██╗██║     ███████╗╚██████╔╝██║   ██║
  ╚═════╝  ╚══▀▀═╝ ╚══════╝    ╚══════╝╚═╝  ╚═╝╚═╝     ╚══════╝ ╚═════╝ ╚═╝   ╚═╝
```

**Schema-driven GraphQL exploitation tool — introspects first, attacks everything.**

![Python](https://img.shields.io/badge/Python-3.8+-3776AB?style=flat-square&logo=python&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-green?style=flat-square)
![Platform](https://img.shields.io/badge/Platform-Linux%20%7C%20Windows%20%7C%20macOS-lightgrey?style=flat-square)
![Use Case](https://img.shields.io/badge/Use%20Case-CTF%20%7C%20Pentest%20%7C%20Bug%20Bounty-red?style=flat-square)

</div>

---

## Overview

`graphql_exploit.py` is a schema-aware GraphQL security testing tool. It pulls the full introspection schema from a live endpoint and uses it to drive every subsequent attack module — no field names, argument names, or query structures are hardcoded. Give it a URL and it figures out the rest.

Built for HTB machines, CTF challenges, and authorised penetration tests.

---

## Attack Modules

| Module | Flag | What it does |
|---|---|---|
| **Introspection** | `--introspect` | Dumps full schema — query fields, mutations, types, args |
| **API Key Hunt** | `--apikeys` | Finds fields returning keys/tokens, ranks by privilege (admin > moderator > user > guest), tests each against auth-gated queries |
| **IDOR** | `--idor` | Queries all schema fields with and without auth keys, probes with common values, flags sensitive data |
| **SQL Injection** | `--sqli` | 5-tier chain: detect → confirm → column count → DB fingerprint → full table/data dump |
| **XSS** | `--xss` | 4-tier: reflection detect → standard payloads → WAF bypass → type-mismatch (String into Int args) |
| **Mutations** | `--mutations` | Enumerates all mutations, tests privilege escalation via role fields |
| **SSTI** | `--ssti` | Detects template injection (Jinja2, Mako, etc.) via math evaluation probes |
| **NoSQL Injection** | `--nosql` | Operator injection probes (`$gt`, `$where`, etc.) with confirmation |
| **Full Chain** | `--all` | Runs everything above in order |

**Flag extraction** is automatic — every response is scanned for `HTB{...}`, `flag{...}`, `ctf{...}`, and 32-char hex strings.

---

## Installation

```bash
git clone https://github.com/zingerw1/GraphQL_Tool.git
cd GraphQL_Tool
pip install requests
```

> Requires Python 3.8+. On Windows use `python -X utf8` to handle the banner correctly.

---

## Usage

```
python3 graphql_exploit.py --url <endpoint> [options]
```

### Arguments

```
Required:
  --url         GraphQL endpoint (e.g. http://target/graphql)

Auth:
  --token       Bearer token
  --cookie      Cookie string (e.g. session=abc123)
  --apikey      Known API key to inject into queries

Attack modules:
  --all         Full automated chain (recommended first run)
  --introspect  Schema dump only
  --apikeys     API key hunt
  --idor        IDOR on all schema fields
  --sqli        SQLi full 5-tier chain
  --xss         XSS reflection testing
  --mutations   Mutation enumeration + privilege escalation
  --ssti        Server-Side Template Injection probes
  --nosql       NoSQL injection probes

SQLi overrides:
  --field       Override injectable field name
  --arg         Override injectable argument name

Misc:
  --payloads    Path to gql_payloads.json (auto-detected if omitted)
```

---

## Examples

**First run — let the tool figure everything out:**
```bash
python3 graphql_exploit.py --url http://target/graphql --all
```

**Authenticated with a Bearer token:**
```bash
python3 graphql_exploit.py --url http://target/graphql --token eyJhbG... --all
```

**Authenticated with a cookie:**
```bash
python3 graphql_exploit.py --url http://target/graphql --cookie "session=abc123" --all
```

**Schema dump only (no attacks):**
```bash
python3 graphql_exploit.py --url http://target/graphql --introspect
```

**Hunt API keys, then re-run with the best one:**
```bash
python3 graphql_exploit.py --url http://target/graphql --apikeys
python3 graphql_exploit.py --url http://target/graphql --apikey <found_key> --all
```

**SQLi only, with a field override:**
```bash
python3 graphql_exploit.py --url http://target/graphql --sqli --field userByName --arg name
```

---

## How It Works

### 1 — Introspection
Queries `__schema` to build a complete map of the API: every query field, mutation, object type, input type, and argument — including their types and whether they're required.

### 2 — Schema-Driven Attacks
Every module reads from that map:
- **Auth args** (`apiKey`, `token`, `auth`, `key`, etc.) are identified dynamically from the schema and passed correctly in every query — no field names are hardcoded.
- **Injectable args** (String/ID types) are collected per-field and tested individually.
- **Return fields** are built automatically from the object type definitions.

### 3 — SQLi Chain (5 Tiers)

```
Tier 1  DETECT      Error-based probes on all String/ID args
Tier 2  CONFIRM     Confirm with disambiguation payloads
Tier 3  COL COUNT   UNION SELECT with increasing NULLs
Tier 3b REFLECTION  Sentinel probe to map SQL column → GraphQL field
Tier 4  FINGERPRINT @@version / database() / sqlite_version()
Tier 5  DUMP        Tables → columns → data via GROUP_CONCAT / string_agg
```

Blind time-based fallback (SLEEP/WAITFOR) runs automatically if error-based detection fails.

### 4 — API Key Ranking
Keys found in the schema are ranked by role: `admin/root` → priority 0, `moderator/staff` → 1, `user` → 2, `guest/viewer` → 3. The best key is used first in all subsequent modules.

---

## File Structure

```
GraphQL_Tool/
├── graphql_exploit.py   # Main tool
└── gql_payloads.json    # Payload library (SQLi, XSS, SSTI, NoSQL, fallback queries)
```

`gql_payloads.json` contains categorised payloads for each module. The tool loads it automatically if it's in the same directory.

---

## Recommended Workflow

```
1.  --introspect          Confirm endpoint is live, read the schema
2.  --apikeys             Find and rank any exposed API keys
3.  --all --apikey <key>  Full chain with the best key
4.  Check summary         FLAG auto-extracted if present
5.  --sqli --field X      Re-run SQLi on a specific field if needed
```

---

## Legal

For authorised security testing, CTF competitions, and educational use only.
Do not run against systems you do not have permission to test.

---

<div align="center">
Built for CWES Module 17 — GraphQL Attack
</div>
