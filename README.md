Netzwerkbridge aufsetzen:

- alte Verbindung eth0_dynamic stoppen:

nmcli connection down eth0_dynamic

- jetzt Bridge:

nmcli connection add type bridge ifname virbr1 stp no

- bridge mit eth0 verbinden, und bridge starten
nmcli connection add type bridge-slave ifname eth0 master virbr1
nmcli connection up bridge-virbr1
nmcli connection up bridge-slave-eth0

-> damit ist computer an der bridge und damit am Internet

bridge(virtuell) ---bridge-slave---> Netzwerkkarte
damit wir mehrere virtuelle Netzwerkschnittstellen implementieren können, wir haben sonst nur eine Netzwerkkarte
und somit hätten wir auch nur eine Netzwerkschnittstelle, aber virtuell gehen unendlich viele 

das ist cool weil wir somit auch unedlich viele vms mit dem Netzwerk verbinden können --> 
alle können so Verbindung mit dem Internet haben und später dann auch mit dem VPN

#anzeige:(hat alles geklappt))
sudo brctl show virbr1

# bridge in kvm nutzen
virsh net-define --file bridgeInterface.xml
virsh net-start --network bridgeInterface
virsh net-autostart --network bridgeInterface
# wenn was schief läuft:
virsh net-stop --network bridgeInterface
virsh net-destroy --network bridgeInterface
virsh net-undefine --network bridgeInterface

# virtuelle Festplatte die 10 G groß ist erstellen 
sudo qemu-img create -f qcow2 postgresql.qcow2 10G


## VM
Oberbegriff KVM allgemein
-wir setzen eine VM auf
- virt-install auf Shell, virt-manager(GUI)

sudo virt-install --name="PostgresVM" \                                                                                                         [15:36:22]
                 --description "PostgreSQL VM" \
                 --hvm --virt-type=kvm \
                 --os-type=linux \
                 --os-variant=archlinux \
                 --vcpus=1 \
                 --memballoon virtio \
                 --ram=4096 \
                 --disk /var/lib/libvirt/images/postgresql.qcow2,bus=virtio,cache=writethrough,device=disk,format=qcow2,size=4,sparse=true \
                 --cdrom=/home/ros/kvm/usb_stick/archlinux-2018.12.01-x86_64.iso \
                 --graphics vnc,password="foobar",listen=127.0.0.1,port=6901 \
                 --noautoconsole --network model=virtio,network="bridgeInterface"

- Netzwerkanbindung
-- in VM durch Netzwerkmanager gegeben (dhcp holt ip, VMs können miteinander kommunizieren, später noch abschottung und VPN)

- Anbindung eines Betriebssystems (installieren) --> ArchLinux bootable file runterladen und angeben bei disk
 --> man muss dann angeben im virt-manager das man von CDRom booten will (SATA CDROM auswählen zusätzlich)
 --> das ist aber nur um die vm zu starten mit Betriebssystem, 
     im nächsten Schritt wollen wir das Betriebssystem auf die angegebene Festplatte der vm installieren (vhd)
 --> starten einer SSH Verbindung von CDRom Linux damit man das Betriebssystem bequem auch von anderen Rechnern auf die
     Festplatte installieren kann (mit passwd Passwort setzen und IP Adresse merken(14.321), Benutzername ist root)

 - bevor Betriebssystem auf Festplatte installiert werden kann, muss Festplatte mind einmal partitioniert werden, das machen wir
   mit gdisk (Befehl ist gdisk +Festplattenpfad)
 - mit o Partitionstabelle erstellen
 - mit n Partition erstellen
 - 2 Partitionen erstellen (evi Partition und betriebssystempartition) in evi ist bootloader drin der BS läd
 - Dateisystem erstellen --> mkfs.fat -F32 /dev/vda1 auf EVI (muss immer fat 32 sein)
                         -->    ext4 für Betriebssystempartition
                         --> mount /dev/vda2 /mnt mounten unsere vm in den mnt Ordner von 
                         Festplatte /dev/vda2 wird in /mnt Ordner vom CDRom Betriebssystem gemountet
                         mkdir /mnt/boot
                         mount /dev/vda1 /mnt/boot
                         Grund: Das stellt die Verbindung zwischen CDRom Arch mit Festplatte her, damit
                                können wir beliebige Software installieren (mit pacstrap benutzt pacman 
                                im Hintergrund) u.a. unser Betriebssystem
   pacstrap /mnt filesystem base zsh coreutils linux linux-firmware base-devel openssh dhcpcd nano efibootmgr grub efibootmgr dosfstools net-tools lvm2
   -> installiert tools und root-filesystem in /mnt Ordner
   => also Festplattenordner wird auf /mnt Ordner vom CDRom Betriebssystem gemountet, 
      dort befindet sich nämlich unser pacstrap, so können wir vom CDRom BS colle Sachen wie 
      zB rootfilesystem auf unsere Festplatte installieren --> ArchLinux auf Festplatte <3
- einbindung ins VPN
- installier Postgres

!! Alles schön dokumentieren was wir machen!!!
-> dann können wir ein schönes Dockerfile schreiben

#PostgreSQL Installation step-by-step

1. install postgres mit pacman -S postgresql --> Automatischer User Postgres wird angelegt

2. zum postgres user wechseln mit su -l postgres

3. Datenbank initialisieren mit initdb --locale en_US.UTF-8 -D '/var/lib/postgres/data' 
   initdb: error: invalid locale name "en_US.UTF-8" Lösung: /etc/locale.gen editieren

su - postgres -c "initdb --locale en_US.UTF-8 -D '/var/lib/postgres/data'"

4. Datenbank starten mit systemctl start postgresql.service

5. Neuen Nutzer anlegen mit postgres userrolle mit su - postgres -c "createuser webapp"   

6. Neue db erstellen als postgres-user mit su - postgres -c "createdb gameingwebsitedb" 

7. Den erstellten Nutzer mit der erstellten db verbinden su - postgres -c "createdb gameingwebsitedb -O webapp" 

8. Connection mit db mit su - postgres -c "psql -d gamingwebsitedb" 
                         su - postgres -c 'psql -U webapp -d gamingwebsitedb'
(mit user)
9. table erstellen
CREATE TABLE games ( 
GameId INT PRIMARY KEY,
Name VARCHAR,
Description VARCHAR,
Publicationdate date,
Developer VARCHAR,
TrailerUrl VARCHAR
);

8+9 alternative:
curl -sfL http://192.168.14.230:8085/table_layout.txt|su - postgres -c 'psql -U webapp -d gamingwebsitedb'

10. curl -sfL http://192.168.14.230:8085/games_data.csv|su - postgres -c 'psql -U webapp -d gamingwebsitedb -c "\copy games FROM STDIN with delimiter as '\'','\'' CSV HEADER;"'


Container
- Dockerfile zum erstellen des ContainerImages
-- Postgres 
-- Datenanbindung
-- Netzwerkanbindung
- dann starten wir laufenden Container(docker run oder podman run)



