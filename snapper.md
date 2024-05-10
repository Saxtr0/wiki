<h1 align="center">Howto: Ubuntu on btrfs snapper managed</h1>
<div style="page-break-after: always"></div>

<!-- TOC -->

- [Tabella spazio temporale](#tabella-spazio-temporale)
- [Preparazione del disco](#preparazione-del-disco)
- [Preparazione del volume btrfs](#preparazione-del-volume-btrfs)
  - [Creazione del primo volume](#creazione-del-primo-volume)
  - [Il "first root filesystem"](#il-first-root-filesystem)
- [Installazione di Ubuntu](#installazione-di-ubuntu)
- [Attività successive l'installazione](#attività-successive-linstallazione)
  - [Montare i subvolumi](#montare-i-subvolumi)
  - [Aggiornare fstab](#aggiornare-fstab)
  - [Riavvio](#riavvio)
- [Primo avvio](#primo-avvio)

<!-- /TOC -->
<!-- /TOC -->
<!-- /TOC -->
<!-- /TOC -->
<!-- /TOC -->
<!-- /TOC -->
<!-- /TOC -->
<!-- /TOC -->

<div style="page-break-after: always"></div>

# Tabella spazio temporale

L'installazione è stata divisa in diversi passaggi.  
La tabella di seguito pubblicata, inquadra i vari passaggi in relazione all'ambiente di esecuzione e allo stato dell'installazione.  

Passaggio | Ambiente | Installazione in corso
--- | --- | ---
Preparazione del disco | Live | No
Preparazione del volume btrfs | Live | No
Installazione di Ubuntu | Live | Si, in corso
Attività successive l'installazione | Live | No, terminata
Primo avvio | Ubuntu | N/A
Installazione di btrfsassistant | Ubuntu | N/A

# Preparazione del disco

L'installazione proposta prevede che il dispositivo di installazione sia già pronto, e non necessiti di formattazione.  
I requisiti minimi per l'installazione, in termini di storage sono:
- partizione EFI/ESP (almeno 100Mb, in caso di dual boot si condivide quella di windows)
- partizione btrfs (consigliati un minimo di 50Gb)

Il dispositivo di installazione si può preparare anche con gparted nella Live.
Se si sta installando in dual boot, e sia necessario ridurre una (o più) partizione(i) in uso,
si consiglia di utilizzare gli strumenti di windows.


caso di esempio | esempio con gparted
---|---
dual boot | ![](img/2024-05-10-00-52-01.png)
single boot | ![](img/2024-05-10-00-58-45.png)


# Preparazione del volume btrfs

Prima di avviare l'installazione, sarà necessario preparare il volume btrfs.

## Creazione del primo volume

Si monta il filesystem btrfs creato nel precedente step, e si crea il subvolume @ (ID=256) (il secondo subvolume del volume. Il primo è subvolid=5).  

```bash
sudo mount /dev/nvme0n1p5 /mnt  
sudo btrfs subvolume create /mnt/@
```

## Il "first root filesystem"

Si crea la directory:

```bash
sudo mkdir /mnt/@/etc/snapper/configs -p
```

Si modifica il default subvolume, e si rimonta il volume:

```bash
sudo btrfs subvolume set-default /mnt/@
sudo umount /mnt && sudo mount /dev/nvme0n1p5 /mnt
```

Si installa, configura e utilizza snapper per creare il "first root filesystem" (sulla Live)  

```bash
sudo apt install snapper
systemctl stop snapper-timeline.timer  # prima dell'installazione non vogliamo le snapshot di tipo timeline
sudo snapper create-config /mnt
sudo cp /etc/snapper/configs/root  /mnt/etc/snapper/configs/
sudo sed -i s@\"/mnt\"@\"/\"@ /mnt/etc/snapper/configs/root
sudo snapper create -t single -d "first root filesystem" --read-write --from 0
```

Creiamo i subvolumi di interesse, infine si modifica il default subvolume con il "first root filesystem".

```bash
sudo btrfs subvolume create /mnt/home
sudo mkdir /mnt/var/lib -p
sudo btrfs subvolume create /mnt/var/cache
sudo btrfs subvolume create /mnt/var/log
sudo btrfs subvolume create /mnt/var/tmp
sudo btrfs subvolume create /mnt/var/lib/flatpack
sudo btrfs subvolume set-default /mnt/.snapshots/1/snapshot
sudo umount /mnt
sudo apt purge snapper
```

Inizia l'installazione.  

# Installazione di Ubuntu

Scelte di rilievo per il buon esito dell'installazione:  
- Selezionare la procedura "Manuale" durante il setup del disco

![](img/2024-05-10-02-20-15.png)

- utilizzare le partizioni prima identificate/create

![](img/2024-05-10-02-22-34.png)

Attenzione, le partizioni non vanno formattate, il dispositivo è già configurato per l'installazione


# Attività successive l'installazione

## Montare i subvolumi

Si verifica che tutti subvolumi si montino correttamente dove atteso e come atteso

```bash
sudo mount /dev/nvme0n1p5 /mnt
sudo mount -o subvol=@/home /dev/nvme0n1p5 /mnt/home
sudo mount -o subvol=@/var/cache /dev/nvme0n1p5 /mnt/var/cache
sudo mkdir /mnt/var/lib/flatpack -p
sudo mount -o subvol=@/var/log /dev/nvme0n1p5 /mnt/var/log
sudo mount -o subvol=@/var/tmp /dev/nvme0n1p5 /mnt/var/tmp
sudo mount -o subvol=@/var/lib/flatpack /dev/nvme0n1p5 /mnt/var/lib/flatpack
sudo mount -o subvol=@/.snapshots /dev/nvme0n1p5 /mnt/.snapshots
```

## Aggiornare fstab

```bash
line=$(grep -n btrfs /mnt/etc/fstab | cut -d":" -f1)
echo "sudo sed -i '"$line"s/.$/0/' /mnt/etc/fstab" | sh
DISP=$(grep btrfs /mnt/etc/fstab | awk '{print $1}')
echo -n -e "\n# btrfs\n" | sudo tee -a /mnt/etc/fstab
grep btrfs /etc/mtab | grep -v "1/snapshot" | sed s@rw,relatime,ssd,space_cache=v2,subvolid=.*,@defaults,@ | sed s@/mnt@@ | sed s@/dev/nvme0n1p5@$DISP@ | sudo tee -a /mnt/etc/fstab
```

![](img/2024-05-10-04-04-15.png)

## Riavvio

Si smontano tutti i subvolumi montati in /mnt, e si riavvia.  

```bash
sudo umount /mnt/.snapshots 
sudo umount /mnt/var/tmp 
sudo umount /mnt/var/cache
sudo umount /mnt/var/log
sudo umount /mnt/var/lib/flatpack
sudo umount /mnt/home 
sudo umount /mnt
sudo shutdown -r now
```


# Primo avvio

Si installa e configura snapper.  
Il file di configurazione è stato ereditato dal subvolume @.

```bash
sudo apt install snapper
sudo sed -i s/SNAPPER_CONFIGS=\"\"/SNAPPER_CONFIGS=\"root\"/g /etc/default/snapper
sudo btrfs quota enable /
sudo snapper setup-quota
sudo snapper list
```