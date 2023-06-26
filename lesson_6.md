  # Физический уровень PostgreSQL
* Создана ВМ c Ubuntu 22.04 LTS в Oracle VM virtual Box
* Установлен PostgreSQL 15
```
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15
```
* Кластер запущен
```
postgres@ubuntu:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```
* Выполнен вход под пользователя postgres в psql и создана таблица с содержимым
```
postgres@ubuntu:~$ psql
postgres=# CREATE TABLE accounts (
	user_id serial PRIMARY KEY,
	username VARCHAR ( 50 ) UNIQUE NOT NULL,
	password VARCHAR ( 50 ) NOT NULL,
	email VARCHAR ( 255 ) UNIQUE NOT NULL
);
postgres=# INSERT INTO accounts(user_id, username, password, email)
VALUES (1, 'Ivan', 'Pass', 'Ivan@mail.ru');
postgres=# INSERT INTO accounts(user_id, username, password, email)
VALUES (2, 'Poman', 'Pass', 'Poman@mail.ru');
postgres=# INSERT INTO accounts(user_id, username, password, email)
VALUES (3, 'Bogdan', 'Pass', 'Bogdan@mail.ru');
postgres=# exit
```
* Остановлен postgres 
```
postgres@ubuntu:~$ sudo -u postgres pg_ctlcluster 15 main stop
postgres@ubuntu:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 down   postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```
* Создан новый диск к ВМ размером 10G
* Проинициализирован диск и примонтирована файловая система
```
tatiana@ubuntu:~$ df -h
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           197M  1,6M  195M   1% /run
/dev/sda3        24G   15G  8,4G  64% /
tmpfs           982M  1,1M  981M   1% /dev/shm
tmpfs           5,0M  4,0K  5,0M   1% /run/lock
/dev/sda2       512M  6,1M  506M   2% /boot/efi
tmpfs           197M  112K  197M   1% /run/user/1001
/dev/sr0         51M   51M     0 100% /media/tatiana/VBox_GAs_7.0.6
```
_Step 1 — Install Parted_
```
root@ubuntu:~# sudo apt update
root@ubuntu:~# sudo apt install parted
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
parted is already the newest version (3.4-2build1).
parted set to manually installed.
0 upgraded, 0 newly installed, 0 to remove and 4 not upgraded.
```
_Step 2 — Identify the New Disk on the System_
```
root@ubuntu:~# sudo parted -l | grep Error
Error: /dev/sdb: unrecognised disk label
Warning: Unable to open /dev/sr0 read-write (Read-only file system).  /dev/sr0
has been opened read-only.
Error: /dev/sr0: unrecognised disk label

root@ubuntu:~# lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0 244,5M  1 loop /snap/firefox/2800
loop1    7:1    0 349,7M  1 loop /snap/gnome-3-38-2004/140
loop2    7:2    0  63,4M  1 loop /snap/core20/1950
loop3    7:3    0  12,3M  1 loop /snap/snap-store/959
loop4    7:4    0   452K  1 loop /snap/snapd-desktop-integration/83
loop5    7:5    0   242M  1 loop /snap/firefox/2667
loop6    7:6    0  53,3M  1 loop /snap/snapd/19361
loop7    7:7    0 346,3M  1 loop /snap/gnome-3-38-2004/119
loop8    7:8    0  91,7M  1 loop /snap/gtk-common-themes/1535
loop9    7:9    0 466,5M  1 loop /snap/gnome-42-2204/111
loop10   7:10   0  73,8M  1 loop /snap/core22/750
loop11   7:11   0 460,6M  1 loop /snap/gnome-42-2204/102
loop12   7:12   0     4K  1 loop /snap/bare/5
loop14   7:14   0   304K  1 loop /snap/snapd-desktop-integration/49
loop15   7:15   0  73,9M  1 loop /snap/core22/766
loop16   7:16   0  63,5M  1 loop /snap/core20/1891
loop17   7:17   0  45,9M  1 loop /snap/snap-store/638
loop18   7:18   0  53,3M  1 loop /snap/snapd/19457
sda      8:0    0    25G  0 disk 
├─sda1   8:1    0     1M  0 part 
├─sda2   8:2    0   513M  0 part /boot/efi
└─sda3   8:3    0  24,5G  0 part /var/snap/firefox/common/host-hunspell
                                 /
sdb      8:16   0    10G  0 disk 
sr0     11:0    1  50,6M  0 rom  /media/tatiana/VBox_GAs_7.0.6
```
_Step 3 — Partition the New Drive_
```
root@ubuntu:~# sudo parted /dev/sdb mklabel gpt
root@ubuntu:~# sudo parted -a opt /dev/sdb mkpart primary ext4 0% 100%    
root@ubuntu:~# lsblk                                                      
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
...
sdb      8:16   0    10G  0 disk 
└─sdb1   8:17   0    10G  0 part 
...
```
_Step 4 — Create a Filesystem on the New Partition_
```
root@ubuntu:~# sudo mkfs.ext4 -L datapartition /dev/sdb1
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 2620928 4k blocks and 655360 inodes
Filesystem UUID: ecedb020-6d19-4950-9292-7117fe5a5480
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632
Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done 

root@ubuntu:~# sudo lsblk --fs
NAME   FSTYPE  FSVER       LABEL          UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
loop0                                                                                0   100% /snap/firefox/2800
loop1  squashf 4.0                                                                   0   100% /snap/gnome-3-38-2004/140
loop2                                                                                0   100% /snap/core20/1950
loop3  squashf 4.0                                                                   0   100% /snap/snap-store/959
loop4  squashf 4.0                                                                   0   100% /snap/snapd-desktop-integration/83
loop5  squashf 4.0                                                                   0   100% /snap/firefox/2667
loop6  squashf 4.0                                                                   0   100% /snap/snapd/19361
loop7  squashf 4.0                                                                   0   100% /snap/gnome-3-38-2004/119
loop8  squashf 4.0                                                                   0   100% /snap/gtk-common-themes/1535
loop9  squashf 4.0                                                                   0   100% /snap/gnome-42-2204/111
loop10 squashf 4.0                                                                   0   100% /snap/core22/750
loop11 squashf 4.0                                                                   0   100% /snap/gnome-42-2204/102
loop12 squashf 4.0                                                                   0   100% /snap/bare/5
loop14 squashf 4.0                                                                   0   100% /snap/snapd-desktop-integration/49
loop15                                                                               0   100% /snap/core22/766
loop16 squashf 4.0                                                                   0   100% /snap/core20/1891
loop17 squashf 4.0                                                                   0   100% /snap/snap-store/638
loop18                                                                               0   100% /snap/snapd/19457
sda                                                                                           
├─sda1                                                                                        
├─sda2 vfat    FAT32                      9730-FA79                             505,9M     1% /boot/efi
└─sda3 ext4    1.0                        cdaadb3c-295e-4813-b17e-e5d5f791ffc4    8,4G    60% /var/snap/firefox/common/host-hunspell
                                                                                              /
sdb                                                                                           
└─sdb1 ext4    1.0         datapartition  ecedb020-6d19-4950-9292-7117fe5a5480                
sr0    iso9660 Joliet Exte VBox_GAs_7.0.6 2023-01-11-16-28-12-93                     0   100% /media/tatiana/VBox_GAs_7.0.6
```
_Step 5 — Mount the New Filesystem_
```
sudo mkdir -p /mnt/data
sudo mount -o defaults /dev/sdb1 /mnt/data
sudo vi /etc/fstab
```
_В файл добавлены строки:_
```
. . .
#My new disk
LABEL=datapartition /mnt/data ext4 defaults 0 2
```
```
root@ubuntu:~# sudo mount -a

root@ubuntu:~# df -h
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           197M  1,6M  195M   1% /run
/dev/sda3        24G   15G  8,4G  64% /
tmpfs           982M  1,1M  981M   1% /dev/shm
tmpfs           5,0M  4,0K  5,0M   1% /run/lock
/dev/sda2       512M  6,1M  506M   2% /boot/efi
tmpfs           197M  112K  197M   1% /run/user/1001
/dev/sr0         51M   51M     0 100% /media/tatiana/VBox_GAs_7.0.6
/dev/sdb1       9,8G   24K  9,3G   1% /mnt/data

root@ubuntu:~# echo "success" | sudo tee /mnt/data/test_file
success

root@ubuntu:~# cat /mnt/data/test_file
success
```
* Инстанс перезагружен, диск остался примонтированным :
```
root@ubuntu:~# reboot
tatiana@ubuntu:~$ uptime
 15:17:17 up 0 min,  1 user,  load average: 2,16, 0,60, 0,20
tatiana@ubuntu:~$ df -h
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           197M  1,6M  195M   1% /run
/dev/sda3        24G   15G  8,4G  64% /
tmpfs           982M  1,1M  981M   1% /dev/shm
tmpfs           5,0M  4,0K  5,0M   1% /run/lock
/dev/sdb1       9,8G   28K  9,3G   1% /mnt/data
/dev/sda2       512M  6,1M  506M   2% /boot/efi
tmpfs           197M  112K  197M   1% /run/user/1001
/dev/sr0         51M   51M     0 100% /media/tatiana/VBox_GAs_7.0.6
```
* Пользователь postgres назначен владельцем /mnt/data
```
root@ubuntu:~# chown -R postgres:postgres /mnt/data/
root@ubuntu:~# ls -l /mnt
total 4
drwxr-xr-x 3 postgres postgres 4096 июн 26 15:13 data
```
* Перенесено содержимое /var/lib/postgres/15 в /mnt/data
```
root@ubuntu:~# mv /var/lib/postgresql/15 /mnt/data
```
* Попытка запустить кластер не удалась,  т.к. все конфиги для старта кластера не найдены. 
```
root@ubuntu:~# sudo -u postgres pg_ctlcluster 15 main start
Error: /var/lib/postgresql/15/main is not accessible or does not exist
```
* Задание: найти конфигурационный параметр в файлах расположенных в /etc/postgresql/15/main который надо поменять и поменяйте его
```
root@ubuntu:/etc/postgresql/15/main# vi postgresql.conf
```
_В файле изменили параметр data_directory, который задаёт каталог, в котором хранятся данные_ 
```
#------------------------------------------------------------------------------
# FILE LOCATIONS
#------------------------------------------------------------------------------
# The default values of these variables are driven from the -D command-line
# option or PGDATA environment variable, represented here as ConfigDir.
data_directory = '/mnt/data/15/main/'          # use data in another directory
                                        # (change requires restart)
```
* Попытка запустить кластер увенчалась успехом, т.к. указали где искать файлы для запуска кластера 
```
root@ubuntu:/etc/postgresql/15/main# sudo -u postgres pg_ctlcluster 15 main start
root@ubuntu:/etc/postgresql/15/main# pg_lsclusters
Ver Cluster Port Status Owner    Data directory    Log file
15  main    5432 online postgres /mnt/data/15/main /var/log/postgresql/postgresql-15-main.log
```
* Зашли через через psql и проверили содержимое ранее созданной таблицы
```
postgres@ubuntu:~$ psql
postgres=# select * from accounts;
 user_id | username | password |     email      
---------+----------+----------+----------------
       1 | Ivan     | Pass     | Ivan@mail.ru
       2 | Poman    | Pass     | Poman@mail.ru
       3 | Bogdan   | Pass     | Bogdan@mail.ru
(3 rows)
```
