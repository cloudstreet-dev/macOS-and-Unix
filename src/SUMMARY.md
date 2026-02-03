# Summary

[Introduction](./introduction.md)

---

# Part I: Foundations & History

- [The Unix Heritage of macOS](./part1-foundations/README.md)
  - [Darwin: The Open Source Core](./part1-foundations/darwin.md)
  - [From NeXTSTEP to macOS](./part1-foundations/nextstep-history.md)
  - [The XNU Kernel Architecture](./part1-foundations/xnu-kernel.md)
  - [macOS vs Linux vs BSD: Key Differences](./part1-foundations/macos-vs-others.md)
  - [POSIX Compliance and Standards](./part1-foundations/posix-compliance.md)

---

# Part II: Filesystem & Storage

- [Understanding macOS Filesystems](./part2-filesystem/README.md)
  - [APFS: Apple's Modern Filesystem](./part2-filesystem/apfs.md)
  - [HFS+ Legacy and Migration](./part2-filesystem/hfs-plus.md)
  - [Case Sensitivity: Options and Implications](./part2-filesystem/case-sensitivity.md)
  - [Extended Attributes and Resource Forks](./part2-filesystem/extended-attributes.md)
  - [The Metadata Files: .DS_Store and ._AppleDouble](./part2-filesystem/metadata-files.md)
  - [Filesystem Hierarchy: Where macOS Diverges](./part2-filesystem/hierarchy.md)
  - [Disk Management from the Command Line](./part2-filesystem/disk-management.md)
  - [Volumes, Containers, and Snapshots](./part2-filesystem/volumes-containers.md)

---

# Part III: Shell & Terminal

- [Mastering the macOS Shell](./part3-shell/README.md)
  - [The Great Shell Transition: Bash to Zsh](./part3-shell/bash-to-zsh.md)
  - [Terminal.app Deep Dive](./part3-shell/terminal-app.md)
  - [iTerm2 and Alternative Terminals](./part3-shell/iterm2-alternatives.md)
  - [Shell Configuration on macOS](./part3-shell/shell-configuration.md)
  - [Environment Variables and PATH](./part3-shell/environment-variables.md)
  - [macOS-Specific Shell Features](./part3-shell/macos-shell-features.md)
  - [Shell Integration with macOS Services](./part3-shell/shell-integration.md)

---

# Part IV: Package Management

- [Package Management on macOS](./part4-packages/README.md)
  - [Homebrew: Architecture and Internals](./part4-packages/homebrew-architecture.md)
  - [Homebrew Best Practices](./part4-packages/homebrew-best-practices.md)
  - [MacPorts: An Alternative Approach](./part4-packages/macports.md)
  - [Native Installers: pkg and dmg](./part4-packages/native-installers.md)
  - [Managing Python Environments](./part4-packages/python-environments.md)
  - [Managing Ruby Environments](./part4-packages/ruby-environments.md)
  - [Managing Node.js Environments](./part4-packages/node-environments.md)
  - [Comparing Package Managers](./part4-packages/comparison.md)

---

# Part V: System Services & Processes

- [System Services on macOS](./part5-services/README.md)
  - [launchd: The Heart of macOS](./part5-services/launchd-overview.md)
  - [Understanding Property Lists (plists)](./part5-services/property-lists.md)
  - [Creating Launch Agents and Daemons](./part5-services/agents-and-daemons.md)
  - [launchctl Command Reference](./part5-services/launchctl.md)
  - [Process Management: CLI vs GUI](./part5-services/process-management.md)
  - [Background Task Scheduling](./part5-services/task-scheduling.md)
  - [XPC Services and IPC](./part5-services/xpc-services.md)
  - [Comparing launchd to systemd and init](./part5-services/launchd-vs-others.md)

---

# Part VI: Unix Commands & Tools

- [Unix Commands on macOS](./part6-commands/README.md)
  - [BSD vs GNU: The Command Divide](./part6-commands/bsd-vs-gnu.md)
  - [Essential macOS-Specific Commands](./part6-commands/macos-commands.md)
  - [Installing and Using GNU Coreutils](./part6-commands/gnu-coreutils.md)
  - [Text Processing: sed, awk, and grep](./part6-commands/text-processing.md)
  - [File Operations and Manipulation](./part6-commands/file-operations.md)
  - [Networking Commands](./part6-commands/networking-commands.md)
  - [System Information Commands](./part6-commands/system-info.md)
  - [Writing Portable Shell Scripts](./part6-commands/portable-scripts.md)

---

# Part VII: Development Environment

- [Development on macOS](./part7-development/README.md)
  - [Xcode Command Line Tools](./part7-development/xcode-cli-tools.md)
  - [The Clang/LLVM Toolchain](./part7-development/clang-llvm.md)
  - [Compiling Software from Source](./part7-development/compiling-source.md)
  - [Library Paths and Linking](./part7-development/library-linking.md)
  - [Frameworks vs Unix Libraries](./part7-development/frameworks-vs-libraries.md)
  - [Debugging with LLDB](./part7-development/lldb-debugging.md)
  - [Performance Profiling Tools](./part7-development/profiling-tools.md)
  - [Universal Binaries and Architecture](./part7-development/universal-binaries.md)

---

# Part VIII: System Administration

- [Administering macOS](./part8-administration/README.md)
  - [User and Group Management](./part8-administration/users-groups.md)
  - [The Permissions Model: Unix Meets ACLs](./part8-administration/permissions-acls.md)
  - [System Integrity Protection (SIP)](./part8-administration/sip.md)
  - [Configuring the pf Firewall](./part8-administration/pf-firewall.md)
  - [Unified Logging System](./part8-administration/unified-logging.md)
  - [System Configuration via defaults](./part8-administration/defaults-command.md)
  - [Startup and Boot Process](./part8-administration/startup-boot.md)
  - [Recovery Mode and Troubleshooting](./part8-administration/recovery-mode.md)

---

# Part IX: Networking

- [Networking on macOS](./part9-networking/README.md)
  - [Network Configuration from CLI](./part9-networking/network-configuration.md)
  - [Understanding Network Services](./part9-networking/network-services.md)
  - [Bonjour and mDNS](./part9-networking/bonjour-mdns.md)
  - [VPN Configuration](./part9-networking/vpn-configuration.md)
  - [Sharing Services via Command Line](./part9-networking/sharing-services.md)
  - [Network Diagnostics and Troubleshooting](./part9-networking/diagnostics.md)
  - [Wi-Fi Management from Terminal](./part9-networking/wifi-management.md)

---

# Part X: Security Model

- [Security on macOS](./part10-security/README.md)
  - [Gatekeeper and Code Signing](./part10-security/gatekeeper-signing.md)
  - [Notarization Requirements](./part10-security/notarization.md)
  - [App Sandboxing](./part10-security/sandboxing.md)
  - [Keychain Services from Terminal](./part10-security/keychain-cli.md)
  - [FileVault and Disk Encryption](./part10-security/filevault.md)
  - [Privacy Controls and TCC Database](./part10-security/tcc-privacy.md)
  - [Secure Boot and T2/Apple Silicon](./part10-security/secure-boot.md)
  - [Security Best Practices](./part10-security/best-practices.md)

---

# Part XI: Interoperability

- [Cross-Platform Interoperability](./part11-interoperability/README.md)
  - [File Sharing with Linux and BSD](./part11-interoperability/file-sharing.md)
  - [Running Linux Containers on macOS](./part11-interoperability/linux-containers.md)
  - [Virtual Machines on macOS](./part11-interoperability/virtual-machines.md)
  - [Cross-Platform Script Compatibility](./part11-interoperability/script-compatibility.md)
  - [SSH Configuration and Usage](./part11-interoperability/ssh.md)
  - [Remote Access: VNC and ARD](./part11-interoperability/remote-access.md)
  - [Working with NFS and SMB](./part11-interoperability/nfs-smb.md)

---

# Part XII: Performance & Optimization

- [Performance and Optimization](./part12-performance/README.md)
  - [Activity Monitor: Beyond the GUI](./part12-performance/activity-monitor.md)
  - [Command-Line Performance Tools](./part12-performance/cli-performance-tools.md)
  - [Intel vs Apple Silicon Considerations](./part12-performance/architecture-considerations.md)
  - [Memory Management Deep Dive](./part12-performance/memory-management.md)
  - [Disk I/O Optimization](./part12-performance/disk-io.md)
  - [Power Management and Battery](./part12-performance/power-management.md)
  - [Troubleshooting Performance Issues](./part12-performance/troubleshooting.md)

---

# Appendices

- [Appendix A: Command Reference](./appendices/command-reference.md)
- [Appendix B: Configuration File Locations](./appendices/config-locations.md)
- [Appendix C: Keyboard Shortcuts](./appendices/keyboard-shortcuts.md)
- [Appendix D: Glossary](./appendices/glossary.md)
- [Appendix E: Additional Resources](./appendices/resources.md)
