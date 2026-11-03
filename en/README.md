# The XZ Utils backdoor: how the build chain nearly reached SSH

**CVE-2024-3094 · a detailed supply-chain incident report**

[← Home](../README.md) · [Русская версия →](../ru/README.md)

> Need SOC/IR-level detail? See the [technical appendix: build stages, exploit conditions, safe triage, and detection engineering](technical-deep-dive.md).

> **In brief.** A multi-stage backdoor was planted in XZ Utils 5.6.0 and 5.6.1. On a narrow set of Linux builds, it could alter the behavior of systemd-launched `sshd` and enable remote code execution. This was not a flaw in the SSH protocol and not an attack triggered by someone merely opening an archive: malicious logic entered the library during package building.

## What happened

XZ Utils is a set of compression tools. Its `liblzma` library is used by many programs. In late March 2024, engineer Andres Freund noticed unusual latency during SSH logins on Debian sid, along with Valgrind errors. Tracing those symptoms led him to malicious changes in the published XZ source archives. He disclosed the issue publicly on March 29.

The affected upstream releases were **5.6.0 and 5.6.1**. A key detail is the difference between the Git repository and release tarballs: some malicious build logic appeared in the release archives but not in the corresponding Git source files. Reviewing commits alone would not necessarily reveal what went into the release.

## How the project was infiltrated

Public technical records describe a long effort to gain trust in the project and eventually influence its releases. The project history includes years of contributions, discussions, and pressure on an overburdened maintainer. The social side mattered: the attacker needed trust and influence over software distribution, not simply a network port to exploit.

The actor’s identity, motive, and any state sponsorship have **not been established by public evidence strongly enough to name a sponsor with confidence**. This report does not present attribution theories as fact.

## The attack chain, from source to SSH

A simplified view:

```text
trust and release influence
          ↓
changes to test data + special logic in release tarball
          ↓
build scripts extract a hidden object from test data
          ↓
under matching conditions, the liblzma build is altered
          ↓
the compromised library is linked into system components
          ↓
in some configurations it affects systemd-launched sshd
          ↓
crafted authentication request → key-verification path → code execution
```

### 1. Concealment in source data and the release archive

Unusual compressed files were added to the repository under the appearance of test data. Separate build logic was placed in release tarballs. During package building, it could extract and decode hidden components from those test files and pass them to a shell. This was a **build-time** action, not something triggered by an ordinary user extracting an archive.

### 2. Conditional modification of liblzma

The extracted code interfered with the library build. The mechanism was conditional and depended on the build and packaging environment; it did not activate identically on every system. Under matching conditions, the result was a modified `liblzma`.

### 3. The path from liblzma to sshd

OpenSSH does not normally use XZ as part of the SSH protocol. The risk came from a particular integration: in some Debian and Fedora builds, `sshd` was linked with systemd, which used `liblzma`. That created a path for library code to interfere with SSH server functions. Exploitation required a narrow combination of vulnerable build, launch method, and target request; it did not mean that every Linux host running SSH was vulnerable.

### 4. Key verification and payload execution

Technical analysis found that the backdoor intercepted the RSA signature-verification path and checked authentication data for a special condition. When the conditions matched, it could enable code execution with the privileges of the `sshd` process. Public analyses describe a way to run commands remotely before a normal successful login. This should not be confused with a universal bypass of arbitrary SSH keys: the implementation was targeted and depended on attacker-controlled secrets and specific conditions.

## Why it was dangerous

- `liblzma` is widely used, so the issue hid inside an ordinary dependency.
- Malicious components were split across test data, release tarballs, and build logic.
- Reviewing Git commits alone could miss differences between the repository and the published archive.
- A supply-chain compromise could turn a trusted library into a path to remote code execution.
- Builds and distributions differed: seeing an XZ version number alone does not prove that the vulnerable build configuration reached a machine.

## Scope and limits of what is known

Public warnings focused on development and unstable distribution branches, including Fedora Rawhide and Debian sid; other pre-release and development builds also needed review. Broad deployment across stable releases was stopped before the malicious versions became widespread.

A host cannot be declared compromised based on the XZ version number alone. The distribution’s build, whether the affected library was replaced, and how `sshd` was launched all matter. If exposure is plausible, follow the OS vendor’s advisory and investigate it as an incident rather than relying only on a version check.

## How it was found and stopped

Freund noticed a small but unusual slowdown in SSH logins and additional checking errors. That prompted investigation of the library and release tarball, followed by a public warning on March 29, 2024. Distributions began reverting or replacing affected packages. In May 2024, XZ Utils 5.6.2 was released with the backdoor removed.

## What administrators should do

1. **Check the distribution and package build** against the vendor’s official advisory. Do not rely on the upstream version number alone.
2. **Update or roll back the package** as the vendor instructs, and install the fixed version available for your release.
3. If the host ran an affected build while reachable by an attacker, **treat it as potentially compromised** and follow your organization’s incident-response plan. Review authentication logs and system integrity.
4. If compromise is confirmed or likely, remediate from a trusted environment and assess whether credentials and keys accessible from that host need rotation. Reinstalling the package does not undo commands already run.
5. Do not run random “detection” scripts from the internet as root. Use vendor instructions and tools you can verify.

## What maintainers and build teams can improve

- Compare release tarballs with the tagged source and explain every difference.
- Build releases in isolated, reproducible environments and retain artifact provenance.
- Require independent review for build-system and release-pipeline changes.
- Reduce critical projects’ long-term reliance on a single volunteer through funding, maintainer rotation, and backup maintainers.
- Separate permissions for source changes, artifact release, and release signing.
- Watch for unusual changes to test data, generators, and build scripts.

## Sources

### Primary and official

1. Andres Freund, original oss-security disclosure, March 29, 2024: [backdoor in upstream xz/liblzma leading to ssh server compromise](https://www.openwall.com/lists/oss-security/2024/03/29/4).
2. Debian Security Advisory [DSA-5649-1](https://lists.debian.org/debian-security-announce/2024/msg00057.html).
3. NIST NVD: [CVE-2024-3094](https://nvd.nist.gov/vuln/detail/CVE-2024-3094).
4. Red Hat: [CVE-2024-3094](https://access.redhat.com/security/cve/CVE-2024-3094).
5. XZ Utils: [official releases and the 5.6.2 note removing the backdoor](https://github.com/tukaani-project/xz/releases).

### Technical analyses

6. CERT-EU: [Critical Vulnerability in XZ Utils](https://cert.europa.eu/publications/security-advisories/2024-032/).
7. Kaspersky Securelist: [The XZ backdoor story, part 1](https://securelist.com/xz-backdoor-story-part-1/112354/).
8. Elastic Security Labs: [500ms to midnight](https://www.elastic.co/security-labs/500ms-to-midnight/).

---

*Prepared for education and awareness. Claims and conclusions were checked against the sources above; attribution is intentionally not asserted without evidence. Links and current status last checked: 2026-11-03.*
