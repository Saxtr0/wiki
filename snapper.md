<h1 align="center">Howto: Ubuntu on btrfs snapper managed</h1>
<div style="page-break-after: always"></div>

<!-- TOC -->

- [Tabella spazio temporale](#tabella-spazio-temporale)
- [Preparazione del disco](#preparazione-del-disco)
- [Preparazione del volume btrfs](#preparazione-del-volume-btrfs)
  - [Creazione del subvolume @](#creazione-del-subvolume-)
  - [Il "first root filesystem"](#il-first-root-filesystem)
- [Installazione di Ubuntu](#installazione-di-ubuntu)
- [Attività successive l'installazione](#attività-successive-linstallazione)
  - [Montare i subvolumi](#montare-i-subvolumi)
  - [Aggiornare fstab](#aggiornare-fstab)
  - [Riavvio](#riavvio)
- [Primo avvio](#primo-avvio)
  - [Configurare snapper sul sistema appena avviato](#configurare-snapper-sul-sistema-appena-avviato)
  - [Creare la configurazione per la /home](#creare-la-configurazione-per-la-home)
  - [Installare Btrfs Assistant](#installare-btrfs-assistant)

<!-- /TOC -->
<!-- /TOC -->
<!-- /TOC -->
<!-- /TOC -->
<!-- /TOC -->

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

Nota: questa guida è pensata per essere eseguita in modalità copia e incolla.
Prima di effettuare il copia e incolla verificare che la variabile `BTRFSDEV` della guida, risolve correttamente il device da utilizzare.  
(in alternativa, modificare il device `/dev/nvme0n1p5`, con il proprio device btrfs)

```bash
BTRFSDEV=$(sudo blkid | grep btrfs | cut -d ":" -f1)
echo $BTRFSDEV
```

![](img/2024-05-12-21-06-47.png)





# Preparazione del volume btrfs

Prima di avviare l'installazione, sarà necessario preparare il volume btrfs.  
E' possibile utilizzare un terminale dalla Live.  

## Creazione del subvolume @

Si monta il filesystem btrfs creato nel precedente step, e si crea il subvolume @ (ID=256) (il secondo subvolume del volume. Il primo è subvolid=5).  

```bash
BTRFSDEV=$(sudo blkid | grep btrfs | cut -d ":" -f1)
sudo mount $BTRFSDEV /mnt  
sudo btrfs subvolume create /mnt/@
```

## Il "first root filesystem"

Si crea la directory:

```bash
sudo mkdir /mnt/@/etc/snapper/configs -p
```

Si modifica il default subvolume, e si rimonta il volume:

```bash
BTRFSDEV=$(sudo blkid | grep btrfs | cut -d ":" -f1)
sudo btrfs subvolume set-default /mnt/@
sudo umount /mnt && sudo mount $BTRFSDEV /mnt
```

Si installa, configura e utilizza snapper per creare il "first root filesystem" (sulla Live)  

```bash
sudo apt install snapper -y
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
```

Inizia l'installazione.  

# Installazione di Ubuntu

Eseguire l'installazione seguendo le istruzioni a video.  
Scelte di rilievo da eseguire per il buon esito dell'installazione:  
- Selezionare la procedura "Manuale" durante il setup del disco

![](img/2024-05-10-02-20-15.png)

- utilizzare le partizioni prima identificate/create

![](img/2024-05-10-02-22-34.png)

Attenzione, le partizioni non vanno formattate, il dispositivo è già configurato per l'installazione, durante i passaggi Preparazione del disco, e Preparazione del volume btrfs.


# Attività successive l'installazione

## Montare i subvolumi

Si verifica che tutti subvolumi si montino correttamente dove atteso e come atteso

```bash
BTRFSDEV=$(sudo blkid | grep btrfs | cut -d ":" -f1)
sudo mount $BTRFSDEV /mnt
sudo mount -o subvol=@/home $BTRFSDEV /mnt/home
sudo mount -o subvol=@/var/cache $BTRFSDEV /mnt/var/cache
sudo mkdir /mnt/var/lib/flatpack -p
sudo mount -o subvol=@/var/log $BTRFSDEV /mnt/var/log
sudo mount -o subvol=@/var/tmp $BTRFSDEV /mnt/var/tmp
sudo mount -o subvol=@/var/lib/flatpack $BTRFSDEV /mnt/var/lib/flatpack
sudo mount -o subvol=@/.snapshots $BTRFSDEV /mnt/.snapshots
```

## Aggiornare fstab

Si aggiorna il file `mnt/etc/fstab` perchè monti i subvolumi attesi dove atteso.  

```bash
BTRFSDEV=$(sudo blkid | grep btrfs | cut -d ":" -f1)
line=$(grep -n btrfs /mnt/etc/fstab | cut -d":" -f1)
echo "sudo sed -i '"$line"s/.$/0/' /mnt/etc/fstab" | sh
DISP=$(grep btrfs /mnt/etc/fstab | awk '{print $1}')
echo -n -e "\n# btrfs\n" | sudo tee -a /mnt/etc/fstab
grep btrfs /etc/mtab \
| grep -v "1/snapshot" \
| sed s@rw.*,subvolid=.*,@defaults,@ \
| sed s@/mnt@@ \
| sed s@$BTRFSDEV@$DISP@ \
| sudo tee -a /mnt/etc/fstab
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

## Configurare snapper sul sistema appena avviato

Si installa e configura snapper.  

```bash
sudo apt install snapper -y
```  
Il file di configurazione `/etc/snapper/configs/root`, è stato ereditato dal subvolume @.  
Abilitiamo la configurazione nel file `/etc/default/snapper`.. 

```bash
sudo sed -i s/SNAPPER_CONFIGS=\"\"/SNAPPER_CONFIGS=\"root\"/g /etc/default/snapper
```  

Si riavvia snapper per attivare la configura

```bash
sudo systemctl restart snapperd
```

Si abilita la quota, in questo modo l'algoritmo di pulizia e raccolta delle snapshots, potrà onorare le direttive `SPACE_LIMIT` e `FREE_LIMIT`.  
Inoltre, l'output del comando `snapper list` fornirà informazioni riguardo lo spazio utilizzato.    

```bash
sudo btrfs quota enable /
sudo snapper setup-quota
sudo snapper list
```

![](img/2024-05-12-22-28-40.png)

## Creare la configurazione per la /home

Si crea la configurazione `home`, in modo da usufruire delle snapshots anche per il subvolume /home.  

```bash
sudo snapper -c home create-config /home
```

## Installare Btrfs Assistant

Un tool grafico, potente e intuitivo, che si può utilizzare per gestire `snapper`, è certamente `Btrfs Assistant`.  

Sfortunatamente, questo software, non è disponibile nei repository ufficiali.  
Non esiste neanche un repository PPA.  

Attualmente, l'unico riferimento ufficiale(?) ad un pacchetto Ubuntu è: https://launchpad.net/ubuntu/+source/btrfs-assistant  (la versione 1.8, è vecchia di un anno. La versione corrente è 2.1)

Le istruzioni per l'installazione sono disponibili nella documentazione ufficiale.  

https://gitlab.com/btrfs-assistant/btrfs-assistant  


1. Prerequisiti per l'installazione

```bash
sudo apt install git cmake fonts-noto qt6-base-dev qt6-base-dev-tools \
g++ libbtrfs-dev libbtrfsutil-dev pkexec qt6-svg-dev qt6-tools-dev
```

2. Scaricare i sorgenti 
Per questo punto, si hanno a disposizione due opzioni:  
la versione main, o ultima versione.
Se si vuole installare la versione main:

```bash
git clone https://gitlab.com/btrfs-assistant/btrfs-assistant.git
cd btrfs-assistant
```

Per l'ultima versione:

```bash
wget https://gitlab.com/btrfs-assistant/btrfs-assistant/-/archive/2.1/btrfs-assistant-2.1.tar.gz
tar btrfs-assistant-2.1.tar.gz
xvf cd btrfs-assistant-2.1
```

1. Costruire il software

```bash
cmake -B build -S . -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_BUILD_TYPE='Release'
make -C build
```

4. Installare il software

```bash
sudo make -C build install
```
 
5. Disinstallazione (eventuale)

Perchè il software possa essere disinstallato, è necessario conservare la directory in cui è stato costruito.  
La lista dei files installati, è presente nella sottodirectory `build`.  
Il comando di disinstallazione è:  

```bash
cd build
sudo xargs rm < install_manifest.txt
```


Contenuto del file `install_manifest` dopo l'installazione:

```ascii
/etc/btrfs-assistant.conf
/usr/share/applications/btrfs-assistant.desktop
/usr/share/metainfo/btrfs-assistant.metainfo.xml
/usr/share/polkit-1/actions/org.btrfs-assistant.pkexec.policy
/usr/bin/btrfs-assistant
/usr/bin/btrfs-assistant-launcher
/usr/bin/btrfs-assistant-bin
```

![](img/2024-05-13-00-08-37.png)

![](img/2024-05-13-00-08-57.png)

![](img/2024-05-13-00-05-46.png)

Nel caso si installasse il pacchetto `btrfsmaintenance`, il software `Btrfs Assistant`, potrà essere utilizzato anche per gestire le operazioni di manutenzione del filesystem.  

![](img/2024-05-13-00-18-57.png)





