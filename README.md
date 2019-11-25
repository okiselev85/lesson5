## Проверяем варианты загрузки без пароля тремя способами. Каждое путем нажатия e  при загрузки ВМ вагрант через гуи
#### 1. Добавляем параметр init=/bin/sh     и жмем ctrl+X
#### 2. Добавляем параметр rd.break     и жмем ctrl+X.  Смена пароля в данном случае необязательна, сам факт проверки    работы такого варианта загрузки в аварийном режиме
#### 3. Добавляем параметр rw init=/sysroot/bin/sh     и жмем ctrl+X
* Ставим систему с LVM с переименованием VG и Смотрим статус системы:

*_vgs_* 

* Получили результат:

_VG         #PV #LV #SN Attr   VSize   VFree
VolGroup00   1   2   0 wz--n- <38.97g    0_

* Переименуем VolGroup в MyGroup:
*_vgrename VolGroup00 MyRoot_*
* Volume group "VolGroup00" successfully renamed to "MyRoot"

* правим fstab, grub и grub 2:
*_ /etc/fstab _*

* Created by anaconda on Sat May 12 18:50:26 2018

* Accessible filesystems, by reference, are maintained under '/dev/disk'

* See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info

* /dev/mapper/MyRoot-LogVol00 /                       xfs     defaults        0 0

* UUID=570897ca-e759-4c81-90cf-389da6eee4cc /boot                   xfs     defaults        0 0

* /dev/mapper/MyRoot-LogVol01 swap                    swap    defaults        0 0

* GRUB_TIMEOUT=1
* GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
* GRUB_DEFAULT=saved
* GRUB_DISABLE_SUBMENU=true
* GRUB_TERMINAL_OUTPUT="console"
* GRUB_CMDLINE_LINUX="no_timer_check console=tty0 console=ttyS0,115200n8 net.ifnames=0 biosdevname=0 elevator=noop  crashkernel=auto rd.lvm.lv=MyRoot/LogVol00 rd.lvm.lv=MyRoot/LogVol01 rhgb quiet"GRUB_DISABLE_RECOVERY="true"
* grub2, только измененная строка:

linux16 /vmlinuz-3.10.0-862.2.3.el7.x86_64 root=/dev/mapper/MyRoot-LogVol00 ro no_timer_check console=tty0 console=ttyS0,115200n8 net.ifnames=0 biosdevname=0 elevator=noop crashkernel=auto rd.lvm.lv=MyRoot/LogVol00 rd.lvm.lv=MyRoot/LogVol01 rhgb quiet

# Пересоздаем initrd:
[root@lvm /]# mkinitrd -f -v /boot/initramfs-$(uname -r).img $(uname -r)

# Перезагружаемся и смотрим результат vgs
[vagrant@lvm ~]$ sudo su
[root@lvm vagrant]# vgs
  VG     #PV #LV #SN Attr   VSize   VFree
  MyRoot   1   2   0 wz--n- <38.97g    0

#Добавляем модуль в initrd
#Создаем папку mytest в  usr/lib/dracut/modules.d/   а так же кладем в нее необходимые скрипты c заранее известным содержимым
[root@lvm vagrant]# mkdir /usr/lib/dracut/modules.d/mytest
[root@lvm vagrant]# cd /usr/lib/dracut/modules.d/
[root@lvm modules.d]# cd mytest/
[root@lvm mytest]# vi module-setup.sh
[root@lvm mytest]# vi test.sh
[root@lvm mytest]# ll
total 8
-rw-r--r--. 1 root root 126 Nov 22 14:18 module-setup.sh
-rw-r--r--. 1 root root 314 Nov 22 14:19 test.sh

#Пересобираем образ initrd, смотрим загруженные модули 
dracut -f -v
lsinitrd -m /boot/initramfs-$(uname -r).img | grep test

#Правим grub.cfg(убираем rghb quiet) и перезагружаемся
vi grub.cfg 
reboot

* Пингвин на скриншоте во вложении
