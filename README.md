# 🛡️ Homelab — SysAdmin Linux & Sécurité Défensive

Lab personnel de montée en compétences en **Administration Système Linux** et **Sécurité Défensive**.  
Chaque projet est documenté de façon à reproduire l'environnement et comprendre les choix techniques.

---

## 🏗️ Infrastructure

```
Kali Linux (bare metal)         → 192.168.1.24  — poste principal, pentest
  └── debian-lab (VM KVM)       → 192.168.1.x   — serveur de lab

Serveur Proxmox (bare metal)    → 192.168.1.x   — hyperviseur dédié lab
  ├── wazuh-vm                  → SIEM central
  ├── windows-server            → lab Active Directory (projet 03)
  └── futures VMs...
```

---

## 📁 Projets

| # | Projet | Description | Status |
|---|---|---|---|
| 00 | [Proxmox Setup](./00-Proxmox-Setup/README.md) | Installation et configuration d'un hyperviseur Proxmox VE 8.4 | 🔜 En cours |
| 01 | [Debian Hardening](./01-debian-hardening/README.md) | Durcissement d'un serveur Debian 12 from scratch | ✅ Terminé |
| 02 | [wazuh SIEM](./02-wazuh-SIEM/README.md) | Déploiement d'un SIEM open source pour centraliser les logs | 🔜 Planifié |
| 03 | Active Directory Lab | Montage d'un AD Windows Server, attaque depuis Kali, remédiation | 🔜 Planifié |
| 04 | Ansible Hardening | Automatisation du durcissement via Ansible | 🔜 Planifié |

---

## 🎯 Objectifs

- Maîtriser l'administration système Linux en environnement sécurisé
- Comprendre les vecteurs d'attaque pour mieux les défendre
- Documenter chaque étape pour reproduire et améliorer
- Progresser vers un rôle d'**Ingénieur Systèmes Linux** avec une spécialisation **Sécurité**

---

## 🔧 Stack technique

```
OS          : Debian 12, RHEL, Kali Linux
Hyperviseur : Proxmox VE 8.4, KVM/QEMU
Sécu        : Lynis, Fail2ban, Auditd, rkhunter, Wazuh (soon)
Réseau      : nftables, SSH, OpenVPN, WireGuard
Supervision : Nagios, Zabbix (soon)
Automation  : Bash, Ansible (soon)
```

---

## 👤 Auteur

**4rch3n3my** — Administrateur Systèmes & Réseaux  
Spécialisation : Linux · Sécurité Défensive · Toulouse, FR  
[LinkedIn](https://www.linkedin.com/in/p1ch0tth0m45)
