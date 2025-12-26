# 🖥️ TP1 – Préparation de la Station de Travail (Host L1)

**Objectif du TP :**  
Transformer votre instance Oracle Cloud en une station de travail complète pour la virtualisation imbriquée.  
À la fin du TP, vous aurez :

- ✅ Installé **Ubuntu 24.04 Desktop** sur Oracle Cloud
- ✅ Configuré **XRDP** pour l'accès graphique distant depuis Windows
- ✅ Optimisé le système pour la **virtualisation imbriquée** (Nested Virtualization)
- ✅ Vérifié les capacités matérielles de votre serveur

> 💡 **Pourquoi ce TP ?**  
> Oracle Cloud fournit des instances puissantes, mais par défaut elles n'ont pas d'interface graphique. Ce TP transforme votre serveur en une vraie station de travail accessible depuis n'importe où, optimisée pour héberger plusieurs couches d'hyperviseurs.

---

## 📋 Prérequis

- Une instance Oracle Cloud (AMD 4 OCPUs, 48GB RAM)
- Accès SSH à votre instance
- Un client RDP sur Windows (Bureau à distance intégré)
- Connexion Internet stable

**Caractéristiques de votre serveur Oracle Cloud :**
```bash
# Vérifier les ressources disponibles
free -h          # RAM : ~48GB
nproc            # CPUs : 4
lsblk            # Stockage disponible
```

---

## 🏗️ Architecture et Concept

### Qu'est-ce qu'une Station de Travail L1 (Layer 1) ?

```
┌─────────────────────────────────────────────┐
│  🖥️ Votre PC Windows (Client RDP)          │
│                                             │
│  Connexion via Bureau à Distance (RDP)     │
└──────────────────┬──────────────────────────┘
                   │ Internet
                   ▼
┌─────────────────────────────────────────────┐
│  ☁️ Oracle Cloud Instance                   │
│  ┌───────────────────────────────────────┐  │
│  │ Ubuntu 24.04 Desktop + XRDP (L1)     │  │
│  │                                       │  │
│  │ → Interface graphique complète       │  │
│  │ → Optimisé pour virtualisation       │  │
│  │ → Base pour KVM/QEMU                 │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

**Pourquoi Ubuntu Desktop et pas Server ?**
- 🎨 Interface graphique pour gérer facilement les VMs avec `virt-manager`
- 🖱️ Expérience utilisateur similaire à votre PC local
- 🔧 Outils de développement et de diagnostic visuels

---

## 🚀 Étape 1 – Connexion SSH et Mise à Jour du Système

Connectez-vous à votre instance Oracle Cloud via SSH :

```bash
# Depuis PowerShell ou CMD sur Windows
ssh ubuntu@VOTRE_IP_PUBLIQUE
```

Une fois connecté, mettez à jour le système :

```bash
# Mise à jour de la liste des paquets
sudo apt update

# Mise à niveau de tous les paquets installés
sudo apt upgrade -y

# Nettoyage des paquets inutiles
sudo apt autoremove -y
```

**Pourquoi cette étape ?**  
- ✅ Garantir que tous les paquets sont à jour
- ✅ Corriger les failles de sécurité
- ✅ Préparer le système pour les installations suivantes

---

## 🎨 Étape 2 – Installation d'Ubuntu Desktop

Par défaut, Oracle Cloud fournit Ubuntu Server (sans interface graphique). Nous allons installer l'environnement de bureau complet.

```bash
# Installation de l'environnement de bureau Ubuntu (GNOME)
sudo apt install ubuntu-desktop -y
```

⏱️ **Cette installation prend environ 10-15 minutes** selon votre connexion.

**Que fait cette commande ?**
- Installe GNOME Desktop Environment
- Installe tous les outils graphiques (Firefox, Terminal, Fichiers, etc.)
- Configure le gestionnaire de connexion graphique (GDM)

**Vérifier l'installation :**

```bash
# Vérifier que le service graphique est actif
systemctl status gdm3
```

Vous devriez voir `active (running)` en vert.

---

## 🔐 Étape 3 – Installation et Configuration de XRDP

XRDP permet de se connecter à distance via le protocole RDP (Remote Desktop Protocol), natif sur Windows.

### 3.1 – Installation de XRDP

```bash
# Installer XRDP et ses dépendances
sudo apt install xrdp -y

# Ajouter l'utilisateur xrdp au groupe ssl-cert pour les certificats
sudo adduser xrdp ssl-cert

# Démarrer et activer XRDP au démarrage
sudo systemctl enable xrdp
sudo systemctl start xrdp
```

### 3.2 – Configuration de XRDP pour GNOME

Par défaut, XRDP peut avoir des problèmes avec GNOME. Nous allons créer une configuration optimale :

```bash
# Créer un fichier de configuration pour la session GNOME
echo "gnome-session" > ~/.xsession

# Donner les permissions appropriées
chmod +x ~/.xsession

# Configurer XRDP pour utiliser la session GNOME
sudo sed -i 's/^new_cursors=true/new_cursors=false/' /etc/xrdp/xrdp.ini
```

### 3.3 – Configuration du Pare-feu Oracle Cloud

**Important :** Oracle Cloud bloque tous les ports par défaut. Nous devons ouvrir le port RDP (3389).

```bash
# Ouvrir le port 3389 dans le pare-feu local (UFW)
sudo ufw allow 3389/tcp
sudo ufw enable
sudo ufw status
```

**Configuration dans l'interface Oracle Cloud :**

1. Connectez-vous à la console Oracle Cloud
2. Allez dans **Networking** → **Virtual Cloud Networks**
3. Sélectionnez votre VCN
4. Cliquez sur **Security Lists** → **Default Security List**
5. Cliquez sur **Add Ingress Rules**
6. Ajoutez :
   - **Source CIDR :** `0.0.0.0/0` (ou votre IP pour plus de sécurité)
   - **IP Protocol :** TCP
   - **Destination Port Range :** `3389`
   - **Description :** RDP Access

### 3.4 – Redémarrer XRDP

```bash
# Redémarrer le service XRDP pour appliquer les changements
sudo systemctl restart xrdp

# Vérifier que XRDP écoute sur le port 3389
sudo netstat -tulpn | grep 3389
```

Vous devriez voir :
```
tcp6       0      0 :::3389                 :::*                    LISTEN      1234/xrdp
```

---

## 🖥️ Étape 4 – Connexion via Bureau à Distance (Windows)

### 4.1 – Ouvrir le Bureau à Distance Windows

1. Appuyez sur `Windows + R`
2. Tapez `mstsc` et appuyez sur Entrée
3. Dans **Ordinateur**, entrez : `VOTRE_IP_PUBLIQUE:3389`
4. Cliquez sur **Connexion**

### 4.2 – Authentification

- **Nom d'utilisateur :** `ubuntu` (ou votre nom d'utilisateur)
- **Mot de passe :** Votre mot de passe SSH

### 4.3 – Première Connexion

🎉 **Félicitations !** Vous devriez maintenant voir le bureau Ubuntu complet dans une fenêtre Windows !

**Optimisations recommandées dans RDP :**
- Résolution : Adaptez à votre écran (1920x1080 recommandé)
- Couleurs : True Color (32 bits)
- Expérience : Décochez "Arrière-plan du bureau" pour de meilleures performances

---

## ⚡ Étape 5 – Optimisation pour la Virtualisation Imbriquée

### 5.1 – Vérifier le Support de la Virtualisation

```bash
# Vérifier si le CPU supporte la virtualisation matérielle
egrep -c '(vmx|svm)' /proc/cpuinfo
```

- **Résultat > 0 :** ✅ Virtualisation supportée
- **Résultat = 0 :** ❌ Virtualisation non disponible (vérifier les paramètres Oracle Cloud)

```bash
# Vérifier si KVM est disponible
kvm-ok
```

Si `kvm-ok` n'est pas installé :
```bash
sudo apt install cpu-checker -y
kvm-ok
```

Vous devriez voir :
```
INFO: /dev/kvm exists
KVM acceleration can be used
```

### 5.2 – Activer la Virtualisation Imbriquée pour KVM

```bash
# Vérifier le module KVM chargé
lsmod | grep kvm
```

Pour AMD (votre cas avec 4 OCPUs AMD) :
```bash
# Vérifier si nested est activé
cat /sys/module/kvm_amd/parameters/nested
```

Si le résultat est `N` ou `0`, activez-le :

```bash
# Créer un fichier de configuration pour activer nested
echo "options kvm_amd nested=1" | sudo tee /etc/modprobe.d/kvm-nested.conf

# Recharger le module KVM
sudo modprobe -r kvm_amd
sudo modprobe kvm_amd

# Vérifier à nouveau
cat /sys/module/kvm_amd/parameters/nested
```

Résultat attendu : `Y` ou `1` ✅

**Pour Intel (si applicable) :**
```bash
echo "options kvm_intel nested=1" | sudo tee /etc/modprobe.d/kvm-nested.conf
sudo modprobe -r kvm_intel
sudo modprobe kvm_intel
cat /sys/module/kvm_intel/parameters/nested
```

### 5.3 – Optimisations Système

```bash
# Augmenter les limites de mémoire pour les VMs
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Optimiser le swap (réduire l'utilisation du swap)
echo "vm.swappiness=10" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Vérifier les paramètres appliqués
sysctl vm.max_map_count
sysctl vm.swappiness
```

**Pourquoi ces optimisations ?**
- `max_map_count` : Permet aux VMs d'allouer plus de zones mémoire
- `swappiness=10` : Privilégie l'utilisation de la RAM plutôt que le swap (important avec 48GB de RAM)

---

## 🔍 Étape 6 – Vérification Complète du Système

### 6.1 – Informations Système

```bash
# Afficher toutes les informations système
neofetch
```

Si `neofetch` n'est pas installé :
```bash
sudo apt install neofetch -y
neofetch
```

### 6.2 – Vérifier les Ressources

```bash
# CPU
lscpu | grep -E "Model name|CPU\(s\)|Virtualization"

# RAM
free -h

# Stockage
df -h

# Réseau
ip addr show
```

### 6.3 – Créer un Script de Diagnostic

Créez un script pour vérifier rapidement l'état du système :

```bash
# Créer le script
cat > ~/check-virt.sh << 'EOF'
#!/bin/bash
echo "=== 🖥️ Vérification de la Station de Travail L1 ==="
echo ""
echo "📊 CPU et Virtualisation :"
egrep -c '(vmx|svm)' /proc/cpuinfo
kvm-ok
echo ""
echo "💾 RAM Disponible :"
free -h | grep Mem
echo ""
echo "💿 Stockage :"
df -h / | tail -1
echo ""
echo "🔧 Nested Virtualization (AMD) :"
cat /sys/module/kvm_amd/parameters/nested 2>/dev/null || echo "Module KVM Intel ou non chargé"
echo ""
echo "🌐 XRDP Status :"
systemctl is-active xrdp
echo ""
echo "✅ Vérification terminée !"
EOF

# Rendre le script exécutable
chmod +x ~/check-virt.sh

# Exécuter le script
~/check-virt.sh
```

---

## 🎯 Étape 7 – Configuration de l'Environnement de Travail

### 7.1 – Installer les Outils Essentiels

```bash
# Outils système et réseau
sudo apt install -y \
    htop \
    iotop \
    net-tools \
    curl \
    wget \
    git \
    vim \
    tree \
    tmux \
    screen

# Outils de virtualisation (préparation pour TP2)
sudo apt install -y \
    qemu-kvm \
    libvirt-daemon-system \
    libvirt-clients \
    bridge-utils \
    virt-manager
```

**Explication des paquets :**
- `htop` : Moniteur système interactif
- `net-tools` : Outils réseau (ifconfig, netstat, etc.)
- `qemu-kvm` : Émulateur et hyperviseur (pour TP2)
- `virt-manager` : Interface graphique pour gérer les VMs

### 7.2 – Ajouter votre Utilisateur au Groupe Libvirt

```bash
# Ajouter l'utilisateur au groupe libvirt et kvm
sudo usermod -aG libvirt $USER
sudo usermod -aG kvm $USER

# Vérifier les groupes
groups $USER
```

**Déconnectez-vous et reconnectez-vous** (RDP) pour que les changements prennent effet.

### 7.3 – Configurer le Terminal

```bash
# Installer Terminator (terminal avancé)
sudo apt install terminator -y

# Configurer bash avec des alias utiles
cat >> ~/.bashrc << 'EOF'

# Alias personnalisés pour la virtualisation
alias vms='virsh list --all'
alias vmstart='virsh start'
alias vmstop='virsh shutdown'
alias vminfo='virsh dominfo'
alias checkvm='~/check-virt.sh'

# Alias système
alias ll='ls -lah'
alias update='sudo apt update && sudo apt upgrade -y'
EOF

# Recharger la configuration
source ~/.bashrc
```

---

## 📊 Étape 8 – Test de Performance

### 8.1 – Test CPU

```bash
# Installer sysbench
sudo apt install sysbench -y

# Test CPU (calcul de nombres premiers)
sysbench cpu --cpu-max-prime=20000 --threads=4 run
```

**Résultats attendus :**
- Events per second : ~1000-2000 (selon le CPU AMD)
- Total time : ~10 secondes

### 8.2 – Test Mémoire

```bash
# Test de bande passante mémoire
sysbench memory --memory-total-size=10G --threads=4 run
```

### 8.3 – Test Disque

```bash
# Préparer le test
sysbench fileio --file-total-size=5G prepare

# Exécuter le test de lecture/écriture aléatoire
sysbench fileio --file-total-size=5G --file-test-mode=rndrw --threads=4 run

# Nettoyer
sysbench fileio --file-total-size=5G cleanup
```

---

## 🔒 Étape 9 – Sécurisation (Optionnel mais Recommandé)

### 9.1 – Changer le Port RDP par Défaut

```bash
# Modifier le port XRDP (exemple : 3390 au lieu de 3389)
sudo sed -i 's/port=3389/port=3390/' /etc/xrdp/xrdp.ini

# Redémarrer XRDP
sudo systemctl restart xrdp

# Mettre à jour le pare-feu
sudo ufw allow 3390/tcp
sudo ufw delete allow 3389/tcp
```

**N'oubliez pas de mettre à jour la règle dans Oracle Cloud Security List !**

### 9.2 – Configurer un Pare-feu Applicatif

```bash
# Vérifier l'état du pare-feu
sudo ufw status verbose

# Règles recommandées
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 3389/tcp  # ou 3390 si vous avez changé
sudo ufw enable
```

---

## 🧹 Étape 10 – Nettoyage et Optimisation Finale

```bash
# Nettoyer les paquets inutiles
sudo apt autoremove -y
sudo apt autoclean

# Vider le cache APT
sudo apt clean

# Vérifier l'espace disque libéré
df -h /
```

---

## 📝 Récapitulatif du TP1

Dans ce TP, vous avez appris à :

- ✅ Installer **Ubuntu 24.04 Desktop** sur Oracle Cloud
- ✅ Configurer **XRDP** pour l'accès distant graphique
- ✅ Activer la **virtualisation imbriquée** (Nested Virtualization)
- ✅ Optimiser le système pour la virtualisation
- ✅ Installer les outils essentiels pour la gestion de VMs
- ✅ Tester les performances de votre serveur
- ✅ Sécuriser l'accès distant

### 🎯 Checklist de Vérification Finale

Avant de passer au TP2, vérifiez que :

- [ ] Vous pouvez vous connecter via RDP depuis Windows
- [ ] `kvm-ok` retourne "KVM acceleration can be used"
- [ ] `cat /sys/module/kvm_amd/parameters/nested` retourne `Y` ou `1`
- [ ] `virsh list --all` fonctionne sans erreur
- [ ] Vous avez 48GB de RAM disponible (`free -h`)
- [ ] L'espace disque est suffisant (>50GB libres)

### 📚 Concepts Clés Appris

| Concept | Description |
|---------|-------------|
| **L1 Host** | Système hôte principal qui hébergera les hyperviseurs |
| **XRDP** | Protocole permettant l'accès graphique distant via RDP |
| **Nested Virtualization** | Capacité d'exécuter des hyperviseurs dans des VMs |
| **KVM** | Hyperviseur Linux basé sur le noyau (Type-1) |
| **QEMU** | Émulateur utilisé avec KVM pour la virtualisation |

---

## 🚀 Prochaine Étape : TP2

Vous êtes maintenant prêt pour le **TP2 : Installation et Fondations KVM** où vous allez :

- Installer et configurer QEMU/KVM complet
- Comprendre la différence entre hyperviseurs Type-1 et Type-2
- Créer votre première machine virtuelle
- Découvrir `virt-manager` (interface graphique)

**Votre station de travail L1 est opérationnelle ! 🎉**
