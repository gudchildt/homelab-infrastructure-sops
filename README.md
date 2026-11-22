
# Homelab Infrastructure | Technical SOPs

A centralized collection of Standard Operating Procedures (SOPs) for deploying, configuring, securing, maintaining, and troubleshooting homelab infrastructure.

This repository documents practical systems administration workflows with an emphasis on reproducibility, operational security, and technical accuracy.

---

## Documentation Library

### Proxmox Virtual Environment (VE)

| Category | Standard Operating Procedure | Status |
|---|---|---|
| Networking | [Tailscale VPN Installation and Automatic Reconnection](proxmox/networking/Proxmox_Tailscale_Installation_Auto_Reconnect_SOP.md) | Published |

**Browse:** [Proxmox Networking Documentation](proxmox/networking/README.md)

---

## Technology Areas

Documentation is organized around the following infrastructure disciplines:

- **Virtualization:** Proxmox VE, virtual machines, Linux containers, and cluster administration.
- **Networking:** VPNs, DNS, routing, and network troubleshooting.
- **Storage:** NAS architecture, TrueNAS, ZFS, and backup strategies.
- **Cybersecurity:** Hardening, access control, monitoring, and security validation.
- **Automation:** Infrastructure scripts, service management, and operational workflows.
- **Hardware:** Server configuration, maintenance, and troubleshooting.

Additional procedures will be published as they are developed and reviewed.

---

## Documentation Standards

Each SOP aims to include:

1. Purpose, scope, and prerequisites.
2. Technical requirements and dependencies.
3. Step-by-step implementation instructions.
4. Command references and flag explanations.
5. Expected results and verification procedures.
6. Security considerations and operational safeguards.
7. Troubleshooting and recovery procedures.
8. Revision history and supporting references.

---

## Security and Privacy

This repository is intended for public technical documentation.

- Use generic hostnames, addresses, and environment placeholders.
- Never publish passwords, API keys, authentication tokens, or private keys.
- Remove sensitive information from screenshots and terminal outputs.
- Review procedures before applying them in production.
- Validate security-sensitive configurations against official documentation.

---

## Disclaimer

These procedures are provided for educational and technical documentation purposes. Commands and configurations should be reviewed and tested for compatibility with the target environment before deployment.

---

## Documentation Maintenance

Procedures may be updated as software versions, security recommendations, and operational requirements change.

Refer to each SOP's revision history for documented changes.
