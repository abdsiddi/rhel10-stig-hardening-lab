# RHEL 10 STIG Hardening Lab (OpenSCAP + Ansible)

This is a writeup of a lab I did on my own VM to practice compliance scanning and hardening on Red Hat Enterprise Linux. I run VMware Workstation instead of VirtualBox or Proxmox, and I used RHEL 10 instead of RHEL 9, so almost none of the commands from the tutorial I followed worked exactly as shown. This repo documents what I actually ran, what broke, and how I fixed it.

I am a final year cybersecurity student, so this was mainly to build a real skill (reading a compliance scan and actually fixing what it finds) instead of just watching a video about it.

## Purpose of the lab

The goal was to take a fresh RHEL install, scan it against a security baseline, and then actually fix what the scan flags, instead of just running a tool and looking at the output. This is basically what an ISSO, compliance analyst, or systems hardening person does before a system can be authorized to run in a government or enterprise environment.

Specifically the lab covers:

- Scanning a Linux system with OpenSCAP against the DISA STIG profile
- Understanding that a failed check is not just "red bad, green good" but ties back to an actual NIST 800-53 control
- Fixing one setting by hand (disabling direct root login over SSH) to see the before and after
- Using Ansible to automate the remaining fixes instead of doing them one by one
- Re-scanning and comparing the before and after report
- Troubleshooting all the environment issues that showed up along the way, since nothing about my setup matched the tutorial exactly

## My setup

- Hypervisor: VMware Workstation (not VirtualBox/Proxmox like most tutorials assume)
- OS: RHEL 10.0 (not RHEL 9, which is what most existing SSG content and tutorials are built for)
- Tools: openscap-scanner, scap-security-guide, ansible-core
- Profile used: `xccdf_org.ssgproject.content_profile_stig`

## Step by step, including everything that went wrong

### 1. Installing RHEL 10 in VMware

VMware's guest OS list only went up to RHEL 9 in the dropdown, since RHEL 10 was too new to be listed yet. I just selected RHEL 9 as the guest OS type and pointed it at the RHEL 10 ISO anyway. It installed and ran fine, this only affects some default hardware settings, not what actually gets installed.

During the install I had to manually fix four warning icons on the Installation Summary screen before I could begin install: register with Red Hat, pick a software set, enable the root account, and create a user.

![Install summary screen](screenshots/01-rhel10-install-summary.png)
![Registering with Red Hat](screenshots/02-rhel10-connect-to-redhat.png)
![Software selection, minimal install](screenshots/03-rhel10-software-selection-minimal.png)
![Installation progress](screenshots/04-rhel10-installation-progress.png)
![First boot login](screenshots/05-rhel10-first-boot-login.png)

### 2. Updating and installing OpenSCAP

```bash
sudo dnf update -y
sudo dnf install -y openscap-scanner scap-security-guide
sudo dnf install -y ansible-core
```

![dnf update](screenshots/06-rhel10-dnf-update.png)
![Installing OpenSCAP packages](screenshots/07-openscap-packages-installation.png)
![Installing ansible-core](screenshots/08-ansible-core-installation.png)

Checked what content actually existed for RHEL 10, since I already knew RHEL 10 content ships differently than RHEL 9:

```bash
ls /usr/share/xml/scap/ssg/content/
```

![Verifying ssg-rhel10-ds.xml exists](screenshots/09-verify-ssg-rhel10-ds-xml.png)

### 3. Snapshot before touching anything

Before changing any settings I took a VMware snapshot named `baseline-clean`. STIG remediation can genuinely lock you out of SSH or break the system, so this was my undo button in case anything went wrong.

VM menu > Snapshot > Take Snapshot in the GUI.

![Taking the snapshot in VMware](screenshots/10-vmware-take-snapshot.png)
![Snapshot baseline-clean confirmed](screenshots/11-vmware-snapshot-baseline-clean.png)

### 4. Baseline scan, and the first round of errors

This is where things stopped matching the tutorial. I tried running the scan as a multi line command with backslashes, copy pasted, and it broke because of stray whitespace after the line continuation characters. The shell read `--profile` as if it was a filename.

```
OpenSCAP Error: Unable to open file: ' --profile'
```

![Multi line paste syntax errors](screenshots/12-oscap-eval-syntax-errors.png)

Fixed it by just running it as one single line instead of multi line with backslashes. Then hit a second error, this time pointing at the wrong xml file entirely, since I had typed the plain RHEL 9 style filename instead of the RHEL 10 data stream file:

```
OpenSCAP Error: Unable to open file: '/usr/share/xml/scap/ssg/content/ssg-rhel10-xccdf.xml'
```

![Wrong xml file referenced](screenshots/13-oscap-eval-wrong-xml-file.png)

Checked the content directory again to confirm the exact filename:

```bash
ls /usr/share/xml/scap/ssg/content/
```

![Listing the ssg content directory](screenshots/14-ls-ssg-content-directory.png)

Only `ssg-rhel10-ds.xml` existed, it ships as a data stream file, not the plain xccdf file the tutorial used (that one was built for RHEL 9). Swapped the filename and the scan finally ran properly, dumping the full baseline results including all the audit related checks that would come up again later:

```bash
sudo oscap xccdf eval --profile xccdf_org.ssgproject.content_profile_stig --results scan-results.xml --report baseline-report.html /usr/share/xml/scap/ssg/content/ssg-rhel10-ds.xml
```

![Baseline scan results, auditd section](screenshots/15-oscap-scan-results-auditd.png)
![Baseline scan results continued](screenshots/16-oscap-scan-results-auditd-continued.png)

### 5. Manual fix: disabling root login over SSH

This is the single check the tutorial focuses on to show how one setting maps to a real control (CCI to NIST 800-53).

```bash
echo 'PermitRootLogin no' | sudo tee /etc/ssh/sshd_config.d/01-stig-rootlogin.conf
sudo sshd -t && sudo systemctl restart sshd.service
```

First attempt at this had a typo (`sytemctl` instead of `systemctl`), then after fixing that the config still came back as `yes` when checked with `sshd -T`. Turned out I had a leftover config file from an earlier attempt, `01-permitrootlogin.conf`, sitting in the same drop in directory. Since `p` comes before `s` alphabetically, OpenSSH read that one first and my file never took effect. Deleted the conflicting file and it finally showed `no`.

![Fixing sshd PermitRootLogin, finding and removing the conflicting file](screenshots/17-fix-sshd-permitrootlogin.png)

This was actually a good lesson, leftover config files silently overriding your intended setting is a real thing that happens in production hardening work too, not just a lab mistake.

### 6. Verifying the single fix with OpenSCAP

Found the exact rule ID for this specific check and ran a single rule scan instead of the whole profile:

```bash
oscap xccdf eval --profile xccdf_org.ssgproject.content_profile_stig --oval-results /usr/share/xml/scap/ssg/content/ssg-rhel10-ds.xml 2>&1 | grep -i root_login
```

![Finding the exact oscap rule ID](screenshots/18-find-oscap-rule-id.png)

```bash
sudo oscap xccdf eval --rule xccdf_org.ssgproject.content_rule_sshd_disable_root_login --results rootlogin-check.xml --report rootlogin-check.html /usr/share/xml/scap/ssg/content/ssg-rhel10-ds.xml
```

Result came back pass, this was the "green check" moment the tutorial talks about.

![Single rule eval, passing](screenshots/19-oscap-single-rule-eval-pass.png)

### 7. Generating and running the Ansible remediation

Instead of fixing every failed check by hand, generated an Ansible playbook straight from the scan content. First attempt at reading the remediation scripts came up empty since I ran `less` before the generate command had actually finished writing the file.

![Missing remediation script, generate hadn't finished](screenshots/20-missing-remediation-scripts.png)

```bash
sudo oscap xccdf generate fix --profile xccdf_org.ssgproject.content_profile_stig --fix-type ansible --output stig-remediate.yml /usr/share/xml/scap/ssg/content/ssg-rhel10-ds.xml
```

![Generating the ansible fix](screenshots/21-oscap-generate-ansible-fix.png)

Once it actually generated (2000+ lines), read through it before running anything:

![Viewing the stig-remediate playbook](screenshots/22-view-stig-remediate-playbook.png)
![Stopped scrolling partway through with less](screenshots/23-less-playbook-stopped.png)

This part had the most errors of the whole lab.

**Error 1**, running the playbook failed with a missing Ansible module:

```
ERROR! couldn't resolve module/action 'community.general.ini_file'
```

![Missing community.general module](screenshots/24-ansible-missing-community-general.png)
![Same error again on retry](screenshots/25-ansible-missing-community-general-duplicate.png)

Installed the missing collection:

```bash
ansible-galaxy collection install community.general
```

![Installing community.general](screenshots/26-install-community-general-collection.png)

**Error 2**, still failed on the same module even after installing it, because I ran the playbook with `sudo`, which uses root's environment, not the user environment the collection had just been installed into. Ansible could not see the collection it had literally just installed.

![Error persists after install because of sudo environment](screenshots/27-ansible-error-persists-with-sudo.png)

**Error 3**, the generated yml file was owned by root since it came from a `sudo oscap` command, so my regular user could not even read it.

```
Permission denied: '/home/abdsiddi/stig-remediate.yml'
```

![Permission denied reading the playbook](screenshots/28-ansible-playbook-permission-denied.png)

**Error 4**, next missing module, this time `ansible.posix.sysctl`.

![Missing ansible.posix.sysctl module](screenshots/29-ansible-missing-posix-sysctl.png)

Fixed the ownership and installed the posix collection together:

```bash
sudo chown abdsiddi:abdsiddi stig-remediate.yml
ansible-galaxy collection install ansible.posix
```

![chown the playbook and install ansible.posix](screenshots/30-chown-playbook-and-install-posix.png)

**Error 5**, ran without sudo this time using `--ask-become-pass` and it got much further, but failed on one task (installing `aide`) with "This command has to be run under the root user", meaning become was not fully escalating for that task.

![become not escalating to root](screenshots/31-ansible-become-root-failure.png)

Went back to running it with sudo, but that brought back the original missing collection error, since sudo uses root's own collection path, and the collections had only been installed under my regular user.

![sudo brings back the missing collection error](screenshots/32-ansible-sudo-missing-collection-error.png)

**Actual fix**, installed the collections again specifically under root, then ran the whole playbook as root with sudo, skipping `become` entirely since it was already root:

```bash
sudo ansible-galaxy collection install ansible.posix community.general community.crypto
sudo ansible-playbook stig-remediate.yml --connection=local -i localhost, --skip-tags gui,desktop
```

![Installing collections as root](screenshots/33-install-collections-as-root.png)

This finally ran clean.

![Ansible STIG playbook running successfully](screenshots/34-ansible-stig-playbook-success.png)

### 8. Getting the reports off the VM

Since RHEL was CLI only, I had to scp the HTML reports to my Windows host to actually view them in a browser. Looked up the VM's IP first, then hit an scp syntax error on the first attempt.

![IP lookup and scp syntax error](screenshots/35-ip-lookup-and-scp-syntax-error.png)

Next attempt failed with permission denied on the local side, turned out I was sitting in a Windows system folder that regular users can't write to.

![scp local permission denied](screenshots/36-scp-local-permission-denied.png)

Also had a version where the report file itself was owned by root again since it came from a `sudo oscap` command, which blocked the transfer from the remote side too.

![scp attempt on the scan report](screenshots/37-scp-post-scan-report.png)

Fixed all of it by moving to a normal folder like Desktop on the Windows side and re-chowning the file back to my regular user on the VM side before sending it.

![scp finally succeeding](screenshots/38-scp-post-scan-report-success.png)

### 9. First real results: 71.8 percent

Compared the pass and fail counts directly in the terminal before even opening the HTML:

```bash
grep -c 'result>pass' full-scan-after.xml
grep -c 'result>fail' full-scan-after.xml
```

![Comparing scan result counts](screenshots/39-grep-compare-scan-results.png)

Pulled both the baseline and the after report over to my Windows host with PowerShell:

![Downloading reports with scp from PowerShell](screenshots/40-powershell-scp-download-reports.png)

Opened the baseline HTML report first, official OpenSCAP score was:

![Baseline HTML report, 48 percent](screenshots/41-oscap-html-report-baseline-48-percent.png)

**Baseline score: 48%**

Then opened the report after the first Ansible remediation run:

![HTML report after first remediation, 71.82 percent](screenshots/42-oscap-html-report-first-remediation-71-percent.png)

**Score after first remediation: 71.82%**

### 10. Chasing down the remaining fails

Looked at the failures left, grouped by severity. Most of them were medium severity audit rules (things like recording chmod, chown, sudo usage, kernel module loading, close to 70 of the fails were all audit related). There were also 5 high severity fails:

- Enable FIPS Mode
- Set kernel parameter crypto.fips_enabled to 1
- Verify system was booted with fips=1
- Encrypt Partitions
- Set the Boot Loader Admin Username to a Non-Default Value
- Set Boot Loader Password in grub2

FIPS mode and partition encryption are basically not fixable after the fact, Red Hat expects those to be set during install, so those got documented as known limitations instead of chased further.

The GRUB password one was fixable live with `grub2-setpassword`.

The big win was the audit rules. Checked and the rules were already written to disk by the Ansible run, just not loaded into the running system:

```bash
sudo auditctl -l
sudo augenrules --load
sudo systemctl restart auditd
```

This actually failed too, but in a good way:

```
/sbin/augenrules: Audit system is in immutable mode - exiting with no changes
Failed to restart auditd.service: Operation refused, unit auditd.service may be requested by dependency only
```

This was one of the STIG controls doing its job, "Make the auditd Configuration Immutable" was one of the passed rules from the Ansible run, so the audit rules genuinely cannot be changed live anymore without a reboot. That's the point of the control. Rebooted the VM, confirmed the rules loaded with `auditctl -l`, and rescanned.

```bash
sudo oscap xccdf eval --profile xccdf_org.ssgproject.content_profile_stig --results full-scan-after2.xml --report full-scan-after2.html /usr/share/xml/scap/ssg/content/ssg-rhel10-ds.xml
grep -c 'result>pass' full-scan-after2.xml
grep -c 'result>fail' full-scan-after2.xml
```

![grep counts on the second remediation scan](screenshots/43-grep-counts-second-remediation-xml.png)

### 11. Final result: 95.37 percent

Hit "permission denied" on the new report file too (root owned again from the sudo scan), fixed the same way as before with chown, then pulled it to the host and opened it.

```bash
sudo chown abdsiddi:abdsiddi full-scan-after2.html full-scan-after2.xml
```

![chown on the second remediation reports](screenshots/44-chown-second-remediation-reports.png)

![Final HTML report, 95.37 percent](screenshots/45-oscap-html-report-final-95-percent.png)

**Final score: 95.37%**
438 passed, 19 failed
Remaining fails: 5 high (FIPS mode + partition encryption, both install time only), 9 medium, 5 low (mostly separate partition rules for /home, /tmp, /var, which also need a reinstall with custom partitioning to fix)

## Summary of results

| Stage | Score |
|---|---|
| Baseline, fresh install | 48% |
| After manual SSH fix + first Ansible run | 71.82% |
| After fixing audit rule loading + reboot | 95.37% |

## What I actually learned from this

- A scan is not the end of the job, reading the failures and knowing which ones are actually fixable versus which ones need a reinstall is the real skill
- Config file precedence in `/etc/ssh/sshd_config.d/` matters, the first file alphabetically wins, leftover files can silently override you
- `sudo` and normal user Ansible runs use completely different collection paths and environments, this cost me the most time in the whole lab
- Files created with `sudo` end up owned by root and will block you later when you try to read, scp, or edit them as your normal user
- Immutable audit configuration is a real STIG control, not a bug, once it's on you need a reboot to change audit rules again
- FIPS mode and disk encryption really do need to be decided at install time, no clean way around that after the fact

## Known limitations / what I would do differently next time

- Would enable FIPS mode and set up encrypted/separate partitions (/home, /tmp, /var, /var/log, /var/log/audit) during the RHEL install itself instead of trying to retrofit them after
- SSSD related checks (certmap, smartcard, trust anchor) need an actual identity provider like AD or FreeIPA to fully pass, out of scope for a single standalone VM
- Would install the ansible collections under root from the very start next time to skip a good chunk of the troubleshooting above
