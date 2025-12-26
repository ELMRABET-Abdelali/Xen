# 🏗️ TP3 – Architecture Réseau et Stockage (Le Lab Complet)

**Objectif du TP :**  
Construire une infrastructure complète avec stockage partagé et réseaux multiples.  
À la fin du TP, vous aurez :

- ✅ Créé un **serveur de stockage NFS** dans une VM
- ✅ Configuré des **réseaux isolés** et **NAT**
- ✅ Déployé **3 VMs Ubuntu Light** interconnectées
- ✅ Maîtrisé les **volumes partagés** et **snapshots**
- ✅ Compris la **communication inter-VM**
- ✅ Construit une **architecture de lab professionnelle**

> 💡 **Pourquoi ce TP ?**  
> Ce TP simule une infrastructure réelle : un serveur de stockage centralisé (NFS) servant plusieurs machines clientes. Vous apprendrez à segmenter les réseaux, partager des ressources, et gérer une architecture multi-VM cohérente.

---

## 📋 Prérequis

- ✅ TP1 et TP2 terminés
- ✅ KVM fonctionnel avec au moins une VM de test
- ✅ Au moins 40GB d'espace disque libre
- ✅ Connexion RDP active à votre instance Oracle Cloud

**Vérification rapide :**
```bash
# Vérifier les ressources disponibles
~/check-kvm.sh

# Vérifier l'espace disque
df -h /var/lib/libvirt/images
```

---

## 🏗️ Architecture Cible du Lab

### Vue d'Ensemble

```
┌────────────────────────────────────────────────────────────────┐
│                    Oracle Cloud Instance (L1)                  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                   Réseau NAT (Internet)                  │  │
│  │                   192.168.122.0/24                       │  │
│  │                   virbr0 (default)                       │  │
│  │                                                          │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │  │
│  │  │ Ubuntu-01   │  │ Ubuntu-02   │  │ Ubuntu-03   │    │  │
│  │  │ Client NFS  │  │ Client NFS  │  │ Client NFS  │    │  │
│  │  │ .122.10     │  │ .122.11     │  │ .122.12     │    │  │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘    │  │
│  │         │                │                │            │  │
│  │         └────────────────┴────────────────┘            │  │
│  │                          │                             │  │
│  └──────────────────────────┼─────────────────────────────┘  │
│                             │                                │
│  ┌──────────────────────────┼─────────────────────────────┐  │
│  │              Réseau Isolé (Stockage)                   │  │
│  │              192.168.100.0/24                          │  │
│  │              virbr1 (isolated-net)                     │  │
│  │                          │                             │  │
│  │  ┌───────────────────────┴────────────────────┐       │  │
│  │  │        VM Serveur de Stockage NFS          │       │  │
│  │  │        192.168.100.10                      │       │  │
│  │  │        /srv/nfs/shared (100GB)             │       │  │
│  │  └────────────────────────────────────────────┘       │  │
│  └────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
```

### Concepts Clés

#### 🔷 Réseau NAT (default)
- **Rôle :** Accès Internet pour les VMs clientes
- **Plage :** 192.168.122.0/24
- **Passerelle :** 192.168.122.1 (hôte L1)
- **DHCP :** Oui (automatique)

#### 🔶 Réseau Isolé (isolated-net)
- **Rôle :** Communication privée pour le stockage
- **Plage :** 192.168.100.0/24
- **Passerelle :** Aucune (isolé)
- **DHCP :** Oui (pour simplifier)

#### 💾 Serveur NFS
- **Rôle :** Stockage centralisé partagé
- **Export :** `/srv/nfs/shared`
- **Clients :** Ubuntu-01, Ubuntu-02, Ubuntu-03
- **Montage :** `/mnt/shared` sur chaque client

---

## 🚀 Étape 1 – Créer les Réseaux Virtuels

### 1.1 – Vérifier le Réseau NAT par Défaut

```bash
# Vérifier que le réseau default existe et est actif
virsh net-list --all

# Afficher la configuration
virsh net-dumpxml default
```

### 1.2 – Créer le Réseau Isolé pour le Stockage

```bash
# Créer le fichier de configuration du réseau isolé
cat > ~/storage-network.xml << 'EOF'
<network>
  <name>storage-net</name>
  <bridge name='virbr1'/>
  <ip address='192.168.100.1' netmask='255.255.255.0'>
    <dhcp>
      <range start='192.168.100.10' end='192.168.100.100'/>
    </dhcp>
  </ip>
</network>
EOF

# Définir le réseau dans libvirt
virsh net-define ~/storage-network.xml

# Démarrer le réseau
virsh net-start storage-net

# Activer au démarrage
virsh net-autostart storage-net

# Vérifier
virsh net-list --all
```

**Résultat attendu :**
```
 Name          State    Autostart   Persistent
------------------------------------------------
 default       active   yes         yes
 storage-net   active   yes         yes
```

### 1.3 – Vérifier les Bridges Réseau

```bash
# Lister les interfaces réseau
ip addr show | grep virbr

# Vérifier virbr0 (default)
ip addr show virbr0

# Vérifier virbr1 (storage-net)
ip addr show virbr1
```

**Pourquoi deux réseaux ?**
- **Séparation des flux :** Trafic Internet vs trafic stockage
- **Sécurité :** Le réseau de stockage est isolé
- **Performance :** Éviter la congestion sur un seul réseau

---

## 💾 Étape 2 – Créer le Serveur de Stockage NFS

### 2.1 – Télécharger Ubuntu Server Cloud Image

Pour gagner du temps, nous utilisons une image cloud pré-installée :

```bash
# Créer un répertoire pour les images cloud
mkdir -p ~/cloud-images

# Télécharger Ubuntu 24.04 Cloud Image
cd ~/cloud-images
wget https://cloud-images.ubuntu.com/releases/24.04/release/ubuntu-24.04-server-cloudimg-amd64.img

# Vérifier le téléchargement
ls -lh ubuntu-24.04-server-cloudimg-amd64.img
```

**Qu'est-ce qu'une Cloud Image ?**
- Image pré-installée et optimisée
- Prête à l'emploi (pas d'installation manuelle)
- Configurée via cloud-init (automatisation)

### 2.2 – Créer le Disque pour le Serveur NFS

```bash
# Créer un répertoire pour les disques du serveur NFS
sudo mkdir -p /var/lib/libvirt/images/nfs-server

# Copier l'image cloud comme disque de base
sudo cp ~/cloud-images/ubuntu-24.04-server-cloudimg-amd64.img \
    /var/lib/libvirt/images/nfs-server/nfs-server-os.qcow2

# Redimensionner le disque OS à 20GB
sudo qemu-img resize \
    /var/lib/libvirt/images/nfs-server/nfs-server-os.qcow2 20G

# Créer un disque de données pour le stockage NFS (100GB)
sudo qemu-img create -f qcow2 \
    /var/lib/libvirt/images/nfs-server/nfs-server-data.qcow2 100G

# Vérifier
sudo qemu-img info /var/lib/libvirt/images/nfs-server/nfs-server-os.qcow2
sudo qemu-img info /var/lib/libvirt/images/nfs-server/nfs-server-data.qcow2
```

### 2.3 – Créer le Fichier cloud-init

Cloud-init permet de configurer automatiquement la VM au premier démarrage :

```bash
# Créer le fichier de configuration cloud-init
cat > ~/nfs-server-cloud-init.yaml << 'EOF'
#cloud-config
hostname: nfs-server
fqdn: nfs-server.local

# Utilisateur par défaut
users:
  - name: ubuntu
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    ssh_authorized_keys:
      - ssh-rsa AAAAB3... # Optionnel : ajoutez votre clé SSH
    lock_passwd: false
    passwd: $6$rounds=4096$saltsalt$hashed_password  # Remplacer par un hash

# Mot de passe simple pour le lab (INSECURE - uniquement pour le lab !)
chpasswd:
  list: |
    ubuntu:password123
  expire: False

# Packages à installer
packages:
  - nfs-kernel-server
  - nfs-common
  - net-tools
  - htop

# Commandes à exécuter au premier démarrage
runcmd:
  - systemctl enable nfs-server
  - systemctl start nfs-server
  - mkdir -p /srv/nfs/shared
  - chown nobody:nogroup /srv/nfs/shared
  - chmod 777 /srv/nfs/shared
  - echo "/srv/nfs/shared 192.168.100.0/24(rw,sync,no_subtree_check,no_root_squash)" >> /etc/exports
  - exportfs -a
  - systemctl restart nfs-server

# Mise à jour du système
package_update: true
package_upgrade: true
EOF

# Générer l'ISO cloud-init
sudo apt install cloud-image-utils -y
cloud-localds ~/nfs-server-cloud-init.iso ~/nfs-server-cloud-init.yaml

# Déplacer l'ISO dans le répertoire libvirt
sudo mv ~/nfs-server-cloud-init.iso /var/lib/libvirt/images/nfs-server/
```

**Que fait cloud-init ?**
1. Configure le hostname : `nfs-server`
2. Crée l'utilisateur `ubuntu` avec mot de passe `password123`
3. Installe le serveur NFS
4. Crée le répertoire de partage `/srv/nfs/shared`
5. Configure l'export NFS pour le réseau 192.168.100.0/24
6. Démarre le service NFS

### 2.4 – Créer la VM Serveur NFS

```bash
# Créer la VM avec virt-install
sudo virt-install \
    --name nfs-server \
    --ram 2048 \
    --vcpus 2 \
    --disk path=/var/lib/libvirt/images/nfs-server/nfs-server-os.qcow2,device=disk,bus=virtio \
    --disk path=/var/lib/libvirt/images/nfs-server/nfs-server-data.qcow2,device=disk,bus=virtio \
    --disk path=/var/lib/libvirt/images/nfs-server/nfs-server-cloud-init.iso,device=cdrom \
    --os-variant ubuntu24.04 \
    --network network=storage-net,model=virtio \
    --graphics none \
    --console pty,target_type=serial \
    --import \
    --noautoconsole

# Attendre 2 minutes que cloud-init se termine
echo "⏳ Attente de l'initialisation cloud-init (2 minutes)..."
sleep 120

# Vérifier que la VM est démarrée
virsh list
```

### 2.5 – Se Connecter au Serveur NFS

```bash
# Trouver l'adresse IP du serveur NFS
virsh domifaddr nfs-server

# Si la commande ci-dessus ne fonctionne pas, se connecter via console
virsh console nfs-server

# Login : ubuntu
# Password : password123

# Vérifier l'IP (devrait être 192.168.100.x)
ip addr show

# Vérifier que NFS fonctionne
sudo systemctl status nfs-server

# Vérifier les exports NFS
sudo exportfs -v

# Sortir de la console : Ctrl + ]
```

**Résultat attendu :**
```
/srv/nfs/shared
        192.168.100.0/24(rw,wdelay,no_root_squash,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash)
```

---

## 🖥️ Étape 3 – Créer les VMs Clientes Ubuntu Light

### 3.1 – Préparer les Disques pour les 3 VMs

```bash
# Créer un répertoire pour chaque VM cliente
for i in {1..3}; do
    sudo mkdir -p /var/lib/libvirt/images/ubuntu-0$i
done

# Copier l'image cloud pour chaque VM
for i in {1..3}; do
    sudo cp ~/cloud-images/ubuntu-24.04-server-cloudimg-amd64.img \
        /var/lib/libvirt/images/ubuntu-0$i/ubuntu-0$i-os.qcow2
    
    # Redimensionner à 15GB
    sudo qemu-img resize \
        /var/lib/libvirt/images/ubuntu-0$i/ubuntu-0$i-os.qcow2 15G
done

# Vérifier
ls -lh /var/lib/libvirt/images/ubuntu-0*/
```

### 3.2 – Créer les Fichiers cloud-init pour Chaque VM

```bash
# Fonction pour créer le cloud-init d'une VM cliente
create_client_cloudinit() {
    local vm_num=$1
    local vm_name="ubuntu-0$vm_num"
    
    cat > ~/${vm_name}-cloud-init.yaml << EOF
#cloud-config
hostname: ${vm_name}
fqdn: ${vm_name}.local

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
  - nfs-common
  - net-tools
  - htop
  - curl

runcmd:
  - mkdir -p /mnt/shared
  - echo "192.168.100.10:/srv/nfs/shared /mnt/shared nfs defaults 0 0" >> /etc/fstab
  - mount -a
  - echo "✅ NFS monté avec succès" > /home/ubuntu/nfs-status.txt

package_update: true
package_upgrade: true
EOF

    # Générer l'ISO cloud-init
    cloud-localds ~/${vm_name}-cloud-init.iso ~/${vm_name}-cloud-init.yaml
    
    # Déplacer dans le répertoire libvirt
    sudo mv ~/${vm_name}-cloud-init.iso /var/lib/libvirt/images/${vm_name}/
}

# Créer les cloud-init pour les 3 VMs
for i in {1..3}; do
    create_client_cloudinit $i
done
```

### 3.3 – Créer les 3 VMs Clientes

```bash
# Fonction pour créer une VM cliente
create_client_vm() {
    local vm_num=$1
    local vm_name="ubuntu-0$vm_num"
    
    sudo virt-install \
        --name ${vm_name} \
        --ram 1024 \
        --vcpus 1 \
        --disk path=/var/lib/libvirt/images/${vm_name}/${vm_name}-os.qcow2,device=disk,bus=virtio \
        --disk path=/var/lib/libvirt/images/${vm_name}/${vm_name}-cloud-init.iso,device=cdrom \
        --os-variant ubuntu24.04 \
        --network network=default,model=virtio \
        --network network=storage-net,model=virtio \
        --graphics none \
        --console pty,target_type=serial \
        --import \
        --noautoconsole
    
    echo "✅ VM ${vm_name} créée"
}

# Créer les 3 VMs
for i in {1..3}; do
    create_client_vm $i
    sleep 10  # Attendre entre chaque création
done

# Attendre que cloud-init se termine sur toutes les VMs
echo "⏳ Attente de l'initialisation cloud-init (3 minutes)..."
sleep 180

# Vérifier que toutes les VMs sont démarrées
virsh list
```

**Résultat attendu :**
```
 Id   Name         State
-----------------------------
 1    nfs-server   running
 2    ubuntu-01    running
 3    ubuntu-02    running
 4    ubuntu-03    running
```

---

## 🔍 Étape 4 – Vérifier la Configuration Réseau

### 4.1 – Vérifier les Adresses IP

```bash
# Afficher les IPs de toutes les VMs
for vm in nfs-server ubuntu-01 ubuntu-02 ubuntu-03; do
    echo "=== $vm ==="
    virsh domifaddr $vm
    echo ""
done
```

### 4.2 – Tester la Connectivité Réseau

```bash
# Se connecter à ubuntu-01
virsh console ubuntu-01

# Login : ubuntu / password123

# Vérifier les interfaces réseau
ip addr show

# Vous devriez voir :
# - ens3 : 192.168.122.x (réseau NAT)
# - ens4 : 192.168.100.x (réseau stockage)

# Tester la connectivité vers le serveur NFS
ping -c 3 192.168.100.10

# Tester la connectivité Internet
ping -c 3 8.8.8.8

# Sortir : Ctrl + ]
```

### 4.3 – Créer un Script de Diagnostic Réseau

```bash
# Créer un script pour tester la connectivité de toutes les VMs
cat > ~/test-network.sh << 'EOF'
#!/bin/bash

echo "=== 🌐 Test de Connectivité Réseau ==="
echo ""

for vm in ubuntu-01 ubuntu-02 ubuntu-03; do
    echo "📡 Test de $vm vers nfs-server (192.168.100.10)..."
    
    virsh domifaddr $vm | grep -oP '192\.168\.100\.\d+' | while read ip; do
        echo "  IP stockage : $ip"
    done
    
    echo ""
done

echo "✅ Test terminé"
EOF

chmod +x ~/test-network.sh
~/test-network.sh
```

---

## 💾 Étape 5 – Vérifier le Montage NFS

### 5.1 – Vérifier sur Chaque VM Cliente

```bash
# Fonction pour vérifier le montage NFS
check_nfs_mount() {
    local vm_name=$1
    
    echo "=== Vérification de $vm_name ==="
    
    # Se connecter et vérifier
    virsh console $vm_name << 'CONSOLE_EOF'
ubuntu
password123
df -h | grep nfs
ls -la /mnt/shared
cat /home/ubuntu/nfs-status.txt
exit
CONSOLE_EOF
    
    echo ""
}

# Vérifier les 3 VMs
for i in {1..3}; do
    check_nfs_mount ubuntu-0$i
done
```

**Résultat attendu :**
```
192.168.100.10:/srv/nfs/shared  100G  1.5G   94G   2% /mnt/shared
✅ NFS monté avec succès
```

### 5.2 – Tester le Partage de Fichiers

```bash
# Se connecter au serveur NFS
virsh console nfs-server

# Login : ubuntu / password123

# Créer un fichier de test
echo "Hello from NFS Server!" > /srv/nfs/shared/test-file.txt
ls -l /srv/nfs/shared/

# Sortir : Ctrl + ]

# Se connecter à ubuntu-01
virsh console ubuntu-01

# Login : ubuntu / password123

# Vérifier que le fichier est visible
cat /mnt/shared/test-file.txt

# Créer un fichier depuis le client
echo "Hello from ubuntu-01!" > /mnt/shared/client-01.txt

# Sortir : Ctrl + ]

# Vérifier sur le serveur NFS
virsh console nfs-server
ls -l /srv/nfs/shared/
# Vous devriez voir test-file.txt ET client-01.txt

# Sortir : Ctrl + ]
```

🎉 **Le partage NFS fonctionne dans les deux sens !**

---

## 📸 Étape 6 – Gestion des Snapshots

### 6.1 – Créer des Snapshots de Toutes les VMs

```bash
# Fonction pour créer un snapshot
create_snapshot() {
    local vm_name=$1
    local snap_name="initial-config"
    
    virsh snapshot-create-as $vm_name \
        --name $snap_name \
        --description "État initial après configuration NFS" \
        --atomic
    
    echo "✅ Snapshot créé pour $vm_name"
}

# Créer des snapshots pour toutes les VMs
for vm in nfs-server ubuntu-01 ubuntu-02 ubuntu-03; do
    create_snapshot $vm
done

# Lister les snapshots
echo ""
echo "=== 📸 Snapshots Créés ==="
for vm in nfs-server ubuntu-01 ubuntu-02 ubuntu-03; do
    echo "--- $vm ---"
    virsh snapshot-list $vm
    echo ""
done
```

### 6.2 – Tester la Restauration d'un Snapshot

```bash
# Se connecter à ubuntu-01 et modifier un fichier
virsh console ubuntu-01
# Login : ubuntu / password123
echo "Modification de test" > /mnt/shared/modification.txt
exit

# Restaurer le snapshot
virsh snapshot-revert ubuntu-01 initial-config

# Vérifier que le fichier a disparu
virsh console ubuntu-01
ls /mnt/shared/modification.txt  # Devrait afficher "No such file"
exit
```

**Utilité des snapshots :**
- Sauvegarder l'état avant des modifications risquées
- Revenir rapidement à un état stable
- Tester des configurations sans risque

---

## 🔧 Étape 7 – Gestion Avancée des Volumes

### 7.1 – Créer un Volume Partagé Supplémentaire

```bash
# Se connecter au serveur NFS
virsh console nfs-server

# Login : ubuntu / password123

# Créer un nouveau répertoire de partage
sudo mkdir -p /srv/nfs/backup
sudo chown nobody:nogroup /srv/nfs/backup
sudo chmod 777 /srv/nfs/backup

# Ajouter l'export NFS
echo "/srv/nfs/backup 192.168.100.0/24(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports

# Recharger les exports
sudo exportfs -a

# Vérifier
sudo exportfs -v

# Sortir : Ctrl + ]
```

### 7.2 – Monter le Nouveau Volume sur les Clients

```bash
# Se connecter à ubuntu-01
virsh console ubuntu-01

# Login : ubuntu / password123

# Créer le point de montage
sudo mkdir -p /mnt/backup

# Monter le volume
sudo mount -t nfs 192.168.100.10:/srv/nfs/backup /mnt/backup

# Vérifier
df -h | grep backup

# Rendre le montage permanent
echo "192.168.100.10:/srv/nfs/backup /mnt/backup nfs defaults 0 0" | sudo tee -a /etc/fstab

# Sortir : Ctrl + ]
```

---

## 📊 Étape 8 – Monitoring et Performance

### 8.1 – Installer des Outils de Monitoring

```bash
# Se connecter au serveur NFS
virsh console nfs-server

# Login : ubuntu / password123

# Installer nfsstat et iotop
sudo apt install nfs-kernel-server sysstat iotop -y

# Afficher les statistiques NFS
nfsstat -s

# Afficher les connexions NFS actives
sudo showmount -a

# Sortir : Ctrl + ]
```

### 8.2 – Tester les Performances NFS

```bash
# Se connecter à ubuntu-01
virsh console ubuntu-01

# Login : ubuntu / password123

# Installer fio (outil de benchmark disque)
sudo apt install fio -y

# Test d'écriture séquentielle
fio --name=write-test \
    --directory=/mnt/shared \
    --size=1G \
    --bs=1M \
    --rw=write \
    --numjobs=1 \
    --time_based \
    --runtime=30s \
    --group_reporting

# Test de lecture séquentielle
fio --name=read-test \
    --directory=/mnt/shared \
    --size=1G \
    --bs=1M \
    --rw=read \
    --numjobs=1 \
    --time_based \
    --runtime=30s \
    --group_reporting

# Sortir : Ctrl + ]
```

### 8.3 – Créer un Dashboard de Monitoring

```bash
# Créer un script de monitoring sur l'hôte L1
cat > ~/monitor-lab.sh << 'EOF'
#!/bin/bash

echo "=== 📊 Monitoring du Lab ==="
echo ""

echo "🖥️ VMs en cours d'exécution :"
virsh list
echo ""

echo "💾 Utilisation du stockage :"
virsh pool-info default
echo ""

echo "🌐 Réseaux actifs :"
virsh net-list
echo ""

echo "📈 Utilisation CPU/RAM de l'hôte :"
free -h | grep Mem
uptime
echo ""

echo "💿 Espace disque :"
df -h /var/lib/libvirt/images | tail -1
echo ""

echo "✅ Monitoring terminé"
EOF

chmod +x ~/monitor-lab.sh
~/monitor-lab.sh
```

---

## 🧪 Étape 9 – Tests de Résilience

### 9.1 – Test de Panne du Serveur NFS

```bash
# Arrêter le serveur NFS
virsh shutdown nfs-server

# Attendre 10 secondes
sleep 10

# Essayer d'accéder au partage depuis ubuntu-01
virsh console ubuntu-01
# Login : ubuntu / password123
ls /mnt/shared  # Devrait bloquer ou afficher une erreur
# Ctrl + C pour annuler
exit

# Redémarrer le serveur NFS
virsh start nfs-server

# Attendre 30 secondes
sleep 30

# Vérifier que le partage est à nouveau accessible
virsh console ubuntu-01
ls /mnt/shared  # Devrait fonctionner
exit
```

### 9.2 – Test de Déconnexion Réseau

```bash
# Déconnecter ubuntu-01 du réseau de stockage
virsh detach-interface ubuntu-01 network --mac $(virsh domiflist ubuntu-01 | grep storage-net | awk '{print $5}') --config

# Redémarrer ubuntu-01
virsh reboot ubuntu-01

# Attendre 30 secondes
sleep 30

# Vérifier que le montage NFS échoue
virsh console ubuntu-01
df -h | grep nfs  # Ne devrait rien afficher
exit

# Reconnecter au réseau de stockage
virsh attach-interface ubuntu-01 network storage-net --model virtio --config

# Redémarrer ubuntu-01
virsh reboot ubuntu-01

# Attendre 30 secondes
sleep 30

# Vérifier que le montage NFS fonctionne à nouveau
virsh console ubuntu-01
df -h | grep nfs  # Devrait afficher le montage
exit
```

---

## 🎯 Étape 10 – Automatisation avec Scripts

### 10.1 – Script de Démarrage du Lab

```bash
# Créer un script pour démarrer tout le lab
cat > ~/start-lab.sh << 'EOF'
#!/bin/bash

echo "🚀 Démarrage du Lab Complet..."
echo ""

# Démarrer les réseaux
echo "🌐 Démarrage des réseaux virtuels..."
virsh net-start default 2>/dev/null || echo "  default déjà démarré"
virsh net-start storage-net 2>/dev/null || echo "  storage-net déjà démarré"
echo ""

# Démarrer le serveur NFS
echo "💾 Démarrage du serveur NFS..."
virsh start nfs-server 2>/dev/null || echo "  nfs-server déjà démarré"
sleep 20
echo ""

# Démarrer les VMs clientes
echo "🖥️ Démarrage des VMs clientes..."
for i in {1..3}; do
    virsh start ubuntu-0$i 2>/dev/null || echo "  ubuntu-0$i déjà démarré"
    sleep 5
done
echo ""

# Afficher l'état
echo "✅ Lab démarré !"
virsh list
EOF

chmod +x ~/start-lab.sh
```

### 10.2 – Script d'Arrêt du Lab

```bash
# Créer un script pour arrêter tout le lab
cat > ~/stop-lab.sh << 'EOF'
#!/bin/bash

echo "🛑 Arrêt du Lab Complet..."
echo ""

# Arrêter les VMs clientes
echo "🖥️ Arrêt des VMs clientes..."
for i in {1..3}; do
    virsh shutdown ubuntu-0$i
done

# Attendre 30 secondes
sleep 30

# Arrêter le serveur NFS
echo "💾 Arrêt du serveur NFS..."
virsh shutdown nfs-server

# Attendre 20 secondes
sleep 20

# Afficher l'état
echo "✅ Lab arrêté !"
virsh list --all
EOF

chmod +x ~/stop-lab.sh
```

### 10.3 – Script de Nettoyage Complet

```bash
# Créer un script pour supprimer tout le lab (ATTENTION : DESTRUCTIF !)
cat > ~/cleanup-lab.sh << 'EOF'
#!/bin/bash

echo "⚠️  ATTENTION : Ce script va SUPPRIMER tout le lab !"
read -p "Êtes-vous sûr ? (oui/non) : " confirm

if [ "$confirm" != "oui" ]; then
    echo "Annulation."
    exit 0
fi

echo "🗑️ Suppression du Lab..."
echo ""

# Arrêter et supprimer les VMs
for vm in nfs-server ubuntu-01 ubuntu-02 ubuntu-03; do
    echo "Suppression de $vm..."
    virsh destroy $vm 2>/dev/null
    virsh undefine $vm --remove-all-storage 2>/dev/null
done

# Supprimer les réseaux
echo "Suppression du réseau storage-net..."
virsh net-destroy storage-net 2>/dev/null
virsh net-undefine storage-net 2>/dev/null

# Supprimer les répertoires
echo "Suppression des répertoires..."
sudo rm -rf /var/lib/libvirt/images/nfs-server
sudo rm -rf /var/lib/libvirt/images/ubuntu-0*

echo "✅ Lab supprimé !"
EOF

chmod +x ~/cleanup-lab.sh
```

---

## 📝 Récapitulatif du TP3

Dans ce TP, vous avez appris à :

- ✅ Créer des **réseaux virtuels multiples** (NAT + Isolé)
- ✅ Déployer un **serveur NFS** avec stockage partagé
- ✅ Créer des **VMs clientes** avec cloud-init
- ✅ Configurer le **montage NFS automatique**
- ✅ Gérer des **snapshots** pour la sauvegarde
- ✅ Tester la **résilience** de l'infrastructure
- ✅ Automatiser le **démarrage/arrêt** du lab
- ✅ Monitorer les **performances** et l'utilisation

### 🎯 Checklist de Vérification Finale

Avant de passer au TP4, vérifiez que :

- [ ] Les 4 VMs sont démarrées (`virsh list`)
- [ ] Les 2 réseaux sont actifs (`virsh net-list`)
- [ ] Le serveur NFS exporte `/srv/nfs/shared` (`sudo exportfs -v` sur nfs-server)
- [ ] Les 3 clients montent `/mnt/shared` (`df -h | grep nfs` sur ubuntu-01/02/03)
- [ ] Les fichiers créés sur un client sont visibles sur les autres
- [ ] Les snapshots sont créés pour toutes les VMs

### 📚 Concepts Clés Appris

| Concept | Description |
|---------|-------------|
| **NFS** | Network File System - Partage de fichiers en réseau |
| **Cloud-init** | Automatisation de la configuration des VMs |
| **Réseau Isolé** | Réseau privé sans accès Internet |
| **Réseau NAT** | Réseau avec accès Internet via l'hôte |
| **Bridge** | Interface réseau virtuelle (virbr0, virbr1) |
| **Snapshot** | Sauvegarde de l'état d'une VM |
| **FSTAB** | Fichier de configuration des montages permanents |

### 🛠️ Commandes Essentielles à Retenir

| Commande | Description |
|----------|-------------|
| `virsh net-define <xml>` | Créer un réseau virtuel |
| `virsh net-start <réseau>` | Démarrer un réseau |
| `cloud-localds <iso> <yaml>` | Créer une ISO cloud-init |
| `virt-install --import` | Créer une VM depuis une image existante |
| `exportfs -a` | Recharger les exports NFS |
| `showmount -a` | Afficher les clients NFS connectés |
| `virsh snapshot-create-as` | Créer un snapshot |

---

## 🚀 Prochaine Étape : TP4

Vous êtes maintenant prêt pour le **TP4 : Hyperviseur Imbriqué - Xen & Xen Orchestra** où vous allez :

- Installer **XCP-ng** (Xen) dans une VM KVM
- Comprendre la **virtualisation imbriquée** (nested)
- Déployer **Xen Orchestra** pour la gestion Web
- Créer des VMs **dans Xen** (L3 - troisième niveau)
- Comparer les performances **KVM vs Xen**

**Votre infrastructure de lab est opérationnelle ! 🎉**
