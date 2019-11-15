# stands-03-lvm

Стенд для домашнего занятия "Файловые системы и LVM"
#Скачиваю себе на машину необходимые файлы из гита
git clone git@gitlab.com:olegmichael/stands-03-lvm.git


#Стартуем вагрант
vagrant up

#подключаемся к вм и ставим xfsdump
vagrant ssh
sudo su
yum install xfsdump


#смотрим как смонтированы диски - все ок
lvmdiscscan
lsblk

#Создаем том и логический рздел на sdb
pvcreate /dev/sdb
vgcreate vg_root /dev/sdb

#Создаем файловую систему и монтируем 
lvcreate -n lv_root -l +100%FREE /dev/vg_root
mkfs.xfs /dev/vg_root/lv_root
mount /dev/vg_root/lv_root /mnt

#Копируем данные с раздела / в mnt
xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt
vagrant 

#Вносим изменения в grub для перехода в раздел /
for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
chroot /mnt/
grub2-mkconfig -o /boot/grub2/grub.cfg


#Обновляем образ initrd
cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done

#вносим изменения в конфиг grub для монтирования нужного root(меняем d.lvm.lv=VolGroup00/LogVol00 на rd.lvm.lv=vg_root/lv_root)
nano /boot/grub2/grub.cfg     

#Выходим из vagrant и рестартнем машину
exit
exit
exit
vagrant halt
vagrant up
vagrant ssh
sudo su

#Меняем размер VG и возвращаем рута. Удаляем старый LV и создаем новый на 8 Гб
lvremove /dev/VolGroup00/LogVol00
lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00


#Создаем файловую систему,монтируем и копируем данные с раздела / в mnt
mkfs.xfs /dev/VolGroup00/LogVol00
mount /dev/VolGroup00/LogVol00 /mnt
xfsdump -J - /dev/vg_root/lv_root | xfsrestore -J - /mnt

#Вносим изменения в grub для перехода в раздел /
for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
chroot /mnt/
grub2-mkconfig -o /boot/grub2/grub.cfg
cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done


#Создаем зеркало на sdc и sdd
pvcreate /dev/sdc /dev/sdd
vgcreate vg_var /dev/sdc /dev/sdd
lvcreate -L 950M -m1 -n lv_var vg_var

#Создаем файловую систему на зеркале и переносим туда /var
mkfs.ext4 /dev/vg_var/lv_var
cp -aR /var/* /mnt/      # rsync -avHPSAX /var/ /mnt/

#Сохраняем содержимое var в каталог oldvar
mkdir /tmp/oldvar && mv /var/* /tmp/oldvar

#Монтируем новый var  в каталог /var
umount /mnt
mount /dev/vg_var/lv_var /var

#Делаем монтирование /var в автоматическом режиме
echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab

#Перезагружаемся и в новый root и удаляем временную VolGroup
exit
exit
exit
vagrant halt
vagrant up
vagrant ssh
sudo su
vremove /dev/vg_root/lv_root
vgremove /dev/vg_root
pvremove /dev/sdb

#Создаем LV  и файловую систему под /home, делаем автомонтирование
lvcreate -n LogVol_Home -L 2G /dev/VolGroup00
mkfs.xfs /dev/VolGroup00/LogVol_Home
mount /dev/VolGroup00/LogVol_Home /mnt/
cp -aR /home/* /mnt/ 
rm -rf /home/*
umount /mnt
mount /dev/VolGroup00/LogVol_Home /home/
echo "`blkid | grep Home | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab

#Создаем файлы в /home и делаем snapshot
lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home

#Удаляем несколько файлов
rm -f /home/file{11..20}

#Восстанавливаемся из снапшота
umount /home
lvconvert --merge /dev/VolGroup00/home_snap
mount /home

