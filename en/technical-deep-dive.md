# Technical appendix: CVE-2024-3094 for security teams

[← Incident overview](README.md) · [Русский разбор →](../ru/technical-deep-dive.md) · [Home](../README.md)

This appendix is for security engineers, Linux administrators, package maintainers, and CI/CD teams. It lays out the technical model, scope boundaries, and a safe triage workflow. It is not an exploit guide and contains no payload or instructions for reproducing the backdoor.

## 1. Threat model and executive technical summary

**Class:** source and build-artifact supply-chain compromise.  
**Component:** XZ Utils / `liblzma`, upstream versions 5.6.0 and 5.6.1.  
**Target path:** particular Linux OpenSSH `sshd` builds launched through systemd and linked into `liblzma`.  
**Impact:** conditional code execution in the SSH server process before normal authentication.

The phrase “XZ is vulnerable” hides the important conditions. The incident was not simply the presence of XZ on a host, nor something triggered by a user opening an archive. The chain required a malicious release tarball, attacker logic executing during package build, a resulting library, a vulnerable `sshd` integration, and a suitable request. A CVE-version match is a useful triage signal, not proof of host compromise.

## 2. Technical chain, stage by stage

### A. Project access and release trust

The attacker spent a long period building credibility in the project and eventually gained influence over changes and distribution. The boundary under attack was trust: who could modify critical files, prepare releases, and control artifacts. Public history confirms years of contributions and gradual influence, but does not establish the actor’s identity or sponsor with confidence.

### B. Git tree versus release tarball

A consequential stage was present in the source archives for releases 5.6.0/5.6.1 but did not match the ordinary files in the visible Git tree. The disclosure points to malicious modifications to `m4/build-to-host.m4`: configure logic searched for a disguised test file and, under conditions, began an extraction chain. Further stages were embedded in the test data.

**Release-engineering lesson:** a signed tag or commit review does not prove that a published tarball is byte-for-byte identical to the tagged source tree. Compare the actual artifact and source tree, and verify who produced the archive and by what process.

### C. Conditional build-time activation

The malicious configure fragment joined data from files that appeared to be tests, decoded it, and passed it to an interpreter. A later stage altered the generated library Makefile, allowing a concealed prebuilt object file to be linked into `liblzma`. The “test” containers remained in the source archive, so a superficial review of the library’s C code did not expose the full execution path.

The logic included environment checks; it was not expected to activate identically on every platform and build. Public analyses describe a tarball-specific injection and targeted packaging/build conditions. Exact applicability depends on the package and its provenance. Do not generalize one upstream artifact’s behavior to every build that carries the same version number.

### D. From liblzma to the dynamic linker

The compromised build used dynamic-loader symbol-resolution mechanisms. In his initial analysis, Freund described resolver functions and a temporary audit hook. During symbol resolution, the PLT path for `RSA_public_decrypt` in the main `sshd` binary was changed. This gave the library a way to intercept a call in OpenSSH’s context, rather than through an ordinary XZ API.

That is why the dependency path looked surprising: the issue was not “SSH uses an archive tool to process SSH,” but a specific Linux package dependency graph and early symbol resolution at process startup.

### E. Authentication path and code execution

The intercepted path inspected RSA-authentication input and checked for a specific condition associated with attacker control. When the conditions matched, the backdoor could pass a command for execution with the privileges of the `sshd` process. This bypassed the normal successful-authentication decision; it was not a universal cracking of arbitrary user accounts or SSH keys.

Public reverse-engineering analyses document the targeted design and its constraints, but do not justify claiming widespread real-world exploitation. Exploitability and confirmed exploitation of a particular host are different claims.

## 3. Conditions to assess separately

For a specific host, answer each question:

1. Did it use XZ Utils 5.6.0/5.6.1, or a derived/rebuilt library, during the period under review?
2. Where did the package come from: the distribution repository, a third-party repository, or an internal build pipeline?
3. Did that exact artifact contain the malicious release/build logic, and did its build conditions match?
4. Was `sshd` built/launched through the affected systemd path and linked to that same `liblzma`?
5. Was the daemon reachable by a potential attacker during the exposure window?
6. Is there evidence of suspicious authentication attempts, command execution, follow-on access, or host changes?

A positive package-version match does not equal “the host was hacked.” A negative answer to one field does not close the question if the artifact came from a third-party build.

## 4. Safe Linux host triage

First preserve the host, timestamp, image/disk snapshot, running processes, and package source according to your incident-response procedure. If compromise is plausible, do not update or “clean” the only copy before collecting available evidence.

### Establish package version and provenance

Debian/Ubuntu:

```sh
dpkg-query -W -f='${Package} ${Version} ${Status}\n' xz-utils liblzma5
apt-cache policy xz-utils liblzma5
```

RPM systems:

```sh
rpm -q xz xz-libs
rpm -qi xz-libs
```

Compare the result with the advisory for **your exact distribution and release**. An upstream version string without the distro release/revision and package source is incomplete.

### Inspect SSH server dependencies

On a trusted copy of the binary, or as part of controlled local triage:

```sh
readelf -d /usr/sbin/sshd | grep -E 'NEEDED|liblzma|libsystemd'
```

This only inventories ELF dependencies. A systemd/liblzma dependency is assessment context, not proof of infection. Account for static builds, alternate binary paths, containers, chroots, and vendor patches. Use distribution tooling to determine the libraries actually loaded, and collect results from a trusted environment.

### Review logs and behavior

Correlate the exposure window with `sshd`, systemd/journald, EDR/auditd, and network telemetry. Look beyond successful logins: inspect unexpected SSH-server child processes, short anomalous sessions, commands, new accounts/keys, `authorized_keys` changes, systemd units, cron, shell history, and outbound connections. These signals are not specific to XZ; correlate them with expected host activity and the timeline.

Public YARA signatures and indicators are an additional layer, but:

- hashes or signatures may not cover every build or variant;
- no match does not prove a host is clean;
- a known string/IOC can be changed;
- scanning a live suspect does not replace analysis of trusted images and package artifacts.

Do not turn a published IOC into a blocking rule until you have checked context, license, and false-positive risk.

## 5. Response guidance

### If version or provenance is unclear

- Restrict access as far as business operations allow.
- Preserve logs and package/repository metadata.
- Check the vendor advisory and inventory potentially exposed nodes through asset inventory/SBOM.
- Ask the base-image and CI owners for artifact provenance and build logs.

### If an affected build is confirmed but exploitation is not

- Roll back/update as directed by the vendor.
- Preserve artifacts showing package composition and source.
- Assess SSH network exposure, time window, and attacker reachability.
- Review logs and EDR telemetry for the full interval.
- Document why the incident is closed as “exposure without evidence of exploitation,” if that is the conclusion.

### If execution or follow-on access is suspected

Treat the host as potentially compromised. Isolate it under your IR plan, preserve forensic data, scope possible lateral movement, rebuild from a trusted image, and rotate secrets available from the host from a clean environment. Reinstalling `xz` does not remove persistence or undo commands that already ran.

## 6. Detection engineering: what to observe

### Useful telemetry

- package-manager inventory and transaction history;
- SBOM/container image and base-image source;
- file integrity/hashes for `liblzma`, `sshd`, and packages;
- process lineage for `sshd` and child processes;
- SSH auth logs, auditd/eBPF/EDR process events;
- outbound network events from SSH or unexpected shells;
- CI records: downloaded tarballs, hashes, build logs, signatures, provenance attestations.

### Analytic idea

Correlate: a potentially affected package was present **during the relevant window** + an SSH listener was reachable + unusual process/authentication events occurred. Do not build a detection around a process name or one 500 ms slowdown: latency helped the original researcher, but is not a reliable production IOC.

Elastic published YARA/osquery/KQL examples and behavioral detection references for suspicious execution through SSH. Treat these as reference material for your telemetry, not universally precise signatures.

## 7. Short timeline

- **February 2024:** XZ Utils 5.6.0 is released.
- **March 2024:** 5.6.1 is released; packages begin to reach some development/unstable branches.
- **March 28–29, 2024:** Andres Freund investigates SSH/Valgrind anomalies and finds the compromise.
- **March 29, 2024:** public disclosure on oss-security; distributions warn users and roll packages back.
- **May 2024:** XZ Utils 5.6.2 release notes state that the backdoor was removed.

These dates summarize upstream and public disclosure. Determine a system’s actual exposure window from the distribution’s package records.

## 8. What this incident changed for AppSec/ProdSec

**Release engineering:** verify that a signed source archive matches its tag, publish provenance, build in an isolated environment, and do not execute untrusted build helper scripts.

**Code review:** separately scrutinize build scripts, m4/autoconf, generators, binary test files, CI changes, and archives. A diff of C source does not cover all executable build logic.

**Identity and governance:** MFA/2FA for maintainers is a baseline, not a complete control. Add separation of duties, second-person release review, recovery-channel controls, and prompt access removal when roles change.

**Reproducibility:** reproducible builds do not automatically detect a backdoor, but they improve the chance of noticing a mismatch between declared source and released artifact. Compare independent builds and retain provenance.

**OSS resilience:** dependence on one person for a critical project is an operational risk. Fund maintenance, establish backup maintainers, and make role handoffs safe.

## 9. Do not confuse this incident with

- a vulnerability in the SSH protocol;
- an exploit triggered by simply opening an `.xz` file;
- proof that every Linux host running XZ or OpenSSH was exposed;
- confirmation of widespread exploitation in real networks;
- a conventional memory-corruption bug in the archive parser.

It is an example of how compromise of a trusted project role and build pipeline can bypass a naïve review of application source code.

## Technical references

- Andres Freund, primary analysis including loader/hook details: https://www.openwall.com/lists/oss-security/2024/03/29/4
- Follow-up clarifying build-script stages: https://openwall.com/lists/oss-security/2024/03/30/43
- Red Hat CVE record and scope statement: https://access.redhat.com/security/cve/CVE-2024-3094
- Debian DSA-5649-1: https://lists.debian.org/debian-security-announce/2024/msg00057.html
- Elastic Security Labs: reverse engineering, YARA, and hunting references: https://www.elastic.co/security-labs/500ms-to-midnight
- CERT-EU advisory: https://cert.europa.eu/publications/security-advisories/2024-032/
- XZ Utils 5.6.2 release note: https://github.com/tukaani-project/xz/releases/tag/v5.6.2

---

*Appendix version: 2026-11-03. For operational decisions, current guidance from your OS vendor takes precedence.*
