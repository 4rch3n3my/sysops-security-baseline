# 🛡️ Homelab Sécurité — Durcissement Debian 12

Projet de lab personnel orienté **Administration Système Linux & Sécurité Défensive**.  
Objectif : déployer une VM Debian 12 from scratch et la durcir selon les bonnes pratiques, en utilisant Lynis comme outil de benchmark.

---

## 🏗️ Architecture du lab

```
Hôte : Kali Linux (bare metal)
Virtualisation : KVM / QEMU / virt-manager

VM : debian-lab
  └── Debian 12 (Bookworm) — serveur headless
  └── Accès : SSH uniquement depuis l'hôte Kali
  └── Réseau : NAT (virbr0)
```

---

## 📊 Résultats Lynis

| Étape | Score | Tests effectués |
|---|---|---|
| Avant durcissement | 62 / 100 | 246 |
| Après durcissement | 76 / 100 | 252 |

**Progression : +14 points**

---

## ✅ Actions de durcissement réalisées

### 1. SSH — `/etc/ssh/sshd_config`

| Paramètre | Avant | Après | Raison |
|---|---|---|---|
| `PermitRootLogin` | yes | no | Empêche l'accès root direct |
| `Port` | 22 | 2222 | Réduit le bruit des bots |
| `MaxAuthTries` | 6 | 3 | Limite le bruteforce |
| `MaxSessions` | 10 | 2 | Réduit la surface d'abus |
| `X11Forwarding` | yes | no | Inutile sur serveur headless |
| `AllowTcpForwarding` | yes | no | Évite le tunnel détourné |
| `AllowAgentForwarding` | yes | no | Inutile sur ce serveur |
| `Compression` | yes | no | Vulnérable à CRIME/BREACH |
| `TCPKeepAlive` | yes | no | Remplacé par ClientAlive chiffré |
| `LogLevel` | INFO | VERBOSE | Plus de détails pour investigation |
| `Banner` | - | /etc/issue.net | Bannière légale avant connexion |

---

### 2. Fail2ban — `/etc/fail2ban/jail.local`

Protection automatique contre le bruteforce SSH.

```ini
[sshd]
enabled  = true
port     = 2222
backend  = systemd
maxretry = 3
bantime  = 1h
findtime = 10m
```

> `backend = systemd` : nécessaire sur Debian 12 car les logs SSH sont gérés par journald, pas dans un fichier classique.

---

### 3. Protocoles réseau inutiles — `/etc/modprobe.d/blacklist-protocols.conf`

Désactivation de protocoles non utilisés pour réduire la surface d'attaque.

```bash
blacklist dccp   # Protocole transport alternatif, jamais utilisé
blacklist sctp   # Utilisé en VoIP, inutile ici
blacklist rds    # Protocole Oracle clustering
blacklist tipc   # Protocole clustering Linux
```

---

### 4. Durcissement kernel — `/etc/sysctl.d/99-hardening.conf`

```bash
net.ipv4.ip_forward = 0                    # Ce serveur ne route pas
net.ipv4.conf.all.accept_redirects = 0     # Protège contre le route poisoning
net.ipv4.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv4.icmp_echo_ignore_broadcasts = 1   # Protège contre les attaques Smurf
net.ipv4.tcp_syncookies = 1                # Protège contre le SYN flood
net.ipv4.conf.all.accept_source_route = 0  # Désactive source routing
fs.suid_dumpable = 0                       # Désactive les core dumps SUID
kernel.kptr_restrict = 2                   # Cache les pointeurs kernel dans /proc
kernel.dmesg_restrict = 1                  # Restreint l'accès aux logs kernel
```

---

### 5. Politique de mots de passe

**`/etc/login.defs`**
```bash
PASS_MAX_DAYS   90     # Expiration après 90 jours
PASS_MIN_DAYS   7      # Délai minimum avant changement
PASS_WARN_AGE   14     # Avertissement 14 jours avant expiration
ENCRYPT_METHOD  SHA512 # Algorithme de hachage
UMASK           027    # Permissions restrictives par défaut
```

**`/etc/pam.d/common-password`** — complexité via pam_pwquality
```bash
password requisite pam_pwquality.so retry=3 minlen=12 difok=3 \
  ucredit=-1 lcredit=-1 dcredit=-1 ocredit=-1 reject_username
```

| Option | Signification |
|---|---|
| `minlen=12` | 12 caractères minimum |
| `ucredit=-1` | Au moins 1 majuscule |
| `lcredit=-1` | Au moins 1 minuscule |
| `dcredit=-1` | Au moins 1 chiffre |
| `ocredit=-1` | Au moins 1 caractère spécial |
| `reject_username` | Interdit d'inclure son nom dans le mdp |

---

### 6. Auditd — surveillance des fichiers sensibles

**`/etc/audit/rules.d/hardening.rules`**

```bash
-w /etc/passwd      -p wa -k identity
-w /etc/shadow      -p wa -k identity
-w /etc/ssh/sshd_config -p wa -k sshd_config
-w /etc/sudoers     -p wa -k sudoers
-w /usr/bin/sudo    -p x  -k sudo_usage
```

Commandes utiles pour investiguer :
```bash
ausearch -k identity          # Modifications des comptes
ausearch -k sshd_config       # Modifications SSH
ausearch -k sudo_usage        # Usage de sudo
aureport --summary            # Rapport global
aureport --auth               # Rapport authentifications
```

---

### 7. rkhunter — scanner rootkit

```bash
rkhunter --propupd            # Initialisation de la base de référence
rkhunter --check --skip-keypress  # Scan complet
```

**Résultat : 498 rootkits vérifiés — 0 détecté**

---

### 8. Firewall nftables — `/etc/nftables.conf`

Politique restrictive : tout est bloqué en entrée sauf ce qui est explicitement autorisé.

```bash
table inet filter {

    chain input {
        type filter hook input priority 0; policy drop;

        # Loopback — communications internes
        iif "lo" accept

        # Connexions déjà établies
        ct state established,related accept

        # SSH uniquement
        tcp dport 2222 accept

        # ICMP — ping autorisé en lab
        icmp type echo-request accept
        icmpv6 type echo-request accept
    }

    chain forward {
        type filter hook forward priority 0; policy drop;
    }

    chain output {
        type filter hook output priority 0; policy accept;
    }
}
```

| Chaîne | Politique | Raison |
|---|---|---|
| `input` | DROP par défaut | Rien n'entre sauf ce qui est autorisé |
| `forward` | DROP par défaut | Ce serveur ne route pas |
| `output` | ACCEPT par défaut | La VM peut contacter l'extérieur (apt, etc.) |

> Choix de `inet` plutôt que `ip` : couvre IPv4 **et** IPv6 en un seul ruleset.

---

### 9. Bannières légales

**`/etc/issue`** et **`/etc/issue.net`**
```
***************************************************************************
SYSTEME PRIVE - ACCES NON AUTORISE INTERDIT
Toute connexion est surveillée et enregistrée.
Les contrevenants seront poursuivis.
***************************************************************************
```

---

## 🔧 Outils utilisés

| Outil | Rôle |
|---|---|
| KVM / virt-manager | Virtualisation de la VM |
| Lynis | Audit et benchmark sécurité |
| Fail2ban | Protection bruteforce |
| Auditd | Surveillance système |
| rkhunter | Détection rootkits |
| pam_pwquality | Politique mots de passe |
| nftables | Firewall — politique restrictive entrante |

---

## 📚 Prochaines étapes

- [ ] Déploiement Wazuh (SIEM) pour centraliser les logs
- [ ] Montage d'un Active Directory Windows Server en lab
- [ ] Attaque AD depuis Kali + remédiation
- [ ] Automatisation du durcissement avec Ansible

---

## 👤 Auteur

**4rch3n3my** — Administrateur Systèmes & Réseaux  
Spécialisation : Linux · Sécurité Défensive  
[LinkedIn](https://www.linkedin.com/in/p1ch0tth0m45)
