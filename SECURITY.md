# Security Policy

This policy defines how security vulnerabilities in the Universal Name Generator — Wordlists repository are reported, investigated, remediated, and disclosed. Security maintenance applies to the latest version distributed through GitHub Releases; preceding versions and unofficial distributions are not supported.

## 📑 Table of Contents

- [Supported Versions](#-supported-versions)
- [Reporting a Vulnerability](#-reporting-a-vulnerability)
- [Scope](#-scope)
- [Disclosure Policy](#-disclosure-policy)

## 🛡️ Supported Versions

Use this table to indicate which project versions currently receive security maintenance.

| Version | Distribution Channel | Supported |
|---------|--------------------|-----------|
| Latest version | GitHub Releases | ✅ |
| Latest version | Unofficial third-party distribution channels | ❌ |
| Preceding versions | Any distribution channel | ❌ |

## 🚨 Reporting a Vulnerability

Please do not disclose suspected vulnerabilities publicly before maintainers have had an opportunity to validate and remediate them.

To report a vulnerability:
- [GitHub Security Advisories](https://github.com/hmlendea/universal-name-generator-wordlists/security/advisories)
- Contact the maintainers directly

## 📌 Scope

The original maintainers are not responsible for changes, modifications, or security issues introduced by downstream forks.

The subsequent report categories are in scope for this repository:
- Malicious, misleading, or otherwise harmful entries introduced into the curated wordlist files under `wordlists/`
- Defects in the generation schema definitions in `GenerationSchemas.xml` that could cause downstream generators to produce unsafe output
- Integrity of the release archives published through GitHub Releases

The subsequent categories are out of scope unless explicitly stated to the contrary:
- Vulnerabilities in the Universal Name Generator application, API, or other downstream consumers of these wordlists
- Generated name content, ordinary data-quality issues, and reports without a reproducible security impact
- Issues requiring access to maintainer infrastructure, private credentials, or unavailable third-party systems

## 📢 Disclosure Policy

This project follows coordinated disclosure:
1. Vulnerabilities are investigated privately.
2. A remediation plan is prepared and validated.
3. Public disclosure is published after a fix, mitigation, or agreed risk decision is available.
4. Credit is attributed in accordance with reporter preference and project policy.
