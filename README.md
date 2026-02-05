# KubeSecureBox

Infrastructure Kubernetes securisee sur Raspberry Pi pour le deploiement de services IA et automatisation.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Reseau Tailscale                         │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │  Pi Master  │  │  Pi Worker  │  │  Pi Worker  │         │
│  │   (Pi 4)    │  │   (Pi 5)    │  │   (Pi 5)    │         │
│  │             │  │             │  │             │         │
│  │ K3s Server  │  │ K3s Agent   │  │ K3s Agent   │         │
│  │ NFS Server  │  │             │  │             │         │
│  │ HDD 4To     │  │             │  │             │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Prerequis

### Sur votre PC (machine de controle)

1. **WSL2** (Windows) ou Linux/macOS
2. **Ansible** installe :
   ```bash
   sudo apt update && sudo apt install ansible python3-pip -y
   pip install passlib
   ```
3. **Cle SSH Ed25519** :
   ```bash
   ssh-keygen -t ed25519 -C "name"
   ```

### Comptes a creer

| Service   | URL                          | Usage                     |
|-----------|------------------------------|---------------------------|
| Tailscale | https://tailscale.com        | VPN prive                 |
| Brevo     | https://brevo.com            | Alertes email             |
| GitHub    | https://github.com           | Versionning config        |

## Installation

### 1. Preparer les Raspberry Pi

1. Flasher **Raspberry Pi OS Lite 64-bit** (Bookworm)
2. Configurer via Raspberry Pi Imager :
   - Hostname: `name_master`, `name_worker`, `name_worker`
   - Utilisateur: `name_user`
   - SSH active avec mot de passe (temporaire)
3. Connecter en Ethernet et noter les IPs

### 2. Configurer l'inventaire

Editer `inventory/hosts.ini` avec vos IPs :

```ini
[masters]
pi-master ansible_host=192.x.x.x

[workers]
pi-worker-01 ansible_host=192.x.x.x
pi-worker-02 ansible_host=192.x.x.x
```

### 3. Configurer les secrets

```bash
# Copier l'exemple
cp inventory/group_vars/vault.yml.example inventory/group_vars/vault.yml

# Editer avec vos valeurs
nano inventory/group_vars/vault.yml

# Chiffrer avec Ansible Vault
ansible-vault encrypt inventory/group_vars/vault.yml
```

```bash
# Copier l'exemple
cp inventory/group_vars/all.yml.example inventory/group_vars/all.yml

# Editer avec vos valeurs
nano inventory/group_vars/all.yml

```

```bash
# Copier l'exemple
cp inventory/hosts.ini.example inventory/hosts.ini

# Editer avec vos valeurs
nano inventory/hosts.ini

```

### 4. Executer les playbooks

```bash
# Phase 0 - Bootstrap (avec mot de passe SSH)
ansible-playbook -i inventory/hosts.ini playbooks/00-bootstrap.yml --ask-pass

# Phase 1 - Securisation
ansible-playbook -i inventory/hosts.ini playbooks/01-security.yml --ask-vault-pass

# Phase 2 - Stockage
ansible-playbook -i inventory/hosts.ini playbooks/02-storage.yml --ask-vault-pass

# Phase 3 - Kubernetes
ansible-playbook -i inventory/hosts.ini playbooks/03-k3s.yml --ask-vault-pass
```

## Apres l'installation

### Connexion SSH

```bash
# Via IP locale (apres securisation)
ssh -p {port} name_user@192.x.x.x

# Via Tailscale (recommande)
ssh -p {port} name_user@100.x.x.x
```

### Utiliser kubectl

```bash
scp -P {port} name_user@100.x.x.x:/home/name_user/.kube/config ~/KubeSecureBox/kubeconfig-{name_master}

# Copier le kubeconfig genere
echo 'export KUBECONFIG=~/KubeSecureBox/kubeconfig-{name_master}' >> ~/.bashrc
source ~/.bashrc

# Verifier le cluster
kubectl get nodes
```

## Structure du projet

```
kubesecurebox/
├── ansible.cfg              # Configuration Ansible
├── inventory/
│   ├── hosts.ini            # Liste des serveurs
│   └── group_vars/
│       ├── all.yml          # Variables globales
│       ├── masters.yml      # Config masters
│       ├── workers.yml      # Config workers
│       └── vault.yml        # Secrets (chiffre)
├── roles/
│   ├── base/                # Mise a jour systeme
│   ├── security/            # SSH, UFW, Fail2Ban
│   ├── tailscale/           # VPN
│   ├── kubernetes-prep/     # Preparation K8s
│   ├── storage/             # HDD, NFS
│   ├── monitoring/          # Node Exporter
│   ├── audit/               # Lynis
│   ├── alerting/            # Discord, Email
│   └── k3s/                 # Kubernetes
└── playbooks/
    ├── 00-bootstrap.yml     # Initialisation
    ├── 01-security.yml      # Securisation
    ├── 02-storage.yml       # Stockage
    ├── 03-k3s.yml           # Kubernetes
    └── site.yml             # Tout en un
```

## Securite

### Mesures implementees

- SSH sur port 2222, cle uniquement
- Pare-feu UFW (deny all, allow Tailscale)
- Fail2Ban (ban 24h apres 3 tentatives)
- Mises a jour automatiques
- Audit Lynis hebdomadaire
- Bluetooth/WiFi desactives

### Score Lynis attendu

Avec cette configuration, vous devriez obtenir un score entre **70 et 85/100**.

## Alertes

| Evenement            | Discord | Email |
|----------------------|---------|-------|
| Demarrage systeme    | ✓       |       |
| Connexion SSH        | ✓       |       |
| IP bannie            | ✓       | ✓     |
| Score Lynis < 70     | ✓       | ✓     |
| Alerte HDD           | ✓       | ✓     |
| Service K3s down     | ✓       | ✓     |

## License

MIT
