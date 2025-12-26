# 🔷 TP4 – Hyperviseur Imbriqué - Xen & Xen Orchestra

**Objectif du TP :**  
Installer XCP-ng (Xen) en tant que VM dans KVM et le gérer via Xen Orchestra.  
À la fin du TP, vous aurez :

- ✅ Compris la **virtualisation imbriquée** (Nested Virtualization)
- ✅ Installé **XCP-ng** (hyperviseur Xen) dans une VM KVM
- ✅ Déployé **Xen Orchestra** (interface Web de gestion)
- ✅ Créé une **VM de niveau 3** (VM dans Xen dans KVM)
- ✅ Maîtrisé la gestion via **interface Web**
- ✅ Comparé les performances **KVM vs Xen**

> 💡 **Pourquoi Xen ?**  
> Xen est un hyperviseur Type-1 bare-metal utilisé par AWS, Alibaba Cloud, et de nombreux fournisseurs cloud. XCP-ng est la version open-source de Citrix Hypervisor, offrant une alternative gratuite et puissante à VMware ESXi.

---

## 📋 Prérequis

- ✅ TP1, TP2, et TP3 terminés
- ✅ Virtualisation imbriquée activée (`cat /sys/module/kvm_amd/parameters/nested` = Y)
- ✅ Au moins 60GB d'espace disque libre
- ✅ 16GB de RAM disponible pour la VM XCP-ng

**Vérification rapide :**
```bash
# Vérifier la virtualisation imbriquée
cat /sys/module/kvm_amd/parameters/nested

# Vérifier les ressources
free -h
df -h /var/lib/libvirt/images
```

---

## 🏗️ Architecture de la Virtualisation Imbriquée

### Les 3 Niveaux de Virtualisation

```
┌─────────────────────────────────────────────────────────────┐
│  Niveau 3 (L3) - VM dans Xen                               │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  VM Ubuntu Test (dans XCP-ng)                        │  │
│  │  - OS : Ubuntu 24.04                                  │  │
│  │  - RAM : 1GB                                          │  │
│  │  - vCPU : 1                                           │  │
│  └───────────────────────────────────────────────────────┘  │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────┴──────────────────────────────────┐
│  Niveau 2 (L2) - VM XCP-ng (Hyperviseur Xen dans KVM)     │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  XCP-ng 8.3 (Xen Hypervisor)                         │  │
│  │  - RAM : 16GB                                         │  │
│  │  - vCPU : 4 (avec nested activé)                     │  │
│  │  - Disque : 50GB                                      │  │
│  │  - Réseau : Bridge vers virbr0                       │  │
│  └───────────────────────────────────────────────────────┘  │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────┴──────────────────────────────────┐
│  Niveau 1 (L1) - Hôte KVM (Oracle Cloud)                  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Ubuntu 24.04 Desktop + KVM                          │  │
│  │  - RAM : 48GB                                         │  │
│  │  - CPU : 4 OCPUs AMD                                  │  │
│  │  - Nested Virtualization : Activée                   │  │
│  └───────────────────────────────────────────────────────┘  │
└──────────────────────────┬──────────────────────────────────┘
                           │
                    Matériel Physique
```

### Pourquoi 3 Niveaux ?

| Niveau | Rôle | Technologie |
|--------|------|-------------|
| **L1** | Hôte physique | Ubuntu + KVM |
| **L2** | Hyperviseur imbriqué | XCP-ng (Xen) |
| **L3** | VM finale | Ubuntu/Windows/etc. |

**Cas d'usage réels :**
- 🧪 **Formation :** Apprendre plusieurs hyperviseurs sans matériel supplémentaire
- 🔬 **Tests :** Tester des configurations d'hyperviseurs
- ☁️ **Cloud :** Certains fournisseurs cloud utilisent la virtualisation imbriquée

---

## 💿 Étape 1 – Télécharger XCP-ng

### 1.1 – Télécharger l'ISO XCP-ng

```bash
# Créer un répertoire pour XCP-ng
mkdir -p ~/iso/xcp-ng

# Télécharger XCP-ng 8.3 (dernière version stable)
cd ~/iso/xcp-ng
wget https://updates.xcp-ng.org/8/8.3/xcp-ng-8.3.0.iso

# Vérifier le téléchargement
ls -lh xcp-ng-8.3.0.iso
```

**Taille attendue :** ~1.2GB

**Qu'est-ce que XCP-ng ?**
- Fork open-source de Citrix Hypervisor
- Basé sur Xen (hyperviseur Type-1)
- Compatible avec Xen Orchestra
- Gratuit et sans limitations

---

## 🖥️ Étape 2 – Créer la VM pour XCP-ng

### 2.1 – Créer le Disque Virtuel

```bash
# Créer un répertoire pour XCP-ng
sudo mkdir -p /var/lib/libvirt/images/xcp-ng

# Créer un disque de 50GB (format raw pour de meilleures performances)
sudo qemu-img create -f raw \
    /var/lib/libvirt/images/xcp-ng/xcp-ng-disk.img 50G

# Vérifier
sudo qemu-img info /var/lib/libvirt/images/xcp-ng/xcp-ng-disk.img
```

**Pourquoi format raw ?**
- Meilleures performances pour un hyperviseur
- Pas besoin de snapshots (XCP-ng gère ses propres snapshots)

### 2.2 – Créer la VM avec virt-install

```bash
# Créer la VM XCP-ng avec nested virtualization
sudo virt-install \
    --name xcp-ng \
    --ram 16384 \
    --vcpus 4 \
    --cpu host-passthrough,cache.mode=passthrough \
    --disk path=/var/lib/libvirt/images/xcp-ng/xcp-ng-disk.img,bus=sata,cache=none \
    --network network=default,model=e1000 \
    --graphics vnc,listen=0.0.0.0,port=5901 \
    --cdrom ~/iso/xcp-ng/xcp-ng-8.3.0.iso \
    --os-variant centos7.0 \
    --boot cdrom,hd \
    --noautoconsole
```

**Explication des options importantes :**

| Option | Description |
|--------|-------------|
| `--cpu host-passthrough` | Expose toutes les fonctionnalités CPU à la VM (nécessaire pour nested) |
| `cache.mode=passthrough` | Désactive le cache pour de meilleures performances |
| `--network model=e1000` | Carte réseau Intel E1000 (compatible XCP-ng) |
| `--graphics vnc,port=5901` | VNC sur port 5901 pour l'installation |
| `--os-variant centos7.0` | XCP-ng est basé sur CentOS |

### 2.3 – Se Connecter à la VM pour l'Installation

```bash
# Ouvrir virt-manager
virt-manager &

# Ou se connecter via VNC depuis Windows :
# Adresse : VOTRE_IP_PUBLIQUE:5901
```

---

## 🔧 Étape 3 – Installer XCP-ng

### 3.1 – Processus d'Installation (via virt-manager ou VNC)

1. **Écran de Bienvenue**
   - Sélectionner : **"Install XCP-ng"**
   - Appuyer sur **Entrée**

2. **Clavier**
   - Sélectionner : **French** (ou votre clavier)
   - Appuyer sur **OK**

3. **Accepter l'EULA**
   - Lire et accepter la licence
   - Sélectionner **Accept EULA**

4. **Disque d'Installation**
   - Sélectionner le disque : **sda (50GB)**
   - Choisir : **Enable thin provisioning** (optionnel)
   - Appuyer sur **OK**

5. **Source d'Installation**
   - Sélectionner : **Local media**
   - Appuyer sur **OK**

6. **Vérification**
   - Choisir : **Skip verification** (pour gagner du temps)

7. **Mot de Passe Root**
   - Entrer un mot de passe : `password123` (pour le lab)
   - Confirmer
   - Appuyer sur **OK**

8. **Configuration Réseau**
   - **Management Interface :** eth0
   - **Configuration :** DHCP (automatique)
   - Appuyer sur **OK**

9. **Hostname et DNS**
   - **Hostname :** `xcp-ng.local`
   - **DNS :** Automatique (DHCP)
   - Appuyer sur **OK**

10. **Fuseau Horaire**
    - Sélectionner : **Europe/Paris** (ou votre fuseau)
    - Appuyer sur **OK**

11. **NTP (Synchronisation de l'heure)**
    - Laisser par défaut ou ajouter : `pool.ntp.org`
    - Appuyer sur **OK**

12. **Installation**
    - Vérifier le résumé
    - Sélectionner **Install XCP-ng**
    - Appuyer sur **OK**

⏱️ **L'installation prend environ 10-15 minutes.**

### 3.2 – Premier Démarrage

Après l'installation :

1. La VM redémarre automatiquement
2. **Éjecter le CDROM :**
   - Dans virt-manager : Détails → SATA CDROM → Déconnecter
3. Redémarrer la VM si nécessaire

**Écran de Connexion XCP-ng :**
```
XCP-ng 8.3.0

xcp-ng login: root
Password: password123
```

### 3.3 – Vérifier l'Installation

```bash
# Se connecter en root
# Password: password123

# Vérifier la version de XCP-ng
cat /etc/xcp-ng-release

# Vérifier l'adresse IP
ip addr show eth0

# Vérifier que Xen fonctionne
xl info

# Lister les VMs (vide pour l'instant)
xl list
```

**Résultat attendu de `xl info` :**
```
host                   : xcp-ng
release                : 4.19.x
version                : #1 SMP
machine                : x86_64
nr_cpus                : 4
max_cpu_id             : 3
nr_nodes               : 1
total_memory           : 16384
free_memory            : 15000
```

🎉 **XCP-ng est installé et fonctionne !**

---

## 🌐 Étape 4 – Déployer Xen Orchestra (Interface Web)

### 4.1 – Qu'est-ce que Xen Orchestra ?

**Xen Orchestra (XO)** est l'interface Web de gestion pour XCP-ng, équivalent à :
- vCenter pour VMware
- Proxmox Web UI pour Proxmox
- Cockpit pour KVM

**Fonctionnalités :**
- Gestion graphique des VMs
- Monitoring en temps réel
- Backups automatiques
- Gestion du stockage et des réseaux

### 4.2 – Installer Xen Orchestra depuis les Sources

XCP-ng fournit un script d'installation automatique pour Xen Orchestra.

**Option 1 : Installer XO dans une VM Ubuntu séparée (Recommandé)**

```bash
# Sur l'hôte L1 (Ubuntu Desktop), créer une VM pour Xen Orchestra
# Télécharger Ubuntu Server Cloud Image (si pas déjà fait au TP3)
cd ~/cloud-images
wget -nc https://cloud-images.ubuntu.com/releases/24.04/release/ubuntu-24.04-server-cloudimg-amd64.img

# Créer le répertoire pour XO
sudo mkdir -p /var/lib/libvirt/images/xen-orchestra

# Copier l'image cloud
sudo cp ~/cloud-images/ubuntu-24.04-server-cloudimg-amd64.img \
    /var/lib/libvirt/images/xen-orchestra/xo-disk.qcow2

# Redimensionner à 30GB
sudo qemu-img resize /var/lib/libvirt/images/xen-orchestra/xo-disk.qcow2 30G

# Créer le fichier cloud-init pour XO
cat > ~/xo-cloud-init.yaml << 'EOF'
#cloud-config
hostname: xen-orchestra
fqdn: xen-orchestra.local

users:
  - name: ubuntu
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    lock_passwd: false

chpasswd:
  list: |
    ubuntu:password123
  expire: False

packages:
  - git
  - curl
  - wget
  - build-essential
  - redis-server
  - libpng-dev
  - python3-minimal
  - libvhdi-utils
  - lvm2
  - nfs-common

runcmd:
  # Installer Node.js 20.x
  - curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
  - apt-get install -y nodejs
  # Installer Yarn
  - npm install -g yarn
  # Cloner Xen Orchestra
  - cd /opt && git clone https://github.com/vatesfr/xen-orchestra
  - cd /opt/xen-orchestra && yarn
  - cd /opt/xen-orchestra/packages/xo-server && cp sample.config.toml .xo-server.toml
  # Créer un service systemd
  - |
    cat > /etc/systemd/system/xo-server.service << 'SERVICE_EOF'
    [Unit]
    Description=Xen Orchestra Server
    After=network.target

    [Service]
    Type=simple
    User=root
    WorkingDirectory=/opt/xen-orchestra/packages/xo-server
    ExecStart=/usr/bin/node ./bin/xo-server
    Restart=always

    [Install]
    WantedBy=multi-user.target
    SERVICE_EOF
  - systemctl daemon-reload
  - systemctl enable xo-server
  - systemctl start xo-server

package_update: true
package_upgrade: true
EOF

# Générer l'ISO cloud-init
cloud-localds ~/xo-cloud-init.iso ~/xo-cloud-init.yaml
sudo mv ~/xo-cloud-init.iso /var/lib/libvirt/images/xen-orchestra/

# Créer la VM Xen Orchestra
sudo virt-install \
    --name xen-orchestra \
    --ram 4096 \
    --vcpus 2 \
    --disk path=/var/lib/libvirt/images/xen-orchestra/xo-disk.qcow2,device=disk,bus=virtio \
    --disk path=/var/lib/libvirt/images/xen-orchestra/xo-cloud-init.iso,device=cdrom \
    --os-variant ubuntu24.04 \
    --network network=default,model=virtio \
    --graphics none \
    --console pty,target_type=serial \
    --import \
    --noautoconsole

# Attendre 5 minutes que l'installation se termine
echo "⏳ Installation de Xen Orchestra en cours (5 minutes)..."
sleep 300
```

### 4.3 – Vérifier l'Installation de Xen Orchestra

```bash
# Trouver l'IP de la VM Xen Orchestra
virsh domifaddr xen-orchestra

# Se connecter à la VM
virsh console xen-orchestra
# Login: ubuntu / Password: password123

# Vérifier que le service XO est actif
sudo systemctl status xo-server

# Vérifier que le port 80 est ouvert
sudo netstat -tulpn | grep :80

# Sortir : Ctrl + ]
```

### 4.4 – Accéder à l'Interface Web de Xen Orchestra

```bash
# Trouver l'IP de Xen Orchestra
XO_IP=$(virsh domifaddr xen-orchestra | grep -oP '192\.168\.122\.\d+' | head -1)
echo "Xen Orchestra disponible sur : http://$XO_IP"
```

**Depuis votre navigateur (dans la session RDP) :**

1. Ouvrir Firefox
2. Aller à : `http://IP_DE_XO` (exemple : http://192.168.122.50)
3. **Première connexion :**
   - Email : `admin@admin.net`
   - Mot de passe : `admin`
4. Changer le mot de passe lors de la première connexion

🎉 **Xen Orchestra est accessible !**

---

## 🔗 Étape 5 – Connecter XCP-ng à Xen Orchestra

### 5.1 – Ajouter le Serveur XCP-ng dans XO

1. **Dans l'interface Xen Orchestra :**
   - Cliquer sur **Settings** (⚙️ en haut à droite)
   - Aller dans **Servers**
   - Cliquer sur **+ Add Server**

2. **Configuration du serveur :**
   - **Label :** `XCP-ng Lab`
   - **Host :** `IP_DE_XCPNG` (trouver avec `virsh domifaddr xcp-ng`)
   - **Username :** `root`
   - **Password :** `password123`
   - **Unauthorized certificates :** Cocher (pour le lab)
   - Cliquer sur **Connect**

3. **Vérification :**
   - Le serveur devrait apparaître en **Connected** (vert)
   - Aller dans **Home** → Vous devriez voir le serveur XCP-ng

### 5.2 – Explorer l'Interface Xen Orchestra

**Sections principales :**

| Section | Description |
|---------|-------------|
| **Home** | Vue d'ensemble de l'infrastructure |
| **VMs** | Liste et gestion des VMs |
| **Hosts** | Serveurs XCP-ng connectés |
| **Pools** | Groupes de serveurs |
| **Storage** | Stockage (SR - Storage Repository) |
| **Network** | Réseaux virtuels |
| **Backup** | Gestion des sauvegardes |

---

## 🖥️ Étape 6 – Créer une VM dans XCP-ng (Niveau 3)

### 6.1 – Télécharger une Image ISO dans XCP-ng

**Méthode 1 : Via l'interface XO**

1. Aller dans **Storage** → Sélectionner le SR par défaut
2. Cliquer sur **Import**
3. Entrer l'URL : `https://releases.ubuntu.com/24.04/ubuntu-24.04-live-server-amd64.iso`
4. Attendre le téléchargement

**Méthode 2 : Via SSH sur XCP-ng**

```bash
# Se connecter à XCP-ng
virsh console xcp-ng
# Login: root / Password: password123

# Créer un répertoire pour les ISOs
mkdir -p /opt/isos

# Télécharger Ubuntu Server
cd /opt/isos
wget https://releases.ubuntu.com/24.04/ubuntu-24.04-live-server-amd64.iso

# Créer un Storage Repository ISO
xe sr-create name-label="ISO Library" type=iso device-config:location=/opt/isos device-config:legacy_mode=true content-type=iso

# Vérifier
xe sr-list

# Sortir : Ctrl + ]
```

### 6.2 – Créer la VM via Xen Orchestra

1. **Dans Xen Orchestra :**
   - Cliquer sur **New** → **VM**

2. **Sélectionner le Template :**
   - Choisir : **Ubuntu Jammy 22.04** (ou similaire)
   - Cliquer sur **Next**

3. **Informations de la VM :**
   - **Name :** `ubuntu-test-l3`
   - **Description :** `VM de test niveau 3`
   - **vCPUs :** 1
   - **RAM :** 1024 MB (1GB)
   - Cliquer sur **Next**

4. **Installation :**
   - **ISO :** Sélectionner `ubuntu-24.04-live-server-amd64.iso`
   - Cliquer sur **Next**

5. **Disques :**
   - **Disk 1 :** 10 GB
   - Cliquer sur **Next**

6. **Réseau :**
   - **Network :** Pool-wide network associated with eth0
   - Cliquer sur **Next**

7. **Résumé :**
   - Vérifier la configuration
   - Cliquer sur **Create**

8. **Démarrer la VM :**
   - Cliquer sur **Start**
   - Cliquer sur **Console** pour voir l'écran

### 6.3 – Installer Ubuntu dans la VM L3

Suivez l'installation standard d'Ubuntu Server :

1. **Langue :** English
2. **Clavier :** French
3. **Type d'installation :** Ubuntu Server (minimal)
4. **Réseau :** DHCP
5. **Stockage :** Use entire disk
6. **Profil :**
   - Nom : `testuser`
   - Serveur : `ubuntu-test-l3`
   - Mot de passe : `password123`
7. **SSH :** Installer OpenSSH server
8. **Snaps :** Aucun

⏱️ **Installation : ~10 minutes**

### 6.4 – Premier Démarrage de la VM L3

Après l'installation :

1. Redémarrer la VM
2. Se connecter via la console XO :
   - Login : `testuser`
   - Password : `password123`

3. Vérifier le système :
```bash
# Vérifier l'OS
cat /etc/os-release

# Vérifier le CPU (devrait montrer 1 vCPU)
lscpu

# Vérifier la RAM (devrait montrer ~1GB)
free -h

# Vérifier le réseau
ip addr show
ping -c 3 8.8.8.8
```

🎉 **Vous avez créé une VM de niveau 3 ! (VM dans Xen dans KVM)**

---

## 📊 Étape 7 – Comparaison des Performances

### 7.1 – Installer sysbench dans les 3 Niveaux

**Sur L1 (Hôte Ubuntu) :**
```bash
sudo apt install sysbench -y
```

**Sur L2 (XCP-ng) :**
```bash
# XCP-ng est basé sur CentOS, utiliser yum
virsh console xcp-ng
# Login: root / Password: password123
yum install -y epel-release
yum install -y sysbench
```

**Sur L3 (VM Ubuntu dans Xen) :**
```bash
# Via la console XO
sudo apt update
sudo apt install sysbench -y
```

### 7.2 – Test CPU sur les 3 Niveaux

**Sur L1 :**
```bash
sysbench cpu --cpu-max-prime=20000 --threads=4 run | grep "events per second"
```

**Sur L2 (XCP-ng) :**
```bash
sysbench cpu --cpu-max-prime=20000 --threads=4 run | grep "events per second"
```

**Sur L3 (VM dans Xen) :**
```bash
sysbench cpu --cpu-max-prime=20000 --threads=1 run | grep "events per second"
```

### 7.3 – Résultats Attendus

| Niveau | Events/sec | Performance Relative |
|--------|-----------|----------------------|
| **L1 (Hôte)** | ~2000 | 100% (référence) |
| **L2 (XCP-ng dans KVM)** | ~1800 | ~90% |
| **L3 (VM dans Xen)** | ~1600 | ~80% |

**Analyse :**
- Chaque niveau de virtualisation ajoute ~10% d'overhead
- Les performances restent acceptables grâce au nested virtualization
- En production, on évite généralement plus de 2 niveaux

---

## 🔧 Étape 8 – Gestion Avancée avec Xen Orchestra

### 8.1 – Créer un Snapshot

1. **Dans XO :**
   - Aller dans **VMs**
   - Sélectionner `ubuntu-test-l3`
   - Onglet **Snapshots**
   - Cliquer sur **+ New Snapshot**
   - **Name :** `snapshot-initial`
   - Cliquer sur **Create**

### 8.2 – Cloner une VM

1. **Dans XO :**
   - Sélectionner `ubuntu-test-l3`
   - Onglet **General**
   - Cliquer sur **Clone**
   - **Name :** `ubuntu-test-l3-clone`
   - **Mode :** Full copy
   - Cliquer sur **Clone**

### 8.3 – Exporter une VM

1. **Dans XO :**
   - Sélectionner `ubuntu-test-l3`
   - Onglet **General**
   - Cliquer sur **Export**
   - **Format :** XVA (Xen Virtual Appliance)
   - Cliquer sur **Export**

### 8.4 – Configurer un Backup Automatique

1. **Dans XO :**
   - Aller dans **Backup** → **New**
   - **Name :** `backup-daily`
   - **VMs :** Sélectionner `ubuntu-test-l3`
   - **Schedule :** Daily at 02:00
   - **Retention :** 7 backups
   - **Remote :** Local (ou configurer NFS)
   - Cliquer sur **Create**

---

## 🌐 Étape 9 – Configuration Réseau Avancée dans XCP-ng

### 9.1 – Créer un Réseau Privé dans XCP-ng

```bash
# Se connecter à XCP-ng
virsh console xcp-ng
# Login: root / Password: password123

# Créer un réseau privé (sans accès externe)
xe network-create name-label="Private Network" description="Réseau isolé pour tests"

# Lister les réseaux
xe network-list

# Sortir : Ctrl + ]
```

### 9.2 – Attacher une VM au Réseau Privé

1. **Dans XO :**
   - Sélectionner `ubuntu-test-l3`
   - Arrêter la VM
   - Onglet **Network**
   - Cliquer sur **+ Add Network**
   - Sélectionner **Private Network**
   - Démarrer la VM

2. **Vérifier dans la VM :**
```bash
# Devrait montrer 2 interfaces réseau
ip addr show
```

---

## 🧪 Étape 10 – Tests et Diagnostics

### 10.1 – Script de Diagnostic Complet

```bash
# Sur l'hôte L1, créer un script de diagnostic
cat > ~/check-nested-virt.sh << 'EOF'
#!/bin/bash

echo "=== 🔷 Diagnostic Virtualisation Imbriquée ==="
echo ""

echo "📊 Niveau 1 (L1 - Hôte KVM) :"
echo "  CPU : $(nproc) cores"
echo "  RAM : $(free -h | grep Mem | awk '{print $2}')"
echo "  Nested : $(cat /sys/module/kvm_amd/parameters/nested)"
echo ""

echo "📊 Niveau 2 (L2 - XCP-ng) :"
if virsh list | grep -q xcp-ng; then
    echo "  État : Running ✅"
    XCP_IP=$(virsh domifaddr xcp-ng | grep -oP '192\.168\.122\.\d+' | head -1)
    echo "  IP : $XCP_IP"
else
    echo "  État : Stopped ❌"
fi
echo ""

echo "📊 Xen Orchestra :"
if virsh list | grep -q xen-orchestra; then
    echo "  État : Running ✅"
    XO_IP=$(virsh domifaddr xen-orchestra | grep -oP '192\.168\.122\.\d+' | head -1)
    echo "  IP : $XO_IP"
    echo "  URL : http://$XO_IP"
else
    echo "  État : Stopped ❌"
fi
echo ""

echo "🖥️ VMs KVM actives :"
virsh list --name
echo ""

echo "✅ Diagnostic terminé"
EOF

chmod +x ~/check-nested-virt.sh
~/check-nested-virt.sh
```

### 10.2 – Vérifier la Chaîne Complète

```bash
# Test de connectivité L1 → L2 → L3
echo "=== Test de Connectivité ==="

# Ping L1 → L2 (XCP-ng)
XCP_IP=$(virsh domifaddr xcp-ng | grep -oP '192\.168\.122\.\d+' | head -1)
echo "L1 → L2 (XCP-ng) :"
ping -c 3 $XCP_IP

# Depuis XCP-ng, ping vers une VM L3
# (nécessite de connaître l'IP de la VM L3)
```

---

## 📝 Récapitulatif du TP4

Dans ce TP, vous avez appris à :

- ✅ Installer **XCP-ng** (Xen) dans une VM KVM
- ✅ Comprendre la **virtualisation imbriquée** sur 3 niveaux
- ✅ Déployer **Xen Orchestra** pour la gestion Web
- ✅ Créer une **VM de niveau 3** (VM dans Xen dans KVM)
- ✅ Gérer les VMs via **interface Web** (snapshots, clones, backups)
- ✅ Comparer les **performances** entre les niveaux
- ✅ Configurer des **réseaux privés** dans XCP-ng

### 🎯 Checklist de Vérification Finale

Avant de passer au TP5, vérifiez que :

- [ ] XCP-ng est installé et accessible (`virsh list`)
- [ ] Xen Orchestra est accessible via navigateur
- [ ] XCP-ng est connecté à Xen Orchestra (vert)
- [ ] Vous avez créé au moins une VM L3 dans XCP-ng
- [ ] Les performances L3 sont acceptables (~80% de L1)
- [ ] Vous pouvez créer des snapshots et clones

### 📚 Concepts Clés Appris

| Concept | Description |
|---------|-------------|
| **XCP-ng** | Hyperviseur Xen open-source (fork de Citrix Hypervisor) |
| **Xen Orchestra** | Interface Web de gestion pour XCP-ng |
| **Nested Virtualization** | Virtualisation sur plusieurs niveaux (L1→L2→L3) |
| **host-passthrough** | Mode CPU qui expose toutes les fonctionnalités au guest |
| **XVA** | Format d'export de VMs Xen (Xen Virtual Appliance) |
| **SR** | Storage Repository - Stockage dans XCP-ng |

### 🛠️ Commandes Essentielles à Retenir

| Commande | Description |
|----------|-------------|
| `xl info` | Informations sur l'hyperviseur Xen |
| `xl list` | Lister les VMs Xen |
| `xe vm-list` | Lister les VMs (commande xe) |
| `xe sr-list` | Lister les Storage Repositories |
| `xe network-list` | Lister les réseaux |
| `xe network-create` | Créer un réseau |

---

## 🚀 Prochaine Étape : TP5

Vous êtes maintenant prêt pour le **TP5 : Hyperviseur Imbriqué - Proxmox VE** où vous allez :

- Installer **Proxmox VE** dans une VM KVM
- Découvrir l'**interface Web Proxmox**
- Créer des **VMs et conteneurs LXC**
- Comparer **Proxmox vs XCP-ng**
- Gérer le **stockage ZFS** (optionnel)

**Vous maîtrisez maintenant Xen et la virtualisation imbriquée ! 🎉**
