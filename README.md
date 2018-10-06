# ArchLinux_LVM_on_LUKS_UEFI
Install Arch Linux LVM on LUKS + EFI Stub

#####Install Arch Linux UEFI LVM LUKS
Инструкция расчитана на установку с использованием одного физического диска (`/dev/sda`).
LVM на LUKS
UEFI грузит ядро напрямую
___

Правильное время, что бы время создания файловых систем было правильное
```
timedatectl set-ntp true && timedatectl set-timezone Europe/Moscow
```
Создаем EFI раздел размером 100Mb, тип `EF00` и именуем `PARTLABEL` в `esp`
```
sgdisk -n=1::+100M --typecode=1:EF00 --change-name=1:"esp" /dev/sda
```
Создаем второй раздел с типом `8309` (Linux LUKS)
```
sgdisk -n=2:: --typecode=2:8309 /dev/sda
```

Форматируем EFI раздел
```
mkfs.vfat -F32 /dev/sda1
```
Форматируем партицию `/dev/sda2` в `LUKS`
```
cryptsetup -y --verbose --cipher twofish-xts-plain64 --key-size 512 --hash sha512 --pbkdf argon2i --iter-time 5000 --use-random --type luks2 luksFormat /dev/sda2
```
Подтверждаем то что данные на этой партиции будут аннигилированы.
Задаем пароль.
```
WARNING!
========
This will overwrite data on /dev/sda2 irrevocably.

Are you sure? (Type uppercase yes): YES
Enter passphrase for /dev/sda2: 
Verify passphrase: 
```
Открываем созданную партицию LUKS.
`haba_haba` - это имя блочного устройства в которое будет отображаться расшифрованная партиция
```
cryptsetup open /dev/sda2 haba_haba
```

Смотрим что получилось
```
lsblk
```
```
NAME          MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
loop0           7:0    0 459.6M  1 loop  /run/archiso/sfs/airootfs
sda             8:0    0     2G  0 disk  
├─sda1          8:1    0   100M  0 part  
└─sda2          8:2    0   1.9G  0 part  
  └─haba_haba 254:0    0   1.9G  0 crypt 

```
Инициализируем контейнер для использования в LVM2
```
pvcreate /dev/mapper/haba_haba
```
Создаем volume group
```
vgcreate vg00 /dev/mapper/haba_haba 
```
Создаем логический том для `swap`, 500Мб
```
lvcreate -L500M -n swap vg00
```

Создаем логический том для root партиции, на все оставшееся место
```
lvcreate -l 100%FREE -n root vg00
```
Форматируем
```
mkfs.ext4 -L root /dev/mapper/vg00-root
mkswap -L swap /dev/mapper/vg00-swap
```
Включаем swap
```
swapon /dev/mapper/vg00-swap
```
Монтируем корень нашей ФС
```
mount /dev/mapper/vg00-root /mnt/
```
Создаем папки
```
mkdir /mnt/{boot,esp}
```
Устанавливаем базовую систему и сразу нужные пакеты
```
pacstrap -i /mnt base sudo extra/vim openssh bash-completion
```
Генерируем конфиг для монтирования дисков при загрузке
```
genfstab -U -p /mnt >> /mnt/etc/fstab
```

Переходим в наше будущее окружение
```
arch-chroot /mnt
```
Даем имя нашему хосту
```
echo "digital_resistance" > /etc/hostname
```
Устанавливаем часовой пояс
```
ln -sf /usr/share/zoneinfo/Europe/Moscow /etc/localtime 
```
Синхронизируем аппаратное время (bios) с системным
```
hwclock --systohc
```
Ставим пароль для root
```
passwd root
```
Настраиваем локали
```
sed -i 's/#en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/g' /etc/locale.gen
sed -i 's/#ru_RU.UTF-8 UTF-8/ru_RU.UTF-8 UTF-8/g' /etc/locale.gen
locale-gen
echo -e "LANG=en_US.UTF-8\nLC_CTYPE=ru_RU.UTF-8\nLC_NUMERIC=ru_RU.UTF-8\nLC_TIME=ru_RU.UTF-8\nLC_COLLATE=ru_RU.UTF-8\nLC_MONETARY=ru_RU.UTF-8\nLC_MESSAGES=en_US.UTF-8\nLC_PAPER=ru_RU.UTF-8\nLC_NAME=ru_RU.UTF-8\nLC_ADDRESS=ru_RU.UTF-8\nLC_TELEPHONE=ru_RU.UTF-8\nLC_MEASUREMENT=ru_RU.UTF-8\nLC_IDENTIFICATION=ru_RU.UTF-8\nLC_ALL=" > /etc/locale.conf
```

Настраиваем кириллицу в консоле
```
echo KEYMAP=ru > /etc/vconsole.conf
echo FONT=cyr-sun16 >> /etc/vconsole.conf
```
Добавляем пользователя от которого будем работать
```
useradd -m -g users -G wheel,games,power,optical,storage,scanner,lp,audio,video -s /bin/bash head01
passwd head01
```
Настраиваем `sudo`
```
visudo
```
Раскомментировать (убрать `#` перед строкой)
```
%wheel ALL=(ALL) ALL
```
Настраиваем хуки
```
vim /etc/mkinitcpio.conf
```
Добавить хуки `encrypt` и `lvm` (последовательность имеет значение)
```
HOOKS=(base udev autodetect modconf block encrypt lvm2 filesystems keyboard fsck)
```
Генерируем ramfs образ с нашими хуками
```
mkinitcpio -p linux
```
Выходим из окружения
```
exit
```
Копируем ядро и образ initramfs (`Файловая система в памяти для начальной инициализации`)
на раздел `esp`
```
mkdir ./esp
mount PARTLABEL=esp ./esp
mkdir ./esp/EFI
cp -ai /mnt/boot/* ./esp/EFI && rm /mnt/boot/*
```
Добавить в `/etc/fstab` для того что бы раздел с `esp` у нас монтировался прямо в `/boot`
```
PARTLABEL=esp					/esp		vfat		rw 0 0 
/esp/EFI/						/boot		none		defaults,bind	0 0
```
Создаем запись в nvram нашего UEFI, на основе которой будет происходить загрузка
```
efibootmgr --create --label "Arch_Linux" --loader '\EFI\vmlinuz-linux' -u 'cryptdevice=UUID=48eacdb6-e829-4549-986c-a93cc09cf2a7:cryptroot root=UUID=edcc6bf2-7f44-4b60-8c1d-2a290900fc08 rw add_efi_memmap initrd=\EFI\initramfs-linux.img'
```
Перезагружаемся, проверяем что все работает.
```
reboot
```
Ставим в автозапуск нужные службы и одновременно стартуем их
```
sudo systemctl enable --now dhcpcd.service
sudo systemctl enable --now sshd
```
