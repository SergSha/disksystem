<h3>### Дисковая подсистема ###</h3>

<h4>Описание домашнего задания</h4>

<ul>
<li>добавить в Vagrantfile еще дисков;</li>
<li>собрать R0/R5/R10 на выбор;</li>
<li>прописать собранный рейд в конф, чтобы рейд собирался при загрузке;</li>
<li>сломать/починить raid;</li>
<li>создать GPT раздел и 5 партиций. В качестве проверки принимаются - измененный Vagrantfile, скрипт для создания рейда, конф для автосборки рейда при загрузке.</li>
<li>Доп. задание - Vagrantfile, который сразу собирает систему с подключенным рейдом и смонтированными разделами. После перезагрузки стенда разделы должны автоматически примонтироваться.</li>
</ul>
<br />



<h4># Добавить в Vagrantfile еще дисков</h4>

<p>В домашней директории создадим директорию disksystem, в которой будут храниться настройки виртуальной машины:</p>

<pre>[user@localhost otus]$ mkdir ./disksystem
[user@localhost otus]$</pre>

<p>Перейдём в директорию disksystem:</p>

<pre>[user@localhost otus]$ cd ./disksystem/
[user@localhost disksystem]$</pre>

<p>Создадим файл Vagrantfile:</p>

<pre>[user@localhost disksystem]$ vi ./Vagrantfile</pre>

<p>Заполним следующим содержимым:</p>

<pre># -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
  :otuslinux => {
        :box_name => "centos/7",
        :ip_addr => '192.168.56.101',
        :disks => {
                :sata1 => {
                        :dfile => './sata1.vdi',
                        :size => 250,
                        :port => 1
                },
                :sata2 => {
                        :dfile => './sata2.vdi',
                        :size => 250, # Megabytes
                        :port => 2
                },
                :sata3 => {
                        :dfile => './sata3.vdi',
                        :size => 250,
                        :port => 3
                },
                :sata4 => {
                        :dfile => './sata4.vdi',
                        :size => 250, # Megabytes
                        :port => 4
                }

        }


  },
}

Vagrant.configure("2") do |config|

  MACHINES.each do |boxname, boxconfig|

      config.vm.define boxname do |box|

          box.vm.box = boxconfig[:box_name]
          box.vm.host_name = boxname.to_s
          #box.vm.network "forwarded_port", guest: 3260, host: 3260+offset

          box.vm.network "private_network", ip: boxconfig[:ip_addr]

          box.vm.provider :virtualbox do |vb|
                  vb.customize ["modifyvm", :id, "--memory", "1024"]
                  needsController = false
                  boxconfig[:disks].each do |dname, dconf|
                          unless File.exist?(dconf[:dfile])
                                vb.customize ['createhd', '--filename', dconf[:dfile], '--variant', 'Fixed', '--size', dconf[:size]]
                                needsController =  true
                          end

                  end
                  if needsController == true
                     vb.customize ["storagectl", :id, "--name", "SATA", "--add", "sata" ]
                     boxconfig[:disks].each do |dname, dconf|
                         vb.customize ['storageattach', :id,  '--storagectl', 'SATA', '--port', dconf[:port], '--device', 0, '--type', 'hdd', '--medium', dconf[:dfile]]
                     end
                  end
          end
          box.vm.provision "shell", inline: <<-SHELL
              mkdir -p ~root/.ssh
              cp ~vagrant/.ssh/auth* ~root/.ssh
              yum install -y mdadm smartmontools hdparm gdisk
          SHELL

      end
  end
end
</pre>

<p>После блока sata4 добавим блок sata5:</p>

<pre>
...
                :sata4 => {
                        :dfile => './sata4.vdi',
                        :size => 250, # Megabytes
                        :port => 4
                },
                :sata5 => {
                        :dfile => './sata5.vdi',
                        :size => 250, # Megabytes
                        :port => 5
                }
...
</pre>

<p>Запустим систему:</p>

<pre>[user@localhost disksystem]$ vagrant up</pre>

<p>и войдём в неё:</p>

<pre>[user@localhost disksystem]$ vagrant ssh
[vagrant@otuslinux ~]$</pre>

<p>Заходим под правми root:</p>

<pre>[vagrant@otuslinux ~]$ sudo -i
[root@otuslinux ~]#</pre>

<h4># Собрать RAID0/1/5/10 - на выбор</h4>

<p>Далее нужно определиться какого уровня RAID будем собирать. Для это посмотрим какие блочные устройства у нас есть и исходя из их кол-во, размера и поставленной задачи определимся.<br />Сделать это можно несколькими способами:</p>

<pre>[root@otuslinux ~]# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   40G  0 disk 
`-sda1   8:1    0   40G  0 part /
sdb      8:16   0  250M  0 disk 
sdc      8:32   0  250M  0 disk 
sdd      8:48   0  250M  0 disk 
sde      8:64   0  250M  0 disk 
sdf      8:80   0  250M  0 disk 
[root@otuslinux ~]#</pre>

<pre>[root@otuslinux ~]# lsscsi
[0:0:0:0]    disk    ATA      VBOX HARDDISK    1.0   /dev/sda 
[3:0:0:0]    disk    ATA      VBOX HARDDISK    1.0   /dev/sdb 
[4:0:0:0]    disk    ATA      VBOX HARDDISK    1.0   /dev/sdc 
[5:0:0:0]    disk    ATA      VBOX HARDDISK    1.0   /dev/sdd 
[6:0:0:0]    disk    ATA      VBOX HARDDISK    1.0   /dev/sde 
[7:0:0:0]    disk    ATA      VBOX HARDDISK    1.0   /dev/sdf 
[root@otuslinux ~]#</pre>

<pre>[root@otuslinux ~]# lshw -short | grep disk
/0/100/1.1/0.0.0    /dev/sda  disk        42GB VBOX HARDDISK
/0/100/d/0          /dev/sdb  disk        262MB VBOX HARDDISK
/0/100/d/1          /dev/sdc  disk        262MB VBOX HARDDISK
/0/100/d/2          /dev/sdd  disk        262MB VBOX HARDDISK
/0/100/d/3          /dev/sde  disk        262MB VBOX HARDDISK
/0/100/d/0.0.0      /dev/sdf  disk        262MB VBOX HARDDISK
[root@otuslinux ~]#</pre>

<pre>[root@otuslinux ~]# fdisk -l

Disk /dev/sdf: 262 MB, 262144000 bytes, 512000 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/sda: 42.9 GB, 42949672960 bytes, 83886080 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x0009ef1a

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048    83886079    41942016   83  Linux

Disk /dev/sdb: 262 MB, 262144000 bytes, 512000 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/sdc: 262 MB, 262144000 bytes, 512000 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/sde: 262 MB, 262144000 bytes, 512000 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/sdd: 262 MB, 262144000 bytes, 512000 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

[root@otuslinux ~]#</pre>

<p>Занулим на всякий случай суперблоки:</p>

<pre>[root@otuslinux ~]# mdadm --zero-superblock --force /dev/sd{b,c,d,e,f}
mdadm: Unrecognised md component device - /dev/sdb
mdadm: Unrecognised md component device - /dev/sdc
mdadm: Unrecognised md component device - /dev/sdd
mdadm: Unrecognised md component device - /dev/sde
mdadm: Unrecognised md component device - /dev/sdf
[root@otuslinux ~]#</pre>

<p>Создаём рейд (в данном случае выбран RAID 5) следующей командой:</p>

<pre>[root@otuslinux ~]# mdadm --create --verbose --force /dev/md0 -l 5 -n 5 /dev/sd{b,c,d,e,f}
mdadm: layout defaults to left-symmetric
mdadm: layout defaults to left-symmetric
mdadm: chunk size defaults to 512K
mdadm: size set to 253952K
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md0 started.
[root@otuslinux ~]#</pre>

<p>Проверим, что RAID собрался корректно:</p>

<pre>[root@otuslinux ~]# cat /proc/mdstat 
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid5 sdf[4] sde[3] sdd[2] sdc[1] sdb[0]
      1015808 blocks super 1.2 level 5, 512k chunk, algorithm 2 [5/5] [UUUUU]
      
unused devices: <none>
[root@otuslinux ~]#</pre>

<p>Как видим, level 5 - RAID 5, размер одного chunk 512 кб, [UUUUU] - 5 юнитов.</p>

<pre>[root@otuslinux ~]# mdadm --detail /dev/md0 
/dev/md0:
           Version : 1.2
     Creation Time : Sun Jun 19 15:56:48 2022
        Raid Level : raid5
        Array Size : 1015808 (992.00 MiB 1040.19 MB)
     Used Dev Size : 253952 (248.00 MiB 260.05 MB)
      Raid Devices : 5
     Total Devices : 5
       Persistence : Superblock is persistent

       Update Time : Sun Jun 19 15:56:51 2022
             State : clean 
    Active Devices : 5
   Working Devices : 5
    Failed Devices : 0
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : otuslinux:0  (local to host otuslinux)
              UUID : 6695ff37:328cd15c:534aaa7f:b684dd54
            Events : 17

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       2       8       48        2      active sync   /dev/sdd
       3       8       64        3      active sync   /dev/sde
       4       8       80        4      active sync   /dev/sdf
[root@otuslinux ~]#</pre>

<h4># Создание конфигурационного файла mdadm.conf</h4>

<p>Для того, чтобы ОС запомнила какой RAID массив требуется создать и какие компоненты в него входят создадим файл mdadm.conf.<br />Сначала убедимся, что информация верна:</p>

<pre>[root@otuslinux ~]# mdadm --detail --scan --verbose
ARRAY /dev/md0 level=raid5 num-devices=5 metadata=1.2 name=otuslinux:0 UUID=6695ff37:328cd15c:534aaa7f:b684dd54
   devices=/dev/sdb,/dev/sdc,/dev/sdd,/dev/sde,/dev/sdf
[root@otuslinux ~]#</pre>

<p>Создаём файл mdadm.conf</p>

<pre>[root@otuslinux ~]# mkdir /etc/mdadm/
[root@otuslinux ~]#</pre>

<pre>[root@otuslinux ~]# echo "DEVICE partitions" > /etc/mdadm/mdadm.conf
[root@otuslinux ~]#</pre>

<pre>[root@otuslinux ~]# mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf 
[root@otuslinux ~]#</pre>

<p>Содержимое полученного файла mdadm.conf</p>

<pre>[root@otuslinux ~]# cat /etc/mdadm/mdadm.conf 
DEVICE partitions
ARRAY /dev/md0 level=raid5 num-devices=5 metadata=1.2 name=otuslinux:0 UUID=6695ff37:328cd15c:534aaa7f:b684dd54
[root@otuslinux ~]#</pre>

<p>Перезагрузим систему:</p>

<pre>[root@otuslinux ~]# shutdown -r now
Connection to 127.0.0.1 closed by remote host.
Connection to 127.0.0.1 closed.</pre>

<p>Заходим в систему:</p>

<pre>[user@localhost disksystem]$ vagrant ssh
Last login: Sun Jun 19 15:38:04 2022 from 10.0.2.2
[vagrant@otuslinux ~]$</pre>

<p>Проверяем, загрузился ли RAID:</p>

<pre>[vagrant@otuslinux ~]$ sudo mdadm -D /dev/md0 
/dev/md0:
           Version : 1.2
     Creation Time : Sun Jun 19 15:56:48 2022
        Raid Level : raid5
        Array Size : 1015808 (992.00 MiB 1040.19 MB)
     Used Dev Size : 253952 (248.00 MiB 260.05 MB)
      Raid Devices : 5
     Total Devices : 5
       Persistence : Superblock is persistent

       Update Time : Sun Jun 19 15:56:51 2022
             State : clean 
    Active Devices : 5
   Working Devices : 5
    Failed Devices : 0
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : otuslinux:0  (local to host otuslinux)
              UUID : 6695ff37:328cd15c:534aaa7f:b684dd54
            Events : 17

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       2       8       48        2      active sync   /dev/sdd
       3       8       64        3      active sync   /dev/sde
       4       8       80        4      active sync   /dev/sdf
[vagrant@otuslinux ~]$</pre>

<p>Как видим, RAID загрузился и работает.</p>

<h4># Сломать/починить RAID</h4>

<p>Сделаем это можно искусственно “сломав” одно из блочных устройств опцией --fail, но предварительно зайдём под правами root:</p>

<pre>[vagrant@otuslinux ~]$ sudo -i
[root@otuslinux ~]#</pre>

<pre>[root@otuslinux ~]# mdadm /dev/md0 --fail /dev/sdc
mdadm: set /dev/sdc faulty in /dev/md0
[root@otuslinux ~]#</pre>

<p>Смотрим как это отразилось на RAID:</p>

<pre>[root@otuslinux ~]# cat /proc/mdstat 
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid5 sdf[4] sde[3] sdb[0] sdd[2] sdc[1](F)
      1015808 blocks super 1.2 level 5, 512k chunk, algorithm 2 [5/4] [U_UUU]
      
unused devices: <none>
[root@otuslinux ~]#</pre>

<p>Из этого [U_UUU], видим, что отсутствует второй юнит.</p>

<pre>[root@otuslinux ~]# mdadm -D /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Sun Jun 19 15:56:48 2022
        Raid Level : raid5
        Array Size : 1015808 (992.00 MiB 1040.19 MB)
     Used Dev Size : 253952 (248.00 MiB 260.05 MB)
      Raid Devices : 5
     Total Devices : 5
       Persistence : Superblock is persistent

       Update Time : Sun Jun 19 17:21:10 2022
             State : clean, degraded 
    Active Devices : 4
   Working Devices : 4
    Failed Devices : 1
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : otuslinux:0  (local to host otuslinux)
              UUID : 6695ff37:328cd15c:534aaa7f:b684dd54
            Events : 19

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       -       0        0        1      removed
       2       8       48        2      active sync   /dev/sdd
       3       8       64        3      active sync   /dev/sde
       4       8       80        4      active sync   /dev/sdf

       1       8       32        -      faulty   /dev/sdc
[root@otuslinux ~]#</pre>

<p>Из этого тоже видно, что диск sdc в массиве RAID не работает.</p>

<p>Удалим этот "сломанный" диск из массива RAID, имитируя "горячее" отключение:</p>

<pre>[root@otuslinux ~]# mdadm /dev/md0 --remove /dev/sdc
mdadm: hot removed /dev/sdc from /dev/md0
[root@otuslinux ~]#</pre>

<p>Теперь имитируем вставку нового диска и добавим его в RAID:</p>

<pre>[root@otuslinux ~]# mdadm /dev/md0 --add /dev/sdc
mdadm: added /dev/sdc
[root@otuslinux ~]#</pre>

<p>Диск должен пройти стадию rebuilding. Процесс rebuilding можно увидеть с помощью следующих команд:</p>

<pre>[root@otuslinux ~]# cat /proc/mdstat 
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid5 sdc[5] sdf[4] sde[3] sdb[0] sdd[2]
      1015808 blocks super 1.2 level 5, 512k chunk, algorithm 2 [5/5] [UUUUU]
      
unused devices: <none>
[root@otuslinux ~]#</pre>

<pre>[root@otuslinux ~]# mdadm -D /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Sun Jun 19 15:56:48 2022
        Raid Level : raid5
        Array Size : 1015808 (992.00 MiB 1040.19 MB)
     Used Dev Size : 253952 (248.00 MiB 260.05 MB)
      Raid Devices : 5
     Total Devices : 5
       Persistence : Superblock is persistent

       Update Time : Sun Jun 19 17:44:34 2022
             State : clean 
    Active Devices : 5
   Working Devices : 5
    Failed Devices : 0
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : otuslinux:0  (local to host otuslinux)
              UUID : 6695ff37:328cd15c:534aaa7f:b684dd54
            Events : 61

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       5       8       32        1      active sync   /dev/sdc
       2       8       48        2      active sync   /dev/sdd
       3       8       64        3      active sync   /dev/sde
       4       8       80        4      active sync   /dev/sdf
[root@otuslinux ~]#</pre>

<p>Из-за очень маленького объёма занятого прострарства процесс перестроения RAID происходит практиччески мгновенно.</p>

<h4># Создать GPT раздел, пять партиций и смонтировать их на диск</h4>

<p>Создаем раздел GPT на созданном RAID:</p>

<pre>[root@otuslinux ~]# parted -s /dev/md0 mklabel gpt
[root@otuslinux ~]#</pre>

<p>Создаём партиции:</p>

<pre>[root@otuslinux ~]# parted /dev/md0 mkpart primary ext4 0% 20%
Information: You may need to update /etc/fstab.

[root@otuslinux ~]# parted /dev/md0 mkpart primary ext4 20% 40%           
Information: You may need to update /etc/fstab.

[root@otuslinux ~]# parted /dev/md0 mkpart primary ext4 40% 60%          
Information: You may need to update /etc/fstab.

[root@otuslinux ~]# parted /dev/md0 mkpart primary ext4 60% 80%          
Information: You may need to update /etc/fstab.

[root@otuslinux ~]# parted /dev/md0 mkpart primary ext4 80% 100%         
Information: You may need to update /etc/fstab.

[root@otuslinux ~]#</pre>

<p>На этих созданных партициях создаём файловые системы:</p>

<pre>[root@otuslinux ~]# for i in $(seq 1 5); do mkfs.ext4 /dev/md0p$i; done
mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=1024 (log=0)
Fragment size=1024 (log=0)
Stride=512 blocks, Stripe width=2048 blocks
50200 inodes, 200704 blocks
10035 blocks (5.00%) reserved for the super user
First data block=1
Maximum filesystem blocks=33816576
25 block groups
8192 blocks per group, 8192 fragments per group
2008 inodes per group
Superblock backups stored on blocks: 
	8193, 24577, 40961, 57345, 73729

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done 

mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=1024 (log=0)
Fragment size=1024 (log=0)
Stride=512 blocks, Stripe width=2048 blocks
50800 inodes, 202752 blocks
10137 blocks (5.00%) reserved for the super user
First data block=1
Maximum filesystem blocks=33816576
25 block groups
8192 blocks per group, 8192 fragments per group
2032 inodes per group
Superblock backups stored on blocks: 
	8193, 24577, 40961, 57345, 73729

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done 

mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=1024 (log=0)
Fragment size=1024 (log=0)
Stride=512 blocks, Stripe width=2048 blocks
51200 inodes, 204800 blocks
10240 blocks (5.00%) reserved for the super user
First data block=1
Maximum filesystem blocks=33816576
25 block groups
8192 blocks per group, 8192 fragments per group
2048 inodes per group
Superblock backups stored on blocks: 
	8193, 24577, 40961, 57345, 73729

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done 

mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=1024 (log=0)
Fragment size=1024 (log=0)
Stride=512 blocks, Stripe width=2048 blocks
50800 inodes, 202752 blocks
10137 blocks (5.00%) reserved for the super user
First data block=1
Maximum filesystem blocks=33816576
25 block groups
8192 blocks per group, 8192 fragments per group
2032 inodes per group
Superblock backups stored on blocks: 
	8193, 24577, 40961, 57345, 73729

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done 

mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=1024 (log=0)
Fragment size=1024 (log=0)
Stride=512 blocks, Stripe width=2048 blocks
50200 inodes, 200704 blocks
10035 blocks (5.00%) reserved for the super user
First data block=1
Maximum filesystem blocks=33816576
25 block groups
8192 blocks per group, 8192 fragments per group
2008 inodes per group
Superblock backups stored on blocks: 
	8193, 24577, 40961, 57345, 73729

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done 

[root@otuslinux ~]#</pre>

<p>и смонтируем их по каталогам:</p>

<pre>[root@otuslinux ~]# mkdir -p /raid/part{1,2,3,4,5}
[root@otuslinux ~]# for i in $(seq 1 5); do mount /dev/md0p$i /raid/part$i; done
[root@otuslinux ~]#</pre>

<p>Смотрим содержимое директории /raid:</p>

<pre>[root@otuslinux ~]# ls -l /raid/
total 5
drwxr-xr-x. 3 root root 1024 Jun 19 18:06 part1
drwxr-xr-x. 3 root root 1024 Jun 19 18:06 part2
drwxr-xr-x. 3 root root 1024 Jun 19 18:06 part3
drwxr-xr-x. 3 root root 1024 Jun 19 18:06 part4
drwxr-xr-x. 3 root root 1024 Jun 19 18:06 part5
[root@otuslinux ~]#</pre>

<p>Смотрим список примонтированных только созданных партиций:</p>

<pre>[root@otuslinux ~]# mount | grep part
/dev/md0p1 on /raid/part1 type ext4 (rw,relatime,seclabel,stripe=2048,data=ordered)
/dev/md0p2 on /raid/part2 type ext4 (rw,relatime,seclabel,stripe=2048,data=ordered)
/dev/md0p3 on /raid/part3 type ext4 (rw,relatime,seclabel,stripe=2048,data=ordered)
/dev/md0p4 on /raid/part4 type ext4 (rw,relatime,seclabel,stripe=2048,data=ordered)
/dev/md0p5 on /raid/part5 type ext4 (rw,relatime,seclabel,stripe=2048,data=ordered)
[root@otuslinux ~]#</pre>

<p>Чтобы эти партиции автоматически монтировались при загрузке системы, добавим их в /etc/fstab:</p>

<pre>[root@otuslinux ~]# for i in $(seq 1 5); do echo "/dev/md0p$i /raid/part$i ext4 defaults 0 0" >> /etc/fstab; done
[root@otuslinux ~]#</pre>

<p>Проверим содержимое fstab:</p>

<pre>[root@otuslinux ~]# cat /etc/fstab 

#
# /etc/fstab
# Created by anaconda on Thu Apr 30 22:04:55 2020
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
UUID=1c419d6c-5064-4a2b-953c-05b2c67edb15 /                       xfs     defaults        0 0
/swapfile none swap defaults 0 0
#VAGRANT-BEGIN
# The contents below are automatically generated by Vagrant. Do not modify.
#VAGRANT-END
/dev/md0p1 /raid/part1 ext4 defaults 0 0
/dev/md0p2 /raid/part2 ext4 defaults 0 0
/dev/md0p3 /raid/part3 ext4 defaults 0 0
/dev/md0p4 /raid/part4 ext4 defaults 0 0
/dev/md0p5 /raid/part5 ext4 defaults 0 0
[root@otuslinux ~]#</pre>

<p>Перезагрузим систему:</p>

<pre>[root@otuslinux ~]# shutdown -r now
Connection to 127.0.0.1 closed by remote host.
Connection to 127.0.0.1 closed.
[user@localhost disksystem]$</pre>

<p>Снова заходим в систему:</p>

<pre>[user@localhost disksystem]$ vagrant ssh
Last login: Sun Jun 19 16:31:47 2022 from 10.0.2.2
[vagrant@otuslinux ~]$</pre>

<p>Смотрим список примонтированных партиций:</p>

<pre>[vagrant@otuslinux ~]$ mount | grep part
/dev/md0p2 on /raid/part2 type ext4 (rw,relatime,seclabel,stripe=2048,data=ordered)
/dev/md0p1 on /raid/part1 type ext4 (rw,relatime,seclabel,stripe=2048,data=ordered)
/dev/md0p3 on /raid/part3 type ext4 (rw,relatime,seclabel,stripe=2048,data=ordered)
/dev/md0p4 on /raid/part4 type ext4 (rw,relatime,seclabel,stripe=2048,data=ordered)
/dev/md0p5 on /raid/part5 type ext4 (rw,relatime,seclabel,stripe=2048,data=ordered)
[vagrant@otuslinux ~]$</pre>

<p>Как видим, после загрузки партиции примонтировались успешно.</p>

<h4># Доп. задание - Vagrantfile, который сразу собирает систему с подключенным рейдом и смонтированными разделами. После перезагрузки стенда разделы должны автоматически примонтироваться</h4>

<p><a href+"https://github.com/SergSha/disksystem/blob/ab80acb66308f48867703b1aba8c6a77a9589dc4/Vagrantfile">Vagrantfile</a></p>
