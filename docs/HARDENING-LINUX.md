# Node hardening guideline: Amazon Linux 2023 on EKS

| Item | Value |
|---|---|
| Written | 2026-10-24. Revised 2026-10-24 after the owner's decisions (section 7.7). |
| Status | Guideline for the agent that writes the Ansible role and the validation tool. It is not the role. |
| Applies to | One EKS managed-node-group node, Amazon Linux 2023 EKS-optimized AMI, `c6i.xlarge`, amd64, Kubernetes 1.37 (see `docs/SPEC.md` sections 2 and 4). |
| Benchmark | CIS Amazon Linux 2023 Benchmark v1.0.0 (released 2023-06-26), Level 1 Server. |
| Control ID format | `CIS-AL2023-1.0.0:<number>`, for example `CIS-AL2023-1.0.0:3.2.1`. The table rows below show only `<number>`. |
| Owner decisions | Level 1 Server only. MOTD wording approved. Controls 1.8 and 4.6.6 skipped. PAM (4.4.x) left unchanged. SSH by a downloaded AWS key pair only, EC2 Instance Connect off (confirmed 2026-10-24, section 7.5). |
| Recommendation | Use the upstream `ansible-lockdown/AMAZON2023-CIS` role, pinned to tag `1.5.0`, with an overrides file and a thin wrapper playbook. Do not write our own role. See section 6.5. |

## 1. Summary of findings

1. **Current OS.** Amazon Linux 2023 (AL2023) is the current generally available Amazon Linux for EKS nodes. The EKS AMI repository builds AL2023 images from source AMI `al2023-ami-minimal-2023.12.20260928.0` in its release `v20260930`. An "Amazon Linux 2027" template exists in the same repository, but its source AMI filter is `al2027-preview-ami-minimal-2027.*-kernel-7.*` with a note to pin a kernel "before AL2027 GA", so it is a preview and not usable for this build. Amazon Linux 2 reached end of support on 2026-06-30. (Sources S1, S2, S7.)
2. **Benchmark.** CIS lists Amazon Linux 2023 at version 1.0.0 and no newer Amazon Linux entry. The same page lists Amazon Linux 2 at 4.0.0 and an "Amazon Linux 2 STIG" at 2.0.1. (S10.) I found no CIS Amazon Linux 2023 version newer than 1.0.0.
3. **Maintained CIS automation exists.** `ansible-lockdown/AMAZON2023-CIS` (MIT) implements CIS AL2023 v1.0.0, last pushed 2026-10-19, latest release `1.5.0` on 2026-10-08. ComplianceAsCode (OpenSCAP) also ships CIS AL2023 v1.0.0 profiles and generated Ansible playbooks in release v0.1.82. (S12, S14.) Because a maintained CIS playbook exists, the STIG fallback is not needed. It is still documented in section 6.3 and 6.4 because you asked for it.
4. **Neither automation is EKS-aware.** Run unmodified, the upstream role would: set net.ipv4.ip_forward to 0 (control 3.2.1), switch SELinux to enforcing (controls 1.6.1.3 and 1.6.1.4, through a default variable), run a full `dnf update` (1.8), fail the whole run on the root-password assertion (4.6.6), point time sync at public pools (2.1.2), and set a 100G journald free-space floor (5.1.2.6). Each has an override in section 5.2 and a row in section 7.
5. **Scope.** The control table in section 5.3 covers all 179 Level 1 Server controls of the benchmark (the upstream role and ComplianceAsCode agree on the count). The 64 Level 2 Server controls are not selected; section 5.4 lists the ones that would conflict with Kubernetes if they were added later.
6. **Not verified on a node.** Nothing here was run on an EKS node. Every "verify" command is taken from the upstream goss audit content or derived from it, and every claim about runtime behaviour is marked UNVERIFIED where I could not confirm it from a source I read. The full CIS PDF could not be read (registration required), so control titles come from two public implementations (S12, S14) and may differ in wording from the PDF.

## 2. Platform facts

| Fact | Value | Source |
|---|---|---|
| AL2023 support | Released March 2023. Standard support ends 2027-06-30. Maintenance (security and critical fixes) ends 2029-06-30. | S1 |
| AL2 support | End of support 2026-06-30. | S2 |
| EKS and AL2 | AWS documentation says EKS stopped publishing AL2 AMIs on 2025-11-26, and that new managed node groups on Kubernetes 1.30 or newer default to AL2023. I read the second statement on the EKS AMI page (S5). The first came from a search result summary of the AL2 transition FAQ; I did not open that page (S6). | S5, S6 |
| AL2023 minor in EKS AMIs | 2023.12, build 20260928, in EKS AMI release v20260930. Kernel 6.18 for Kubernetes 1.36 and 1.37, kernel 6.12 for 1.33 to 1.35. | S7 |
| Defaults that matter | SELinux is enabled in permissive mode. `/tmp` is tmpfs. The systemd journal is the default log, not rsyslog. Cron is replaced by systemd timers. | S3, S4 |
| What the EKS AMI already sets | Sysctls `net.ipv4.ip_forward = 1`, `net.bridge.bridge-nf-call-iptables = 1`, `vm.overcommit_memory = 1`, `kernel.panic = 10`, `kernel.panic_on_oops = 1`, a reserved port list, large inotify limits. Modules `ip_tables` and `iptable_nat` loaded at boot. `update-motd.timer` is disabled. cloud-init package updates are turned off. Kernel, containerd and `amazon-ec2-net-utils` packages are version-locked. | S7 (`templates/al2023`) |
| Ansible on the node | The AL2023 2023.12 package list shows `ansible-core` 2.15.3 and `ansible` 8.3.0. The upstream CIS role requires ansible-core 2.16.1 or newer (`min_ansible_version` in its `vars/main.yml`). So the distribution package is too old for the role. | S9, S12 |

## 3. Method and honesty notes

- **Control numbers** are taken from the two public implementations of CIS AL2023 v1.0.0: the upstream role (tag `1.5.0`) and the ComplianceAsCode control file `controls/cis_al2023.yml` (release v0.1.82). They list the same 243 controls (179 Level 1 Server, 64 Level 2 Server) and the same Level split, except two controls (5.2.4.2 and 5.2.4.3) that the role leaves without a level tag.
- **Titles** are shortened and reworded labels, not benchmark text.
- **Ansible implementation** is the role's rule variable (`amzn2023cis_rule_<number with underscores>`) and the Ansible modules its tasks use, read from the role source. In the modules column, `cg.` means `community.general` and `posix.` means `ansible.posix`; everything else is `ansible.builtin`.
- **Verify** commands are my rendering of the upstream goss audit checks (`ansible-lockdown/AMAZON2023-CIS-Audit`, branch `benchmark_1.0.0`), shortened. The goss file is the authority for exact regexes. I did not run any of them.
- **Safe for EKS** is my judgement from the sources named in each row. "Safe" means I found no conflict, not that it was tested.
- **Names.** Per the project rules, this document names only standards, AWS products and open-source projects.

## 4. What must be installed, removed or disabled

This is the end state for a node counted as hardened at CIS AL2023 v1.0.0 Level 1 Server, minus the deviations in section 7.

### 4.1 Present and required (do not remove)

Installed by the EKS AMI build (`templates/al2023/provisioners/install-worker.sh`, S7): `amazon-ssm-agent`, `chrony`, `containerd`, `runc`, `kubelet`, `nodeadm`, `iptables-nft`, `nftables`, `conntrack`, `socat`, `ipvsadm`, `nfs-utils`, `jq`, `unzip`, `wget`. `ec2-instance-connect` is also installed by the AMI build but is removed by this guideline (section 7.5, decided 2026-10-24). Hardening must leave the rest installed and running. `amazon-ssm-agent` is how the hardening is delivered, so no control may disable it or block its network access.

### 4.2 Added by Level 1 controls

| Package or service | Control | Note |
|---|---|---|
| `aide` | 1.3.1 | Plus a scheduled check (1.3.2). Needs path exclusions on a container host (section 7). |
| `cronie` and `crond` | 4.1.1 | AL2023 normally uses systemd timers, so this is an addition. |
| `libselinux` | 1.6.1.1 | Normally already present. |
| `sudo` | 4.3.1 | Normally already present. |
| `chrony` | 2.1.1, 2.1.2 | Already present. Configure it for Amazon Time Sync. |
| `rsyslog` | 5.1.1.x | Only if `amzn2023cis_syslog_service: rsyslog`. This guideline chooses `journald`, so rsyslog is not installed. |
| `systemd-journal-remote` | 5.1.2.1.x | Skipped: no remote log host exists (section 7). Control 5.1.2.1.4 (journald does not receive remote logs) still applies. |

Level 2 would also add `auditd` (5.2.x) and a firewall utility (3.4.x). They are not in scope.

### 4.3 Must be absent or masked (Level 1)

Absent (package names are those the upstream role removes): `avahi`, `avahi-autoipd`, `cups`, `dhcp-server`, `bind`, `vsftpd`, `tftp-server`, `httpd`, `nginx`, `dovecot`, `cyrus-imapd`, `samba`, `squid`, `net-snmp`, `telnet-server`, `dnsmasq`, `telnet`, `openldap-clients`, `ftp`, `setroubleshoot`, `mcstrans`. Control 2.2.1 (`xorg-x11-server-common`) is Level 2 and not selected.

Masked, not removed: `nfs-server` (2.2.16), `rpcbind.socket` and `rpcbind` (2.2.17), `rsyncd` (2.2.18). The benchmark accepts "absent or masked". On this node `nfs-utils` must stay installed because it provides the NFS mount helper that EFS and NFS volumes use (the EKS AMI installs it on purpose), so mask.

No mail transfer agent may listen on a non-loopback address (2.2.15).

### 4.4 Must be disabled in configuration

Kernel modules `cramfs`, `freevxfs`, `jffs2`, `hfs`, `hfsplus` and `usb-storage` (1.1.1.3 to 1.1.1.7, 1.1.9). Core dump storage (1.5.3, 1.5.4). Remote root login, host-based auth, empty passwords, user environment in SSH (4.2.7 to 4.2.10). Network parameters in 3.2.2 and 3.3.1 to 3.3.9 (not 3.2.1, which is a deviation).

### 4.5 Left alone on purpose

SELinux stays in its AMI default (enabled, permissive). IP forwarding stays on. IPv6 stays enabled. Kernel modules the CNI and kube-proxy load are not blocked. Details in section 7.

## 5. Controls

### 5.1 How to read the table

Each row is one Level 1 Server control of CIS Amazon Linux 2023 Benchmark v1.0.0. Columns:

- **ID**: benchmark control number. Stable ID is `CIS-AL2023-1.0.0:<ID>`.
- **Requirement**: my wording.
- **Ansible implementation**: the upstream role variable that enables the control and the modules its task uses (task names in the role start with `<ID> | PATCH |` or `<ID> | AUDIT |`).
- **Verify**: the check, rendered from the upstream goss audit.
- **EKS**: `Safe`, `Safe, test` (apply, then run the named test), `Override` (apply with a changed value), `Needs input` (a value only the owner can give), `SKIP` (deliberate deviation, recorded in `docs/DECISIONS.md`) or `N/A` (not applicable on this node; the tool records not-applicable).

Source for every row: CIS AL2023 v1.0.0 as implemented by S12 (role) and S14 (control file), verification from S13. Level for every row: 1 (Server). Source type: CIS, not STIG.

Totals for the 179 rows: Safe 133, Safe-with-test 4, Override 13, Needs input 1, SKIP 8, N/A 13 (single-volume node, no separate partition), N/A alternative-chosen 7 (rsyslog set, journald chosen).

### 5.2 Overrides file

The upstream role is configured only through variables. This file is the starting point for the role author. Every variable name below exists in the role at tag `1.5.0` (checked in `defaults/main/main.yml` and `defaults/main/audit.yml`). The values are the decisions of this guideline.

```yaml
# ansible/group_vars/all/cis_al2023_overrides.yml
# Level selection. SSM's AWS-ApplyAnsiblePlaybooks has no tags parameter,
# so select the level with variables and with ansible.cfg [tags] skip (section 8.3).
amzn2023cis_level_1: true
amzn2023cis_level_2: false

# 1.6.1.3 / 1.6.1.4: the role's default 'enforcing' breaks the EKS AMI.
amzn2023cis_selinux_enforce: permissive

# 3.2.1: pod networking and kube-proxy need IP forwarding.
amzn2023cis_rule_3_2_1: false
# 1.8: nodes are patched by replacement, not in place.
amzn2023cis_rule_1_8: false
# 4.6.6: root has no password on EC2; the role asserts it does and aborts.
amzn2023cis_rule_4_6_6: false
# 4.4.1 / 4.4.2: owner decision, PAM stays unchanged.
amzn2023cis_rule_4_4_1: false
amzn2023cis_rule_4_4_2: false
# 5.1.2.1.1 - 5.1.2.1.3: no remote journal host exists.
amzn2023cis_rule_5_1_2_1_1: false
amzn2023cis_rule_5_1_2_1_2: false
amzn2023cis_rule_5_1_2_1_3: false
# 5.2.4.2 / 5.2.4.3 are Level 2 but carry no level tag in the role.
amzn2023cis_rule_5_2_4_2: false
amzn2023cis_rule_5_2_4_3: false

# Logging: AL2023 uses journald.
amzn2023cis_syslog_service: journald
amzn2023cis_journald_systemmaxuse: 500M      # example, size to the volume
amzn2023cis_journald_systemkeepfree: 1G      # role default 100G
# (SystemMaxUse / KeepFree also exist for runtime: amzn2023cis_journald_runtimemaxuse / _runtimekeepfree)

# Time: Amazon Time Sync.
amzn2023cis_time_synchronization_servers:
  - 169.254.169.123

# Keep these as masks. Never 'remove' (nfs-utils is needed for EFS/NFS volumes).
amzn2023cis_nfs_server: mask
amzn2023cis_rpc_server: mask
amzn2023cis_rsync_server: mask

# Keep IPv6 on (IPv6 and dual-stack clusters).
amzn2023cis_ipv6_required: true

# 4.2.4: the default is derived from the connecting user (root under SSM).
# SSH access design is in section 7.5 (decided 2026-10-24).
amzn2023cis_sshd_allowusers: ec2-user

# 6.1.11: keep the file-permission fix on, but see section 7 about the sticky-bit step.
amzn2023cis_no_world_write_adjust: false

# Banner / MOTD text (section 6.4). Must not contain the words the audit rejects.
amzn2023cis_warning_banner: |
  Managed node. Authorized administrators only. Activity is logged.

# No reboot from inside an SSM run.
skip_reboot: true

# Boot mode: UNVERIFIED for this AMI. Check with 'test -d /sys/firmware/efi'.
amzn2023cis_legacy_boot: false
```

Notes for the author:

- `amzn2023cis_rule_*` variables only take effect when the matching section variable (`amzn2023cis_section1` to `amzn2023cis_section6`) is true; leave all six true.
- The role runs `ansible.posix.selinux` for 1.6.1.3 and 1.6.1.4 with `state: '{{ amzn2023cis_selinux_enforce }}'`. With `permissive`, the benchmark's "mode is not disabled" requirement (1.6.1.4) still passes.
- The role template for chrony (`templates/etc/chrony.conf.j2`) is Debian-derived (it references `/etc/chrony/chrony.keys` and sets `local stratum 10`) and is written to `/etc/chrony.d/chrony.conf`. Whether that file coexists cleanly with the AL2023 `chrony.conf` is UNVERIFIED; test `chronyc sources` and `chronyc tracking` after the run. If it misbehaves, set `amzn2023cis_rule_2_1_2: false` and ship our own small template under the same control ID.

### 5.3 Level 1 Server control table

Legend for the EKS column is in section 5.1. Rows with `Override`, `SKIP`, `Needs input` or `Safe, test` have an entry in section 7.


### 5.1 Section 1: Initial setup

| ID | Requirement | Ansible implementation | Verify (command -> expected) | EKS |
|---|---|---|---|---|
| 1.1.1.3 | mounting of cramfs filesystems is disabled | `amzn2023cis_rule_1_1_1_3`; lineinfile,cg.modprobe | `modprobe -n -v cramfs \| grep -E '(cramfs\|install)'` -> install /bin/true | Safe: Module is not used by kubelet, containerd or the VPC CNI. |
| 1.1.1.4 | mounting of freevxfs filesystems is disabled | `amzn2023cis_rule_1_1_1_4`; lineinfile,cg.modprobe | `modprobe -n -v freevxfs \| grep -E '(freevxfs\|install)'` -> install /bin/true; `/etc/modprobe.d/blacklist.conf` matches `/^blacklist freevxfs/` | Safe: Module is not used by kubelet, containerd or the VPC CNI. |
| 1.1.1.5 | mounting of jffs2 filesystems is disabled | `amzn2023cis_rule_1_1_1_5`; lineinfile,cg.modprobe | `modprobe -n -v jffs2 \| grep -E '(jffs2\|install)'` -> install /bin/true; `/etc/modprobe.d/blacklist.conf` matches `/^blacklist jffs2/` | Safe: Module is not used by kubelet, containerd or the VPC CNI. |
| 1.1.1.6 | mounting of hfs filesystems is disabled | `amzn2023cis_rule_1_1_1_6`; lineinfile,cg.modprobe | `modprobe -n -v hfs \| grep -E '(hfs\|install)'` -> install /bin/true; `/etc/modprobe.d/blacklist.conf` matches `/^blacklist hfs/` | Safe: Module is not used by kubelet, containerd or the VPC CNI. |
| 1.1.1.7 | mounting of hfsplus filesystems is disabled | `amzn2023cis_rule_1_1_1_7`; lineinfile,cg.modprobe | `modprobe -n -v hfsplus \| grep -E '(hfsplus\|install)'` -> install /bin/true; `/etc/modprobe.d/blacklist.conf` matches `/^blacklist hfsplus/` | Safe: Module is not used by kubelet, containerd or the VPC CNI. |
| 1.1.2.1 | /tmp is a separate partition | `amzn2023cis_rule_1_1_2_1`; audit/warn only | `findmnt /tmp` returns a line | Safe: Audit only. AL2023 mounts /tmp and /dev/shm as tmpfs (AL2023 docs). |
| 1.1.2.2 | nodev option set on /tmp partition | `amzn2023cis_rule_1_1_2_2`; set_fact on mount options, then handler with ansible.posix.mount (skipped if no such mount) | `findmnt -no OPTIONS /tmp` includes `nodev` | Safe, test: /tmp is tmpfs on AL2023. noexec on /tmp is the risky one: confirm nodeadm, the SSM agent and Ansible do not execute from /tmp (UNVERIFIED). Ansible's default remote tmp is under the user's home, not /tmp. |
| 1.1.2.3 | noexec option set on /tmp partition | `amzn2023cis_rule_1_1_2_3`; set_fact on mount options, then handler with ansible.posix.mount (skipped if no such mount) | `findmnt -no OPTIONS /tmp` includes `noexec` | Safe, test: /tmp is tmpfs on AL2023. noexec on /tmp is the risky one: confirm nodeadm, the SSM agent and Ansible do not execute from /tmp (UNVERIFIED). Ansible's default remote tmp is under the user's home, not /tmp. |
| 1.1.2.4 | nosuid option set on /tmp partition | `amzn2023cis_rule_1_1_2_4`; set_fact on mount options, then handler with ansible.posix.mount (skipped if no such mount) | `findmnt -no OPTIONS /tmp` includes `nosuid` | Safe, test: /tmp is tmpfs on AL2023. noexec on /tmp is the risky one: confirm nodeadm, the SSM agent and Ansible do not execute from /tmp (UNVERIFIED). Ansible's default remote tmp is under the user's home, not /tmp. |
| 1.1.3.2 | nodev option set on /var partition | `amzn2023cis_rule_1_1_3_2`; set_fact on mount options, then handler with ansible.posix.mount (skipped if no such mount) | `findmnt -no OPTIONS /var` includes `nodev` | N/A: Option applies to a separate partition. The node has a single root volume, so the role task is skipped and the tool records not-applicable. UNVERIFIED that no such mounts exist: confirm with `findmnt`. |
| 1.1.3.3 | nosuid option set on /var partition | `amzn2023cis_rule_1_1_3_3`; set_fact on mount options, then handler with ansible.posix.mount (skipped if no such mount) | `findmnt -no OPTIONS /var` includes `nosuid` | N/A: Option applies to a separate partition. The node has a single root volume, so the role task is skipped and the tool records not-applicable. UNVERIFIED that no such mounts exist: confirm with `findmnt`. |
| 1.1.4.2 | noexec option set on /var/tmp partition | `amzn2023cis_rule_1_1_4_2`; set_fact on mount options, then handler with ansible.posix.mount (skipped if no such mount) | `findmnt -no OPTIONS /var/tmp` includes `noexec` | N/A: Option applies to a separate partition. The node has a single root volume, so the role task is skipped and the tool records not-applicable. UNVERIFIED that no such mounts exist: confirm with `findmnt`. |
| 1.1.4.3 | nosuid option set on /var/tmp partition | `amzn2023cis_rule_1_1_4_3`; set_fact on mount options, then handler with ansible.posix.mount (skipped if no such mount) | `findmnt -no OPTIONS /var/tmp` includes `nosuid` | N/A: Option applies to a separate partition. The node has a single root volume, so the role task is skipped and the tool records not-applicable. UNVERIFIED that no such mounts exist: confirm with `findmnt`. |
| 1.1.4.4 | nodev option set on /var/tmp partition | `amzn2023cis_rule_1_1_4_4`; set_fact on mount options, then handler with ansible.posix.mount (skipped if no such mount) | `findmnt -no OPTIONS /var/tmp` includes `nodev` | N/A: Option applies to a separate partition. The node has a single root volume, so the role task is skipped and the tool records not-applicable. UNVERIFIED that no such mounts exist: confirm with `findmnt`. |
| 1.1.5.2 | nodev option set on /var/log partition | `amzn2023cis_rule_1_1_5_2`; set_fact on mount options, then handler with ansible.posix.mount (skipped if no such mount) | `findmnt -no OPTIONS /var/log` includes `nodev` | N/A: Option applies to a separate partition. The node has a single root volume, so the role task is skipped and the tool records not-applicable. UNVERIFIED that no such mounts exist: confirm with `findmnt`. |
| 1.1.5.3 | noexec option set on /var/log partition | `amzn2023cis_rule_1_1_5_3`; set_fact on mount options, then handler with ansible.posix.mount (skipped if no such mount) | `findmnt -no OPTIONS /var/log` includes `noexec` | N/A: Option applies to a separate partition. The node has a single root volume, so the role task is skipped and the tool records not-applicable. UNVERIFIED that no such mounts exist: confirm with `findmnt`. |
| 1.1.5.4 | nosuid option set on /var/log partition | `amzn2023cis_rule_1_1_5_4`; set_fact on mount options, then handler with ansible.posix.mount (skipped if no such mount) | `findmnt -no OPTIONS /var/log` includes `nosuid` | N/A: Option applies to a separate partition. The node has a single root volume, so the role task is skipped and the tool records not-applicable. UNVERIFIED that no such mounts exist: confirm with `findmnt`. |
| 1.1.6.2 | noexec option set on /var/log/audit partition | `amzn2023cis_rule_1_1_6_2`; set_fact on mount options, then handler with ansible.posix.mount (skipped if no such mount) | `findmnt -no OPTIONS /var/log/audit` includes `noexec` | N/A: Option applies to a separate partition. The node has a single root volume, so the role task is skipped and the tool records not-applicable. UNVERIFIED that no such mounts exist: confirm with `findmnt`. |
| 1.1.6.3 | nodev option set on /var/log/audit partition | `amzn2023cis_rule_1_1_6_3`; set_fact on mount options, then handler with ansible.posix.mount (skipped if no such mount) | `findmnt -no OPTIONS /var/log/audit` includes `nodev` | N/A: Option applies to a separate partition. The node has a single root volume, so the role task is skipped and the tool records not-applicable. UNVERIFIED that no such mounts exist: confirm with `findmnt`. |
| 1.1.6.4 | nosuid option set on /var/log/audit partition | `amzn2023cis_rule_1_1_6_4`; set_fact on mount options, then handler with ansible.posix.mount (skipped if no such mount) | `findmnt -no OPTIONS /var/log/audit` includes `nosuid` | N/A: Option applies to a separate partition. The node has a single root volume, so the role task is skipped and the tool records not-applicable. UNVERIFIED that no such mounts exist: confirm with `findmnt`. |
| 1.1.7.2 | nodev option set on /home partition | `amzn2023cis_rule_1_1_7_2`; set_fact on mount options, then handler with ansible.posix.mount (skipped if no such mount) | `findmnt -no OPTIONS /home` includes `nodev` | N/A: Option applies to a separate partition. The node has a single root volume, so the role task is skipped and the tool records not-applicable. UNVERIFIED that no such mounts exist: confirm with `findmnt`. |
| 1.1.7.3 | nosuid option set on /home partition | `amzn2023cis_rule_1_1_7_3`; set_fact on mount options, then handler with ansible.posix.mount (skipped if no such mount) | `findmnt -no OPTIONS /home` includes `nosuid` | N/A: Option applies to a separate partition. The node has a single root volume, so the role task is skipped and the tool records not-applicable. UNVERIFIED that no such mounts exist: confirm with `findmnt`. |
| 1.1.8.1 | /dev/shm is a separate partition | `amzn2023cis_rule_1_1_8_1`; audit/warn only | `findmnt /dev/shm` returns a line | Safe: Audit only. AL2023 mounts /tmp and /dev/shm as tmpfs (AL2023 docs). |
| 1.1.8.2 | nodev option set on /dev/shm partition | `amzn2023cis_rule_1_1_8_2`; set_fact on mount options, then handler with ansible.posix.mount (skipped if no such mount) | `findmnt -no OPTIONS /dev/shm` includes `nodev` | Safe: Affects the host /dev/shm only. Pod /dev/shm mounts are separate tmpfs mounts made by the runtime. |
| 1.1.8.3 | noexec option set on /dev/shm partition | `amzn2023cis_rule_1_1_8_3`; set_fact on mount options, then handler with ansible.posix.mount (skipped if no such mount) | `findmnt -no OPTIONS /dev/shm` includes `noexec` | Safe: Affects the host /dev/shm only. Pod /dev/shm mounts are separate tmpfs mounts made by the runtime. |
| 1.1.8.4 | nosuid option set on /dev/shm partition | `amzn2023cis_rule_1_1_8_4`; set_fact on mount options, then handler with ansible.posix.mount (skipped if no such mount) | `findmnt -no OPTIONS /dev/shm` includes `nosuid` | Safe: Affects the host /dev/shm only. Pod /dev/shm mounts are separate tmpfs mounts made by the runtime. |
| 1.1.9 | usb-storage is disabled | `amzn2023cis_rule_1_1_9`; lineinfile,cg.modprobe | `modprobe -n -v usb-storage \| grep -E '(usb-storage\|install)'` -> install /bin/true; `/etc/modprobe.d/blacklist.conf` matches `/^blacklist usb-storage/` | Safe: Module is not used by kubelet, containerd or the VPC CNI. |
| 1.2.1 | GPG keys are configured (manual) | `amzn2023cis_rule_1_2_1`; shell | `rpm -q gpg-pubkey --qf '%{name}-%{version}-%{release} --> %{summary} '` | Safe: Audit only (manual review of keys and repositories). |
| 1.2.2 | gpgcheck is globally activated | `amzn2023cis_rule_1_2_2`; replace | `if [ `grep -c -E '^\s*gpgcheck.*0' /etc/yum.conf` -ge 1 ]; then echo FAIL; elif [ `grep -c -E '^\s*gpgcheck.*1` -> !/FAIL/; `if [ `grep -c -E '^\s*gpgcheck.*0' /etc/yum.repos.d/*.repo` -ge 1 ]; then echo FAIL; elif [ `grep -c -E '^\s*g` -> !/FAIL/ | Safe: AL2023 repositories are signed. The role README warns that third-party repos without gpgcheck break; none are used. |
| 1.2.3 | package manager repositories are configured (manual) | `amzn2023cis_rule_1_2_3`; command | `yum repolist` | Safe: Audit only (manual review of keys and repositories). |
| 1.3.1 | AIDE is installed | `amzn2023cis_rule_1_3_1`; package,package_facts,command,wait_for,copy | `rpm -q aide` -> installed | Override: AIDE scans every file. Exclude /var/lib/containerd, /var/lib/kubelet, /var/lib/cni and /var/log from aide.conf and do not build the database inside the SSM run synchronously (large I/O on a 4 vCPU node). Variable names for exclusions: UNVERIFIED in the role; may need a template override. |
| 1.3.2 | filesystem integrity is regularly checked | `amzn2023cis_rule_1_3_2`; cron | `grep -rs aide /etc/cron.* /etc/crontab /var/spool/cron/*` -> !/^#/ | Override: AIDE scans every file. Exclude /var/lib/containerd, /var/lib/kubelet, /var/lib/cni and /var/log from aide.conf and do not build the database inside the SSM run synchronously (large I/O on a 4 vCPU node). Variable names for exclusions: UNVERIFIED in the role; may need a template override. |
| 1.3.3 | cryptographic mechanisms are used to protect the integrity of audit tools | `amzn2023cis_rule_1_3_3`; blockinfile | `grep /sbin/au /etc/aide.conf` -> /sbin/auditctl p+i+n+u+g+s+b+acl+xattrs+sha512 ; /sbin/audit | Override: AIDE scans every file. Exclude /var/lib/containerd, /var/lib/kubelet, /var/lib/cni and /var/log from aide.conf and do not build the database inside the SSM run synchronously (large I/O on a 4 vCPU node). Variable names for exclusions: UNVERIFIED in the role; may need a template override. |
| 1.4.1 | permissions on bootloader config are configured | `amzn2023cis_rule_1_4_1`; file,lineinfile | `stat -c '%a %U %G' /boot/grub2/grub.cfg` -> `0600 root root`; `grep /boot/efi /etc/fstab` -> /.*\s/boot/efi vfat defaults,noatime,uid=0,gid=0,umask=0027, | Needs input: Set `amzn2023cis_legacy_boot` to match the instance (UEFI vs BIOS). Path `/boot/grub2/grub.cfg` vs the EFI path: UNVERIFIED on the EKS AMI. |
| 1.5.1 | address space layout randomization (ASLR) is enabled | `amzn2023cis_rule_1_5_1`; set_fact flag, then template `/etc/sysctl.d/60-*.conf` in tasks/post.yml + handler `Reload sysctl` | `sysctl -n kernel.randomize_va_space` -> `2`; `grep "kernel\.randomize_va_space" /etc/sysctl.conf /etc/sysctl.d/* \| cut -d ":" -f2` -> /^kernel.randomize_va_space(\| )=(\| )2/ | Safe |
| 1.5.2 | ptrace_scope is restricted | `amzn2023cis_rule_1_5_2`; set_fact flag, then template `/etc/sysctl.d/60-*.conf` in tasks/post.yml + handler `Reload sysctl` | `sysctl -n kernel.yama.ptrace_scope` -> `1`; `grep "kernel\.yama\.ptrace_scope" /etc/sysctl.conf /etc/sysctl.d/* \| cut -d ":" -f2` -> /^kernel.yama.ptrace_scope(\s\|)=(\s\|)1/ | Safe: ptrace_scope=1 still lets processes with CAP_SYS_PTRACE (for example a debug container) trace. Affects ad hoc host debugging only. |
| 1.5.3 | core dump storage is disabled | `amzn2023cis_rule_1_5_3`; lineinfile | `/etc/systemd/coredump.conf` matches `/^Storage=none/` | Safe |
| 1.5.4 | core dump backtraces are disabled | `amzn2023cis_rule_1_5_4`; lineinfile | `/etc/systemd/coredump.conf` matches `/^ProcessSizeMax=0/` | Safe |
| 1.6.1.1 | SELinux is installed | `amzn2023cis_rule_1_6_1_1`; package | `rpm -q libselinux` -> installed | Safe |
| 1.6.1.2 | SELinux is not disabled in bootloader configuration | `amzn2023cis_rule_1_6_1_2`; replace | `/boot/grub2/grub.cfg` matches , not `/selinux=0/` | Safe |
| 1.6.1.3 | SELinux policy is configured | `amzn2023cis_rule_1_6_1_3`; posix.selinux | `/etc/selinux/config` matches `/SELINUXTYPE=(targeted\|mls)/` | Override: The role sets the SELinux state to `amzn2023cis_selinux_enforce`, which defaults to `enforcing`. Set it to `permissive`. Enforcing breaks node bootstrap on the EKS AMI (container-selinux missing; awslabs/amazon-eks-ami issue 2006). Permissive satisfies this control. |
| 1.6.1.4 | SELinux mode is not disabled | `amzn2023cis_rule_1_6_1_4`; posix.selinux | `/etc/selinux/config` matches `/SELINUX=(enforcing\|permissive)/`; `getenforce` -> /SELINUX=(Enforcing\|Permissive)/ | Override: The role sets the SELinux state to `amzn2023cis_selinux_enforce`, which defaults to `enforcing`. Set it to `permissive`. Enforcing breaks node bootstrap on the EKS AMI (container-selinux missing; awslabs/amazon-eks-ami issue 2006). Permissive satisfies this control. |
| 1.6.1.6 | no unconfined services exist | `amzn2023cis_rule_1_6_1_6`; shell | `ps -eZ \| grep unconfined_service_t` -> !/./ | Safe |
| 1.6.1.7 | SETroubleshoot is not installed | `amzn2023cis_rule_1_6_1_7`; package | `rpm -q setroubleshoot` -> not installed | Safe |
| 1.6.1.8 | MCS Translation Service (mcstrans) is not installed | `amzn2023cis_rule_1_6_1_8`; package | `rpm -q mcstrans` -> not installed | Safe |
| 1.7.1 | message of the day is configured properly | `amzn2023cis_rule_1_7_1`; template | `stat -c '%a %U %G' /etc/motd` -> `0644 root root` | Override: Banner text comes from `amzn2023cis_warning_banner`; the role default is generic, not government text. Replace it with the custom text in section 6.4. The upstream audit rejects the words CentOS, RHEL, Red Hat, x86_64, Linux and Amazon in these files. |
| 1.7.2 | local login warning banner is configured properly | `amzn2023cis_rule_1_7_2`; file,template | `/etc/issue` matches , not `[Cc]ent[Oo][Ss]` | Override: Banner text comes from `amzn2023cis_warning_banner`; the role default is generic, not government text. Replace it with the custom text in section 6.4. The upstream audit rejects the words CentOS, RHEL, Red Hat, x86_64, Linux and Amazon in these files. |
| 1.7.3 | remote login warning banner is configured properly | `amzn2023cis_rule_1_7_3`; file,template | `/etc/issue.net` matches , not `[Cc]ent[Oo][Ss]` | Override: Banner text comes from `amzn2023cis_warning_banner`; the role default is generic, not government text. Replace it with the custom text in section 6.4. The upstream audit rejects the words CentOS, RHEL, Red Hat, x86_64, Linux and Amazon in these files. |
| 1.7.4 | permissions on /etc/motd are configured | `amzn2023cis_rule_1_7_4`; file | `stat -c '%a %U %G' /etc/motd` -> `0644 root root` | Safe |
| 1.7.5 | permissions on /etc/issue are configured | `amzn2023cis_rule_1_7_5`; file | `stat -c '%a %U %G' /etc/issue` -> `0644 root root` | Safe |
| 1.7.6 | permissions on /etc/issue.net are configured | `amzn2023cis_rule_1_7_6`; file | `stat -c '%a %U %G' /etc/issue.net` -> `0644 root root` | Safe |
| 1.8 | updates, patches, and additional security software are installed (manual) | `amzn2023cis_rule_1_8`; package | `yum check-update` -> !/[0-9].* packages available/ ; !/upgrade/ | SKIP: Owner decision 2026-10-24: approved. The task runs `package: name='*' state=latest`. That updates the node in place, which fights the AMI's version locks (kernel, containerd, amazon-ec2-net-utils) and the rule that nodes change by replacement. Skip and patch by rolling the node group to a new AMI release. |
| 1.9 | system-wide crypto policy is not legacy | `amzn2023cis_rule_1_9`; shell | `/etc/crypto-policies/config` matches `/^(\|\s+)(DEFAULT\|FUTURE\|FIPS)/`, not `/^(\|\s+)LEGACY/` | Safe: DEFAULT policy. The EKS AMI only enables FIPS when built with `enable_fips`. |

### 5.2 Section 2: Services

| ID | Requirement | Ansible implementation | Verify (command -> expected) | EKS |
|---|---|---|---|---|
| 2.1.1 | time synchronization is in use | `amzn2023cis_rule_2_1_1`; package | `rpm -q chrony` -> installed | Safe: chrony is installed on the EKS AMI. |
| 2.1.2 | chrony is configured | `amzn2023cis_rule_2_1_2`; template,lineinfile | `grep -Pr -- '^\h*(server\|pool)\h+[^#\n\r]+' /etc/chrony.conf /etc/chrony.d/*.conf` -> /^(server\|pool)\s{{ . }}/ | Override: Set `amzn2023cis_time_synchronization_servers` to `169.254.169.123` (Amazon Time Sync). The default list is public pools. The role template is Debian-derived and writes /etc/chrony.d/chrony.conf: test that chronyd starts and `chronyc sources` shows a reachable source (UNVERIFIED). |
| 2.2.2 | avahi is not installed | `amzn2023cis_rule_2_2_2`; package | `rpm -q avahi avahi-autoipd` -> not installed (role package names; corrected from upstream audit) | Safe |
| 2.2.3 | a print server is not installed | `amzn2023cis_rule_2_2_3`; package | `rpm -q cups` -> not installed | Safe |
| 2.2.4 | a dhcp server is not installed | `amzn2023cis_rule_2_2_4`; package | `rpm -q dhcp-server` -> not installed (upstream audit checks `dhcp`; corrected) | Safe |
| 2.2.5 | a dns server is not installed | `amzn2023cis_rule_2_2_5`; package | `rpm -q bind` -> not installed | Safe |
| 2.2.6 | an ftp server is not installed | `amzn2023cis_rule_2_2_6`; package | `rpm -q vsftpd` -> not installed (upstream audit file has a template glitch; corrected) | Safe |
| 2.2.7 | a tftp server is not installed | `amzn2023cis_rule_2_2_7`; package | `rpm -q tftp-server` -> not installed | Safe |
| 2.2.8 | a web server is not installed | `amzn2023cis_rule_2_2_8`; package | `rpm -q httpd` -> not installed; `rpm -q nginx` -> not installed | Safe |
| 2.2.9 | IMAP and POP3 server is not installed | `amzn2023cis_rule_2_2_9`; package | `rpm -q dovecot cyrus-imapd` -> not installed (upstream audit has the typo `cyrus-impad`; corrected) | Safe |
| 2.2.10 | Samba is not installed | `amzn2023cis_rule_2_2_10`; package | `rpm -q samba` -> not installed | Safe |
| 2.2.11 | HTTP Proxy Server is not installed | `amzn2023cis_rule_2_2_11`; package | `rpm -q squid` -> not installed | Safe |
| 2.2.12 | net-snmp is not installed or the snmpd service is not enabled | `amzn2023cis_rule_2_2_12`; package,systemd | `rpm -q net-snmp` -> not installed | Safe |
| 2.2.13 | telnet-server is not installed | `amzn2023cis_rule_2_2_13`; package | `rpm -q telnet-server` -> not installed | Safe |
| 2.2.14 | dnsmasq is not installed | `amzn2023cis_rule_2_2_14`; package | `rpm -q dnsmasq` -> not installed | Safe |
| 2.2.15 | mail transfer agent is configured for local-only mode | `amzn2023cis_rule_2_2_15`; lineinfile | `ss -lntu \| grep -E ":25\s" \| grep -E -v "\s(127.0.0.1\|\[?::1\]?):25\s"` -> !/./ | Safe |
| 2.2.16 | nfs-utils is not installed or the nfs-server service is masked | `amzn2023cis_rule_2_2_16`; package,systemd | `systemctl is-enabled nfs-server` -> `masked`, OR `rpm -q nfs-utils` -> not installed (upstream audit demands the package absent; corrected, see section 8.5) | Override: The EKS AMI installs nfs-utils (client tools for EFS and NFS volumes). Keep the default `amzn2023cis_nfs_server: mask`, never `remove`. The upstream audit still expects the package absent, so the tool must accept 'nfs-server masked' as pass (see section 8). |
| 2.2.17 | rpcbind is not installed or the rpcbind services are masked | `amzn2023cis_rule_2_2_17`; package,systemd | `systemctl is-enabled rpcbind.socket rpcbind.service` -> `masked`, OR `rpm -q rpcbind` -> not installed | Safe: Keep `mask`, not `remove`. rpcbind is only needed for NFSv3 mounts; EFS uses NFSv4. |
| 2.2.18 | rsync-daemon is not installed or the rsyncd service is masked | `amzn2023cis_rule_2_2_18`; package,systemd | `systemctl is-enabled rsyncd` -> `masked`, OR `rpm -q rsync-daemon` -> not installed (upstream audit checks `rsync`; corrected) | Safe: Keep `mask`, not `remove`. rpcbind is only needed for NFSv3 mounts; EFS uses NFSv4. |
| 2.3.1 | telnet client is not installed | `amzn2023cis_rule_2_3_1`; package | `rpm -q telnet` -> not installed | Safe |
| 2.3.2 | LDAP client is not installed | `amzn2023cis_rule_2_3_2`; package | `rpm -q openldap-clients` -> not installed | Safe |
| 2.3.3 | FTP client is not installed | `amzn2023cis_rule_2_3_3`; package | `rpm -q ftp` -> not installed (upstream audit checks `ypbind`; corrected) | Safe |
| 2.4 | nonessential services listening on the system are removed or masked (manual) | `amzn2023cis_rule_2_4`; command | `echo "Warning - Ensure all services are required - remove or mask if not"` -> !/Warning/ | Safe: Manual review of listening services. Expected listeners on a node are kubelet, containerd, kube-proxy, the CNI, sshd (if EC2 Instance Connect is kept) and the SSM agent. |

### 5.3 Section 3: Network

| ID | Requirement | Ansible implementation | Verify (command -> expected) | EKS |
|---|---|---|---|---|
| 3.1.1 | IPv6 status is identified (manual) | `amzn2023cis_rule_3_1_1`; set_fact flag, then template `/etc/sysctl.d/60-*.conf` in tasks/post.yml + handler `Reload sysctl` | `grep "^\s*linux" /boot/grub2/grub.cfg \| grep -v ipv6.disable=1` -> !/./; `sysctl -n net.ipv6.conf.all.disable_ipv6` -> `1` | Safe: Keep `amzn2023cis_ipv6_required: true`. Disabling IPv6 would break IPv6 or dual-stack clusters and costs nothing to avoid. |
| 3.2.1 | IP forwarding is disabled | `amzn2023cis_rule_3_2_1`; set_fact flag, then template `/etc/sysctl.d/60-*.conf` in tasks/post.yml + handler `Reload sysctl` | `sysctl -n net.ipv4.ip_forward` -> `0`; `sysctl -n net.ipv6.conf.all.forwarding` -> `0` | SKIP: BLOCKING. Sets net.ipv4.ip_forward=0. The EKS AMI sets it to 1 in /etc/sysctl.d/99-kubernetes-cri.conf and pod networking and kube-proxy need it. Set `amzn2023cis_rule_3_2_1: false` (or `amzn2023cis_is_router: true`, which also skips 3.2.2). |
| 3.2.2 | packet redirect sending is disabled | `amzn2023cis_rule_3_2_2`; set_fact flag, then template `/etc/sysctl.d/60-*.conf` in tasks/post.yml + handler `Reload sysctl` | `sysctl -n net.ipv4.conf.all.send_redirects` -> `0`; `sysctl -n net.ipv4.conf.default.send_redirects` -> `0` | Safe |
| 3.3.1 | source routed packets are not accepted | `amzn2023cis_rule_3_3_1`; set_fact flag, then template `/etc/sysctl.d/60-*.conf` in tasks/post.yml + handler `Reload sysctl` | `sysctl -n net.ipv4.conf.all.accept_source_route` -> `0`; `sysctl -n net.ipv4.conf.default.accept_source_route` -> `0` | Safe |
| 3.3.2 | ICMP redirects are not accepted | `amzn2023cis_rule_3_3_2`; set_fact flag, then template `/etc/sysctl.d/60-*.conf` in tasks/post.yml + handler `Reload sysctl` | `sysctl -n net.ipv4.conf.all.accept_redirects` -> `0`; `sysctl -n net.ipv4.conf.default.accept_redirects` -> `0` | Safe |
| 3.3.3 | secure ICMP redirects are not accepted | `amzn2023cis_rule_3_3_3`; set_fact flag, then template `/etc/sysctl.d/60-*.conf` in tasks/post.yml + handler `Reload sysctl` | `sysctl -n net.ipv4.conf.all.secure_redirects` -> `0`; `sysctl -n net.ipv4.conf.default.secure_redirects` -> `0` | Safe |
| 3.3.4 | suspicious packets are logged | `amzn2023cis_rule_3_3_4`; set_fact flag, then template `/etc/sysctl.d/60-*.conf` in tasks/post.yml + handler `Reload sysctl` | `sysctl -n net.ipv4.conf.all.log_martians` -> `1`; `sysctl -n net.ipv4.conf.default.log_martians` -> `1` | Safe |
| 3.3.5 | broadcast ICMP requests are ignored | `amzn2023cis_rule_3_3_5`; set_fact flag, then template `/etc/sysctl.d/60-*.conf` in tasks/post.yml + handler `Reload sysctl` | `sysctl -n net.ipv4.icmp_echo_ignore_broadcasts` -> `1` | Safe |
| 3.3.6 | bogus ICMP responses are ignored | `amzn2023cis_rule_3_3_6`; set_fact flag, then template `/etc/sysctl.d/60-*.conf` in tasks/post.yml + handler `Reload sysctl` | `sysctl -n net.ipv4.icmp_ignore_bogus_error_responses` -> `1` | Safe |
| 3.3.7 | Reverse Path Filtering is enabled | `amzn2023cis_rule_3_3_7`; set_fact flag, then template `/etc/sysctl.d/60-*.conf` in tasks/post.yml + handler `Reload sysctl` | `sysctl -n net.ipv4.conf.all.rp_filter` -> `1`; `sysctl -n net.ipv4.conf.default.rp_filter` -> `1` | Safe, test: rp_filter=1 (strict). The VPC CNI init sets the primary interface to loose (2) itself (aws/amazon-vpc-cni-k8s, cmd/aws-vpc-cni-init), and the kernel uses the larger of the all and interface values. Secondary ENIs used for pod traffic inherit `default`. Run a cross-node pod-to-pod and pod-to-service test after applying; UNVERIFIED. |
| 3.3.8 | TCP SYN Cookies is enabled | `amzn2023cis_rule_3_3_8`; set_fact flag, then template `/etc/sysctl.d/60-*.conf` in tasks/post.yml + handler `Reload sysctl` | `sysctl -n net.ipv4.tcp_syncookies` -> `1` | Safe |
| 3.3.9 | IPv6 router advertisements are not accepted | `amzn2023cis_rule_3_3_9`; set_fact flag, then template `/etc/sysctl.d/60-*.conf` in tasks/post.yml + handler `Reload sysctl` | `sysctl -n net.ipv6.conf.all.accept_ra` -> `0`; `sysctl -n net.ipv6.conf.default.accept_ra` -> `0` | Safe: IPv6 router advertisements off. Re-check if the cluster is ever moved to IPv6. |

### 5.4 Section 4: Access, authentication and authorization

| ID | Requirement | Ansible implementation | Verify (command -> expected) | EKS |
|---|---|---|---|---|
| 4.1.1 | cron daemon is installed and enabled | `amzn2023cis_rule_4_1_1`; service | `rpm -q cronie` -> installed; `systemctl is-enabled crond` / `is-active crond` -> enabled=true, running=true | Safe: Installs cronie. AL2023 uses systemd timers by default, so this adds a package and a service the node does not need; it is required by the benchmark. |
| 4.1.2 | permissions on /etc/crontab are configured | `amzn2023cis_rule_4_1_2`; file | `stat -c '%a %U %G' /etc/crontab` -> `0600 root root` | Safe |
| 4.1.3 | permissions on /etc/cron.hourly are configured | `amzn2023cis_rule_4_1_3`; file | `stat -c '%a %U %G' /etc/cron.hourly` -> `0700 root root` | Safe |
| 4.1.4 | permissions on /etc/cron.daily are configured | `amzn2023cis_rule_4_1_4`; file | `stat -c '%a %U %G' /etc/cron.daily` -> `0700 root root` | Safe |
| 4.1.5 | permissions on /etc/cron.weekly are configured | `amzn2023cis_rule_4_1_5`; file | `stat -c '%a %U %G' /etc/cron.weekly` -> `0700 root root` | Safe |
| 4.1.6 | permissions on /etc/cron.monthly are configured | `amzn2023cis_rule_4_1_6`; file | `stat -c '%a %U %G' /etc/cron.monthly` -> `0700 root root` | Safe |
| 4.1.7 | permissions on /etc/cron.d are configured | `amzn2023cis_rule_4_1_7`; file | `stat -c '%a %U %G' /etc/cron.d` -> `0700 root root` | Safe |
| 4.1.8 | cron is restricted to authorized users | `amzn2023cis_rule_4_1_8`; file | `test -e /etc/cron.deny` -> absent; `stat -c '%a %U %G' /etc/cron.allow` -> `0600 root root` | Safe |
| 4.1.9 | at is restricted to authorized users | `amzn2023cis_rule_4_1_9`; file | `test -e /etc/at.deny` -> absent; `stat -c '%a %U %G' /etc/at.allow` -> `0600 root root` | Safe |
| 4.2.1 | permissions on /etc/ssh/sshd_config are configured | `amzn2023cis_rule_4_2_1`; file | `stat -c '%a %U %G' /etc/ssh/sshd_config` -> `0600 root root` | Safe: sshd stays for key-based access (section 7.5); SSM Session Manager does not use it. 4.2.12 and 4.2.13 are L2 in the role's tags (not selected). |
| 4.2.2 | permissions on SSH private host key files are configured | `amzn2023cis_rule_4_2_2`; file | `userkeys=$(sudo find /etc/ssh/ -name *_key -type f ! -user root ); echo $userkeys` -> !/./; `groupkeys=$(sudo find /etc/ssh/ -name *_key -type f ! -group root ); echo $groupkeys` -> !/./ | Safe: sshd stays for key-based access (section 7.5); SSM Session Manager does not use it. 4.2.12 and 4.2.13 are L2 in the role's tags (not selected). |
| 4.2.3 | permissions on SSH public host key files are configured | `amzn2023cis_rule_4_2_3`; file | `userkeys=$(sudo find /etc/ssh/ -name *_key.pub -type f ! -user root ); echo $userkeys` -> !/./; `groupkeys=$(sudo find /etc/ssh/ -name *_key.pub -type f ! -group root ); echo $groupkeys` -> !/./ | Safe: sshd stays for key-based access (section 7.5); SSM Session Manager does not use it. 4.2.12 and 4.2.13 are L2 in the role's tags (not selected). |
| 4.2.4 | SSH access is limited | `amzn2023cis_rule_4_2_4`; lineinfile | `grep -Ei "^(Allow\|Deny)(Users\|Groups)" /etc/ssh/sshd_config /etc/ssh/sshd_config.d/*.conf` -> /(?i)AllowUsers: {{ .Vars.amzn2023cis_sshd_allowusers }}/ ;  | Override: Set `amzn2023cis_sshd_allowusers: ec2-user` (the default is derived from the connecting user, root under SSM, and would lock out `ec2-user`). Applies to the downloaded-key SSH access in section 7.5 (decided 2026-10-24). |
| 4.2.5 | SSH LogLevel is appropriate | `amzn2023cis_rule_4_2_5`; lineinfile | `/etc/ssh/sshd_config` matches `/LogLevel (VERBOSE\|INFO)/`, not `/^LogLevel DEBUG/` | Safe: sshd stays for key-based access (section 7.5); SSM Session Manager does not use it. 4.2.12 and 4.2.13 are L2 in the role's tags (not selected). |
| 4.2.6 | SSH PAM is enabled | `amzn2023cis_rule_4_2_6`; lineinfile | `/etc/ssh/sshd_config` matches `/^UsePAM yes/`, not `/^UsePAM no/` | Safe: sshd stays for key-based access (section 7.5); SSM Session Manager does not use it. 4.2.12 and 4.2.13 are L2 in the role's tags (not selected). |
| 4.2.7 | SSH root login is disabled | `amzn2023cis_rule_4_2_7`; lineinfile | `/etc/ssh/sshd_config` matches `/^PermitRootLogin no/`, not `/^PermitRootLogin yes/` | Safe: sshd stays for key-based access (section 7.5); SSM Session Manager does not use it. 4.2.12 and 4.2.13 are L2 in the role's tags (not selected). |
| 4.2.8 | SSH HostbasedAuthentication is disabled | `amzn2023cis_rule_4_2_8`; lineinfile | `/etc/ssh/sshd_config` matches `HostbasedAuthentication no`, not `/^HostbasedAuthentication yes/` | Safe: sshd stays for key-based access (section 7.5); SSM Session Manager does not use it. 4.2.12 and 4.2.13 are L2 in the role's tags (not selected). |
| 4.2.9 | SSH PermitEmptyPasswords is disabled | `amzn2023cis_rule_4_2_9`; lineinfile | `/etc/ssh/sshd_config` matches `/^PermitEmptyPasswords no/`, not `/^PermitEmptyPasswords yes/` | Safe: sshd stays for key-based access (section 7.5); SSM Session Manager does not use it. 4.2.12 and 4.2.13 are L2 in the role's tags (not selected). |
| 4.2.10 | SSH PermitUserEnvironment is disabled | `amzn2023cis_rule_4_2_10`; lineinfile | `/etc/ssh/sshd_config` matches `/^PermitUserEnvironment no/`, not `/^PermitUserEnvironment yes/` | Safe: sshd stays for key-based access (section 7.5); SSM Session Manager does not use it. 4.2.12 and 4.2.13 are L2 in the role's tags (not selected). |
| 4.2.11 | SSH IgnoreRhosts is enabled | `amzn2023cis_rule_4_2_11`; lineinfile | `/etc/ssh/sshd_config` matches `/IgnoreRhosts yes/`, not `/^IgnoreRhosts no/` | Safe: sshd stays for key-based access (section 7.5); SSM Session Manager does not use it. 4.2.12 and 4.2.13 are L2 in the role's tags (not selected). |
| 4.2.14 | system-wide crypto policy is not over-ridden | `amzn2023cis_rule_4_2_14`; shell,replace | `grep -Ei "^\s*Crypto_policy" /etc/sysconfig/sshd /etc/ssh/sshd_config /etc/ssh/sshd_config.d/*` -> !/^CRYPTO_POLICY/ | Safe: The EKS AMI build appends a Ciphers line to sshd_config (install-worker.sh); that is not the CRYPTO_POLICY override this control checks. UNVERIFIED that the role leaves that line alone. |
| 4.2.15 | SSH warning banner is configured | `amzn2023cis_rule_4_2_15`; lineinfile | `/etc/ssh/sshd_config` matches `/^Banner /etc/issue.net/`, not `/^Banner none/` | Safe: sshd stays for key-based access (section 7.5); SSM Session Manager does not use it. 4.2.12 and 4.2.13 are L2 in the role's tags (not selected). |
| 4.2.16 | SSH MaxAuthTries is set to 4 or less | `amzn2023cis_rule_4_2_16`; lineinfile | `/etc/ssh/sshd_config` matches `/MaxAuthTries [1-4]/`, not `/MaxAuthTries (0\|[5-9])/` | Safe: sshd stays for key-based access (section 7.5); SSM Session Manager does not use it. 4.2.12 and 4.2.13 are L2 in the role's tags (not selected). |
| 4.2.17 | SSH MaxStartups is configured | `amzn2023cis_rule_4_2_17`; lineinfile | `/etc/ssh/sshd_config` matches `/^MaxStartups 10:30:60/` | Safe: sshd stays for key-based access (section 7.5); SSM Session Manager does not use it. 4.2.12 and 4.2.13 are L2 in the role's tags (not selected). |
| 4.2.18 | SSH MaxSessions is set to 10 or less | `amzn2023cis_rule_4_2_18`; lineinfile | `/etc/ssh/sshd_config` matches `/^MaxSessions ([2-9]\|10)/`, not `/^MaxSessions (1\|[1-9]{2,5})/` | Safe: sshd stays for key-based access (section 7.5); SSM Session Manager does not use it. 4.2.12 and 4.2.13 are L2 in the role's tags (not selected). |
| 4.2.19 | SSH LoginGraceTime is set to one minute or less | `amzn2023cis_rule_4_2_19`; lineinfile | `/etc/ssh/sshd_config` matches `/^LoginGraceTime ([1-9]\|[1-5][0-9])/` | Safe: sshd stays for key-based access (section 7.5); SSM Session Manager does not use it. 4.2.12 and 4.2.13 are L2 in the role's tags (not selected). |
| 4.2.20 | SSH Idle Timeout Interval is configured | `amzn2023cis_rule_4_2_20`; lineinfile | `/etc/ssh/sshd_config` matches `/^ClientAliveInterval {{ .Vars.amzn2023cis_sshd_clientaliveinterval }}/` and `/^ClientAliveCountMax {{ .Vars.amzn2023cis_sshd_clientalivecountmax }}/` | Safe: sshd stays for key-based access (section 7.5); SSM Session Manager does not use it. 4.2.12 and 4.2.13 are L2 in the role's tags (not selected). |
| 4.3.1 | sudo is installed | `amzn2023cis_rule_4_3_1`; package | `rpm -q sudo` -> installed | Safe |
| 4.3.2 | sudo commands use pty | `amzn2023cis_rule_4_3_2`; lineinfile | `/etc/sudoers` matches `/^Defaults.*\suse_pty/`; `ls -1 /etc/sudoers.d/* > /dev/null 2>&1; if [ '$?' = '0' ]; then grep -c '^Defaults.*\suse_pty' /etc/sudoers.d` -> /[0-99]/ | Safe |
| 4.3.3 | sudo log file exists | `amzn2023cis_rule_4_3_3`; lineinfile | `/etc/sudoers` matches `/^Defaults\s+logfile="/var/log/sudo.log"/`; `grep -c '^Defaults logfile=/var/log/*.log' /etc/sudoers.d/.*/` -> /[1:99]/ | Safe |
| 4.3.4 | re-authentication for privilege escalation is not disabled globally | `amzn2023cis_rule_4_3_4`; replace | `/etc/sudoers` matches , not `/!authenticate/`; `grep authenticate /etc/sudoers.d/*` -> !/!authenticate/ | Safe: Removes `!authenticate` only. The SSM agent's sudoers.d file uses NOPASSWD, which this does not touch (UNVERIFIED). |
| 4.3.5 | sudo authentication timeout is configured correctly | `amzn2023cis_rule_4_3_5`; shell,lineinfile,replace | `grep -rP "timestamp_timeout=\K[0-9]*" /etc/sudoers*` -> !/timestamp_timeout=(-1\|1[6-9]\|[2-9][0-9]\|[1-9][0-9]{2,})/ ; | Safe |
| 4.3.6 | access to the su command is restricted | `amzn2023cis_rule_4_3_6`; group,lineinfile | `/etc/pam.d/su` matches `/^auth.*required.*pam_wheel.so\suse_uid\sgroup={{ .Vars.amzn2023cis_sugroup }}/`; `grep {{ .Vars.amzn2023cis_sugroup }} /etc/group` -> /^{{ .Vars.amzn2023cis_sugroup }}:x:\d+:$/ ; !/^{{ .Vars.amz | Safe: Uses an empty group (`nosugroup`), so nobody gains su access. |
| 4.4.1 | custom authselect profile is used (manual) | `amzn2023cis_rule_4_4_1`; command | `head -1 /etc/authselect/authselect.conf \| grep 'custom/'` -> /^custom\/.*/ | SKIP: Owner decision 2026-10-24: PAM is left unchanged. Set `amzn2023cis_rule_4_4_1: false`. Reason: nodes have no password logins (keys and SSM only), and an authselect change is the one change that can lock out every interactive login. Note 4.5.1 to 4.5.4 also edit PAM files (section 7.7, question 1). |
| 4.4.2 | authselect includes with-faillock | `amzn2023cis_rule_4_4_2`; shell,command,lineinfile | `/etc/pam.d/password-auth` matches `/auth\s+required\s+pam_faillock.so/` and `/auth\s+required\s+pam_faillock.so preauth silent/`; `/etc/pam.d/system-auth` matches `/auth\s+required\s+pam_faillock.so/` and `/auth\s+required\s+pam_faillock.so preauth silent/` | SKIP: Owner decision 2026-10-24: PAM is left unchanged. Set `amzn2023cis_rule_4_4_2: false`. Reason: nodes have no password logins (keys and SSM only), and an authselect change is the one change that can lock out every interactive login. Note 4.5.1 to 4.5.4 also edit PAM files (section 7.7, question 1). |
| 4.5.1 | password creation requirements are configured | `amzn2023cis_rule_4_5_1`; lineinfile | `/etc/security/pwquality.conf` matches `{{ .Vars.amzn2023cis_pam_password_minlen }}` and `{{ .Vars.amzn2023cis_pam_password_minclass }}`; `/etc/pam.d/password-auth` matches `/password\s+requisite\s+pam_pwquality.so try_first_pass local_users_only enforce_for_root retry=[3-5]/` | Safe: PAM changes affect interactive password logins; the node uses key and SSM access. |
| 4.5.2 | lockout for failed password attempts is configured | `amzn2023cis_rule_4_5_2`; lineinfile | `/etc/pam.d/password-auth` matches `/^account\s*required\s*pam_faillock.so/`; `/etc/pam.d/system-auth` matches `/^account\s*required\s*pam_faillock.so/` | Safe: PAM changes affect interactive password logins; the node uses key and SSM access. |
| 4.5.3 | password reuse is limited | `amzn2023cis_rule_4_5_3`; lineinfile | `/etc/pam.d/system-auth` matches `/^password\s+(requisite\|sufficient)\s+pam_pwhistory.so.*remember=\b(?:[1-5]\d+\|5)\b/` | Safe: PAM changes affect interactive password logins; the node uses key and SSM access. |
| 4.5.4 | password hashing algorithm is SHA-512 | `amzn2023cis_rule_4_5_4`; replace | `/etc/pam.d/password-auth` matches `/^password\s+sufficient\s+pam_unix.so sha512/`, not `/^password\s+sufficient\s+pam_unix.so md5/`; `/etc/pam.d/system-auth` matches `/^password\s+sufficient\s+pam_unix.so sha512/`, not `/^password\s+sufficient\s+pam_unix.so md5/` | Safe: PAM changes affect interactive password logins; the node uses key and SSM access. |
| 4.6.1.1 | password expiration is 365 days or less | `amzn2023cis_rule_4_6_1_1`; lineinfile,shell,user | `/etc/login.defs` matches `/^PASS_MAX_DAYS\s+{{ .Vars.amzn2023cis_pass_max_days }}/`, not `/^PASS_MAX_DAYS\s+(36[6-9]\|[4-9][0-9][0-9]\|[1-9][0-9][0-9][0-9]\|[1-9][0-9][0-9][0-9][0-9])/` | Safe |
| 4.6.1.2 | minimum days between password changes is configured | `amzn2023cis_rule_4_6_1_2`; lineinfile,shell,user | `/etc/login.defs` matches `/^PASS_MIN_DAYS\s+{{ .Vars.amzn2023cis_pass_min_days }}/`, not `/^PASS_MIN_DAYS\s+0/` | Safe |
| 4.6.1.3 | password expiration warning days is 7 or more | `amzn2023cis_rule_4_6_1_3`; lineinfile,shell,command | `/etc/login.defs` matches `/^PASS_WARN_AGE\s*{{ .Vars.amzn2023cis_pass_warn_age }}/`, not `/^PASS_WARN_AGE\s*[1-6]/` | Safe |
| 4.6.1.4 | inactive password lock is 30 days or less | `amzn2023cis_rule_4_6_1_4`; shell,command | `useradd -D \| grep INACTIVE` -> /^INACTIVE=[1-30]/; `grep -E ^[^:]+:[^\!*] /etc/shadow \| grep -vw root \| cut -d : -f1,7` -> !/./ | Safe |
| 4.6.1.5 | all users last password change date is in the past | `amzn2023cis_rule_4_6_1_5`; command,shell | `for usr in `cut -d: -f1 /etc/shadow`; do [ "$(date --date="$(chage --list $usr \| grep "^Last password change" ` -> !Failed | Safe |
| 4.6.2 | system accounts are secured | `amzn2023cis_rule_4_6_2`; user | `/awk -F: '$3<1000' /etc/passwd \| egrep -v 'root\|sync\|halt\|shutdown\|/sbin/nologin\|/bin/false'/` -> !/./; `/awk -F: '$3<1000' /etc/passwd \| egrep -wv 'root' \| cut -d: -f1 \| xargs -I '{}' passwd -S '{}' \| egrep -cvw 'L` -> 0 | Safe |
| 4.6.3 | default user shell timeout is 900 seconds or less | `amzn2023cis_rule_4_6_3`; blockinfile | `/etc/profile.d/tmout.sh` matches `/TMOUT=([1-8][0-9]{0,2}\|900)/` and `/readonly TMOUT/` | Safe: TMOUT=900 ends idle Session Manager shells after 15 minutes. |
| 4.6.4 | default group for the root account is GID 0 | `amzn2023cis_rule_4_6_4`; user | user:root | Safe |
| 4.6.5 | default user umask is 027 or more restrictive | `amzn2023cis_rule_4_6_5`; lineinfile,shell,replace,copy | `/etc/profile` matches `/^(\s+\|)umask\s0(2\|7)(2\|7)/`, not `/^(\s+\|)umask\s[1-7][0-7][0-7]/` | Safe |
| 4.6.6 | root password is set | `amzn2023cis_rule_4_6_6`; audit/warn only | `passwd -S root \| grep "Password set"` -> /^root.*Password set.*crypt/ | SKIP: Owner decision 2026-10-24: approved. The role asserts that root has a password and fails the whole run if not. On EC2 the root password is normally locked (UNVERIFIED). Set `amzn2023cis_rule_4_6_6: false`; setting a root password is worse than the deviation. |

### 5.5 Section 5: Logging

| ID | Requirement | Ansible implementation | Verify (command -> expected) | EKS |
|---|---|---|---|---|
| 5.1.1.1 | rsyslog is installed | `amzn2023cis_rule_5_1_1_1`; package | `rpm -q rsyslog` -> installed; `systemctl is-enabled rsyslog` / `is-active rsyslog` -> enabled=true, running=true | N/A (alt.): Only runs when `amzn2023cis_syslog_service: rsyslog`. Choose `journald` (the AL2023 default), then these are not-applicable and 5.1.2.x applies. |
| 5.1.1.2 | rsyslog service is enabled (manual) | `amzn2023cis_rule_5_1_1_2`; systemd | UNVERIFIED (no upstream check found) | N/A (alt.): Only runs when `amzn2023cis_syslog_service: rsyslog`. Choose `journald` (the AL2023 default), then these are not-applicable and 5.1.2.x applies. |
| 5.1.1.3 | journald is configured to send logs to rsyslog | `amzn2023cis_rule_5_1_1_3`; lineinfile | `/etc/systemd/journald.conf` matches `/^ForwardToSyslog=yes/`, not `/^ForwardToSyslog=[Nn][Oo]/` | N/A (alt.): Only runs when `amzn2023cis_syslog_service: rsyslog`. Choose `journald` (the AL2023 default), then these are not-applicable and 5.1.2.x applies. |
| 5.1.1.4 | rsyslog default file permissions are configured | `amzn2023cis_rule_5_1_1_4`; lineinfile | `/etc/rsyslog.conf` matches `/\$FileCreateMode 0[0,6][0,4]0/` | N/A (alt.): Only runs when `amzn2023cis_syslog_service: rsyslog`. Choose `journald` (the AL2023 default), then these are not-applicable and 5.1.2.x applies. |
| 5.1.1.5 | logging is configured (manual) | `amzn2023cis_rule_5_1_1_5`; command,blockinfile | `/etc/rsyslog.conf` matches `/^mail.\*\s+-/var/log/mail/` and `/^auth,authpriv.\*\s+\/var\/log/secure/` | N/A (alt.): Only runs when `amzn2023cis_syslog_service: rsyslog`. Choose `journald` (the AL2023 default), then these are not-applicable and 5.1.2.x applies. |
| 5.1.1.6 | rsyslog is configured to send logs to a remote log host | `amzn2023cis_rule_5_1_1_6`; blockinfile | `grep "^*.*[^I][^I]*@" /etc/rsyslog.conf /etc/rsyslog.d/*.conf` -> /\*\.\* @@*./ | N/A (alt.): Only runs when `amzn2023cis_syslog_service: rsyslog`. Choose `journald` (the AL2023 default), then these are not-applicable and 5.1.2.x applies. |
| 5.1.1.7 | rsyslog is not configured to receive logs from a remote client | `amzn2023cis_rule_5_1_1_7`; replace | `grep '$ModLoad imtcp' /etc/rsyslog.conf /etc/rsyslog.d/*.conf` -> /.*[^#]?(\| )\$ModLoad imtcp/ ; /.*:#(\| )\$ModLoad imtcp/; `grep '$InputTCPServerRun' /etc/rsyslog.conf /etc/rsyslog.d/*.conf` -> /.*[^#]?(\| )\$InputTCPServerRun/ ; /.*:#(\| )\$InputTCPServer | N/A (alt.): Only runs when `amzn2023cis_syslog_service: rsyslog`. Choose `journald` (the AL2023 default), then these are not-applicable and 5.1.2.x applies. |
| 5.1.2.1.1 | systemd-journal-remote is installed (manual) | `amzn2023cis_rule_5_1_2_1_1`; package | `rpm -q systemd-journal-remote` -> installed | SKIP: Needs a remote journal host. The role default URL is a private placeholder address. No log host exists in this project, so skip and record the deviation. Revisit when central logging exists. |
| 5.1.2.1.2 | systemd-journal-remote is configured (manual) | `amzn2023cis_rule_5_1_2_1_2`; lineinfile | `/etc/systemd/journal-upload.conf` matches `/^URL=/` and `/ServerKeyFile=.*.pem` | SKIP: Needs a remote journal host. The role default URL is a private placeholder address. No log host exists in this project, so skip and record the deviation. Revisit when central logging exists. |
| 5.1.2.1.3 | systemd-journal-remote is enabled (manual) | `amzn2023cis_rule_5_1_2_1_3`; systemd | `systemctl is-enabled systemd-journal-upload.service` / `is-active systemd-journal-upload.service` -> enabled=true, running=true | SKIP: Needs a remote journal host. The role default URL is a private placeholder address. No log host exists in this project, so skip and record the deviation. Revisit when central logging exists. |
| 5.1.2.1.4 | journald is not configured to receive logs from a remote client | `amzn2023cis_rule_5_1_2_1_4`; systemd | `systemctl is-enabled systemd-journal-remote.socket` / `is-active systemd-journal-remote.socket` -> enabled=false, running=false; `systemctl is-enabled systemd-journal-remote.socket` -> /^masked/ | Safe |
| 5.1.2.2 | journald service is enabled | `amzn2023cis_rule_5_1_2_2`; systemd,command | `systemctl is-enabled systemd-journald` / `is-active systemd-journald` -> enabled=true, running=true | Safe |
| 5.1.2.3 | journald is configured to compress large log files | `amzn2023cis_rule_5_1_2_3`; lineinfile | `grep -i compress /etc/systemd/journald.conf` -> /^Compress=yes/ ; !/^Compress=[Nn][Oo]/ | Safe |
| 5.1.2.4 | journald is configured to write logfiles to persistent disk | `amzn2023cis_rule_5_1_2_4`; lineinfile | `grep -i storage /etc/systemd/journald.conf` -> /^Storage=persistent/ | Safe |
| 5.1.2.5 | journald is not configured to send logs to rsyslog (manual) | `amzn2023cis_rule_5_1_2_5`; lineinfile | `grep -i forward /etc/systemd/journald.conf` -> !/^\s*ForwardToSyslog/ | Safe |
| 5.1.2.6 | journald log rotation is configured per site policy (manual) | `amzn2023cis_rule_5_1_2_6`; lineinfile | `grep -Ei "max\|keep" /etc/systemd/journald.conf` -> /^SystemMaxUse=/ ; /^SystemKeepFree=/ | Override: Role default `amzn2023cis_journald_systemkeepfree` is 100G, larger than a typical node root volume; journald would stop writing. Set sizes that fit the volume. |
| 5.1.2.7 | journald default file permissions configured (manual) | `amzn2023cis_rule_5_1_2_7`; lineinfile | `echo "Manual - Please check journald default permissions"` -> !/^Manual/ | Safe: Manual review. |
| 5.1.3 | all logfiles have appropriate permissions and ownership | `amzn2023cis_rule_5_1_3`; file | `find /var/log/ -type f -perm /g+wx,o+rwx -exec ls -l "{}" + \| grep -Ev "[b,u,w]tmp.*\|lastllog"` -> !/.*/ | Safe |
| 5.3 | logrotate is configured (manual) | `amzn2023cis_rule_5_3`; package,systemd,replace | `echo "Manually Check logrotate files are configured with site policy"` -> !/^Manually/ | Safe: Manual review. |

### 5.6 Section 6: System maintenance

| ID | Requirement | Ansible implementation | Verify (command -> expected) | EKS |
|---|---|---|---|---|
| 6.1.1 | permissions on /etc/passwd are configured | `amzn2023cis_rule_6_1_1`; file | `stat -L -c "%a %U %G" /etc/passwd` -> /6[0,4][0,4] root root/ | Safe: The benchmark lists 6.1.2 with the same title as 6.1.1 (the ComplianceAsCode control file does too); the real intent of 6.1.2 is UNVERIFIED. |
| 6.1.2 | permissions on /etc/passwd are configured | `amzn2023cis_rule_6_1_2`; file | `stat -L -c "%a %U %G" /etc/passwd` -> /6[0,4][0,4] root root/ | Safe: The benchmark lists 6.1.2 with the same title as 6.1.1 (the ComplianceAsCode control file does too); the real intent of 6.1.2 is UNVERIFIED. |
| 6.1.3 | permissions on /etc/passwd- are configured | `amzn2023cis_rule_6_1_3`; file | `stat -L -c "%a %U %G" /etc/passwd-` -> /6[0,4][0,4] root root/ | Safe: The benchmark lists 6.1.2 with the same title as 6.1.1 (the ComplianceAsCode control file does too); the real intent of 6.1.2 is UNVERIFIED. |
| 6.1.4 | permissions on /etc/group are configured | `amzn2023cis_rule_6_1_4`; file | `stat -L -c "%a %U %G" /etc/group` -> /6[0,4][0,4] root root/ | Safe: The benchmark lists 6.1.2 with the same title as 6.1.1 (the ComplianceAsCode control file does too); the real intent of 6.1.2 is UNVERIFIED. |
| 6.1.5 | permissions on /etc/group- are configured | `amzn2023cis_rule_6_1_5`; file | `stat -L -c "%a %U %G" /etc/group-` -> /6[0,4][0,4] root root/ | Safe: The benchmark lists 6.1.2 with the same title as 6.1.1 (the ComplianceAsCode control file does too); the real intent of 6.1.2 is UNVERIFIED. |
| 6.1.6 | permissions on /etc/shadow are configured | `amzn2023cis_rule_6_1_6`; file | `stat -L -c "%a %U %G" /etc/shadow` -> /0 root root/ | Safe: The benchmark lists 6.1.2 with the same title as 6.1.1 (the ComplianceAsCode control file does too); the real intent of 6.1.2 is UNVERIFIED. |
| 6.1.7 | permissions on /etc/shadow- are configured | `amzn2023cis_rule_6_1_7`; file | `stat -L -c "%a %U %G" /etc/shadow-` -> /0 root root/ | Safe: The benchmark lists 6.1.2 with the same title as 6.1.1 (the ComplianceAsCode control file does too); the real intent of 6.1.2 is UNVERIFIED. |
| 6.1.8 | permissions on /etc/gshadow are configured | `amzn2023cis_rule_6_1_8`; file | `stat -L -c "%a %U %G" /etc/gshadow` -> /0 root root/ | Safe: The benchmark lists 6.1.2 with the same title as 6.1.1 (the ComplianceAsCode control file does too); the real intent of 6.1.2 is UNVERIFIED. |
| 6.1.9 | permissions on /etc/gshadow- are configured | `amzn2023cis_rule_6_1_9`; file | `stat -L -c "%a %U %G" /etc/gshadow-` -> /0 root root/ | Safe: The benchmark lists 6.1.2 with the same title as 6.1.1 (the ComplianceAsCode control file does too); the real intent of 6.1.2 is UNVERIFIED. |
| 6.1.11 | world writable files and directories are secured | `amzn2023cis_rule_6_1_11`; shell,file | `df --local -P \| awk '{if (NR!=1) print $6}' \| xargs -I '{}' find '{}' -xdev -perm -0002` -> !/./ | Override: The role's file-fix step already excludes containerd and kubelet pod paths, but the sticky-bit step runs over every `df --local` mount. Set `amzn2023cis_no_world_write_adjust: false` and treat the sticky-bit step as a risk on overlay mounts (UNVERIFIED that df lists them here); consider skipping and recording it. |
| 6.1.12 | no unowned or ungrouped files or directories exist | `amzn2023cis_rule_6_1_12`; command | `df --local -P \| awk {'if (NR!=1) print $6'} \| xargs -I '{}' find '{}' -xdev -nouser -nogroup` -> !/./ | Safe: Audit only; a full-disk find, so allow time inside the SSM run. |
| 6.1.13 | SUID and SGID files are reviewed (manual) | `amzn2023cis_rule_6_1_13`; shell | `df --local -P \| awk '{if (NR!=1) print $6}' \| xargs -I '{}' find '{}' -xdev -type f -perm -4000` -> !/./; `df --local -P \| awk '{if (NR!=1) print $6}' \| xargs -I '{}' find '{}' -xdev -type f -perm -2000` -> !/./ | Safe: Audit only; a full-disk find, so allow time inside the SSM run. |
| 6.2.1 | accounts in /etc/passwd use shadowed passwords | `amzn2023cis_rule_6_2_1`; shell,command | `/etc/passwd` matches `/^.*:x:*/`, not `/^.*:[a-w][y-z]:.*` | Safe |
| 6.2.2 | /etc/shadow password fields are not empty | `amzn2023cis_rule_6_2_2`; command | `/etc/shadow` matches , not `/^.*:\!\!:.*/` | Safe |
| 6.2.3 | all groups in /etc/passwd exist in /etc/group | `amzn2023cis_rule_6_2_3`; shell | `pwck -r \| grep 'no group' \| awk '{ gsub("[:\47]",""); print $2}'` -> !/./ | Safe |
| 6.2.4 | no duplicate UIDs exist | `amzn2023cis_rule_6_2_4`; shell | `cat /etc/passwd \| cut -d: -f3 \| uniq -d` -> !/./ | Safe |
| 6.2.5 | no duplicate GIDs exist | `amzn2023cis_rule_6_2_5`; shell | `cat /etc/group \| cut -d: -f3 \| uniq -d` -> !/./ | Safe |
| 6.2.6 | no duplicate user names exist | `amzn2023cis_rule_6_2_6`; shell | `cat /etc/passwd \| cut -d: -f1 \| uniq -d` -> !/./ | Safe |
| 6.2.7 | no duplicate group names exist | `amzn2023cis_rule_6_2_7`; shell | `cat /etc/group \| cut -d: -f1 \| uniq -d` -> !/./ | Safe |
| 6.2.8 | root PATH Integrity | `amzn2023cis_rule_6_2_8`; shell,file | `/bin/bash --login -c 'env \| grep ^PATH=' \| sed -e 's/PATH=//' -e 's/::/:/' -e 's/:$//' -e 's/:/\n/g'` -> !/\\./ | Safe |
| 6.2.9 | root is the only UID 0 account | `amzn2023cis_rule_6_2_9`; command | `awk -F: '($3 == 0) { print $1 }' /etc/passwd` -> root | Safe |
| 6.2.10 | local interactive user home directories are configured | `amzn2023cis_rule_6_2_10`; file,posix.acl | `set -o pipefail; pwck -r \| grep -P "^user '(?P<user>.*)': directory '(?P<dir>.*)' does not exist$"` -> !/./; `getent passwd {1000..60000} \| awk -F: '{ print $1 " " $6 }' \| while read user dir; do if [ $user != 'ftpd' ]; ` -> !/./ | Safe |
| 6.2.11 | local interactive user dot files access is configured | `amzn2023cis_rule_6_2_11`; file | `find /home/ -name "\.*" -perm /g+w,o+w` -> !/./; `find /home/ -name .forward -name .netrc -name .rhosts` -> !/./ | Safe |

### 5.4 Level 2 Server controls (not selected)

The 64 Level 2 Server controls are out of scope for `docs/SPEC.md`. Appendix A lists their role variables so the wrapper can set them all to false. If the owner later asks for Level 2, these would conflict with the node:

| Control(s) | Why it conflicts |
|---|---|
| 1.6.1.5 (SELinux enforcing) | The EKS AMI does not install `container-selinux`; enabling enforcing mode made `nodeadm` fail and the node not join (awslabs/amazon-eks-ami issue 2006, still open as of the last comment in October 2025). S8. |
| 3.4.1.1, 3.4.1.2, 3.4.2.1 to 3.4.2.7 (firewalld or nftables, default deny) | kube-proxy and the VPC CNI program packet-filter rules on the host (the kubelet unit itself runs `iptables -P FORWARD ACCEPT`, S7). A second rule manager with default-deny would drop pod, health-check and load-balancer traffic. Use security groups for the node boundary. The specific rule clashes are UNVERIFIED. |
| 5.2.3.20 (audit config immutable) | After the first run the rules cannot be reloaded without a reboot. The role re-runs on a schedule, so later runs would fail to load changed rules until reboot. Consequence inferred from the control title. UNVERIFIED on a node. |
| 5.2.2.3 (disk-full action) | The role default `amzn2023cis_auditd_admin_space_left_action` is `halt`, which stops the only node if the disk fills. |
| 3.1.2 to 3.1.5 (DCCP, SCTP, RDS, TIPC off) | Would block SCTP-based workloads on the host. Podium does not use SCTP. |
| 4.2.12, 4.2.13 (SSH X11 and TCP forwarding off) | `AllowTcpForwarding no` blocks `ssh -L` over the key-based SSH path. SSM port forwarding is unaffected. |
| 1.1.1.1, 1.1.1.2 (squashfs, udf off) | Probably harmless; whether SOCI or any image path needs squashfs is UNVERIFIED. |
| 1.1.3.1 to 1.1.7.1 (separate partitions) | Needs extra EBS volumes in the launch template. Mount options such as nodev or noexec on `/var` can break container runtimes and CSI helpers. UNVERIFIED. |

## 6. Existing automation

### 6.1 What I looked at

| Project | Benchmark / profile | Licence | Last commit / release | EKS-optimized image support | Coverage |
|---|---|---|---|---|---|
| `ansible-lockdown/AMAZON2023-CIS` (S12) | CIS Amazon Linux 2023 v1.0.0. README: "Based on CIS AMAZON 2023 Benchmark v1.0.0 - 26-06-2023". `benchmark_version: 1.0.0` in defaults. | MIT | Latest release `1.5.0`, 2026-10-08 (commit `3a89ab3853926c11327591d58416c020c00a8717`). Previous releases 1.4.0 (2026-08-18), 1.3.0 (2026-05-13), 1.2.8 (2026-03-24). Latest commit on `devel` 2026-10-13. Repo pushed 2026-10-19. | Not mentioned anywhere I read. Meta says Amazon Linux 2023 only. It was written against a clean OS install (README). Needed the overrides in section 5.2 to be safe on EKS. | 243 rule variables and tasks, matching the 243 benchmark controls: 179 Level 1 Server, 62 tagged Level 2 Server plus 2 untagged. Some tasks only warn (manual controls). |
| `ansible-lockdown/AMAZON2023-CIS-Audit` (S13) | Same benchmark, goss checks, branch `benchmark_1.0.0` | MIT | Latest commit 2026-09-15; repo pushed 2026-10-01 | Not mentioned | 243 controls have checks (each carries its control number in `meta.CIS_ID`). Has defects (section 8.5). |
| ComplianceAsCode/content, release v0.1.82 (S14) | `cis` and `cis_server_l1` profiles for product `al2023`, "Benchmark for Amazon Linux 2023 v1.0.0, released 2023-06-26". Files shipped: `ssg-al2023-ds.xml`, `ansible/al2023-playbook-cis.yml`, `ansible/al2023-playbook-cis_server_l1.yml`, Bash scripts, HTML guides. | BSD-3-Clause (LICENSE file header; GitHub reports "NOASSERTION") | Release v0.1.82, 2026-09-01. Latest commit touching `controls/cis_al2023.yml`: 2026-10-19. | Not mentioned. An `eks` product also exists in the repo; I did not examine it and believe it targets the Kubernetes benchmark, not the node OS (UNVERIFIED). | Control file: 243 controls, status automated 218, manual 21, partial 1, supported 3. The L1 playbook has 789 top-level tasks and its header says it "attempts to fix every selected rule, even if the system is already compliant". |
| `ansible-lockdown/AMAZON2023-STIG` (S15) | DISA STIG for Amazon Linux 2023, "Version 1, Rel 4" per its README | MIT | Release `1.4.0`, 2026-10-23 | Not mentioned | 187 distinct `AZLX-23-` rule IDs in its tasks. |
| `dev-sec/ansible-collection-hardening` | Generic OS, SSH and service hardening, not a benchmark implementation | Apache-2.0 | Pushed 2026-10-22 | Not evaluated | No benchmark control numbers, so no mapping to a control list. Not considered further. UNVERIFIED for AL2023. |

I did not find another maintained CIS Amazon Linux 2023 playbook. I did not do an exhaustive search of every hardening repository.

### 6.2 Gaps in the upstream CIS role (what the wrapper must cover)

1. **Not EKS-aware.** See section 7. Six defaults would damage the node.
2. **Does not record applied control IDs.** The role writes `/etc/ansible/facts.d/compliance_facts.fact` with the benchmark, version, run date and level flags, but not a per-control list. `docs/SPEC.md` P3.1 requires each applied control to be recorded by ID in a file on the node, so the wrapper needs a small callback plugin or a post-run step (section 8.3).
3. **Idempotency is not zero.** The role's changelog states idempotency "147 -> 4 -> 2 changed over three runs". `docs/SPEC.md` P3.1 requires no changes on the second run. Expect to find and fix or suppress the remaining changing tasks (candidates: the shell and command tasks).
4. **Ansible version.** Needs ansible-core 2.16.1 or newer; the AL2023 repository package was 2.15.3 (S9). The distribution package will not run it.
5. **Collections.** Needs `community.general`, `community.crypto`, `ansible.posix`. The role's `requirements.yml` fetches them from Git. Vendor them into the S3 bundle so the node does not need Git access.
6. **Audit content is fetched at run time** (git clone of the Audit repo, goss binary from a GitHub release) when `run_audit: true`. Vendor both, or run the audit from a separate step.
7. **Check mode is not supported** (README), so do not rely on `--check` in the SSM `Check` parameter.
8. **Small community.** The repository had 48 stars and 2 open issues when read. Plan to carry local fixes.
9. **Changelog heading lags the release number** (the file's latest heading is "1.3.1 based on v1.0.0" while the tag is 1.5.0). Pin by commit, not by reading the changelog.

### 6.3 Where banner, MOTD and issue files are deployed

Neither CIS source deploys government text. Both default to the generic sentence "Authorized users only. All activity may be monitored and reported." The government text appears only in the STIG role.

| Source | Control or task | File written | Text variable | How to exclude |
|---|---|---|---|---|
| CIS role (S12) | 1.7.1 | `/etc/motd` from `templates/etc/motd.j2` | `amzn2023cis_warning_banner` | `amzn2023cis_rule_1_7_1: false`, then install our own template (section 6.4). |
| CIS role | 1.7.2 | `/etc/issue` from `issue.j2` (removes a symlink first) | same variable | Keep, with custom text. CIS requires site-defined text. |
| CIS role | 1.7.3 | `/etc/issue.net` from `issue.net.j2` | same variable | Keep, with custom text. |
| CIS role | 1.7.4 to 1.7.6 | permissions on the three files | none | Keep. |
| CIS role | 4.2.15 | `Banner /etc/issue.net` in `sshd_config` | none | Keep. |
| ComplianceAsCode CIS playbooks (S14) | tasks "Modify the System Login Banner", "... for Remote Connections", "... Message of the Day Banner", "Enable SSH Warning Banner" (tags `banner_etc_issue`, `banner_etc_issue_net`, `banner_etc_motd`, `sshd_enable_warning_banner_net`) | `/etc/issue`, `/etc/issue.net`, `/etc/motd`, and `Banner /etc/issue.net` in `/etc/ssh/sshd_config.d/00-complianceascode-hardening.conf` | `login_banner_contents`, `remote_login_banner_contents`, `motd_banner_contents` | `--skip-tags banner_etc_motd` (and the others as needed), or set the three variables. |
| STIG role (S15) | `AZLX-23-002000` (V-274065) | `/etc/issue.net` from `templates/etc/issue.net.j2` | `az2023stig_logon_banner` (default is the US Government notice and consent text) | `az2023stig_002000: false` or `--skip-tags azlx-23-002000` |
| STIG role | `AZLX-23-002005` (V-274066) | `Banner /etc/issue.net` in `sshd_config` | none | `az2023stig_002005: false` or `--skip-tags azlx-23-002005` |

Notes:

- The STIG role has no task for `/etc/motd` or `/etc/issue`. Only `/etc/issue.net` and the sshd `Banner` line.
- ComplianceAsCode has no STIG profile for AL2023 (its `products/al2023/profiles` directory holds only `cis` and `cis_server_l1`). The ComplianceAsCode-generated STIG fallback you described is therefore not available. The only STIG automation I found is S15.
- DISA publishes an Amazon Linux 2023 STIG. Search results showed Ver 1 Rel 4 on the NIST checklist page and V1R3 on a third-party viewer, and the role README says Rel 4 released "01 Jul 2026" while the NIST listing showed 01/07/2026; I could not reconcile these dates and did not open the DISA site (S16). Not needed if the CIS route is used.
- Where CIS and the STIG disagree on a banner, CIS wins: it only asks for site-defined text, without operating-system identifiers, so the custom text below meets it.

### 6.4 Custom MOTD template and task

Wording approved by the owner on 2026-10-24.

Plain text, no government wording, no company names. The upstream audit for 1.7.1 to 1.7.3 rejects the words CentOS, RHEL, Red Hat, x86_64, Linux and Amazon (any case of the first letter), so the text avoids them.

`ansible/templates/motd.j2`:

```
This host is a Kubernetes worker node. Automation builds and changes it.

  Access    Authorized administrators only.
  Logging   Logins and commands on this host are logged.
  Changes   Changes made by hand are overwritten the next time the
            hardening run applies.
  Node      {{ ansible_facts['hostname'] }}
```

Task (placed in the wrapper's `post_tasks`, with `amzn2023cis_rule_1_7_1: false` so the role does not write the file first). The task name starts with the control number so the evidence parser maps it:

```yaml
- name: 1.7.1 | PATCH | Ensure message of the day is configured properly | custom template
  ansible.builtin.template:
    src: motd.j2
    dest: /etc/motd
    owner: root
    group: root
    mode: "0644"
  tags:
    - level1-server
    - banner
    - rule_1.7.1
```

Use the same words (a shorter plain sentence) in `amzn2023cis_warning_banner` for `/etc/issue` and `/etc/issue.net`. Whether AL2023 prints anything else at login from `/etc/motd.d` or `/etc/update-motd.d` is UNVERIFIED; the EKS AMI disables `update-motd.timer` (S7). Check with `ls /etc/motd.d /etc/update-motd.d` on a node.

If the STIG fallback were ever used instead: set `az2023stig_002000: false` and `az2023stig_002005: false` in the vars, run the same MOTD task, and set the sshd `Banner` to `/etc/issue.net` with custom text through a task of our own.

### 6.5 Recommendation: use upstream with overrides

Use `ansible-lockdown/AMAZON2023-CIS` at tag `1.5.0`, vendored into the S3 bundle, driven by:

1. the overrides file (section 5.2) plus the Level 2 switch-offs (Appendix A),
2. a thin wrapper playbook that adds the custom MOTD, the evidence callback and the deviations record,
3. the upstream goss audit (vendored) and, optionally, OpenSCAP as a second opinion (section 8.4).

Reasons:

- It is the only maintained CIS AL2023 v1.0.0 Ansible implementation I found: MIT licence, a release two weeks before this document, 243 of 243 controls with per-control variables and stable task names that start with the control number.
- Using it as-is is not safe (six damaging defaults, section 1 item 4), so the answer is not "as-is".
- Writing our own role would mean re-implementing 179 controls and their idempotency, with no gain: every override we need is a variable or a skipped rule. The cost is better spent on the wrapper and the verification tool.
- The ComplianceAsCode playbook is generated, unconditional ("attempts to fix every selected rule") and does not carry CIS numbers in task tags. It sets the same forwarding sysctl to 0. Use ComplianceAsCode as a cross-check and as an OpenSCAP scanner, not as the remediation.
- The Level 1 value choices differ between the two implementations (for example the faillock deny count is 5 in the role and 3 in the ComplianceAsCode playbook), so the verifier must come from the same family as the remediation, or be tailored (section 8.4).

Risks to accept: small community, non-zero idempotency, no EKS testing statement, Ansible runtime bootstrap on the node (section 7.3). Each has an acceptance item in section 9.

## 7. Exceptions and deviations for Kubernetes and EKS

### 7.1 Controls that conflict or need care

Every `SKIP` and `Override` below is a deviation to record in `docs/DECISIONS.md`. Controls skipped are listed as deliberate deviations, per `docs/SPEC.md` section 4.

| ID(s) | Control | What it would break and why | Safe alternative | Status |
|---|---|---|---|---|
| 3.2.1 | IP forwarding off | Pod-to-pod and pod-to-service routing and kube-proxy need `net.ipv4.ip_forward=1`. The EKS AMI sets it in `/etc/sysctl.d/99-kubernetes-cri.conf`, and the kubelet unit also sets `iptables -P FORWARD ACCEPT` (S7). | Skip. Keep forwarding at 1. Record the check `sysctl -n net.ipv4.ip_forward` equals `1` as the replacement. Do not use `amzn2023cis_is_router: true` unless you accept that it also skips 3.2.2. If the control were applied, the role would write `/etc/sysctl.d/60-netipv4_sysctl.conf` with `net.ipv4.ip_forward = 0` and its handler runs `sysctl --system`; the AMI's `99-` file is read later in name order and would probably win, but that is UNVERIFIED and no reason to apply the control. | SKIP |
| 1.6.1.3, 1.6.1.4 | SELinux state | The role default sets `enforcing`. The EKS AMI lacks `container-selinux`, so enforcing prevents `nodeadm` and the node from joining (S8). | `amzn2023cis_selinux_enforce: permissive` (AMI default; satisfies "not disabled"). Do not add 1.6.1.5. | Override |
| 1.8 | Install all updates | `package: name='*' state=latest` updates in place. The AMI version-locks kernel, containerd and `amazon-ec2-net-utils`, disables cloud-init package updates to avoid version skew, and the project replaces nodes rather than patching them. | Skip. Patch by moving the node group to a newer AMI release. | SKIP (owner approved 2026-10-24) |
| 3.3.7 | Strict reverse-path filter | The VPC CNI init sets the primary interface to loose mode itself (S18); the kernel takes the larger of the `all` and per-interface values. Secondary ENIs follow `default` (1). Whether this breaks cross-node pod traffic here is untested. | Apply, then test pod-to-pod across nodes (needs two nodes) and pod-to-service on the single node. If it fails, set `net.ipv4.conf.default.rp_filter = 2` in a drop-in and record the deviation. UNVERIFIED. | Test |
| 2.2.16, 2.2.17, 2.2.18 | NFS, rpcbind, rsync | `nfs-utils` is installed by the EKS AMI for EFS and NFS volumes. Removing it breaks those mounts. | Mask `nfs-server`, `rpcbind.socket`, `rpcbind` and `rsyncd`; never remove the package. The upstream audit insists the package is absent (section 8.5). | Override |
| 4.6.6 | Root password set | The role asserts it and aborts the whole run when root has no password. EC2 root accounts normally have none (UNVERIFIED on this AMI). | Skip. Do not set a root password. Root access is by SSM only. | SKIP (owner approved 2026-10-24) |
| 5.1.1.x / 5.1.2.x | Logging backend | AL2023 uses journald; rsyslog is not installed by default (S3). | `amzn2023cis_syslog_service: journald`. 5.1.1.x become not-applicable. | Override |
| 5.1.2.1.1 to 5.1.2.1.3 | Journal upload to a remote host | No log host exists. The role default URL is a private placeholder. | Skip. Revisit when central logging exists. Keep 5.1.2.1.4. | SKIP |
| 5.1.2.6 | Journal size limits | Default `SystemKeepFree=100G` exceeds a typical root volume; journald would stop writing persistent logs. | Set `SystemMaxUse` and `SystemKeepFree` to values that fit the volume (section 5.2). | Override |
| 1.3.1 to 1.3.3 | AIDE | Scans every file; on a container host that includes image layers and pod volumes, giving a slow first database build and constant noise. | Exclude `/var/lib/containerd`, `/var/lib/kubelet`, `/var/lib/cni`, `/var/log` and `/proc`-like paths from the AIDE configuration. Build the database asynchronously. The role variables for exclusions are UNVERIFIED; a template override may be needed. | Override |
| 1.1.2.2 to 1.1.2.4 | `/tmp` options | `/tmp` is tmpfs on AL2023 (S3). `noexec` is the one to prove: nothing in the SSM agent, `nodeadm` or Ansible path should execute from `/tmp`. | Apply, run a full association cycle and a node replacement, and confirm the association still reports Success. | Test |
| 1.1.3.2 to 1.1.7.3 | Options on separate partitions | The node has one root volume, so the role skips these. | Record as not-applicable. If separate volumes are added, test `/var` options with containerd and the CSI driver first. UNVERIFIED. | N/A |
| 1.4.1 | Bootloader permissions | Depends on UEFI or BIOS boot and the grub path on this AMI. | Set `amzn2023cis_legacy_boot` after checking `test -d /sys/firmware/efi`. UNVERIFIED. | Needs input |
| 2.1.2 | chrony configuration | Default servers are public pools. Amazon Time Sync at 169.254.169.123 works without internet. The role's template is Debian-derived. | Override the server list; test `chronyc sources`. | Override |
| 4.2.4 | SSH access limit | The default allow-list comes from the connecting user, which is root under SSM, and would lock out `ec2-user`. | Set `amzn2023cis_sshd_allowusers: ec2-user` (see section 7.5). | Override |
| 4.4.1, 4.4.2 | authselect profile | Owner decision 2026-10-24: PAM unchanged. The custom-profile step runs only when `amzn2023cis_authselect_custom_profile_create` is true (default false); 4.4.2 selects a faillock feature. PAM changes can lock out interactive logins. | Skip both (`amzn2023cis_rule_4_4_1: false`, `amzn2023cis_rule_4_4_2: false`). Access is by key and SSM, not password. | SKIP |
| 4.6.3 | Shell idle timeout | `TMOUT=900` ends idle Session Manager shells after 15 minutes. | Accept. | Safe, noted |
| 6.1.11 to 6.1.13 | World-writable and unowned file scans | The file-fix step excludes `containerd` and `kubelet/pods` paths, but the sticky-bit step runs over every `df --local` mount, which may include container overlay mounts. The `find` scans are heavy on a busy node. | Keep `amzn2023cis_no_world_write_adjust: false`. Consider skipping the sticky-bit step. Run the heavy audits less often than the schedule. UNVERIFIED that `df` lists overlay mounts on this node. | Override |
| 3.1.1 | IPv6 | Disabling IPv6 breaks IPv6 and dual-stack clusters. | Keep `amzn2023cis_ipv6_required: true`. | Safe, noted |
| 1.5.2 | ptrace scope | `kernel.yama.ptrace_scope=1` limits host-level debugging tools. Processes with `CAP_SYS_PTRACE` are still allowed. | Accept. | Safe, noted |

### 7.2 Things no hardening role may touch

- **Kubelet and container runtime.** Do not restart `kubelet`, `containerd` or `amazon-ssm-agent` from the role. Do not change their unit files. Do not remount `/var/lib/containerd` or `/var/lib/kubelet`.
- **Pod networking.** Do not block the modules `ip_tables`, `iptable_nat` (the AMI loads them, S7), `overlay`, `br_netfilter`, `veth`, `vxlan` and the `nf_*`, `xt_*` and `ip_vs*` families. The benchmark does not ask for this at Level 1; it matters if anyone adds extra module blacklisting. The AMI sysctls `net.bridge.bridge-nf-call-iptables`, `vm.overcommit_memory`, `kernel.panic` and `kernel.panic_on_oops` must not be changed.
- **IMDS and bootstrap.** Nothing may block the instance metadata service from the host; `nodeadm` reads user data from it.
- **SSM agent access.** No Level 1 control disables the agent. Controls that touch accounts (4.6.x, 4.3.x) could affect the `ssm-user` account the agent creates for Session Manager; I found no conflict but did not test (UNVERIFIED). The node needs outbound HTTPS to the SSM endpoints throughout.
- **EKS bootstrap files.** `/etc/eks`, `/etc/systemd/system` and `/usr/bin` carry SELinux contexts set at AMI build (`configure-selinux.sh`, S7). A role that edits files there should not change contexts.

### 7.3 Node replacement and the SSM association

Sequence, from `docs/SPEC.md` sections 4 and 7 (P3.2): the group launches a node, `nodeadm` joins it to the cluster, the SSM agent registers it, and the association (targeting the hardening tag) applies the playbook at first registration and on its schedule. I did not verify State Manager's trigger on newly registered instances against AWS documentation; the SPEC states it and acceptance item P3.2.3 tests it.

Consequences for the role:

1. **There is an unhardened window.** The node can run pods between join and the first successful run. The single-node project accepts this. The mitigation, if wanted later, is a startup taint in the node configuration that the association removes at the end; this is not designed here and the kubelet field names in `nodeadm` are UNVERIFIED.
2. **The run must be safe with pods running.** No reboot (`skip_reboot: true`). The restarts it may cause are `sshd`, `chronyd` and `systemd-journald`. Avoid the `/tmp` busy-remount warning by not changing `/tmp` options after the first run.
3. **The run must be repeatable.** It runs on a schedule on the same node as well as on new nodes. See the idempotency gap (section 6.2 item 3).
4. **Evidence must leave the node.** A replaced node takes its files with it. Send command output to S3 using the association's output option (the console documents an S3 output bucket and prefix, written with the instance profile's permissions; S17) and copy the evidence files out in the same run.
5. **Ansible has to exist first.** The distribution `ansible-core` is 2.15.3 and the role needs 2.16.1 or newer. How `AWS-ApplyAnsiblePlaybooks` with `InstallDependencies` behaves on AL2023 is not documented in what I read (S17 lists only Ansible as a dependency for Amazon Linux). Options: a pip-installed `ansible-core` bundled in the S3 package, or a newer Python and ansible-core installed through user data. Decide and test in P3.2. UNVERIFIED.
6. **Network.** The node needs S3 for the bundle. With everything vendored it needs nothing from GitHub.
7. **Association parameters.** The document takes `SourceType`, `SourceInfo`, `InstallDependencies`, `PlaybookFile`, `ExtraVariables`, `Check`, `Verbose` and `TimeoutSeconds` (S17). It has no tags parameter, which is why level selection is done with variables and `ansible.cfg`. Which user the playbook runs as is not stated in what I read; assume root and keep `become: true`.

### 7.4 What must be done in the launch template or user data instead

These cannot be fixed by Ansible after the fact, or are better set before boot. The AWS hardening document owns the detail; this is the list the Ansible role depends on.

| Item | Why it is not an Ansible job | Note |
|---|---|---|
| Instance profile with SSM managed-instance permissions and the EKS node permissions | The agent cannot register without it | Per SPEC P2.2.3 |
| Tag the association targets | The association selects nodes by tag | Per SPEC P2.2.3 |
| Root volume size and encryption | Chosen at launch. The EKS AMI builds with a 20 GiB default (S7); size for images and logs. | |
| Metadata options (require IMDSv2) | A launch-template setting. A search summary of the AL2 transition FAQ says AL2023 enables IMDSv2 by default (S6; not confirmed); the hop-limit choice affects whether pods can reach metadata and is for the infra document. | Hop-limit effect UNVERIFIED |
| EC2 key pair by name in the launch template; inbound 22 only from the operator's address | The key pair is created outside Terraform (section 7.5). The security group is an infra setting. Decided 2026-10-24. | Details in section 7.5 |
| Node configuration (`NodeConfig` in user data, MIME multipart) | `nodeadm` reads it at boot; the AMI also reads drop-ins from `/etc/eks/nodeadm.d/` (S7 docs) | Do not run `nodeadm init` by hand |
| FIPS | Set at AMI build (`enable_fips`) | Not used |
| Kernel boot arguments (`audit=1` and similar, Level 2) | Need a reboot to apply | Not in scope |
| Separate volumes for `/var`, `/var/log`, `/home` | Block device mappings | Not in scope |

### 7.5 SSH access (decided 2026-10-24)

**Owner statement (2026-10-24):** keep EC2 Instance Connect, but SSH should only allow a key that is generated in AWS, downloaded and stored locally. **The owner confirmed on 2026-10-24 the reading below: a regular EC2 key pair, EC2 Instance Connect off, SSM Session Manager as the second way in.** (Instance Connect works by pushing temporary keys, which contradicts "only the downloaded key", so the service is dropped.)

Decision:

1. A regular EC2 key pair is created by a script with `aws ec2 create-key-pair`. The private key is written to a local file at mode 0400 and never printed. It does not go through Terraform, so it is not in Terraform state. (That the private key is returned only once, at creation, is general AWS behaviour I did not re-check in this task: UNVERIFIED.)
2. Terraform's launch template refers only to the key pair name.
3. Inbound port 22 on the node security group is open only to the operator's address. (Infra document.)
4. EC2 Instance Connect is off, because it works by pushing temporary keys at connection time.
5. SSM Session Manager stays as the second path in.

How Instance Connect is wired (read from the `aws/aws-ec2-instance-connect-config` source, S23, not from an AL2023 node): the package installs `/opt/aws/bin/eic_run_authorized_keys` and two helper scripts, creates a locked system user `ec2-instance-connect`, and in its RPM post-install step writes a systemd drop-in `sshd.service.d/ec2-instance-connect.conf` under `/lib/systemd/system/` that starts sshd with `-o "AuthorizedKeysCommand /opt/aws/bin/eic_run_authorized_keys %u %f" -o "AuthorizedKeysCommandUser ec2-instance-connect"`. Its pre-uninstall step removes the drop-in and restarts sshd. The AL2023 package may differ from this spec file: UNVERIFIED.

Implementation for the role author:

| Step | Ansible | Note |
|---|---|---|
| Remove Instance Connect | `ansible.builtin.package: name=ec2-instance-connect state=absent` | This removes the drop-in through the package's own uninstall step. The EKS AMI installs this package on purpose in `install-worker.sh` (S7); that nothing else on the node depends on it is UNVERIFIED. |
| Belt and braces: sshd drop-in | `ansible.builtin.copy` to `/etc/ssh/sshd_config.d/00-node-access.conf`, mode 0600, with the lines below, then a handler that validates (`sshd -t`) and restarts sshd | The ComplianceAsCode AL2023 product declares `sshd_distributed_config: "true"` and its playbooks write `/etc/ssh/sshd_config.d/00-complianceascode-hardening.conf` (S14), so a drop-in directory is in use on AL2023. sshd uses the first value it reads, so a `00-` file wins over later files and over the main file. That the stock `sshd_config` includes the directory is UNVERIFIED; check with `sshd -T`. |
| Name the task like a control | Task name starts with `LOCAL-SSH-1 \| PATCH \|` | So the evidence parser can map it (section 8.3). These three IDs are local controls, not CIS. |

Drop-in contents:

```
AuthorizedKeysCommand none
AuthorizedKeysCommandUser none
AuthorizedKeysFile .ssh/authorized_keys
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
AllowUsers ec2-user
```

Where the key comes from: cloud-init installs the key pair's public key into the default user's `~/.ssh/authorized_keys` at first boot (standard behaviour, UNVERIFIED on this AMI). The EKS AMI build deletes `/home/ec2-user/.ssh/authorized_keys` and `/root/.ssh/authorized_keys` and checks they are gone (`validate.sh`, S7), so no key is baked into the image. A replaced node gets the key again from the launch template, so the same local key keeps working.

CIS interaction: 4.2.1 to 4.2.20 are unchanged. 4.2.7 (root login off) and 4.2.9 (no empty passwords) agree with the drop-in. 4.2.4 is the `AllowUsers` line. The CIS role writes into `/etc/ssh/sshd_config` directly with `lineinfile`; where it and the drop-in set the same option to the same value there is no conflict, and a mismatch would show up in the checks below.

Local controls to add to the control list (source `local`, not CIS):

| ID | Requirement | Verify |
|---|---|---|
| `LOCAL-SSH-1` | EC2 Instance Connect is not installed | `rpm -q ec2-instance-connect` -> not installed; `test -e /lib/systemd/system/sshd.service.d/ec2-instance-connect.conf` -> absent |
| `LOCAL-SSH-2` | sshd does not use an external authorized-keys command and password logins are off | `sshd -T` shows `authorizedkeyscommand none`, `passwordauthentication no`, `kbdinteractiveauthentication no`, `pubkeyauthentication yes`, `permitrootlogin no` |
| `LOCAL-SSH-3` | Keys are read only from the user's file | `sshd -T` shows `authorizedkeysfile .ssh/authorized_keys`; `sshd -T` shows `allowusers ec2-user` |

(The `sshd -T` output key names are lower-cased option names; confirm the exact output on a node: UNVERIFIED.)

### 7.6 Deviation register (final list of departures from upstream)

One row per override, skipped control or exception. "Upstream default" is the value in the role at tag `1.5.0` unless stated. "Kind" is `skip` (control not applied), `override` (control applied with a different value), `exception` (applied as upstream, but the upstream audit or a verifier must be corrected), `addition` (not in upstream), `na` (not applicable on this node). The `Tool verdict` column is what the validation tool should report when the node matches the "We set" column.

| ID | Upstream default | We set | Why | Kind | Tool verdict |
|---|---|---|---|---|---|
| All 64 Level 2 Server controls (Appendix A) and 5.2.4.2, 5.2.4.3 | `amzn2023cis_level_2: true`, rule variables true | `amzn2023cis_level_2: false`; each Level 2 rule variable false | Owner decision: Level 1 Server only | skip | not-applicable (out of scope; not in the control list) |
| 3.2.1 | `amzn2023cis_rule_3_2_1: true` (sets `net.ipv4.ip_forward = 0`) | `false`; `net.ipv4.ip_forward` stays 1 | Pod networking and kube-proxy need forwarding | skip | not-applicable if `ip_forward` is 1 |
| 1.8 | `amzn2023cis_rule_1_8: true` (`dnf` update of everything) | `false` | Nodes are replaced, not patched; AMI version locks. Owner approved 2026-10-24 | skip | not-applicable |
| 4.6.6 | `amzn2023cis_rule_4_6_6: true` (asserts root password set; aborts run) | `false` | EC2 root has no password (UNVERIFIED on this AMI); setting one is worse. Owner approved 2026-10-24 | skip | not-applicable |
| 4.4.1, 4.4.2 | rules true (4.4.1 acts only if `amzn2023cis_authselect_custom_profile_create`, default false) | `amzn2023cis_rule_4_4_1: false`, `amzn2023cis_rule_4_4_2: false` | Owner decision 2026-10-24: leave PAM unchanged; no password logins | skip | not-applicable |
| 5.1.2.1.1, 5.1.2.1.2, 5.1.2.1.3 | rules true; upload URL is a private placeholder address | `false` | No remote journal host exists | skip | not-applicable |
| 1.6.1.3, 1.6.1.4 | `amzn2023cis_selinux_enforce: enforcing` | `permissive` | Enforcing breaks the EKS AMI (container-selinux missing, S8) | override | pass if SELinux is enabled and not disabled |
| 5.1.1.1 to 5.1.1.7 | `amzn2023cis_syslog_service: rsyslog` | `journald` | AL2023 uses journald; these controls are the alternative | override | not-applicable (alternative chosen) |
| 5.1.2.6 | `systemmaxuse: 10M`, `systemkeepfree: 100G` | e.g. `500M` and `1G` (size to the volume) | 100G free-space floor stops journald writing on a small volume | override | pass |
| 2.1.2 | public NTP pools | `169.254.169.123` | Amazon Time Sync, no internet needed | override | pass |
| 1.7.1 | `amzn2023cis_rule_1_7_1: true` with `motd.j2` and generic text | `amzn2023cis_rule_1_7_1: false` plus our MOTD task (section 6.4) | Owner-approved wording | override | pass |
| 1.7.2, 1.7.3, 4.2.15 | generic sentence in `amzn2023cis_warning_banner` | our short text, no words the audit rejects | Site-defined banner | override | pass |
| 4.2.4 | `amzn2023cis_sshd_allowusers` derived from connecting user | `ec2-user` | Root under SSM would lock out the SSH user. Decided 2026-10-24 (section 7.5) | override | pass |
| 6.1.11 | `amzn2023cis_no_world_write_adjust: true` | `false` | Avoid touching container overlay mounts | override | pass |
| 1.3.1, 1.3.2, 1.3.3 | AIDE scans everything | exclude container and log paths; build the database asynchronously | I/O and noise on a container host. Exclusion variables UNVERIFIED | override | pass |
| 2.2.16, 2.2.17, 2.2.18 | `mask` (already the default) | keep `mask`; never `remove` | `nfs-utils` is needed for EFS and NFS volumes. The upstream audit expects the package absent | exception | pass if masked or absent |
| 3.3.7 | strict reverse-path filter | apply, then test pod traffic; fallback `default.rp_filter = 2` | VPC CNI interaction untested | exception (test) | pass |
| 1.1.2.2 to 1.1.2.4 | apply | apply, then test association and replacement | `noexec` on `/tmp` is untested here | exception (test) | pass |
| 1.4.1 | `amzn2023cis_legacy_boot: false` | set from the real boot mode | Depends on UEFI or BIOS | override | pass |
| 1.1.3.2 to 1.1.7.3 (13 controls) | apply to separate partitions if they exist | no change; the role skips them | Single root volume | na | not-applicable |
| 3.1.1 | `amzn2023cis_ipv6_required: true` | keep `true` | IPv6 and dual-stack clusters | exception | pass |
| Manual controls (1.2.1, 1.2.3, 2.4, 3.1.1, 4.4.1, 5.1.1.2, 5.1.1.5, 5.1.2.5, 5.1.2.6, 5.1.2.7, 5.3, 6.1.13; 1.8 and 5.1.2.1.1 to 5.1.2.1.3 are above) | role warns or checks | no change | Benchmark marks them Manual | exception | not-applicable with reason `manual` unless a real check is given |
| 2.2.2, 2.2.4, 2.2.6, 2.2.9, 2.3.3, 2.2.18 | upstream goss package names are wrong (section 8.5) | use the corrected checks | Audit defects | exception | pass or fail on the corrected check |
| `LOCAL-SSH-1` to `LOCAL-SSH-3` | not in upstream | remove `ec2-instance-connect`; sshd drop-in | Owner decision 2026-10-24 on SSH | addition | pass |
| Skip tags and Ansible runtime | none | `ansible.cfg` `[tags] skip`, pinned ansible-core 2.16.1+, vendored collections | SSM document has no tags parameter; AL2023 package too old | addition | not a control |

### 7.7 Open questions for the owner

1. PAM: you asked for 4.4.x to be left alone. Controls 4.5.1 to 4.5.4 (password quality, lockout, reuse, hashing) also edit the PAM files `system-auth` and `password-auth` according to the upstream audit. Should they be skipped too, or is "PAM unchanged" meant to cover only 4.4.x? Until answered, this guideline applies 4.5.x.
(Decisions already given and recorded: MOTD wording approved; 1.8 and 4.6.6 skipped; 4.4.x skipped; SSH by downloaded key pair with Instance Connect off; Level 1 Server only.)

## 8. Verification

### 8.1 What the validation tool needs from the node

Three kinds of node-side evidence, each keyed by the benchmark control number:

| Evidence | Produced by | Answers | Format |
|---|---|---|---|
| A. Ansible run record | `ansible-playbook` with a JSON callback | Was the control applied, skipped, changed or failed in this run? | JSON (or JSON Lines) |
| B. State check results | goss audit (`AMAZON2023-CIS-Audit`) run on the node after the Ansible run | Is the node in the required state right now? | goss JSON |
| C. Optional scanner results | OpenSCAP `oscap xccdf eval` | Independent second opinion per rule | ARF or XCCDF XML |

The tool reads the control list (section 8.6) and, per control ID, combines A and B into `pass`, `fail` or `not-applicable`. It reports no percentage.

### 8.2 Per-control checks

The "Verify" column of section 5.3 is the check for each Level 1 control. For skipped controls the replacement check is in section 7.1 and in the control list.

### 8.3 Capturing Ansible output

- **Callback.** `ansible.posix.json` (one JSON document at the end of the run) and `ansible.posix.jsonl` (one JSON object per line) both exist in the `ansible.posix` collection (checked in the collection source). Prefer `jsonl`: a truncated or interrupted run still leaves usable lines. Configure in an `ansible.cfg` shipped beside the playbook:

  ```ini
  [defaults]
  stdout_callback = ansible.posix.jsonl
  [tags]
  skip = level2-server,level1-workstation,level2-workstation
  ```

  `[tags] skip` is the documented ini key for the skip-tags setting (ansible-core `base.yml`, `TAGS_SKIP`, env `ANSIBLE_SKIP_TAGS`). That the SSM document runs `ansible-playbook` from a directory where this `ansible.cfg` is picked up is UNVERIFIED; if it is not, rely on the variable switch-offs in Appendix A and on SSM's S3 output of the run text.
- **Output size.** SSM command output is limited in size; send it to S3 with the association's S3 output option, and do not depend on the console copy. The exact limit is not stated in what I read.
- **Control ID from the task name.** Upstream task names begin with the control number: `<number> | PATCH | <title>` or `<number> | AUDIT | ...`. The parser regex is `^(\d+(?:\.\d+)*) \| (PATCH|AUDIT) \|`. Some names have extra suffixes after further `|` separators; group by the leading number. Our own tasks (the MOTD task) follow the same pattern.
- **Mapping task results to a state per control.** `failed` anywhere in the control: fail. `ok` or `changed`: applied. `skipped` for every task of the control: skipped (expected only for controls in the deviation list or marked not-applicable). Tasks that use `set_fact` or `debug` (the sysctl and mount-option controls only set facts, and the files are written later in `tasks/post.yml` and in handlers, whose tasks are tagged `always` rather than with a control number) say nothing about the real state, so for those controls the goss result is the evidence, not the Ansible result.
- **Per-control file on the node.** Required by `docs/SPEC.md` P3.1. The upstream role does not write one. Add a small callback plugin to the bundle (about 40 lines) that appends one JSON line per control result to `/var/lib/node-hardening/controls.jsonl` with fields `ts`, `run_id`, `control`, `state` (`ok`, `changed`, `skipped`, `failed`), `task`. Also write `/var/lib/node-hardening/run.json` with the benchmark name and version, role tag and commit, wrapper version, instance ID (from the facts, not a secret), start and end time, and whether a reboot is pending (`change_requires_reboot`).
- **goss run.** The upstream `run_audit.sh` calls `goss -g <audit dir>/goss.yml --vars <vars file> --vars-inline '<json>' v --max-concurrent N -f json -o pretty > <out>`. From the role, `run_audit: true` writes `/opt/<hostname>-AMAZON2023-CIS-1.0.0_pre_scan_<epoch>.json` and `..._post_scan_<epoch>.json` (pattern from `defaults/main/audit.yml`; `audit_only: true` skips remediation). The goss JSON output has `results[]` entries and a `summary` object. From the goss source (`outputs/json.go`): each result has `successful` (boolean), `summary-line`, `summary-line-compact` and `duration`; the summary has `test-count`, `failed-count`, `skipped-count`, `total-duration` and `summary-line`. That each result also carries `title` and the `meta` block (which holds `CIS_ID`) comes from goss's result structure, not from a file I read: UNVERIFIED, confirm on the first run. If `meta` is absent, fall back to the leading number in `title` (upstream titles start with the number, for example `3.2.1 | Ensure IP forwarding is disabled | IPv4`).

### 8.4 OpenSCAP route

- Datastream: `ssg-al2023-ds.xml` in ComplianceAsCode release v0.1.82. Profile IDs (from the playbook headers): `xccdf_org.ssgproject.content_profile_cis_server_l1` and `xccdf_org.ssgproject.content_profile_cis`.
- Command (standard `oscap` options, not run here): `oscap xccdf eval --profile xccdf_org.ssgproject.content_profile_cis_server_l1 --results-arf /var/lib/node-hardening/arf.xml --report /var/lib/node-hardening/report.html ssg-al2023-ds.xml`. Whether `openscap-scanner` is in the AL2023 repositories is UNVERIFIED.
- Result states are the XCCDF rule results (`pass`, `fail`, `notapplicable`, `notchecked`, `error` and others); confirm against the produced XML.
- Map rule to CIS number through `controls/cis_al2023.yml`: each control lists its `rules`. Controls with an empty rule list are manual.
- Expect disagreement on parameters. The ComplianceAsCode profile and the role pick different values in places (faillock deny 3 vs 5, for example). Use a tailoring file to align OpenSCAP with our overrides, and to deselect the rules for 3.2.1 (`sysctl_net_ipv4_ip_forward` and the IPv6 forwarding rule), 1.8 and the SELinux enforcing rules. The profile also has the same conflicts as the role, so it must never be used to remediate.
- Treat C as supporting evidence. The tool's verdict should come from A and B so it matches the role's own family.

### 8.5 Known defects in the upstream audit (the tool must correct them)

| Control | Defect | Handling |
|---|---|---|
| 2.2.16 | Expects `nfs-utils` absent; the benchmark and the role accept "masked" | Pass if `nfs-server` is masked or the package is absent. |
| 2.2.17, 2.2.18 | Same as 2.2.16: the upstream test requires the package absent as well as the service state, and 2.2.18 checks the names `rsync` and `rsync` service where the role handles `rsync-daemon` and `rsyncd` | Use the corrected checks in the table. |
| 2.2.4, 2.2.9, 2.2.6, 2.3.3, 2.2.2 | Package name mismatches against the role (`dhcp` vs `dhcp-server`, `cyrus-impad`, a template glitch for `vsftpd`, `ypbind` for the FTP client) | Use the corrected checks in the table. |
| 3.2.1 | Test is correct for the benchmark, wrong for this node | The tool must expect `net.ipv4.ip_forward` equal to 1 and treat the upstream failure as expected. |
| 1.6.1.4 | Upstream test accepts permissive or enforcing; the role would set enforcing | Keep permissive. |
| 6.1.2 | Same title as 6.1.1 in both implementations | Treat as a duplicate check of `/etc/passwd`. |
| Manual controls | The benchmark marks 16 Level 1 controls Manual: 1.2.1, 1.2.3, 1.8, 2.4, 3.1.1, 4.4.1, 5.1.1.2, 5.1.1.5, 5.1.2.1.1 to 5.1.2.1.3, 5.1.2.5, 5.1.2.6, 5.1.2.7, 5.3, 6.1.13 (from S14). The goss tests for 2.4, 5.1.2.7 and 5.3 are `echo` placeholders that say nothing about node state | For a placeholder, the tool reports not-applicable with reason `manual`, never pass. Where upstream has a real check, give the control an explicit `checks` list and keep `disposition: manual` only for the reviewer note. |

### 8.6 Machine-readable control list

One file, one entry per control, committed in the repo (suggested path `ansible/controls/cis-al2023-l1.yml`). The tool reads it, reads evidence A and B, and prints one verdict per `id`. The three entries below show an applied control, a skipped control with a replacement check, and an override with alternative checks. Field names are a proposal for the tool author, not a standard.

```yaml
schema: 1
benchmark:
  name: CIS Amazon Linux 2023 Benchmark
  version: "1.0.0"
  source: CIS
  profile: Level 1 - Server
role:
  name: ansible-lockdown/AMAZON2023-CIS
  tag: "1.5.0"
controls:
  - id: "CIS-AL2023-1.0.0:3.2.2"
    number: "3.2.2"
    level: 1
    title: Packet redirect sending is disabled
    disposition: apply              # apply | override | skip | not-applicable | manual
    ansible:
      rule_var: amzn2023cis_rule_3_2_2
      task_prefix: "3.2.2 |"
      expect: applied               # applied | skipped
    checks:                         # all must hold for pass
      - {kind: sysctl, name: net.ipv4.conf.all.send_redirects, expect: "0"}
      - {kind: sysctl, name: net.ipv4.conf.default.send_redirects, expect: "0"}
    goss: {cis_id: "3.2.2"}

  - id: "CIS-AL2023-1.0.0:3.2.1"
    number: "3.2.1"
    level: 1
    title: IP forwarding is disabled
    disposition: skip
    deviation:
      ref: "DEV-pending"            # replace with the id used in docs/DECISIONS.md
      reason: "Kubernetes pod networking and kube-proxy require IP forwarding"
    ansible:
      rule_var: amzn2023cis_rule_3_2_1
      task_prefix: "3.2.1 |"
      expect: skipped
    checks:                         # replacement check; holds -> not-applicable, fails -> fail
      - {kind: sysctl, name: net.ipv4.ip_forward, expect: "1"}
    goss: {cis_id: "3.2.1", ignore_result: true}

  - id: "CIS-AL2023-1.0.0:2.2.16"
    number: "2.2.16"
    level: 1
    title: nfs-utils is not installed or the nfs-server service is masked
    disposition: override
    ansible:
      rule_var: amzn2023cis_rule_2_2_16
      task_prefix: "2.2.16 |"
      expect: applied
      vars: {amzn2023cis_nfs_server: mask}
    checks_any:                     # any one group holding is a pass
      - [{kind: unit_state, name: nfs-server, expect: masked}]
      - [{kind: package_absent, name: nfs-utils}]
    goss: {cis_id: "2.2.16", ignore_result: true}
```

Verdict rules for the tool:

| Disposition | Condition | Verdict |
|---|---|---|
| apply / override | Ansible shows the control applied (or the control is state-only) and all checks hold in the goss or direct evidence | pass |
| apply / override | any task failed, any check fails, or evidence is missing | fail (with the reason) |
| skip | the replacement check holds and the Ansible record shows the control skipped | not-applicable (with the deviation reference) |
| skip | the replacement check fails, or the control ran when it should have been skipped | fail |
| not-applicable | the precondition holds (for example no separate `/var` mount) | not-applicable |
| manual | always | not-applicable, reason `manual` |

The `kind` values the tool must support, from the checks in section 5.3: `sysctl`, `package_present`, `package_absent`, `unit_state` (`enabled`, `active`, `masked`), `mount_option`, `file_mode` (path, mode, owner, group), `file_contains` and `file_not_contains` (path, regex), `command` (exec, expected exit code, expected output regex), `kernel_module_disabled`.

## 9. Acceptance checklist

Role and wrapper:

- [ ] Role pinned to `ansible-lockdown/AMAZON2023-CIS` tag `1.5.0` (commit `3a89ab3853926c11327591d58416c020c00a8717`), vendored with its three collections; no Git access needed on the node.
- [ ] ansible-core 2.16.1 or newer is available on the node before the playbook runs, and this is tested on a freshly launched EKS node.
- [ ] Overrides file contains every override in section 5.2 and every Level 2 switch-off in Appendix A.
- [ ] Controls 3.2.1, 1.8, 4.6.6, 4.4.1, 4.4.2, 5.1.2.1.1, 5.1.2.1.2 and 5.1.2.1.3 are skipped; `amzn2023cis_selinux_enforce` is `permissive`; the nodes show `net.ipv4.ip_forward = 1` and SELinux not disabled after a run.
- [ ] The node joins the cluster and reports `Ready` after the first run, after a second run, and after a node replacement.
- [ ] A second consecutive run reports no changes (SPEC P3.1.2); any remaining changing task is fixed or listed with a reason.
- [ ] `ansible-lint` and `ansible-playbook --syntax-check` pass (SPEC P3.1.4).
- [ ] The custom MOTD is installed, contains none of the words the upstream audit rejects, and `/etc/motd` is `0644 root root`.
- [ ] The wrapper writes `/var/lib/node-hardening/controls.jsonl` and `run.json`, and the files reach S3.
- [ ] Every deviation in section 7.1 marked `SKIP` or `Override` has an entry in the role README and `docs/DECISIONS.md` (SPEC P3.1.3).
- [ ] Every row of the deviation register (section 7.6) is implemented in the overrides file or wrapper, and appears in the control list with the same disposition.
- [ ] SSH (decided 2026-10-24): the `ec2-instance-connect` package is absent, `sshd -T` shows no authorized-keys command and no password authentication, the downloaded key logs in as `ec2-user`, and a replaced node accepts the same key. The private key never appears in Terraform state, logs or output.
- [ ] The three `Test` items (3.3.7, 1.1.2.2 to 1.1.2.4) were tested and the result is recorded.
- [ ] SSM agent, kubelet and containerd are not restarted by a run (check unit start times before and after).
- [ ] Open items marked UNVERIFIED in this file are resolved or accepted by the owner.

Verification:

- [ ] The control list has 179 CIS entries (one per Level 1 control) plus the three `LOCAL-SSH-` entries, each with an `id`, a disposition and a check.
- [ ] The tool prints pass, fail or not-applicable for every ID and no percentage.
- [ ] The tool treats the defects in section 8.5 correctly, in particular 2.2.16, 3.2.1 and the manual controls.
- [ ] The goss JSON from a real node was inspected once to confirm the `meta` and `title` fields (section 8.3).
- [ ] Evidence from a replaced node reaches S3 without anyone logging in.

## 10. Sources

Read on 2026-10-23 and 2026-10-24. "Read" means I opened and read the page or file; "search result only" means I saw it only in a search summary.

| ID | URL | Version / date | How used |
|---|---|---|---|
| S1 | https://docs.aws.amazon.com/linux/al2023/ug/release-cadence.html | page read 2026-10-24 | AL2023 support dates. Read. |
| S2 | https://aws.amazon.com/amazon-linux-2/faqs/ | page read 2026-10-24 | AL2 end of support 2026-06-30. Read. |
| S3 | https://docs.aws.amazon.com/linux/al2023/ug/compare-with-al2.html | page read 2026-10-24 | AL2023 vs AL2 topic list: `/tmp` tmpfs, journald replaces rsyslog, timers replace cron. Only the topic list and the support section were returned; details of each topic were not read. |
| S4 | https://docs.aws.amazon.com/linux/al2023/ug/selinux.html | page read 2026-10-24 | SELinux enabled, permissive by default. Read. |
| S5 | https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-ami.html | page read 2026-10-24 | Managed node groups on 1.30 or newer default to AL2023. Read. |
| S6 | https://docs.aws.amazon.com/eks/latest/userguide/eks-ami-deprecation-faqs.html | search result only | AL2 AMI publishing stopped 2025-11-26. I did not open the page. |
| S7 | https://github.com/awslabs/amazon-eks-ami (MIT-0), downloaded as a tarball, last push 2026-10-23, CHANGELOG release `v20260930` | read | `templates/al2023` provisioners, sysctls, modules, units; `templates/al2027` preview; `nodeadm` docs; AMI table with source AMI names. |
| S8 | https://github.com/awslabs/amazon-eks-ami/issues/2006 | opened 2024-10-14; last comment 2025-10-14; open | Enforcing mode is unsupported on the EKS AL2023 AMI. Read the issue and comments. |
| S9 | https://docs.aws.amazon.com/linux/al2023/release-notes/all-packages-AL2023.12.html | AL2023.12 package list | `ansible-core` 2.15.3, `ansible` 8.3.0, `aide` 0.18.6, `audit` 3.1.5. Only the first 100,000 characters of a 3.7-million-character page were read; other packages were not checked. |
| S10 | https://www.cisecurity.org/benchmark/amazon_linux | read 2026-10-24 | Versions listed: Amazon Linux 2023 1.0.0, Amazon Linux 2 4.0.0, Amazon Linux 2 STIG 2.0.1. The page has no dates. |
| S11 | https://www.cisecurity.org/insights/blog/cis-benchmarks-may-2026-update | search result only | Mentions an Amazon Linux 2 v4.0.0 update and no AL2023 update, per the search summary. Not opened. |
| S12 | https://github.com/ansible-lockdown/AMAZON2023-CIS (MIT), tag `1.5.0` and branch `devel` | tag commit `3a89ab38...`, 2026-10-08 | Role source: tasks, defaults, templates, changelog, README. Read in full for the controls cited. |
| S13 | https://github.com/ansible-lockdown/AMAZON2023-CIS-Audit (MIT), branch `devel` | commit `0017cb58...`, 2026-09-15 | goss checks and `run_audit.sh`. Parsed all check files. |
| S14 | https://github.com/ComplianceAsCode/content, release v0.1.82 (2026-09-01) | release zip downloaded | `controls/cis_al2023.yml`, `products/al2023`, `ansible/al2023-playbook-cis*.yml`, file listing. BSD-3-Clause per the LICENSE header. |
| S15 | https://github.com/ansible-lockdown/AMAZON2023-STIG (MIT), release 1.4.0 (2026-10-23) | read | Banner tasks and variables, STIG version in README. |
| S16 | https://aws.amazon.com/blogs/compute/disa-stig-for-amazon-linux-2023-is-now-available ; https://ncp.nist.gov/checklist/revision/7430 ; https://www.stigviewer.com/stigs/amazon_linux_2023 | search results only | Existence of the AL2023 STIG and its release numbering. I did not open these pages and did not reach public.cyber.mil. |
| S17 | https://docs.aws.eu/systems-manager/latest/userguide/systems-manager-state-manager-ansible.html | page read 2026-10-24 | `AWS-ApplyAnsiblePlaybooks` parameters, S3 output option. Parameter defaults and run-as user not on the page. |
| S18 | https://github.com/aws/amazon-vpc-cni-k8s, `cmd/aws-vpc-cni-init/main.go` | code search hit 2026-10-24 | The init sets the primary interface `rp_filter` to 2. |
| S19 | https://github.com/ansible-collections/ansible.posix `plugins/callback/`; https://github.com/ansible/ansible `lib/ansible/config/base.yml` | read 2026-10-24 | `json` and `jsonl` callbacks exist; `TAGS_RUN` and `TAGS_SKIP` settings. |
| S20 | https://github.com/goss-org/goss `outputs/json.go` (Apache-2.0) | read 2026-10-24 | goss JSON fields. |
| S21 | CIS Amazon Linux 2023 Benchmark v1.0.0 PDF | not accessible | Needs a free registration. Not read. Control titles and levels come from S12 and S14 only. |
| S23 | https://github.com/aws/aws-ec2-instance-connect-config (Apache-2.0), tarball; last push 2024-06-13 | read | How Instance Connect wires into sshd (`rpmsrc/SPECS/generic.spec`, `src/rpm_systemd`). Source of the upstream package, not the AL2023 build. |
| S24 | https://docs.aws.amazon.com/linux/al2023/ug/ssh-host-key.html | page read 2026-10-24 | AL2023 sshd defaults: `ssh-rsa` signatures disabled, `UseDNS=no`. Other defaults not on the page. |
| S22 | `docs/SPEC.md` (v0.5) and `PROJECT.md` in this repository | read | Scope, Level 1 Server, deviation rule, acceptance criteria P3.1 to P3.3. |

## Appendix A. Level 2 switch-offs for the overrides file

Set to false in the same overrides file. From the Level 2 Server entries of S14, plus 5.2.4.2 and 5.2.4.3 (already listed in section 5.2).

```yaml
amzn2023cis_rule_1_1_1_1: false
amzn2023cis_rule_1_1_1_2: false
amzn2023cis_rule_1_1_3_1: false
amzn2023cis_rule_1_1_4_1: false
amzn2023cis_rule_1_1_5_1: false
amzn2023cis_rule_1_1_6_1: false
amzn2023cis_rule_1_1_7_1: false
amzn2023cis_rule_1_2_4: false
amzn2023cis_rule_1_6_1_5: false
amzn2023cis_rule_2_2_1: false
amzn2023cis_rule_3_1_2: false
amzn2023cis_rule_3_1_3: false
amzn2023cis_rule_3_1_4: false
amzn2023cis_rule_3_1_5: false
amzn2023cis_rule_3_4_1_1: false
amzn2023cis_rule_3_4_1_2: false
amzn2023cis_rule_3_4_2_1: false
amzn2023cis_rule_3_4_2_2: false
amzn2023cis_rule_3_4_2_3: false
amzn2023cis_rule_3_4_2_4: false
amzn2023cis_rule_3_4_2_5: false
amzn2023cis_rule_3_4_2_6: false
amzn2023cis_rule_3_4_2_7: false
amzn2023cis_rule_4_2_12: false
amzn2023cis_rule_4_2_13: false
amzn2023cis_rule_5_2_1_1: false
amzn2023cis_rule_5_2_1_2: false
amzn2023cis_rule_5_2_1_3: false
amzn2023cis_rule_5_2_1_4: false
amzn2023cis_rule_5_2_2_1: false
amzn2023cis_rule_5_2_2_2: false
amzn2023cis_rule_5_2_2_3: false
amzn2023cis_rule_5_2_3_1: false
amzn2023cis_rule_5_2_3_2: false
amzn2023cis_rule_5_2_3_3: false
amzn2023cis_rule_5_2_3_4: false
amzn2023cis_rule_5_2_3_5: false
amzn2023cis_rule_5_2_3_6: false
amzn2023cis_rule_5_2_3_7: false
amzn2023cis_rule_5_2_3_8: false
amzn2023cis_rule_5_2_3_9: false
amzn2023cis_rule_5_2_3_10: false
amzn2023cis_rule_5_2_3_11: false
amzn2023cis_rule_5_2_3_12: false
amzn2023cis_rule_5_2_3_13: false
amzn2023cis_rule_5_2_3_14: false
amzn2023cis_rule_5_2_3_15: false
amzn2023cis_rule_5_2_3_16: false
amzn2023cis_rule_5_2_3_17: false
amzn2023cis_rule_5_2_3_18: false
amzn2023cis_rule_5_2_3_19: false
amzn2023cis_rule_5_2_3_20: false
amzn2023cis_rule_5_2_3_21: false
amzn2023cis_rule_5_2_4_1: false
amzn2023cis_rule_5_2_4_2: false
amzn2023cis_rule_5_2_4_3: false
amzn2023cis_rule_5_2_4_4: false
amzn2023cis_rule_5_2_4_5: false
amzn2023cis_rule_5_2_4_6: false
amzn2023cis_rule_5_2_4_7: false
amzn2023cis_rule_5_2_4_8: false
amzn2023cis_rule_5_2_4_9: false
amzn2023cis_rule_5_2_4_10: false
amzn2023cis_rule_6_1_10: false
```
