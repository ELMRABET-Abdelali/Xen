# 🔧 TP2 – Installation et Fondations KVM

**Objectif du TP :**  
Installer et maîtriser QEMU/KVM, l'hyperviseur Linux de référence.  
À la fin du TP, vous aurez :

- ✅ Compris la différence entre **hyperviseurs Type-1 et Type-2**
- ✅ Installé **QEMU/KVM** et **libvirt** complets
- ✅ Configuré **virt-manager** (interface graphique)
- ✅ Vérifié l'**accélération matérielle** (KVM)
- ✅ Créé votre **première machine virtuelle**
- ✅ Maîtrisé les commandes de base avec `virsh`

> 💡 **Pourquoi KVM ?**  
> KVM (Kernel-based Virtual Machine) est un hyperviseur Type-1 intégré au noyau Linux. Il transforme Linux en hyperviseur bare-metal tout en conservant les fonctionnalités d'un OS complet. C'est la base de nombreuses solutions cloud (OpenStack, Proxmox, etc.).

---

## 📋 Prérequis

- ✅ TP1 terminé (Ubuntu Desktop + XRDP + Nested Virtualization)
- ✅ Connexion RDP active à votre instance Oracle Cloud
- ✅ `kvm-ok` retournant "KVM acceleration can be used"
- ✅ Au moins 30GB d'espace disque libre

**Vérification rapide :**
```bash
# Exécuter le script de vérification du TP1
~/check-virt.sh
```

---

## 🏗️ Architecture et Concepts Fondamentaux

### Type-1 vs Type-2 : Quelle est la Différence ?

#### 🔷 Hyperviseur Type-1 (Bare-Metal)

```
┌─────────────────────────────────────────┐
│         VMs (Invités)                   │
│  ┌──────┐  ┌──────┐  ┌──────┐          │
│  │ VM 1 │  │ VM 2 │  │ VM 3 │          │
│  └──────┘  └──────┘  └──────┘          │
├─────────────────────────────────────────┤
│    Hyperviseur (KVM, Xen, ESXi)        │
├─────────────────────────────────────────┤
│    Matériel Physique (CPU, RAM, etc.)  │
└─────────────────────────────────────────┘
```

**Caractéristiques :**
- ✅ S'exécute **directement sur le matériel**
- ✅ Performances maximales (accès direct au hardware)
- ✅ Utilisé en production (datacenters, cloud)
- 📌 Exemples : KVM, Xen, VMware ESXi, Hyper-V

#### 🔶 Hyperviseur Type-2 (Hosted)

```
┌─────────────────────────────────────────┐
│         VMs (Invités)                   │
│  ┌──────┐  ┌──────┐                    │
│  │ VM 1 │  │ VM 2 │                    │
│  └──────┘  └──────┘                    │
├─────────────────────────────────────────┤
│    Hyperviseur (VirtualBox, VMware WS) │
├─────────────────────────────────────────┤
│    OS Hôte (Windows, Linux, macOS)     │
├─────────────────────────────────────────┤
│    Matériel Physique                   │
└─────────────────────────────────────────┘
```

**Caractéristiques :**
- ✅ S'exécute **sur un OS existant**
- ⚠️ Performances réduites (couche OS intermédiaire)
- ✅ Facile à installer et utiliser
- 📌 Exemples : VirtualBox, VMware Workstation, Parallels

### 🎯 KVM : Le Meilleur des Deux Mondes

**KVM est unique :** Il transforme le noyau Linux en hyperviseur Type-1 tout en conservant un OS complet !

```
┌─────────────────────────────────────────────────┐
│  Applications Linux (virt-manager, SSH, etc.)  │
├─────────────────────────────────────────────────┤
│              VMs (via QEMU/KVM)                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐     │
│  │ Ubuntu   │  │ Proxmox  │  │ XCP-ng   │     │
│  │ Server   │  │ (TP5)    │  │ (TP4)    │     │
│  └──────────┘  └──────────┘  └──────────┘     │
├─────────────────────────────────────────────────┤
│      Noyau Linux + Module KVM (Type-1)        │
├─────────────────────────────────────────────────┤
│      Matériel Physique (Oracle Cloud)         │
└─────────────────────────────────────────────────┘
```

**Avantages de KVM :**
- ⚡ Performances Type-1 (accès direct au hardware via `/dev/kvm`)
- 🛠️ Flexibilité d'un OS complet (outils Linux, SSH, etc.)
- 🆓 Open-source et intégré au noyau Linux
- 🔧 Base de nombreuses solutions professionnelles

---

## 🚀 Étape 1 – Installation Complète de QEMU/KVM

### 1.1 – Installer les Paquets Essentiels

```bash
# Mise à jour du système
sudo apt update

# Installation de QEMU/KVM et outils associés
sudo apt install -y \
    qemu-kvm \
    qemu-system-x86 \
    qemu-utils \
    libvirt-daemon-system \
    libvirt-daemon \
    libvirt-clients \
    bridge-utils \
    virt-manager \
    virtinst \
    ovmf \
    genisoimage
```

**Explication des paquets :**

| Paquet | Rôle |
|--------|------|
| `qemu-kvm` | Émulateur QEMU avec accélération KVM |
| `qemu-system-x86` | Support des architectures x86/x86_64 |
| `qemu-utils` | Utilitaires (qemu-img pour gérer les disques virtuels) |
| `libvirt-daemon-system` | Daemon de gestion des VMs |
| `libvirt-clients` | Outils clients (virsh, virt-install) |
| `bridge-utils` | Gestion des bridges réseau |
| `virt-manager` | Interface graphique pour gérer les VMs |
| `virtinst` | Scripts d'installation de VMs |
| `ovmf` | Firmware UEFI pour les VMs |
| `genisoimage` | Création d'images ISO (cloud-init, etc.) |

### 1.2 – Vérifier l'Installation

```bash
# Vérifier que KVM est chargé
lsmod | grep kvm

# Vérifier la version de QEMU
qemu-system-x86_64 --version

# Vérifier libvirt
virsh version
```

**Résultats attendus :**
```
kvm_amd               # ou kvm_intel
kvm

QEMU emulator version 8.x.x

Compiled against library: libvirt 10.x.x
Using library: libvirt 10.x.x
```

---

## 👥 Étape 2 – Configuration des Permissions

### 2.1 – Ajouter l'Utilisateur aux Groupes Nécessaires

```bash
# Ajouter l'utilisateur aux groupes libvirt et kvm
sudo usermod -aG libvirt $USER
sudo usermod -aG kvm $USER

# Vérifier les groupes
groups $USER
```

Vous devriez voir : `ubuntu libvirt kvm ...`

### 2.2 – Démarrer et Activer les Services

```bash
# Démarrer libvirtd
sudo systemctl start libvirtd

# Activer au démarrage
sudo systemctl enable libvirtd

# Vérifier le statut
sudo systemctl status libvirtd
```

**Résultat attendu :** `active (running)` en vert ✅

### 2.3 – Configurer les Permissions sur /dev/kvm

```bash
# Vérifier les permissions sur /dev/kvm
ls -l /dev/kvm

# Devrait afficher :
# crw-rw---- 1 root kvm ... /dev/kvm
```

**Important :** Déconnectez-vous et reconnectez-vous (RDP) pour que les changements de groupe prennent effet !

```bash
# Après reconnexion, vérifier l'accès
test -r /dev/kvm && echo "✅ Accès KVM OK" || echo "❌ Accès KVM refusé"
```

---

## 🔍 Étape 3 – Vérification de l'Accélération Matérielle

### 3.1 – Test avec kvm-ok

```bash
# Vérifier que KVM est utilisable
kvm-ok
```

**Résultat attendu :**
```
INFO: /dev/kvm exists
KVM acceleration can be used
```

### 3.2 – Vérifier les Capacités de Virtualisation

```bash
# Afficher les capacités du système
virt-host-validate
```

**Analyse des résultats :**

| Message | Signification |
|---------|---------------|
| `QEMU: Checking for hardware virtualization : PASS` | ✅ CPU supporte la virtualisation |
| `QEMU: Checking if device /dev/kvm exists : PASS` | ✅ Module KVM chargé |
| `QEMU: Checking if device /dev/kvm is accessible : PASS` | ✅ Permissions correctes |
| `QEMU: Checking for cgroup 'cpu' controller support : PASS` | ✅ Isolation CPU disponible |
| `QEMU: Checking for cgroup 'memory' controller support : PASS` | ✅ Isolation mémoire disponible |

Si vous voyez des `WARN` ou `FAIL`, consultez les messages pour corriger.

### 3.3 – Test de Performance KVM vs Sans KVM

```bash
# Test AVEC accélération KVM (rapide)
time qemu-system-x86_64 -enable-kvm -m 512 -nographic -kernel /boot/vmlinuz-$(uname -r) -append "console=ttyS0" -initrd /boot/initrd.img-$(uname -r) &
sleep 5
killall qemu-system-x86_64

# Test SANS accélération KVM (lent)
time qemu-system-x86_64 -m 512 -nographic -kernel /boot/vmlinuz-$(uname -r) -append "console=ttyS0" -initrd /boot/initrd.img-$(uname -r) &
sleep 5
killall qemu-system-x86_64
```

**Différence attendue :** Le test avec `-enable-kvm` doit être **beaucoup plus rapide** (10x à 100x).

---

## 🌐 Étape 4 – Configuration Réseau de Base

### 4.1 – Vérifier le Réseau par Défaut

Libvirt crée automatiquement un réseau NAT appelé `default` :

```bash
# Lister les réseaux virtuels
virsh net-list --all
```

**Résultat attendu :**
```
 Name      State    Autostart   Persistent
--------------------------------------------
 default   active   yes         yes
```

### 4.2 – Démarrer le Réseau par Défaut (si inactif)

```bash
# Démarrer le réseau
virsh net-start default

# Activer au démarrage
virsh net-autostart default

# Vérifier la configuration
virsh net-dumpxml default
```

**Configuration typique :**
- **Réseau :** 192.168.122.0/24
- **Passerelle :** 192.168.122.1
- **DHCP :** 192.168.122.2 à 192.168.122.254
- **Mode :** NAT (les VMs peuvent accéder à Internet via l'hôte)

### 4.3 – Vérifier l'Interface Bridge

```bash
# Vérifier que le bridge virbr0 existe
ip addr show virbr0
```

**Résultat attendu :**
```
virbr0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
    inet 192.168.122.1/24 brd 192.168.122.255 scope global virbr0
```

**Pourquoi un bridge ?**  
Le bridge `virbr0` permet aux VMs de communiquer entre elles et avec l'hôte, tout en utilisant le NAT pour accéder à Internet.

---

## 🖥️ Étape 5 – Lancer virt-manager (Interface Graphique)

### 5.1 – Démarrer virt-manager

Depuis votre session RDP Ubuntu Desktop :

```bash
# Lancer virt-manager
virt-manager &
```

Ou via le menu Applications : **Applications** → **Système** → **Virtual Machine Manager**

### 5.2 – Première Configuration

1. **Connexion automatique :** virt-manager se connecte automatiquement à `qemu:///system`
2. **Vue d'ensemble :** Vous devriez voir "QEMU/KVM" dans la liste
3. **Double-cliquez** sur "QEMU/KVM" pour voir les détails

**Interface virt-manager :**
- 📊 **Vue d'ensemble :** Liste des VMs
- 📈 **Graphiques :** CPU, RAM, réseau, stockage
- ⚙️ **Détails :** Configuration matérielle de chaque VM

---

## 💿 Étape 6 – Créer Votre Première Machine Virtuelle

### 6.1 – Télécharger une Image ISO Ubuntu Server

```bash
# Créer un répertoire pour les ISOs
mkdir -p ~/iso

# Télécharger Ubuntu Server 24.04 (version minimale)
cd ~/iso
wget https://releases.ubuntu.com/24.04/ubuntu-24.04-live-server-amd64.iso

# Vérifier le téléchargement
ls -lh ubuntu-24.04-live-server-amd64.iso
```

### 6.2 – Créer la VM via virt-manager (Méthode Graphique)

1. **Cliquer sur "Créer une nouvelle machine virtuelle"** (icône en haut à gauche)

2. **Étape 1 : Méthode d'installation**
   - Sélectionner : **"Local install media (ISO image or CDROM)"**
   - Cliquer **"Forward"**

3. **Étape 2 : Choisir l'ISO**
   - Cliquer **"Browse..."**
   - Cliquer **"Browse Local"**
   - Naviguer vers `~/iso/ubuntu-24.04-live-server-amd64.iso`
   - **OS type :** Ubuntu 24.04 (détecté automatiquement)
   - Cliquer **"Forward"**

4. **Étape 3 : Mémoire et CPU**
   - **Mémoire :** 2048 MB (2GB)
   - **CPUs :** 2
   - Cliquer **"Forward"**

5. **Étape 4 : Stockage**
   - **Créer un disque :** Cocher
   - **Taille :** 20 GB
   - Cliquer **"Forward"**

6. **Étape 5 : Configuration finale**
   - **Nom :** `test-vm-01`
   - **Réseau :** Virtual network 'default' : NAT
   - ✅ Cocher **"Customize configuration before install"**
   - Cliquer **"Finish"**

7. **Personnalisation (optionnel)**
   - **Boot Options :** Vérifier que le CDROM est en premier
   - **Video :** QXL (recommandé)
   - Cliquer **"Begin Installation"**

### 6.3 – Installation d'Ubuntu dans la VM

La VM démarre sur l'ISO Ubuntu. Suivez l'installation standard :

1. **Langue :** English (ou Français)
2. **Keyboard :** French (ou votre clavier)
3. **Type d'installation :** Ubuntu Server (minimal)
4. **Réseau :** DHCP (automatique)
5. **Stockage :** Use entire disk
6. **Profil :**
   - Nom : `testuser`
   - Serveur : `test-vm-01`
   - Mot de passe : `votre_mot_de_passe`
7. **SSH :** Installer OpenSSH server ✅
8. **Snaps :** Aucun (pour l'instant)

⏱️ **L'installation prend environ 5-10 minutes.**

### 6.4 – Premier Démarrage

Après l'installation :

1. La VM redémarre automatiquement
2. **Éjecter le CDROM virtuel :**
   - Dans virt-manager, clic droit sur la VM → **"Shut Down"** → **"Force Off"**
   - Double-clic sur la VM → **"Details"** → **"SATA CDROM 1"**
   - Décocher **"Connect at boot"**
   - Cliquer **"Apply"**
3. Redémarrer la VM : **"Power On"**

4. **Se connecter :**
   - Login : `testuser`
   - Mot de passe : `votre_mot_de_passe`

🎉 **Félicitations ! Votre première VM KVM fonctionne !**

---

## 💻 Étape 7 – Maîtriser virsh (Ligne de Commande)

`virsh` est l'outil en ligne de commande pour gérer les VMs. C'est essentiel pour l'automatisation et l'administration à distance.

### 7.1 – Commandes de Base

```bash
# Lister toutes les VMs
virsh list --all

# Démarrer une VM
virsh start test-vm-01

# Arrêter proprement une VM
virsh shutdown test-vm-01

# Forcer l'arrêt (équivalent à débrancher)
virsh destroy test-vm-01

# Redémarrer une VM
virsh reboot test-vm-01

# Supprimer une VM (attention : irréversible !)
virsh undefine test-vm-01 --remove-all-storage
```

### 7.2 – Informations sur les VMs

```bash
# Afficher les informations d'une VM
virsh dominfo test-vm-01

# Afficher la configuration XML complète
virsh dumpxml test-vm-01

# Statistiques CPU
virsh cpu-stats test-vm-01

# Statistiques mémoire
virsh dommemstat test-vm-01
```

### 7.3 – Console Série (Accès Direct)

```bash
# Se connecter à la console série de la VM
virsh console test-vm-01

# Pour sortir : Ctrl + ]
```

**Pourquoi la console série ?**  
Permet d'accéder à la VM même si le réseau ne fonctionne pas (équivalent d'un écran/clavier physique).

### 7.4 – Gestion des Snapshots

```bash
# Créer un snapshot
virsh snapshot-create-as test-vm-01 \
    --name "snapshot-initial" \
    --description "État juste après installation"

# Lister les snapshots
virsh snapshot-list test-vm-01

# Restaurer un snapshot
virsh snapshot-revert test-vm-01 snapshot-initial

# Supprimer un snapshot
virsh snapshot-delete test-vm-01 snapshot-initial
```

**Utilité des snapshots :**  
Sauvegarder l'état d'une VM avant des modifications risquées (mises à jour, tests, etc.).

---

## 🔧 Étape 8 – Créer une VM via Ligne de Commande (virt-install)

### 8.1 – Créer une VM Ubuntu Minimale

```bash
# Créer une VM avec virt-install
virt-install \
    --name test-vm-02 \
    --ram 2048 \
    --vcpus 2 \
    --disk path=/var/lib/libvirt/images/test-vm-02.qcow2,size=20 \
    --os-variant ubuntu24.04 \
    --network network=default \
    --graphics vnc,listen=0.0.0.0 \
    --cdrom ~/iso/ubuntu-24.04-live-server-amd64.iso \
    --noautoconsole
```

**Explication des options :**

| Option | Description |
|--------|-------------|
| `--name` | Nom de la VM |
| `--ram` | Mémoire en MB |
| `--vcpus` | Nombre de CPUs virtuels |
| `--disk` | Chemin et taille du disque virtuel |
| `--os-variant` | Type d'OS (optimisations automatiques) |
| `--network` | Réseau à utiliser |
| `--graphics` | Type d'affichage (VNC, Spice, etc.) |
| `--cdrom` | ISO d'installation |
| `--noautoconsole` | Ne pas ouvrir la console automatiquement |

### 8.2 – Lister les OS Variants Disponibles

```bash
# Lister tous les OS supportés
osinfo-query os | grep ubuntu

# Afficher les détails d'un OS
osinfo-query os name=ubuntu24.04
```

### 8.3 – Se Connecter à la VM

```bash
# Ouvrir virt-manager et double-cliquer sur test-vm-02
virt-manager
```

Ou via VNC :

```bash
# Trouver le port VNC
virsh vncdisplay test-vm-02

# Résultat : :0 (port 5900)
# Se connecter avec un client VNC à : VOTRE_IP:5900
```

---

## 📊 Étape 9 – Gestion du Stockage

### 9.1 – Pools de Stockage

```bash
# Lister les pools de stockage
virsh pool-list --all

# Afficher les détails du pool par défaut
virsh pool-info default

# Lister les volumes dans un pool
virsh vol-list default
```

### 9.2 – Créer un Nouveau Pool de Stockage

```bash
# Créer un répertoire pour le nouveau pool
sudo mkdir -p /var/lib/libvirt/images/custom-pool

# Définir le pool
virsh pool-define-as custom-pool dir \
    --target /var/lib/libvirt/images/custom-pool

# Construire le pool
virsh pool-build custom-pool

# Démarrer le pool
virsh pool-start custom-pool

# Activer au démarrage
virsh pool-autostart custom-pool

# Vérifier
virsh pool-list --all
```

### 9.3 – Créer un Disque Virtuel

```bash
# Créer un disque qcow2 de 10GB
virsh vol-create-as custom-pool disk-01.qcow2 10G \
    --format qcow2

# Lister les disques
virsh vol-list custom-pool

# Afficher les informations
virsh vol-info --pool custom-pool disk-01.qcow2
```

**Format qcow2 vs raw :**

| Format | Avantages | Inconvénients |
|--------|-----------|---------------|
| **qcow2** | Compression, snapshots, allocation dynamique | Légèrement plus lent |
| **raw** | Performances maximales | Pas de snapshots, taille fixe |

---

## 🌐 Étape 10 – Gestion Réseau Avancée

### 10.1 – Créer un Réseau Isolé

```bash
# Créer un fichier de configuration réseau
cat > ~/isolated-network.xml << 'EOF'
<network>
  <name>isolated-net</name>
  <bridge name='virbr1'/>
  <ip address='192.168.100.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.100.10' end='192.168.100.100'/>
    </dhcp>
  </ip>
</network>
EOF

# Définir le réseau
virsh net-define ~/isolated-network.xml

# Démarrer le réseau
virsh net-start isolated-net

# Activer au démarrage
virsh net-autostart isolated-net

# Vérifier
virsh net-list --all
```

**Différence Isolé vs NAT :**
- **Isolé :** Les VMs communiquent entre elles uniquement (pas d'accès Internet)
- **NAT :** Les VMs peuvent accéder à Internet via l'hôte

### 10.2 – Attacher une VM au Réseau Isolé

```bash
# Méthode 1 : Lors de la création
virt-install ... --network network=isolated-net ...

# Méthode 2 : Modifier une VM existante
virsh attach-interface test-vm-01 network isolated-net \
    --model virtio --config --live
```

---

## 🧪 Étape 11 – Tests et Vérifications

### 11.1 – Script de Diagnostic KVM

```bash
# Créer un script de diagnostic complet
cat > ~/check-kvm.sh << 'EOF'
#!/bin/bash
echo "=== 🔧 Diagnostic KVM Complet ==="
echo ""
echo "📦 Version QEMU/KVM :"
qemu-system-x86_64 --version | head -1
echo ""
echo "📦 Version libvirt :"
virsh version --short
echo ""
echo "✅ Accélération KVM :"
kvm-ok
echo ""
echo "🖥️ VMs en cours d'exécution :"
virsh list
echo ""
echo "💾 Pools de stockage :"
virsh pool-list
echo ""
echo "🌐 Réseaux virtuels :"
virsh net-list
echo ""
echo "📊 Utilisation des ressources :"
free -h | grep Mem
echo ""
echo "✅ Diagnostic terminé !"
EOF

chmod +x ~/check-kvm.sh
~/check-kvm.sh
```

### 11.2 – Tester la Performance d'une VM

Depuis l'intérieur de `test-vm-01` :

```bash
# Se connecter à la VM
virsh console test-vm-01

# Installer sysbench
sudo apt update && sudo apt install sysbench -y

# Test CPU
sysbench cpu --cpu-max-prime=20000 run

# Test mémoire
sysbench memory --memory-total-size=1G run
```

**Comparer avec l'hôte :**  
Les performances devraient être proches (90-95%) grâce à l'accélération KVM.

---

## 📝 Récapitulatif du TP2

Dans ce TP, vous avez appris à :

- ✅ Comprendre la différence entre **hyperviseurs Type-1 et Type-2**
- ✅ Installer **QEMU/KVM** et **libvirt** complets
- ✅ Configurer **virt-manager** pour la gestion graphique
- ✅ Vérifier l'**accélération matérielle KVM**
- ✅ Créer des VMs via **interface graphique** et **ligne de commande**
- ✅ Maîtriser **virsh** pour l'administration
- ✅ Gérer les **pools de stockage** et **réseaux virtuels**
- ✅ Créer des **snapshots** pour sauvegarder l'état des VMs

### 🎯 Checklist de Vérification Finale

Avant de passer au TP3, vérifiez que :

- [ ] `kvm-ok` retourne "KVM acceleration can be used"
- [ ] `virt-host-validate` affiche tous les tests en PASS
- [ ] Vous avez au moins une VM fonctionnelle (`virsh list`)
- [ ] Le réseau `default` est actif (`virsh net-list`)
- [ ] Vous pouvez créer des snapshots (`virsh snapshot-list`)
- [ ] `virt-manager` s'ouvre sans erreur

### 📚 Commandes Essentielles à Retenir

| Commande | Description |
|----------|-------------|
| `virsh list --all` | Lister toutes les VMs |
| `virsh start <vm>` | Démarrer une VM |
| `virsh shutdown <vm>` | Arrêter proprement une VM |
| `virsh console <vm>` | Accéder à la console série |
| `virsh snapshot-create-as <vm> <nom>` | Créer un snapshot |
| `virt-install ...` | Créer une VM en ligne de commande |
| `virsh net-list` | Lister les réseaux virtuels |
| `virsh pool-list` | Lister les pools de stockage |

---

## 🚀 Prochaine Étape : TP3

Vous êtes maintenant prêt pour le **TP3 : Architecture Réseau et Stockage (Le Lab Complet)** où vous allez :

- Créer un **serveur de stockage NFS** dans une VM
- Configurer des **réseaux isolés** et **NAT**
- Déployer **3 VMs Ubuntu Light** interconnectées
- Maîtriser les **volumes partagés** et la **communication inter-VM**
- Construire une **architecture complète** de lab

**Votre fondation KVM est solide ! 🎉**
