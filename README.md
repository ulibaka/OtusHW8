## 1. Попасть в систему без пароля несколькими способами. ##

1. Во время загрузки vm в окне grub при выбере ядра нажимаем: e и можем изменить параметры загрузки.
   В строке начинающей с linux16 удаляем все с конца до console=tty0, меняем ro на rw и в конце дописываем init=/bin/sh. Нажимаем Ctrl+x
2. Аналочно первому варианту, удаляем часть параметров и дописываем rd.break, далее Ctrl-x
   Попадаем в систему в emergency mode. Далее:
 ```
# mount -o remount,rw /sysroot
# chroot /sysroot
# passwd root - назначем новый пароль
# touch /.autorelabel
```
   После перезагрузки указываем пользователя root и новый пароль. Готово.

## 2. Установить систему с LVM, после чего переименовать VG
```
[root@lvm ~]# vgs
  VG         #PV #LV #SN Attr   VSize   VFree  
  VolGroup00   1   3   0 wz--n- <38.97g <27.47g
```
#### Переименовываем volumegroup
```
[root@lvm ~]# vgrename  VolGroup00 OtusHW8
  Volume group "VolGroup00" successfully renamed to "OtusHW8"
```
#### Меняем на новое название в конфигах

```
[root@lvm ~]# sed -i 's/VolGroup00/OtusHW8/g' /etc/fstab 
[root@lvm ~]# sed -i 's/VolGroup00/OtusHW8/g' /etc/default/grub
[root@lvm ~]# sed -i 's/VolGroup00/OtusHW8/g' /boot/grub2/grub.cfg
```

#### Пересоздаем initrd image, чтобы он знал новое название Volume Group

```
[root@lvm ~]# mkinitrd -f -v /boot/initramfs-$(uname -r).img $(uname -r)
...
*** Creating image file ***
*** Creating image file done ***
*** Creating initramfs image file '/boot/initramfs-3.10.0-1160.76.1.el7.x86_64.img' done ***
```

#### reboot
```
[root@lvm ~]# vgs
  VG      #PV #LV #SN Attr   VSize   VFree
  OtusHW8   1   2   0 wz--n- <38.97g    0 
```

## 3. Добавить модуль в initrd

#### создаем директорию с модулем
```
[root@lvm ~]# mkdir /usr/lib/dracut/modules.d/01test
```
#### размещаем в ней два скрипта
```
1. [root@lvm ~]# cat << _OEF_ > /usr/lib/dracut/modules.d/01test/module_setup.sh
> #!/bin/bash
> 
> check() {
>     return 0
> }
> 
> depends() {
>     return 0
> }
> 
> install() {
>     inst_hook cleanup 00 "${moddir}/test.sh"
> }
> _OEF_

[root@lvm ~]# cat > /usr/lib/dracut/modules.d/01test/test.sh 
#!/bin/bash

exec 0<>/dev/console 1<>/dev/console 2<>/dev/console
cat <<'msgend'
Hello! You are in dracut module!
 ___________________
< I'm dracut module >
 -------------------
   \
    \
        .--.
       |o_o |
       |:_/ |
      //   \ \
     (|     | )
    /'\_   _/`\
    \___)=(___/
msgend
sleep 10
echo " continuing...."


```
#### Пересобираем образ initrd
```
mkinitrd -f -v /boot/initramfs-$(uname -r).img $(uname -r)
```

#### Проверяем какие модули загружены
```
[root@lvm 01test]# lsinitrd -m /boot/initramfs-$(uname -r).img | grep test
test
```

#### удаляем в строке linux16 параметры rghb и quiet из /boot/grub2/grub.cfg и перезагружаемся. Через GUI видим устанволенного пингвина. 


