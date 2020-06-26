
В ДЗ выполнено основное задание и первое задание со *

Изменённый Vagrant находится в корне репозитория.
Сделано:

	Уменьшен / до 8G
	/var перенесён в зеркало
	Под /home выделен отдельный том
	Для /home сделан и восстановлен снэпшот

В корневой папке находится скрипт(file typescript) записанный с помощью утилиты script

## Основное ДЗ

Установим  xfsdump
Создадим PV
	
	[root@lvm vagrant]# pvcreate /dev/sdb
	  Physical volume "/dev/sdb" successfully created.
	  
Создадим временную VG и LV на ней

	[root@lvm vagrant]#  vgcreate vg_root /dev/sdb
	  Volume group "vg_root" successfully created
	[root@lvm vagrant]# lvcreate -n lv_root -l +100%FREE /dev/vg_root
	Logical volume "lv_root" created.

Создаём и монтируем ФС.

	[root@lvm vagrant]# mkfs.xfs /dev/vg_root/lv_root
	meta-data=/dev/vg_root/lv_root   isize=512    agcount=4, agsize=655104 blks
			 =                       sectsz=512   attr=2, projid32bit=1
			 =                       crc=1        finobt=0, sparse=0
	data     =                       bsize=4096   blocks=2620416, imaxpct=25
			 =                       sunit=0      swidth=0 blks
	naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
	log      =internal log           bsize=4096   blocks=2560, version=2
			 =                       sectsz=512   sunit=0 blks, lazy-count=1
	realtime =none                   extsz=4096   blocks=0, rtextents=0
	[root@lvm vagrant]#  mount /dev/vg_root/lv_root /mnt

С помощью **xfsdump** делаем бэкап / в /mnt

	root@lvm vagrant]#  xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt
	...
	xfsrestore: Restore Status: SUCCESS
	
Изменяем GRUB для того, чтобы при запуске использовать новый /

	[root@lvm vagrant]# for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
	[root@lvm vagrant]# chroot /mnt/
	[root@lvm /]# grub2-mkconfig -o /boot/grub2/grub.cfg
	Generating grub configuration file ...
	Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
	Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
	done

Обновляем initrd

	[root@lvm boot]# cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done
	...
	*** Creating image file ***
	*** Creating image file done ***
	*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***

Ну и для того, чтобы при загрузке был смонтирован нужно root нужно в файле. Здесь быть внимательным, иначе придётся возится загрузчиком.
/boot/grub2/grub.cfg заменить rd.lvm.lv=VolGroup00/LogVol00 на rd.lvm.lv=vg_root/lv_root

После перезагрузки проверяем блочные устройства

	[root@lvm vagrant]# lsblk
	NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
	sda                       8:0    0   10G  0 disk 
	└─vg_root-lv_root       253:0    0   10G  0 lvm  /
	sdb                       8:16   0    2G  0 disk 
	sdc                       8:32   0    1G  0 disk 
	sdd                       8:48   0    1G  0 disk 
	sde                       8:64   0   40G  0 disk 
	├─sde1                    8:65   0    1M  0 part 
	├─sde2                    8:66   0    1G  0 part /boot
	└─sde3                    8:67   0   39G  0 part 
	  ├─VolGroup00-LogVol00 253:1    0 37.5G  0 lvm  
	  └─VolGroup00-LogVol01 253:2    0  1.5G  0 lvm  [SWAP]

Теперь изменяем размер старой VG и возвращаем на него рут. Для этого удаляем старый LV размером в 40G и создаем новый на 8G повторяем шаги выполненные в начале:

	[root@lvm vagrant]#  lvremove /dev/VolGroup00/LogVol00
	Do you really want to remove active logical volume VolGroup00/LogVol00? [y/n]: n
	  Logical volume VolGroup00/LogVol00 not removed.
	[root@lvm vagrant]# lvs
	  LV       VG         Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
	  LogVol00 VolGroup00 -wi-a----- <37.47g                                                    
	  LogVol01 VolGroup00 -wi-ao----   1.50g                                                    
	  lv_root  vg_root    -wi-ao---- <10.00g                                                    
	[root@lvm vagrant]#  lvremove /dev/VolGroup00/LogVol00
	Do you really want to remove active logical volume VolGroup00/LogVol00? [y/n]: y
	  Logical volume "LogVol00" successfully removed
	[root@lvm vagrant]# lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00
	WARNING: xfs signature detected on /dev/VolGroup00/LogVol00 at offset 0. Wipe it? [y/n]: y
	  Wiping xfs signature on /dev/VolGroup00/LogVol00.
	  Logical volume "LogVol00" created.
	[root@lvm vagrant]# mkfs.xfs /dev/VolGroup00/LogVol00
	meta-data=/dev/VolGroup00/LogVol00 isize=512    agcount=4, agsize=524288 blks
			 =                       sectsz=512   attr=2, projid32bit=1
			 =                       crc=1        finobt=0, sparse=0
	data     =                       bsize=4096   blocks=2097152, imaxpct=25
			 =                       sunit=0      swidth=0 blks
	naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
	log      =internal log           bsize=4096   blocks=2560, version=2
			 =                       sectsz=512   sunit=0 blks, lazy-count=1
	realtime =none                   extsz=4096   blocks=0, rtextents=0
	[root@lvm vagrant]# mount /dev/VolGroup00/LogVol00 /mnt
	[root@lvm vagrant]# xfsdump -J - /dev/vg_root/lv_root | xfsrestore -J - /mnt
	
Так же как в первый раз переконфигурируем grub, за исключением правки /etc/grub2/grub.cfg

	for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
	chroot /mnt/
	grub2-mkconfig -o /boot/grub2/grub.cfg
	cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g;s/.img//g"` --force; done
	
Пока не перезагружаемся и не выходим из под chroot - мы можем заодно перенести /var

	[root@lvm boot]# pvcreate /dev/sdc /dev/sdd
	  Physical volume "/dev/sdc" successfully created.
	  Physical volume "/dev/sdd" successfully created.
	[root@lvm boot]# vgcreate vg_var /dev/sdc /dev/sdd
	  Volume group "vg_var" successfully created
	[root@lvm boot]# lvcreate -L 950M -m1 -n lv_var vg_var
	  Rounding up size to full physical extent 952.00 MiB
	  Logical volume "lv_var" created.
	[root@lvm boot]#  mkfs.ext4 /dev/vg_var/lv_var
	...
	Writing superblocks and filesystem accounting information: done
	root@lvm boot]# mount /dev/vg_var/lv_var /mnt
	[root@lvm boot]# cp -aR /var/* /mnt/ # rsync -avHPSAX /var/ /mnt/
	
сохраняем содержимое старого var, монтируем новЫй var в каталог /var, правим fstab для автоматического монтирования /var:

	[root@lvm boot]# mkdir /tmp/oldvar && mv /var/* /tmp/oldvar
	[root@lvm boot]#  umount /mnt
	[root@lvm boot]#  mount /dev/vg_var/lv_var /var
	[root@lvm boot]# echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab
	
Перезагружаемся
Удаляем временную VG и PV

	[root@lvm vagrant]#  lvremove /dev/vg_root/lv_root
	Do you really want to remove active logical volume vg_root/lv_root? [y/n]: y
	  Logical volume "lv_root" successfully removed
	[root@lvm vagrant]# vgremove /dev/vg_root
	  Volume group "vg_root" successfully removed
	[root@lvm vagrant]#  pvremove /dev/sdb
	  Labels on physical volume "/dev/sdb" successfully wiped.
	
Выделяем том под /home
	
	[root@lvm vagrant]# lvcreate -n LogVol_Home -L 2G /dev/VolGroup00
	[root@lvm vagrant]# mkfs.xfs /dev/VolGroup00/LogVol_Home
	[root@lvm vagrant]# mount /dev/VolGroup00/LogVol_Home /mnt/
	[root@lvm vagrant]# mount /dev/VolGroup00/LogVol_Home /mnt/
	[root@lvm vagrant]#  cp -aR /home/* /mnt/ 
	[root@lvm vagrant]#  rm -rf /home/*
	[root@lvm vagrant]# umount /mnt
	[root@lvm vagrant]#  mount /dev/VolGroup00/LogVol_Home /home/

Правим fstab для автоматического монтирования /home
	
	 echo "`blkid | grep Home | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab

Сгенерируем файлы в /home/:

	touch /home/file{1..20}

Создаём снэпшот

	[root@lvm vagrant]#  lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home
	  Rounding up size to full physical extent 128.00 MiB
	  Logical volume "home_snap" created.

Удаляем часть файлов:

	[root@otuslinux ~]# rm -f /home/file{11..20}
	
Процесс восстановления со снапшота:

	root@lvm vagrant]#  rm -f /home/file{11..20}
	[root@lvm vagrant]# umount /home
	[root@lvm vagrant]# lvconvert --merge /dev/VolGroup00/home_snap
	  Merging of volume VolGroup00/home_snap started.
	  VolGroup00/LogVol_Home: Merged: 100.00%
	[root@lvm vagrant]# mount /home
	
## Развёртывание ZFS

Устанавливаем ZFS по мануалу https://openzfs.github.io/openzfs-docs/Getting%20Started/RHEL%20and%20CentOS.html.
	
	sudo yum install http://download.zfsonlinux.org/epel/zfs-release.el7_8.noarch.rpm
	vim /etc/yum.repos.d/zfs.repo
	sudo yum install zfs
	Disk Requirements:
		At least 48MB more space needed on the /var filesystem.

Увидели, что не хватает места

	root@lvm vagrant]# lvextend -L +50m /dev/vg_var/lv_var /dev/sdd
	  Rounding size to boundary between physical extents: 52.00 MiB.
	  Extending 2 mirror images.
	Insufficient free space: 26 extents needed, but only 16 available
  
	[root@lvm vagrant]# lvcreate -n VolGroup00/vg_var -L 4G /dev/VolGroup00
	[root@lvm vagrant]# mkfs.xfs /dev/VolGroup00/vg_var 
	
	[root@lvm vagrant]# pvcreate /dev/sdc
	  Physical volume "/dev/sdc" successfully created.

Перенесём var на новую VG

	[root@lvm vagrant]# vgcreate vg_var1 /dev/sdc
	  Volume group "vg_var1" successfully created
	  [root@lvm vagrant]# lvs
	  LV          VG         Attr       LSize    Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
	  LogVol00    VolGroup00 -wi-ao----    8.00g                                                    
	  LogVol01    VolGroup00 -wi-ao----    1.50g                                                    
	  LogVol_Home VolGroup00 -wi-ao----    2.00g                                                    
	  lv_var      vg_var     rwi-aor--- 1016.00m                                    100.00          
	  lv_var1     vg_var1    -wi-a-----   <2.00g                                                    

	[root@lvm vagrant]# mkfs.ext4 /dev/vg_var1/lv_var1
	Writing superblocks and filesystem accounting information: done
	
	echo "`blkid | grep var1: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab

После чего убираем строчку из /etc/fstab для пердыдущего тома для var
Убираем ранее созданную VG:
	
	[root@lvm vagrant]# vgremove vg_var
	Do you really want to remove volume group "vg_var" containing 1 logical volumes? [y/n]: y
	Do you really want to remove active logical volume vg_var/lv_var? [y/n]: y
	Logical volume "lv_var" successfully removed
	Volume group "vg_var" successfully removed

	[root@lvm vagrant]# /sbin/modprobe zfs
	[root@lvm vagrant]# zpool create zpool01 raidz sdb sdd sde -f
	[root@lvm vagrant]# zpool list
	NAME      SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
	zpool01  2.75G   208K  2.75G        -         -     0%     0%  1.00x    ONLINE  -
