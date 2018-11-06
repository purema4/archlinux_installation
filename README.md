# Installation d'Arch Linux complet par Marc-Antoine Carrière <purema4@gmail.com>

## Installation avec LVM LUKS BTRFS Systemd-boot iommu EFI

_Attention, c'est vraiment ma configuration personnel._
_Venu le temps de changer des fichiers, je vais essayer de le faire sans aucun outil textuel eg. vim emacs nano. Vous pouvez, très facilement, utiliser ces outils._

_Aussi, j'utilise EFI avec GPT comme table de partition._

### Trucs machins de début

```bash
loadkeys <your-keymap>
timedatectl set-ntp true
```

### Modifier la liste de mirroir.

_J'utilise personnellement le mirroir de l'ÉTS au http://mirror.cedille.club/archlinux/_

Si vous êtes au Québec:
```bash
cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup
grep cedille /etc/pacman.d/mirrorlist.bak > /etc/pacman.d/mirrorlist
```

Sinon:

```bash
cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup
rankmirrors -n 6 /etc/pacman.d/mirrorlist.backup > /etc/pacman.d/mirrorlist
```

### Partition 

La partition ce fait avec `gdisk`

_généralement /dev/sda est le disque par défaut, assurez vous de choisir le bon disque avec fdisk -l au préalable._

Nous allons créer deux partition, une pour l'amorce (boot) et l'autre pour la parti encryption.

```bash
gdisk /dev/sda
```

```
GPT fdisk (gdisk) version 1.0.1

Partition table scan:
  MBR: protective
  BSD: not present
  APM: not present
  GPT: present

Found valid GPT with protective MBR; using GPT.

Command (? for help): o
This option deletes all partitions and creates a new protective MBR.
Proceed? (Y/N): Y

Command (? for help): n
Partition number (1-128, default 1): 
First sector (34-242187466, default = 2048) or {+-}size{KMGTP}: 
Last sector (2048-242187466, default = 242187466) or {+-}size{KMGTP}: +512M
Current type is 'Linux filesystem'
Hex code or GUID (L to show codes, Enter = 8300): EF00
Changed type of partition to 'EFI System'

Command (? for help): n
Partition number (2-128, default 2): 
First sector (34-242187466, default = 1050624) or {+-}size{KMGTP}: 
Last sector (1050624-242187466, default = 242187466) or {+-}size{KMGTP}: 
Current type is 'Linux filesystem'
Hex code or GUID (L to show codes, Enter = 8300): 
Changed type of partition to 'Linux filesystem'

Command (? for help): w
```

### L'encryption et création de volumes logiques!

Ensuite, nous allons créer l'encryption avec les volumes logiques à l'intérieur.

```bash
cryptsetup -v luksFormat /dev/sda2 
```
Ça va vous demander d'écrire YES puis un mot de passe

```bash
cryptsetup luksOpen /dev/sda2 lvm
```
ceci ouvre la partition encrypté sous /dev/mapper/lvm
_le dernier argument, ***notez*** le pour plus tard_

Création du volume physique et du group. Nous allons l'appeler **arch**
```bash
pvcreate /dev/mapper/lvm
vgcreate arch /dev/mapper/lvm
```

Nous allons créer par la suite les volues suivants:

Nom | Espace | Chemin | Commande
--- | ------ | ------ | --------
swap | 16gb *ram de 16gb* | /dev/mapper/arch-swap | `lvcreate -L 16G arch -n swap`
root | Restant | /dev/mapper/arch-root | `lvcreate -l 100%FREE arch -n root`

_Je fais ici seulement deux partitions, car j'utilise btrfs et on peut faire des sous-volumes pour /home /var etc.._

Voilà c'est fait!

### Formatage

Nous allons formater la partition d'amorcage en fat32

```bash
mkfs.vfat /dev/sda1
```

Ensuite, nous allons formater le volume logique _arch-root_ avec btrfs
```bash
mkfs.btrfs -L "Arch Linux" /dev/mapper/arch-root
```

Puis le swap
```bash
mkswap /dev/mapper/arch-swap
```

### Monter les partitions et volumes logiques

_j'utilise la compression zstd avec btrfs, pour pouvoir l'utiliser, **il faut installer zstd en premier** avec `pacman -Syu zstd`_

Bon, je vous recommande de mettre vos propres options différents votre configuration.
```bash
mount -o compress=zstd,defaults,noatime,discard,ssd /dev/mapper/arch-root /mnt
```
Ensuite le swap et la partition d'amorcage
```bash
swapon /dev/mapper/arch-swap
mkdir /mnt/boot && mount /dev/sda1 /mnt/boot
```

### Installation du système!!!
```bash
pacstrap /mnt base base-devel vim networkmanager dialog
```
### Création de FSTAB
```bash
genfstab -pU /mnt >> /mnt/etc/fstab
```

### Chroot
```bash
arch-chroot /mnt
```

### Configuration du système de base
```bash
ln -sf /usr/share/zoneinfo/Région/Ville /etc/localtime
hwclock --systohc

echo <hostname> > /etc/hostname

passwd
useradd -m -G wheel -s /bin/bash <nom d'utilisateur>
password <nom d'utilisateur>
```

### Locale-gen
**Bien important de faire cette étape**
Ok donc c'est pas si compliqué, voici la commande:
```bash
sed -e '/<locale>/ s/^#*//' -i /etc/locale.gen
```
la variable _locale_ est égale à votre locale, vous mettez ce que vous voulez.
Par exemple:
- en_US.UTF-8
- fr_CA.UTF-8
- es_NI.UTF-8

Ensuite
```bash
locale-gen
```

### mkinitcpio
Il faut aller modifier les étapes d'amorcages du kernel.
Modifier /etc/mkinitcpio.conf
```
HOOKS="base udev autodetect modconf block keyboard encrypt lvm2 resume filesystems fsck"
```

Si vous utiliser btrfs, veuillez installer `btrfs-progs`
Executer:

```bash
mkinitcpio -p linux
```

**important** installation de micro code
Si vous êtes avec amd `pacman -S amd-ucode`
Si vous êtes avec intel `pacman -S intel-ucode`

### Systemd-boot

```bash
bootctl --path=/boot install
```

Nous allons modifier le chargeur d'amorcage (bootloader)

```bash
cat > /boot/loader/loader.conf << "EOF"
default arch
timeout 5
editor 0
EOF
```

Nous allons créer l'entré _arch_ pour systemd.

Créer et Modifier le fichier /boot/loader/entries/arch.conf

```
title Arch Linux
linux /vmlinuz-linux
initrd intel-ucode.img OU initrd amd-ucode.img
initrd options cryptdevice=UUID=<Le UUID de la Partition /dev/sda2>:<nom du luks noté plus haut> resume=/dev/mapper/arch-swap root=/dev/mapper/arch-root rw quiet iommu_<cpu>=on iommu=pt
```
Si vous êtes dans vim. faites `:read !blkid /dev/sda2`, le UUID va s'ajouter dans le tampon de modification.

Ouf...

### Redémarrage

```bash
exit
umount -R /mnt
reboot
```
Si tout ce passe bien, l'ordinateur va redémarrer et voud demander pour le mot de passe d'encryption.

Pour installer un environnement graphique, un prochain  tutoriel s'en vient.
