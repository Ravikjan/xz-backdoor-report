# The XZ Backdoor — CVE-2024-3094

<p align="center">
  <strong>A clear, sourced account of the supply-chain attack that nearly reached Linux SSH servers.</strong><br>
  <em>Русский и English — каждый язык в отдельной папке.</em>
</p>

<p align="center">
  <a href="ru/README.md"><img alt="Читать по-русски" src="https://img.shields.io/badge/Читать-по--русски-2563eb?style=for-the-badge"></a>
  <a href="en/README.md"><img alt="Read in English" src="https://img.shields.io/badge/Read-in--English-16a34a?style=for-the-badge"></a>
  <a href="https://nvd.nist.gov/vuln/detail/CVE-2024-3094"><img alt="CVE-2024-3094" src="https://img.shields.io/badge/CVE-2024--3094-critical-b91c1c?style=for-the-badge"></a>
</p>

> **In one sentence:** a carefully planted backdoor in XZ Utils 5.6.0 and 5.6.1 could, on a narrow set of Linux builds, alter how systemd-launched OpenSSH handled authentication and give an attacker remote code execution.

## Start here

| | |
|---|---|
| 🇷🇺 Русский отчёт | [Открыть русскую версию](ru/README.md) |
| 🇬🇧 English report | [Open the English version](en/README.md) |
| 🇷🇺 Русское техническое приложение | [Триаж, механика, detection engineering](ru/technical-deep-dive.md) |
| 🇬🇧 English technical appendix | [Triage, mechanics, detection engineering](en/technical-deep-dive.md) |

Both reports cover the background and timeline, the multi-stage build-time mechanism, the SSH attack path, what was and was not affected, discovery and response, practical checks, and lessons for maintainers and operators.

## Sources and scope

This is an educational incident report, not an attribution claim or a substitute for distribution-specific incident response. It distinguishes documented facts from open questions and links to primary advisories and technical analyses. The incident is tracked as **CVE-2024-3094**.

## Sources at a glance

- [Andres Freund's original disclosure](https://www.openwall.com/lists/oss-security/2024/03/29/4)
- [Debian Security Advisory DSA-5649-1](https://lists.debian.org/debian-security-announce/2024/msg00057.html)
- [NVD: CVE-2024-3094](https://nvd.nist.gov/vuln/detail/CVE-2024-3094)
- [Red Hat: CVE-2024-3094](https://access.redhat.com/security/cve/CVE-2024-3094)
- [XZ Utils release notes](https://github.com/tukaani-project/xz/releases)

---

<p align="center"><sub>Read critically. Verify versions against your distribution's advisory. Last reviewed: 2026-11-03.</sub></p>
