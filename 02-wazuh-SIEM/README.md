# Projet 02 — Wazuh SIEM 4.11.2

Déploiement d'un SIEM open source sur infrastructure Proxmox, avec supervision en temps réel d'un serveur Debian 12 durci.

---

## Objectifs

- Déployer Wazuh (All-in-One) sur une VM Debian 12 hébergée sur Proxmox VE
- Connecter debian-lab comme agent supervisé
- Observer les alertes de sécurité, les vulnérabilités CVE et le score CIS Benchmark
- Comprendre la boucle détection → analyse → remédiation

---

## Infrastructure

```
Proxmox VE 8.4 (192.168.1.50)
└── wazuh-server (VM Debian 12)
    ├── IP : 192.168.1.25
    ├── RAM : 4 Go
    ├── Disque : 50 Go SSD (ZFS)
    └── Services : Wazuh Indexer + Manager + Dashboard

KVM sur Kali Linux (192.168.1.24)
└── debian-lab (VM KVM)
    ├── IP : 192.168.122.42
    └── Wazuh Agent v4.11.2 → connecté à 192.168.1.25
```

---

## Installation Wazuh (All-in-One)

### Prérequis VM

```bash
apt update && apt install -y sudo curl gnupg
```

### Téléchargement des scripts

```bash
curl -sO https://packages.wazuh.com/4.11/wazuh-install.sh
curl -sO https://packages.wazuh.com/4.11/config.yml
```

### Configuration `config.yml`

Toutes les IPs pointent vers la même machine (déploiement all-in-one) :

```yaml
nodes:
  indexer:
    - name: node-1
      ip: "192.168.1.25"
  server:
    - name: wazuh-1
      ip: "192.168.1.25"
  dashboard:
    - name: dashboard
      ip: "192.168.1.25"
```

### Déploiement étape par étape

```bash
# 1. Génération des certificats
bash wazuh-install.sh --generate-config-files

# 2. Wazuh Indexer (moteur de recherche — Elasticsearch fork)
bash wazuh-install.sh --wazuh-indexer node-1

# 3. Wazuh Manager (moteur d'analyse et de règles)
bash wazuh-install.sh --wazuh-server wazuh-1

# 4. Démarrage du cluster Indexer
bash wazuh-install.sh --start-cluster

# 5. Wazuh Dashboard (interface web)
bash wazuh-install.sh --wazuh-dashboard dashboard
```

### Récupération des credentials

```bash
tar -xvf wazuh-install-files.tar
# Le fichier wazuh-passwords.txt contient le mot de passe admin
```

### Accès dashboard

```
URL  : https://192.168.1.25
User : admin
Pass : (voir wazuh-passwords.txt)
```

---

## Installation Agent sur debian-lab

Depuis le dashboard Wazuh : **Agents → Deploy new agent**

Paramètres :
- Package : DEB amd64
- Manager IP : `192.168.1.25`

Commandes générées :

```bash
# Sur debian-lab
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | gpg --no-default-keyring \
  --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
chmod 644 /usr/share/keyrings/wazuh.gpg

echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] \
  https://packages.wazuh.com/4.x/apt/ stable main" \
  > /etc/apt/sources.list.d/wazuh.list

apt update && apt install wazuh-agent

# Configuration du manager
sed -i 's/MANAGER_IP/192.168.1.25/' /var/ossec/etc/ossec.conf

# Démarrage et activation
systemctl daemon-reload
systemctl enable wazuh-agent
systemctl start wazuh-agent
```

### Vérification

```bash
systemctl status wazuh-agent
# Active: active (running)

# Sur wazuh-server, vérifier la connexion de l'agent
/var/ossec/bin/agent_control -l
# Agent 001 — debian-lab — Active
```

---

## Ce que Wazuh supervise sur debian-lab

### Vulnerability Detection

Scan automatique des paquets installés comparés aux bases CVE (NVD, Debian Tracker).

| Sévérité | Nombre |
|----------|--------|
| Critical | 1      |
| High     | 77     |
| Medium   | 281    |
| Low      | 5      |

> Les vulnérabilités critiques et high sont à analyser en priorité. La plupart proviennent de paquets système non mis à jour.

### CIS Benchmark — Score 30%

Wazuh applique automatiquement le benchmark CIS Debian 12 (plus de 170 règles).

Catégories principales en échec :

| Catégorie | Exemples de checks failed |
|-----------|--------------------------|
| Filesystems | cramfs, jffs2, hfs, squashfs non désactivés |
| Partitions | Pas de partition séparée pour /var, /tmp, /home |
| Bootloader | Permissions /boot/grub/grub.cfg incorrectes |
| AppArmor | Non activé dans la config GRUB |
| AIDE | Non installé (File Integrity Monitoring) |
| Sudo | Pas de log file dédié |

> **Note** : certains checks (partitions séparées) nécessitent une réinstallation et sont ignorés dans ce contexte de lab. Le score CIS n'est pas l'objectif final — comprendre les catégories l'est.

### Corrections appliquées post-audit Wazuh

```bash
# Permissions bannières légales
chown root:root /etc/issue /etc/issue.net
chmod 644 /etc/issue /etc/issue.net

# Désactivation filesystems inutiles (complément au projet 01)
cat >> /etc/modprobe.d/blacklist-protocols.conf << EOF
install cramfs /bin/true
install freevxfs /bin/true
install jffs2 /bin/true
install hfs /bin/true
install hfsplus /bin/true
install squashfs /bin/true
install udf /bin/true
EOF

# Log sudo
echo 'Defaults logfile="/var/log/sudo.log"' | sudo tee -a /etc/sudoers

# AppArmor dans GRUB
# Éditer /etc/default/grub :
# GRUB_CMDLINE_LINUX="apparmor=1 security=apparmor"
update-grub
```

---

## Architecture des composants Wazuh

```
┌─────────────────────────────────────────┐
│            Wazuh Dashboard              │  ← Interface web (HTTPS 443)
│         (basé sur OpenSearch)           │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│            Wazuh Indexer                │  ← Stockage et recherche des logs
│         (OpenSearch Engine)             │
└──────────────────┬──────────────────────┘
                   │
┌──────────────────▼──────────────────────┐
│            Wazuh Manager                │  ← Moteur de règles et alertes
│         (analyse + corrélation)         │
└──────────────────┬──────────────────────┘
                   │ TCP 1514/1515
         ┌─────────▼─────────┐
         │   Wazuh Agent     │  ← Sur debian-lab (192.168.122.42)
         │   v4.11.2         │
         └───────────────────┘
```

---

## Prochaines étapes

- [ ] Configurer FIM (File Integrity Monitoring) sur debian-lab
- [ ] Analyser et investiguer la CVE critique détectée
- [ ] Connecter Kali Linux comme second agent
- [ ] Créer des règles d'alerte personnalisées
- [ ] Documenter une investigation complète (détection → analyse → remédiation)

---

## Références

- [Documentation officielle Wazuh](https://documentation.wazuh.com/current/)
- [CIS Debian Linux 12 Benchmark](https://www.cisecurity.org/benchmark/debian_linux)
- [Wazuh Vulnerability Detection](https://documentation.wazuh.com/current/user-manual/capabilities/vulnerability-detection/)
