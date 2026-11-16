ERSÄTT DETTA INNEHÅLL MED ERT INNEHÅLL
```
loadkeys se-lat6 # Svenska tangentbordslayout
```

### Kontorllera om systemet startar i UEFI läge:

```bash
cat /sys/firmware/efi/fw_platform_size
```
om kommandot returnerar 64 körs det i UEFI- läge
(64-bitars UEFI)


### Nätverk

```bash
ip link # listar nätverksgränssnitt och används för att se om nätverkskortet är aktivt.
ping google.com # testar internetanslutning mot google
```

### Tidszon

```bash
timedatectl list-timezones
timedatectl set-timezone Europe/Stockholm
timedatectl
```
listar tillgängliga tidszoner
sätter tid
kontrollerar aktuell tid

### Partitionering och kryptering

Disk 1 (/dev/sda)

```bash
fdisk /dev/sda
```
/dev/sda1 - 1GB för EFI system
/dev/sda2 -resten av disken för root

### kryptera root-partitionen och lås den

```bash
cryptsetup luksFormat /dev/sda2
cryptsetup open /dev/sda2 cryptroot
```

### Skapa filsystem på partitionenra

```bash
mkfs.fat -F32 /dev/sda1
```

EFI/boot-partition.(måste vara FAT 32 för UEFI)

```bash
mkfs.ext4 /dev/mapper/cryptroot
```

root partitionen - ext4 är standard för Linux root

### Montera partitionerna

```bash
mount /dev/mapper/cryptroot /mnt
mkdir /mnt/boot
mount /dev/sda1 /mnt/boot
```
här bestäms vilka mappar i filsystemet som kopplas till respektive partition från diskarna


### Installera grundsystem

Installera baspaket, kärna och firmware på den monterade rootpartitionen:

```bash
pacstrap -K /mnt base linux linux-firmware
```
- -k -> hoppar över kopiering av pacman-cache från live-systemet.
- /mnt -> platsen där systemet ska installeras (i vårat fall rootpartitonen vi monterade).
- base -> grundläggande verktyg som behövs för att systemet ska fungera.
- linux -> Linux-kärnan (hjärtat i systemet)
- linux-firmware -> drivrutiner för hårdvara (nätverk, ljud, grafik).

### Extrapaket

```bash
pacman -S intel-ucode
```
- CPU-mikrokod för intel-processorer

```bash
pacman -S dosfstools e2fsprogs btrfs-progs xfsprogs
```

- Verktyg för att skapa och kontrollera olika filsystem

```bash
pacman -S lvm2 mdadm
```

- Hantera LVM och RAID

```bash
pacman -S sof-firmware linux-firmware-marvell linux-firmware-broadcom
```

- Extra drivrutiner för ljud och nätverkskort

```bash
pacman -S networkmanager dhcpcd wpa_supplicant modemmanager
```

- Program för nätverk och internetanslutning

```bash
pacman -S nano
```

- Enkel texteditor för terminalen

```bash
pacman -S man-db man-pages texinfo
```

- Dokumentation och manualer för kommandon

```bash
pacman -S mlocate
```

- mlocate -> verktyg för att hitta filer i systemet

Efter installation:
```
sudo updatedb   # bygger databas över filer
locate filnamn  # söker efter filen

```

### Användare och sudo

```bash
pacman -S sudo
```
- Installerar sudo så vanliga användare kan köra kommandot som root

```
useradd -m -G wheel -s /bin/bash <användarnamn>
passwd <användarnamn>
```
- Skapar en ny användare med hemkatalog. (-m) sätter den i wheel-gruppen för sudo-rättigheter och ger den bash som standard-shell.
- passwd -> sätter lösenord för användaren

```
EDITOR=nano visudo
```
- Öppnar sudo-konfigurationen
- Ta bort kommentaren (#) framför raden:
```bash
%wheel ALL=(ALL:ALL) ALL
```
-> Detta ger alla i wheel-gruppen rätt att använda sudo


```bash
passwd -l root
```
- Låser rootkonton (inaktiverar root-login)


### Nätverkskonfiguration med systemd-networkd

#### 1.skapa nätverksfil

```bash
sudo nano /etc/systemd/network/20-wired.network
```
Lägg in följande innehåll:

```bash
[Match]
Name=enp*

[Network]
DHCP=yes
```
- [Match] -> anger vilka nätverkskort som filen gäller för (enp* matchar alla “enyethernet”-interface).
- [Network] -> nställningar för nätverket. DHCP=yes betyder att nätverket automatiskt får en IP-adress.

#### 2. Aktivera nätverkstjänster

```bash
sudo systemctl enable --now systemd-networkd systemd-resolved
```

- systemd-networkd → hanterar nätverksinterface och IP-konfiguration.
- systemd-resolved → hanterar DNS.
- --now → startar tjänsterna direkt utan omstart.

#### Länka DNS-fil

``` bash
sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

- Skapar en symbolisk länk så systemet använder systemd-resolved för DNS.
- Viktigt för att internet och namnupplösning ska fungera korrekt.


### Swap-fil
skapa och aktivera en swap-fil (virtuellt minne)

```bash
sudo fallocate -l 2G /swapfile
```
- Skapar en fil på 2 GB som används till swapfil

```bash
sudo chmod 600 /swapfile
```
- Sätter rättigheter så att endast root kan läsa och skriva filen.

```bash
sudo mkswap /swapfile
```
- formaterar filen som swapområde

```bash
sudo swapon /swapfile
```
- Aktivierar swapfilen direkt så systemet kan använda den

### Permanent swap
För att swapfilen ska aktiveras vid varje uppstart
```bash
sudo nano /etc/fstab
```
- /swapfile none swap defaults 0 0 - läggs till längst ner i nano-filen


### Disk 2 - kryptering och extra lagring

#### 1 Partitionera disken

```bash
sudo fdisk /dev/sdb
```
- skapa partition på disk 2 (/dev/sdb)

#### 2 Kryptera disken med LUKS

```bash
sudo cryptsetup luksFormat /dev/sdb1
sudo cryptsetup open /dev/sdb1 storage_crypt
```

- låser upp den krypterade partitionen som /dev/mapper/storage_crypt

#### 3 skapar filsystem och monteringspunkt

```bash
sudo mkfs.ext4 /dev/mapper/storage_crypt
sudo mkdir /storage
```
- skapar en ext4 filsystem på partitionen
- skapar mappen och  monteringspunkten /storage

#### 4 Skapa nyckelfil för automatisk upplåsning

```bash
sudo dd if=/dev/random of=/storagekey bs=1024 count=4
sudo chmod 600 /storagekey
sudo chown root:root /storagekey
sudo cryptsetup luksAddKey /dev/sdb1 /storagekey
```
- Nyckelfilen används för att låsa upp disken automatiskt vid start (när rätt lösenord slagits in)

- Rättigheter sätts så endast root kan läsa den

#### 5 Konfigurera /etc/crypttab

```bash
sudo nano /etc/crypttab
```
lägg till raden i nano-filen

```bash
storage_crypt  UUID=<UUID-för-sdb1>  /storagekey  luks
```
- Berättar för systemet hur den krypterade partitionen ska låsas upp vid boot.

#### 6 Konfigurera /etc/fstab

```
sudo nano /etc/fstab
```
Lägg till raden:

```
/dev/mapper/storage_crypt   /storage   ext4   defaults   0 2
```

- Monterar den krypterade disken automatiskt i /storage vid uppstart


### Wayland -installera GNOME och GDM

#### 1. Installera GDM

```bash
sudo pacman -S gnome gdm
```

- packman -> Arch Linux paket-hanterare
- -S ->installerar paket
- gnome -> skrivbordsmiljön GNOME
- gdm -> GNOME Display Manager, inloggningsskärmen som hanterar Wayland/X11-sessioner

#### 2. Aktivera GDM

```bash
sudo systemctl enable gdm --now

```

- systemctl enable gdm -> GDM startar automatiskt vid boot
- --now -> startar tjänsten direkt utan omstart

#### 3. Starta om systemet

```
sudo reboot
```

- startar om datorn så inställningarna träder i kraft

#### 4.kontrollera att wayland används

```
echo $XDG_SESSION_TYPE
```

- om swaret är wayland -> systemet kör nu wayland

#### Varför wayland?
- Wayland är en modern grafisk server som ersätter X11.
- Fördelar: snabbare rendering, bättre säkerhet och smidigare hantering av flera skärmar.
- GNOME använder Wayland som standard på Arch, så när `$XDG_SESSION_TYPE` visar `wayland` vet du att du kör den moderna grafiska sessionen.

### Schemalagd backup mellan Arch och en annan dator

#### Varför valde vi Restic?
- Vi kikade

OBS Kolla med johan om monteringarna stämmer utifrån det vi satt upp!!
Kolla även upp sudo. om den är inaktiverad!!

