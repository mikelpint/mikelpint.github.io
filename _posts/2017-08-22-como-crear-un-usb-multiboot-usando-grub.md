---
layout: post
title: Como crear un USB multiboot usando GRUB
img: gnulinux.png
date: 2017-08-22 13:35:00 +200
tags: Tutorials
---

En este tutorial aprenderás a crear un USB con el que poder iniciar varias distribuciones GNU/Linux. Sirve para sistemas de 64 y de 32 bits y para sistemas con UEFI o BIOS. También podremos guardar datos en el USB.

Primero, debo agradecerle el contenido de esta entrada a [la wiki de Arch Linux](https://wiki.archlinux.org) y a la gente de [LiGNUx](https://lignux.com).

**Preparación:** Debes disponer de una memoria USB y un sistema GNU/Linux con el paquete `grub2` instalado. El sistema puede ser live o estar instalado. También debes disponer acceso como superusuario ya que todo en esta guía va a ser hecho como el usuario *root*.

**1.** Conecta el pendrive y lista los dispositivos de almacenamiento disponibles mediante el comando `lsblk`.
La salida será algo así:

~~~
NAME                                          MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
sda                                             8:0    0 931,5G  0 disk
├─sda1                                          8:1    0   450M  0 part
├─sda2                                          8:2    0   100M  0 part
├─sda3                                          8:3    0    16M  0 part
├─sda4                                          8:4    0 882,1G  0 part
└─sda5                                          8:5    0  48,8G  0 part
sdb                                             8:16   0   2,7T  0 disk
├─sdb1                                          8:17   0 976,6G  0 part  /run/media/mikel/Datos
├─sdb2                                          8:18   0 195,3G  0 part  /
├─sdb3                                          8:19   0 976,6G  0 part  /run/media/mikel/Juegos
├─sdb4                                          8:20   0     1G  0 part  /boot/efi
├─sdb5                                          8:21   0  19,5G  0 part  [SWAP]
└─sdb6                                          8:22   0 527,9G  0 part
  └─luks-652893b0-a193-43a1-88d0-94883ef555ba 254:0    0 527,9G  0 crypt /run/media/mikel/VMs
sdc                                             8:32   1  29,1G  0 disk  /mnt/MULTIBOOT
sr0                                            11:0    1  1024M  0 rom
~~~

Puedes identificar tu dispositivo por el tamaño y por que tiene un 1 en la columna de extraíbles (RM). En mi caso es /dev/sdc.
Si tu sistema lo ha montado automáticamente desmóntalo.

Después, introduce el comando `fdisk dispositivo>`. Escribe *o* para crear una tabla de particiones de tipo msdos y *w* para guardar.
A continuación, formatea el pendrive como FAT32  con `mkfs.vfat -F 32 <dispositivo>`.

**2.** Monta el USB en un directorio llamado */mnt/MULTIBOOT* con `mount <dispositivo> /mnt/MULTIBOOT`.
Luego, instala GRUB en el USB.

- <h4>Para sistemas (UEFI)<h4>

~~~
grub-install--target=x86_64-efi--efi-directory=/mnt/MULTIBOOT --boot-directory=/mnt/MULTIBOOT/boot --removable --recheck
~~~

- <h4>Para sistemas BIOS<h4>

~~~
grub-install --target=i386-pc --boot-directory=/mnt/MULTIBOOT/boot --recheck <dispodistivo>
~~~

**3.** Dirígete a la carpeta */mnt/MULTIBOOT/boot* y dentro de ella crea una carpeta llamada iso. En esta carpeta guardarás las imágenes ISO de las distribuciones que quieras iniciar.

**4.** Ahora tienes que preparar el menú de GRUB2.
Primero, usa `lsblk -f <dispositivo>` y copia el contenido de la columna UUID.
Después, copia lo siguiente en el archivo `/mnt/MULTIBOOT/grub.cfg` sustituyendo `<UUID>` por lo que copiaste anteriormente.:

~~~
set imgdevpath='/dev/disk/by-uuid/<UUID>'
menu_color_highlight=light-green/black
menu_color_normal=white/black
if [ "${grub_platform}" == "efi" ]; then
    insmod efi_gop
    insmod efi_uga
else
    insmod vbe
    insmod vga
fi
insmod gfxterm
terminal_output gfxterm
loadfont /boot/grub/fonts/unicode.pf2
~~~

## Personalización ##

Puedes reemplazar los colores de la segunda y la tercera línea con los de [esta lista](https://www.gnu.org/software/grub/manual/legacy/color.html).

Si quieres ponerle un fondo al GRUB añade las siguentes líneas al archivo `/mnt/MULTIBOOT/boot/grub/grub.cfg` y quita el carácter # del tipo de tu imagen:

~~~
#insmod png
#insmod jpeg
background_image -m stretch /boot/grub/<imagen>.<extensión>
~~~

Si quieres cambiar la fuente sigue estos pasos:

- **1.** Escoge una fuente que te guste (en mi caso Ubuntu).
- **2.** Hazla compatible con GRUB con este comando:

~~~
grub-mkfont -s <tamaño> -o <archivo de destino>.pf2 <archivo de origen>.ttf`
~~~

- **3.** Copia la fuente al pendrive:

~~~
cp <archivo> /mnt/MULTIBOOT/boot/grub/fonts
~~~

- **4.** Edita el archivo `/mnt/MULTIBOOT/boot/grub/grub.cfg` y sustituye la línea que dice `loadfont /boot/grub/fonts/unicode.pf2>` por la siguiente:

`loadfont /boot/grub/fonts/<fuente>.pf2`

## Entradas ##

Añade las entradas de las distros. Aquí tienes unos cuantos ejemplos:

**NOTA:** No importa el escritorio que tenga la distro.

**NOTA:** Las entradas no han sido probadas en máquinas de 32 bits.

- **Arch Linux**

~~~
menuentry '<nombre de la entrada>' {
    set isofile='/boot/iso/<archivo>.iso'
    loopback loop $isofile
    linux (loop)/arch/boot/x86_64/vmlinuz img_dev=$imgdevpath img_loop=$isofile earlymodules=loop
    initrd (loop)/arch/boot/x86_64/archiso.img
}
~~~

- **Antergos**

~~~
menuentry '<nombre de la entrada>' {
    set isofile='/boot/iso/<archivo>.iso'
    loopback loop $isofile
    linux (loop)/arch/boot/vmlinuz img_dev=$imgdevpath img_loop=$isofile earlymodules=loop
    initrd (loop)/arch/boot/archiso.img
    initrd	(loop)/arch/boot/intel_ucode.img /initramfs-linux.img
}
~~~

- **Manjaro**

    - **32 bits**

    ~~~
    menuentry '<nombre de la entrada>' {
        set isofile='/boot/iso/<archivo>.iso'
        set dri="nonfree"
        search --no-floppy -f --set=root $isofile
        loopback loop $isofile
        linux  (loop)/boot/vmlinuz-i686  img_dev=$imgdevpath img_loop=$isofile driver=$dri
        initrd  (loop)/boot/intel_ucode.img (loop)/boot/initramfs-i686.img
    }
    ~~~

    - **64 bits**
    ~~~
    menuentry '<nombre de la entrada>' {
        set isofile='/boot/iso/<archivo>.iso'
        set dri="nonfree"
        search --no-floppy -f --set=root $isofile
        loopback loop $isofile
        linux  (loop)/boot/vmlinuz-x86_64  img_dev=$imgdevpath img_loop=$isofile driver=$dri
        initrd  (loop)/boot/intel_ucode.img (loop)/boot/initramfs-x86_64.img
    }
    ~~~

- **Debian Live**

~~~
menuentry '<nombre de la entrada>' {
    set isofile='/boot/iso/<archivo>.iso'
    loopback loop $isofile
    linux (loop)/live/vmlinuz boot=live config findiso=$isofile
    initrd (loop)/live/initrd.img
}
~~~

- **Instalación de Debian**

    - **32 bits**

    ~~~
    menuentry '<nombre de la entrada>' {
        set isofile='/boot/iso/<archivo>.iso'
        loopback loop $isofile
        linux (loop)/install/vmlinuz config findiso=$isofile
        initrd (loop)/install/initrd.gz
    }
    ~~~


    - **64 bits**

    ~~~
    menuentry '<nombre de la entrada>' {
        set isofile='/boot/iso/<archivo>.iso'
        loopback loop $isofile
        linux (loop)/install.amd/vmlinuz config findiso=$isofile
        initrd (loop)/install.amd/initrd.gz
    }
    ~~~

- **Devuan Live**

~~~
menuentry '<nombre de la entrada>' {
    set isofile='/boot/iso/<archivo>.iso'
    loopback loop $isofile
    linux (loop)/live/vmlinuz boot=live config findiso=$isofile
    initrd (loop)/live/initrd.img
}
~~~

- **Instalación de Devuan**

    - **32 bits**

    ~~~
    menuentry '<nombre de la entrada>' {
        set isofile='/boot/iso/<archivo>.iso'
        loopback loop $isofile
        linux (loop)/install/vmlinuz config findiso=$isofile
        initrd (loop)/install/initrd.img
    }
    ~~~

    - **64 bits**
    ~~~
    menuentry '<nombre de la entrada>' {
        set isofile='/boot/iso/<archivo>.iso'
        loopback loop $isofile
        linux (loop)/install.amd/vmlinuz config findiso=$isofile
        initrd (loop)/install.amd/initrd.img
    }
    ~~~

- **Ubuntu**

~~~
menuentry '<nombre de la entrada>' {
    set isofile='/boot/iso/<archivo>.iso'
    loopback loop $isofile
    linux (loop)/casper/vmlinuz.efi boot=casper iso-scan/filename=$isofile
    initrd (loop)/casper/initrd.lz
}
~~~

- **Kali Linux**

~~~
menuentry '<nombre de la entrada>' {
    set isofile='/boot/iso/<archivo>.iso'
    loopback loop $isofile
    linux (loop)/live/vmlinuz boot=live findiso=$isofile noconfig=sudo hostname=kali
    initrd (loop)/live/initrd.img
}
~~~

- **Fedora Workstation**

~~~
menuentry '<nombre de la entrada>' {
    set isofile='/boot/iso/<archivo>.iso'
    loopback loop $isofile
    linux (loop)/isolinux/vmlinuz root=live:CDLABEL=<CDLABEL (monta la iso para verlo)> iso-scan/filename=$isofile rd.live.image quiet
    initrd (loop)/isolinux/initrd.img
}
~~~

- **Linux Mint**

~~~
menuentry '<nombre de la entrada>' {
    set isofile='/boot/iso/<archivo>.iso'
    loopback loop $isofile
    linux (loop)/casper/vmlinuz boot=casper iso-scan/filename=$isofile noeject noprompt
    initrd (loop)/casper/initrd.lz
}
~~~

- **Deepin**

~~~
menuentry ‘<nombre de la entrada>’ {
set isofile=’/boot/iso/<archivo>.iso’
loopback loop $isofile
linux (loop)/live/vmlinuz.efi boot=live config findiso=$isofile priority=low
initrd (loop)/live/initrd.lz
}
~~~

- **elementaryOS**

~~~
menuentry ‘<nombre de la entrada>’ {
set isofile=’/boot/iso/<archivo>.iso’
loopback loop $isofile
linux (loop)/casper/vmlinuz boot=casper iso-scan/filename=$isofile
initrd (loop)/casper/initrd.lz
}
~~~

- **GParted**

~~~
menuentry '<nombre de la entrada>' {
    set isofile='/boot/iso/<archivo>.iso'
    loopback loop $isofile
    linux (loop)/live/vmlinuz boot=live union=overlay username=user config components quiet noswap noeject toram=filesystem.squashfs nosplash findiso=$isofile iso-scan/filename=$
    initrd (loop)/live/initrd.img
}
~~~

- **Tails**

~~~
menuentry ‘<nombre de la entrada>’ {
set isofile=’/boot/iso/<archivo>.iso’
loopback loop $isofile
linux (loop)/live/vmlinuz boot=live config findiso=${isofile} live-media=removable apparmor=1 security=apparmor nopersistent noprompt timezone=Etc/UTC block.events_dfl_poll_msecs=1000 noautologin module=Tails
initrd (loop)/live/initrd2.img
}
~~~

- **SystemRescueCD**

~~~
menuentry “<nombre de la entrada>″ {
set isofile=”/boot/iso/<archivo>.iso”
loopback loop $isofile
linux (loop)/isolinux/rescue64 setkmap=es isoloop=$isofile
initrd (loop)/isolinux/initram.igz
}
~~~

- **Clonezilla Live**

~~~
menuentry “<nombre de la entrada>″ {
set isofile=”/boot/iso/<archivo>.iso”
loopback loop $isofile
linux (loop)/live/vmlinuz boot=live live-config noswap findiso=$isofile union=overlay username=user config acpi_osi= noapic nolapic —
initrd (loop)/live/initrd.img
}
~~~

- **Wifislax**

~~~
menuentry "<nombre de la entrada> ($sl_lang)" {
    set isofile='/boot/iso/<archivo>.iso'
    loopback loop $isofile
    linux (loop)/boot/vmlinuz kbd=$sl_kbd tz=$sl_tz locale=$sl_locale xkb=$sl_xkb rw
    initrd (loop)/boot/initrd.xz
}
~~~

    - **En RAM**

    ~~~
    menuentry "<nombre de la entrada> ($sl_lang)" {
    set isofile='/boot/iso/<archivo>.iso'
    loopback loop $isofile
    linux (loop)/boot/vmlinuz kbd=$sl_kbd tz=$sl_tz locale=$sl_locale xkb=$sl_xkb rw toram
    initrd (loop)/boot/initrd.xz
    }
    ~~~

- **Sabayon**

~~~
menuentry '<nombre de la entrada>' {
    insmod loopback
    set isofile=/boot/iso/<archivo>.iso
    loopback loop ${isofile}
    linux (loop)/boot/sabayon root=/dev/ram0 init=/linuxrc isoboot=$isofile cdroot looptype=squashfs loop=/livecd.squashfs overlayfs
    initrd (loop)/boot/sabayon.igz
}
~~~

- **Zorin OS**

~~~
menuentry '<nombre de la entrada>' {
    set isofile='/boot/iso/<archivo>.iso'
    loopback loop $isofile
    linux   (loop)/casper/vmlinuz boot=casper iso-scan/filename=$isofile quiet splash
    initrd  (loop)/casper/initrd.lz
}
~~~

- **Void Linux**

~~~
menuentry '<nombre de la entrada>' {
    set isofile='/boot/iso/<archivo>.iso'
    loopback loop $isofile
    linux (loop)/boot/vmlinuz iso-scan/filename=$isofile root=live:CDLABEL=VOID_LIVE
    initrd (loop)/boot/initrd
}
~~~

- **Reiniciar**

~~~~
menuentry '<nombre de la entrada>'  {
    reboot
}
~~~~

- **Apagar**

~~~
menuentry '<nombre de la entrada>'  {
    halt
}
~~~

- **Configurar el firmware UEFI**

~~~
menuentry '<nombre de la entrada>'  {
    fwconfig
}
~~~

<br>

Para terminar, dejo algunas imágenes que uso como fondo para GRUB.

![](https://orig11.deviantart.net/c3ff/f/2010/089/f/7/feel_free_by_cyric80.jpg)
<br>
<br>
![](https://img08.deviantart.net/d282/i/2016/220/7/9/gnu_linux_wallpaper_by_itsfoss-dad4qq4.png)
<br>
<br>
![](http://www.technocrazed.com/wp-content/uploads/2015/12/Linux-Wallpaper-32.png)
