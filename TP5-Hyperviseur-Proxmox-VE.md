# 🟦 TP5 – Hyperviseur Imbriqué - Proxmox VE

**Objectif du TP :**  
Installer Proxmox VE en tant que VM dans KVM et découvrir ses fonctionnalités.  
À la fin du TP, vous aurez :

- ✅ Installé **Proxmox VE** dans une VM KVM
- ✅ Configuré l'**interface Web Proxmox**
- ✅ Créé des **VMs** et des **conteneurs LXC**
- ✅ Maîtrisé le **stockage** (LVM, Directory, ZFS optionnel)
- ✅ Configuré les **réseaux** (Linux Bridge, VLAN)
- ✅ Comparé **Proxmox vs XCP-ng**

> 💡 **Pourquoi Proxmox ?**  
> Proxmox VE est une solution de virtualisation open-source très populaire, combinant KVM (VMs) et LXC (conteneurs). Son interface Web intuitive et ses fonctionnalités de clustering en font un choix privilégié pour les PME et les homelab.

---

## 📋 Prérequis

- ✅ TP1, TP2, TP3, et TP4 terminés
- ✅ Virtualisation imbriquée activée
- ✅ Au moins 60GB d'espace disque libre
- ✅ 16GB de RAM disponible pour la VM Proxmox

**Vérification rapide :**
```bash
# Vérifier les ressources
free -h
df -h /var/lib/libvirt/images

# Vérifier la virtualisation imbriquée
cat /sys/module/kvm_amd/parameters/nested
```

---

## 🏗️ Architecture Proxmox VE

### Vue d'Ensemble

```
┌─────────────────────────────────────────────────────────────┐
│  Niveau 3 (L3) - VMs et Conteneurs dans Proxmox            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ VM Ubuntu    │  │ VM Windows   │  │ CT Alpine    │     │
│  │ (KVM)        │  │ (KVM)        │  │ (LXC)        │     │
│  │ 2GB RAM      │  │ 4GB RAM      │  │ 512MB RAM    │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────┴──────────────────────────────────┐
│  Niveau 2 (L2) - Proxmox VE dans KVM                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Proxmox VE 8.x                                       │  │
│  │  - KVM + LXC                                          │  │
│  │  - Interface Web (port 8006)                          │  │
│  │  - RAM : 16GB                                         │  │
│  │  - vCPU : 4 (nested)                                  │  │
│  │  - Disque : 60GB                                      │  │
│  └───────────────────────────────────────────────────────┘  │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────┴──────────────────────────────────┐
│  Niveau 1 (L1) - Hôte KVM (Oracle Cloud)                  │
│  Ubuntu 24.04 Desktop + KVM                                │
└─────────────────────────────────────────────────────────────┘
```

### Proxmox VE : KVM + LXC

**Proxmox combine deux technologies :**

| Technologie | Type | Utilisation |
|-------------|------|-------------|
| **KVM** | Virtualisation complète | VMs avec OS complet (Ubuntu, Windows, etc.) |
| **LXC** | Conteneurs système | Conteneurs légers (Alpine, Debian, etc.) |

**Avantages de LXC :**
- ⚡ Démarrage ultra-rapide (< 1 seconde)
- 💾 Consommation mémoire minimale
- 🔧 Isolation au niveau du noyau (partage du kernel hôte)

---

## 💿 Étape 1 – Télécharger Proxmox VE

### 1.1 – Télécharger l'ISO Proxmox

```bash
# Créer un répertoire pour Proxmox
mkdir -p ~/iso/proxmox

# Télécharger Proxmox VE 8.x (dernière version stable)
cd ~/iso/proxmox
wget https://enterprise.proxmox.com/iso/proxmox-ve_8.3-1.iso

# Vérifier le téléchargement
ls -lh proxmox-ve_8.3-1.iso
```

**Taille attendue :** ~1.3GB

**Qu'est-ce que Proxmox VE ?**
- Basé sur Debian Linux
- Interface Web complète (port 8006)
- Gestion de VMs (KVM) et conteneurs (LXC)
- Clustering et haute disponibilité
- Backups intégrés

---

## 🖥️ Étape 2 – Créer la VM pour Proxmox

### 2.1 – Créer le Disque Virtuel

```bash
# Créer un répertoire pour Proxmox
sudo mkdir -p /var/lib/libvirt/images/proxmox

# Créer un disque de 60GB (format qcow2 pour les snapshots)
sudo qemu-img create -f qcow2 \
    /var/lib/libvirt/images/proxmox/proxmox-disk.qcow2 60G

# Vérifier
sudo qemu-img info /var/lib/libvirt/images/proxmox/proxmox-disk.qcow2
```

### 2.2 – Créer la VM avec virt-install

```bash
# Créer la VM Proxmox avec nested virtualization
sudo virt-install \
    --name proxmox \
    --ram 16384 \
    --vcpus 4 \
    --cpu host-passthrough,cache.mode=passthrough \
    --disk path=/var/lib/libvirt/images/proxmox/proxmox-disk.qcow2,bus=virtio,cache=none \
    --network network=default,model=virtio \
    --graphics vnc,listen=0.0.0.0,port=5902 \
    --cdrom ~/iso/proxmox/proxmox-ve_8.3-1.iso \
    --os-variant debian11 \
    --boot cdrom,hd \
    --noautoconsole
```

**Explication des options :**

| Option | Description |
|--------|-------------|
| `--cpu host-passthrough` | Expose toutes les fonctionnalités CPU (nécessaire pour nested) |
| `--network model=virtio` | Carte réseau virtio (meilleures performances) |
| `--graphics vnc,port=5902` | VNC sur port 5902 |
| `--os-variant debian11` | Proxmox est basé sur Debian |

### 2.3 – Se Connecter à la VM pour l'Installation

```bash
# Ouvrir virt-manager
virt-manager &

# Ou se connecter via VNC depuis Windows :
# Adresse : VOTRE_IP_PUBLIQUE:5902
```

---

## 🔧 Étape 3 – Installer Proxmox VE

### 3.1 – Processus d'Installation (via virt-manager ou VNC)

1. **Écran de Bienvenue**
   - Attendre que l'ISO démarre
   - Sélectionner : **"Install Proxmox VE (Graphical)"**
   - Appuyer sur **Entrée**

2. **EULA**
   - Lire et accepter la licence
   - Cliquer sur **I agree**

3. **Disque Cible**
   - **Target Harddisk :** `/dev/vda` (60GB)
   - **Filesystem :** ext4 (recommandé pour nested)
   - Cliquer sur **Next**

   > **Note :** ZFS est plus performant mais consomme plus de RAM. Pour un lab imbriqué, ext4 est suffisant.

4. **Localisation**
   - **Country :** France
   - **Time zone :** Europe/Paris
   - **Keyboard Layout :** French
   - Cliquer sur **Next**

5. **Mot de Passe et Email**
   - **Password :** `password123` (pour le lab)
   - **Confirm :** `password123`
   - **Email :** `admin@lab.local`
   - Cliquer sur **Next**

6. **Configuration Réseau**
   - **Management Interface :** `ens3` (devrait être détecté automatiquement)
   - **Hostname (FQDN) :** `proxmox.local`
   - **IP Address (CIDR) :** Laisser DHCP ou noter l'IP affichée
   - **Gateway :** 192.168.122.1 (passerelle du réseau default)
   - **DNS Server :** 8.8.8.8
   - Cliquer sur **Next**

7. **Résumé**
   - Vérifier la configuration
   - Cliquer sur **Install**

⏱️ **L'installation prend environ 5-10 minutes.**

### 3.2 – Premier Démarrage

Après l'installation :

1. La VM redémarre automatiquement
2. **Éjecter le CDROM :**
   - Dans virt-manager : Détails → SATA CDROM → Déconnecter
3. Redémarrer la VM si nécessaire

**Écran de Connexion Proxmox :**
```
Welcome to the Proxmox Virtual Environment. Please use your web browser to configure this server.

  https://192.168.122.X:8006/

proxmox login: root
Password: password123
```

### 3.3 – Vérifier l'Installation

```bash
# Se connecter en root
# Password: password123

# Vérifier la version de Proxmox
pveversion

# Vérifier l'adresse IP
ip addr show ens3

# Vérifier que les services Proxmox fonctionnent
systemctl status pve-cluster
systemctl status pvedaemon
systemctl status pveproxy

# Vérifier KVM
kvm-ok
```

**Résultat attendu de `pveversion` :**
```
pve-manager/8.3.1/...
```

🎉 **Proxmox VE est installé et fonctionne !**

---

## 🌐 Étape 4 – Accéder à l'Interface Web Proxmox

### 4.1 – Trouver l'IP de Proxmox

```bash
# Sur l'hôte L1
virsh domifaddr proxmox
```

**Résultat attendu :**
```
 Name       MAC address          Protocol     Address
-------------------------------------------------------------------------------
 vnet0      52:54:00:xx:xx:xx    ipv4         192.168.122.50/24
```

### 4.2 – Se Connecter à l'Interface Web

**Depuis votre navigateur (dans la session RDP) :**

1. Ouvrir Firefox
2. Aller à : `https://192.168.122.50:8006`
3. **Avertissement de sécurité :**
   - Cliquer sur **Advanced**
   - Cliquer sur **Accept the Risk and Continue**

4. **Page de Connexion :**
   - **Username :** `root`
   - **Password :** `password123`
   - **Realm :** `Linux PAM standard authentication`
   - **Language :** French (ou English)
   - Cliquer sur **Login**

5. **Message de Souscription :**
   - Proxmox affiche un message concernant la souscription
   - Cliquer sur **OK** (vous pouvez l'ignorer pour un lab)

🎉 **Vous êtes connecté à l'interface Web Proxmox !**

### 4.3 – Désactiver le Message de Souscription (Optionnel)

```bash
# Se connecter à Proxmox via console
virsh console proxmox
# Login: root / Password: password123

# Éditer le fichier JavaScript
sed -i.bak "s/data.status !== 'Active'/false/g" /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js

# Redémarrer le service proxy
systemctl restart pveproxy

# Sortir : Ctrl + ]
```

Rafraîchir la page Web → Le message ne devrait plus apparaître.

---

## 🖥️ Étape 5 – Explorer l'Interface Proxmox

### 5.1 – Vue d'Ensemble de l'Interface

**Panneau de Gauche (Arborescence) :**
- **Datacenter :** Configuration globale
  - **proxmox :** Votre nœud Proxmox
    - **VMs :** Machines virtuelles
    - **Conteneurs :** Conteneurs LXC
    - **Stockage :** Volumes de stockage

**Panneau Central :**
- **Summary :** Vue d'ensemble (CPU, RAM, stockage)
- **Notes :** Notes personnalisées
- **Shell :** Terminal Web

**Panneau de Droite :**
- Détails de l'élément sélectionné

### 5.2 – Vérifier les Ressources

1. Cliquer sur **proxmox** (le nœud)
2. Onglet **Summary** :
   - **CPU :** 4 cores
   - **Memory :** ~16GB
   - **Local :** ~60GB

---

## 💾 Étape 6 – Configurer le Stockage

### 6.1 – Stockages par Défaut

Proxmox crée automatiquement deux stockages :

| Stockage | Type | Utilisation |
|----------|------|-------------|
| **local** | Directory | Images ISO, templates de conteneurs |
| **local-lvm** | LVM-Thin | Disques de VMs |

**Vérifier les stockages :**

1. Cliquer sur **Datacenter** → **Storage**
2. Vous devriez voir :
   - `local` : `/var/lib/vz`
   - `local-lvm` : Volume Group `pve`

### 6.2 – Télécharger des Templates de Conteneurs

```bash
# Via l'interface Web :
# 1. Cliquer sur "local (proxmox)" → "CT Templates"
# 2. Cliquer sur "Templates"
# 3. Sélectionner :
#    - alpine-3.20-default
#    - debian-12-standard
#    - ubuntu-24.04-standard
# 4. Cliquer sur "Download"

# Ou via SSH :
virsh console proxmox
# Login: root / Password: password123

# Télécharger des templates
pveam update
pveam download local alpine-3.20-default_20240908_amd64.tar.xz
pveam download local debian-12-standard_12.7-1_amd64.tar.zst
pveam download local ubuntu-24.04-standard_24.04-2_amd64.tar.zst

# Lister les templates
pveam list local

# Sortir : Ctrl + ]
```

### 6.3 – Télécharger une ISO Ubuntu

```bash
# Via SSH sur Proxmox
virsh console proxmox
# Login: root / Password: password123

# Aller dans le répertoire des ISOs
cd /var/lib/vz/template/iso

# Télécharger Ubuntu Server
wget https://releases.ubuntu.com/24.04/ubuntu-24.04-live-server-amd64.iso

# Vérifier
ls -lh

# Sortir : Ctrl + ]
```

**Via l'interface Web :**
1. Cliquer sur **local (proxmox)** → **ISO Images**
2. Cliquer sur **Upload** ou **Download from URL**
3. Entrer l'URL : `https://releases.ubuntu.com/24.04/ubuntu-24.04-live-server-amd64.iso`

---

## 🐧 Étape 7 – Créer un Conteneur LXC (Niveau 3)

### 7.1 – Créer un Conteneur Alpine Linux

**Via l'interface Web :**

1. **Cliquer sur "Create CT"** (en haut à droite)

2. **General :**
   - **Node :** proxmox
   - **CT ID :** 100 (auto)
   - **Hostname :** `alpine-test`
   - **Password :** `password123`
   - **Confirm password :** `password123`
   - Cocher **Unprivileged container**
   - Cliquer sur **Next**

3. **Template :**
   - **Storage :** local
   - **Template :** `alpine-3.20-default`
   - Cliquer sur **Next**

4. **Disks :**
   - **Storage :** local-lvm
   - **Disk size (GB) :** 4
   - Cliquer sur **Next**

5. **CPU :**
   - **Cores :** 1
   - Cliquer sur **Next**

6. **Memory :**
   - **Memory (MB) :** 512
   - **Swap (MB) :** 512
   - Cliquer sur **Next**

7. **Network :**
   - **Bridge :** vmbr0
   - **IPv4 :** DHCP
   - **IPv6 :** DHCP
   - Cliquer sur **Next**

8. **DNS :**
   - Laisser par défaut (hérite de l'hôte)
   - Cliquer sur **Next**

9. **Confirm :**
   - Vérifier la configuration
   - Cocher **Start after created**
   - Cliquer sur **Finish**

### 7.2 – Se Connecter au Conteneur

1. **Dans l'interface Web :**
   - Cliquer sur **100 (alpine-test)**
   - Cliquer sur **Console**

2. **Login :**
   - Login : `root`
   - Password : `password123`

3. **Tester le conteneur :**
```bash
# Vérifier l'OS
cat /etc/os-release

# Vérifier la RAM (devrait montrer ~512MB)
free -h

# Installer un paquet
apk update
apk add htop

# Tester la connectivité
ping -c 3 8.8.8.8

# Sortir
exit
```

🎉 **Votre premier conteneur LXC fonctionne !**

**Avantages observés :**
- ⚡ Démarrage instantané (< 1 seconde)
- 💾 Consommation mémoire minimale (~50MB)
- 🔧 Accès root complet

---

## 🖥️ Étape 8 – Créer une VM KVM (Niveau 3)

### 8.1 – Créer une VM Ubuntu

**Via l'interface Web :**

1. **Cliquer sur "Create VM"** (en haut à droite)

2. **General :**
   - **Node :** proxmox
   - **VM ID :** 101 (auto)
   - **Name :** `ubuntu-vm-test`
   - Cliquer sur **Next**

3. **OS :**
   - **Use CD/DVD disc image file (iso) :** Cocher
   - **Storage :** local
   - **ISO image :** `ubuntu-24.04-live-server-amd64.iso`
   - **Guest OS Type :** Linux
   - **Version :** 6.x - 2.6 Kernel
   - Cliquer sur **Next**

4. **System :**
   - **Graphic card :** Default
   - **Machine :** Default (i440fx)
   - **BIOS :** Default (SeaBIOS)
   - **SCSI Controller :** VirtIO SCSI single
   - Cocher **Qemu Agent**
   - Cliquer sur **Next**

5. **Disks :**
   - **Bus/Device :** SCSI 0
   - **Storage :** local-lvm
   - **Disk size (GB) :** 15
   - **Cache :** Default (No cache)
   - Cliquer sur **Next**

6. **CPU :**
   - **Sockets :** 1
   - **Cores :** 2
   - **Type :** host (pour nested)
   - Cliquer sur **Next**

7. **Memory :**
   - **Memory (MB) :** 2048
   - Cliquer sur **Next**

8. **Network :**
   - **Bridge :** vmbr0
   - **Model :** VirtIO (paravirtualized)
   - Cliquer sur **Next**

9. **Confirm :**
   - Vérifier la configuration
   - Cocher **Start after created**
   - Cliquer sur **Finish**

### 8.2 – Installer Ubuntu dans la VM

1. **Cliquer sur la VM 101** → **Console**
2. Suivre l'installation standard d'Ubuntu Server :
   - Langue : English
   - Clavier : French
   - Réseau : DHCP
   - Stockage : Use entire disk
   - Profil : `testuser` / `password123`
   - SSH : Installer OpenSSH server

⏱️ **Installation : ~10 minutes**

### 8.3 – Installer l'Agent QEMU

Après l'installation et le redémarrage :

```bash
# Dans la console de la VM
# Login: testuser / Password: password123

# Installer l'agent QEMU
sudo apt update
sudo apt install qemu-guest-agent -y

# Démarrer l'agent
sudo systemctl start qemu-guest-agent
sudo systemctl enable qemu-guest-agent

# Vérifier
sudo systemctl status qemu-guest-agent
```

**Pourquoi l'agent QEMU ?**
- Permet à Proxmox de voir l'IP de la VM
- Améliore la gestion de l'arrêt/redémarrage
- Permet le snapshot avec cohérence des données

---

## 📊 Étape 9 – Comparaison LXC vs KVM

### 9.1 – Test de Démarrage

**Conteneur LXC (alpine-test) :**
```bash
# Dans l'interface Web, arrêter le conteneur
# Cliquer sur "100 (alpine-test)" → "Shutdown"

# Chronométrer le démarrage
# Cliquer sur "Start"
# Temps : ~1 seconde ⚡
```

**VM KVM (ubuntu-vm-test) :**
```bash
# Arrêter la VM
# Cliquer sur "101 (ubuntu-vm-test)" → "Shutdown"

# Chronométrer le démarrage
# Cliquer sur "Start"
# Temps : ~30 secondes 🐢
```

### 9.2 – Test de Consommation Mémoire

| Type | Nom | RAM Allouée | RAM Utilisée | Efficacité |
|------|-----|-------------|--------------|------------|
| **LXC** | alpine-test | 512MB | ~50MB | 90% libre |
| **KVM** | ubuntu-vm-test | 2048MB | ~1800MB | 12% libre |

**Observation :** Les conteneurs LXC sont **beaucoup plus légers** que les VMs KVM.

### 9.3 – Cas d'Usage

| Scénario | Recommandation |
|----------|----------------|
| Serveur Web (Nginx, Apache) | **LXC** (léger, rapide) |
| Base de données (PostgreSQL, MySQL) | **LXC** ou **KVM** |
| Application nécessitant un noyau spécifique | **KVM** (noyau isolé) |
| Windows ou autre OS non-Linux | **KVM** (seule option) |
| Isolation maximale | **KVM** (virtualisation complète) |

---

## 🔧 Étape 10 – Gestion Avancée

### 10.1 – Créer un Snapshot

**Pour une VM :**
1. Cliquer sur **101 (ubuntu-vm-test)**
2. Onglet **Snapshots**
3. Cliquer sur **Take Snapshot**
4. **Name :** `snapshot-initial`
5. **Description :** `État après installation`
6. Cocher **Include RAM** (optionnel)
7. Cliquer sur **Take Snapshot**

**Pour un conteneur :**
1. Cliquer sur **100 (alpine-test)**
2. Onglet **Snapshots**
3. Même procédure

### 10.2 – Cloner une VM/Conteneur

**Cloner le conteneur Alpine :**
1. Cliquer sur **100 (alpine-test)**
2. Cliquer sur **More** → **Clone**
3. **Target node :** proxmox
4. **VM ID :** 102
5. **Name :** `alpine-test-clone`
6. **Mode :** Full Clone
7. Cliquer sur **Clone**

### 10.3 – Créer un Template

**Transformer une VM en template :**
1. Arrêter la VM
2. Cliquer sur **101 (ubuntu-vm-test)**
3. Cliquer sur **More** → **Convert to template**
4. Confirmer

**Utiliser le template :**
1. Clic droit sur le template → **Clone**
2. Créer une nouvelle VM basée sur le template

### 10.4 – Backup et Restore

**Créer un backup :**
1. Cliquer sur **100 (alpine-test)**
2. Onglet **Backup**
3. Cliquer sur **Backup now**
4. **Storage :** local
5. **Mode :** Snapshot (si disponible) ou Stop
6. **Compression :** ZSTD
7. Cliquer sur **Backup**

**Restaurer un backup :**
1. Cliquer sur **local (proxmox)** → **Backups**
2. Sélectionner le backup
3. Cliquer sur **Restore**

---

## 🌐 Étape 11 – Configuration Réseau Avancée

### 11.1 – Créer un Réseau Privé (Linux Bridge)

```bash
# Via SSH sur Proxmox
virsh console proxmox
# Login: root / Password: password123

# Éditer la configuration réseau
nano /etc/network/interfaces

# Ajouter un nouveau bridge
cat >> /etc/network/interfaces << 'EOF'

# Bridge privé pour réseau isolé
auto vmbr1
iface vmbr1 inet static
    address 10.0.0.1/24
    bridge-ports none
    bridge-stp off
    bridge-fd 0
EOF

# Appliquer la configuration
ifreload -a

# Vérifier
ip addr show vmbr1

# Sortir : Ctrl + ]
```

**Via l'interface Web :**
1. Cliquer sur **proxmox** → **System** → **Network**
2. Cliquer sur **Create** → **Linux Bridge**
3. **Name :** `vmbr1`
4. **IPv4/CIDR :** `10.0.0.1/24`
5. **Bridge ports :** (vide pour réseau isolé)
6. Cliquer sur **Create**
7. Cliquer sur **Apply Configuration**

### 11.2 – Attacher une VM au Réseau Privé

1. Arrêter la VM
2. Cliquer sur **101 (ubuntu-vm-test)** → **Hardware**
3. Cliquer sur **Add** → **Network Device**
4. **Bridge :** vmbr1
5. **Model :** VirtIO
6. Cliquer sur **Add**
7. Démarrer la VM

**Vérifier dans la VM :**
```bash
# Devrait montrer 2 interfaces réseau
ip addr show
```

---

## 🧪 Étape 12 – Tests et Diagnostics

### 12.1 – Script de Diagnostic Proxmox

```bash
# Sur l'hôte L1
cat > ~/check-proxmox.sh << 'EOF'
#!/bin/bash

echo "=== 🟦 Diagnostic Proxmox VE ==="
echo ""

echo "📊 VM Proxmox :"
if virsh list | grep -q proxmox; then
    echo "  État : Running ✅"
    PVE_IP=$(virsh domifaddr proxmox | grep -oP '192\.168\.122\.\d+' | head -1)
    echo "  IP : $PVE_IP"
    echo "  URL : https://$PVE_IP:8006"
else
    echo "  État : Stopped ❌"
fi
echo ""

echo "🖥️ VMs et Conteneurs dans Proxmox :"
echo "  (Connectez-vous à l'interface Web pour voir la liste)"
echo ""

echo "✅ Diagnostic terminé"
EOF

chmod +x ~/check-proxmox.sh
~/check-proxmox.sh
```

### 12.2 – Monitoring des Ressources

**Via l'interface Web :**
1. Cliquer sur **proxmox** → **Summary**
2. Observer les graphiques :
   - CPU Usage
   - Memory Usage
   - Network Traffic
   - Disk I/O

**Via SSH :**
```bash
virsh console proxmox
# Login: root / Password: password123

# Afficher l'utilisation des ressources
pvesh get /nodes/proxmox/status

# Lister les VMs et conteneurs
qm list  # VMs
pct list # Conteneurs

# Sortir : Ctrl + ]
```

---

## 📝 Récapitulatif du TP5

Dans ce TP, vous avez appris à :

- ✅ Installer **Proxmox VE** dans une VM KVM
- ✅ Configurer l'**interface Web Proxmox**
- ✅ Créer des **conteneurs LXC** (Alpine, Debian, Ubuntu)
- ✅ Créer des **VMs KVM** avec installation complète
- ✅ Comparer **LXC vs KVM** (performances, cas d'usage)
- ✅ Gérer les **snapshots, clones, templates**
- ✅ Configurer des **réseaux privés** (Linux Bridge)
- ✅ Créer des **backups** automatiques

### 🎯 Checklist de Vérification Finale

Avant de passer au TP6, vérifiez que :

- [ ] Proxmox est installé et accessible via Web (`https://IP:8006`)
- [ ] Vous avez créé au moins un conteneur LXC
- [ ] Vous avez créé au moins une VM KVM
- [ ] Les templates de conteneurs sont téléchargés
- [ ] Vous pouvez créer des snapshots et clones
- [ ] Le réseau privé `vmbr1` est configuré

### 📚 Concepts Clés Appris

| Concept | Description |
|---------|-------------|
| **Proxmox VE** | Plateforme de virtualisation open-source (KVM + LXC) |
| **LXC** | Linux Containers - Conteneurs système légers |
| **KVM** | Kernel-based Virtual Machine - Virtualisation complète |
| **vmbr0** | Bridge réseau par défaut (accès Internet) |
| **local-lvm** | Stockage LVM-Thin pour disques de VMs |
| **CT** | Container (conteneur LXC) |
| **VM** | Virtual Machine (machine virtuelle KVM) |

### 🛠️ Commandes Essentielles à Retenir

| Commande | Description |
|----------|-------------|
| `pveversion` | Version de Proxmox |
| `qm list` | Lister les VMs |
| `pct list` | Lister les conteneurs |
| `qm start <vmid>` | Démarrer une VM |
| `pct start <ctid>` | Démarrer un conteneur |
| `pveam list local` | Lister les templates de conteneurs |
| `pvesh get /nodes/proxmox/status` | Statut du nœud |

---

## 🚀 Prochaine Étape : TP6

Vous êtes maintenant prêt pour le **TP6 : Déploiement Final et Comparaison** où vous allez :

- Déployer une **VM Ubuntu** dans **Proxmox** ET **Xen Orchestra**
- Comparer les **performances** (CPU, RAM, I/O)
- Comparer l'**expérience utilisateur** (interface, fonctionnalités)
- Créer un **tableau comparatif** complet
- Choisir le **meilleur hyperviseur** pour vos besoins

**Vous maîtrisez maintenant Proxmox VE ! 🎉**
