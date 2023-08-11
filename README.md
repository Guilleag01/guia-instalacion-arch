
# Guia de instalación de arch, guia principal

Esta guia se propone mostrar el proceso de instalación básico de Arch Linux en un **SISTEMA UEFI** según el [manual oficial de Arch](https://wiki.archlinux.org/title/Installation_guide).

Esta guia está hecha a 04/08/2023.
## Índice
- [Guia de instalación de arch, guia principal](#guia-de-instalación-de-arch-guia-principal)
  - [Índice](#índice)
  - [Descarga del medio de instalación](#descarga-del-medio-de-instalación)
  - [Establecer la distribución del teclado](#establecer-la-distribución-del-teclado)
  - [Configurar wifi](#configurar-wifi)
  - [Partición de los discos](#partición-de-los-discos)
  - [Formateo de las particiones](#formateo-de-las-particiones)
  - [Montaje de las particiones](#montaje-de-las-particiones)
  - [Instalación del sistema base](#instalación-del-sistema-base)
  - [Fstab](#fstab)


## Descarga del medio de instalación

Se puede descargar el ISO de archi linux desde [la página de descargas de arch](https://archlinux.org/download/).

## Establecer la distribución del teclado

Establecemos la distribución del teclado en español.

```shell
loadkeys es
```

## Configurar wifi

Si se usa una conexión por cable se puede saltar este paso.

Para conectarse por wifi vamos a usar `iwctl`.

Listamos los dispositivos, escaneamos las redes disponibles, listamos las redes encontradas y mos conectamos a la red.

```shell
iwctl device list
iwctl station <device> scan
iwctl station <device> get-networks
iwctl station <device> connect <SSID>
```

Comprobamos la conexión.

```shell
ping gnu.org
```

Si todo ha funcionado correctamente ahora deberíamos tener conexión a internet.

## Partición de los discos

Se supone que se quiere borrar todo lo que se tiene en el disco e instalar únicamente Arch linux.

Entramos en la herramienta de particionado cfdisk

```shell
cfdisk
```

Seleccionamos el tipo de etiqueta `gpt` (Este paso podría no ser necesario).

Si hay particiones existentes se borran con el botón `[ Delete ]`.

Cuando solo se tenga espacio libre se selecciona `[ New ]` y se pulsa enter para crear **la particion del sistema**. Cuando nos pregunte el tamaño escribir `250M`.

Ahora vamos a hacer la partición de swap, esta particion no es necesaria si tienes mucha ram pero es recomendable en cualquier caso. Se hace de igual manera que antes, esta vez la hacemos de `4G`, si tienes mucha ram puede ser recomendable hacer una partición de swap mas grande. Con 32GB de memoria he usado 4G de swap.

Para la última partición no escribimos tamaño, de esta forma se crea una partición que ocupa el resto del disco.

Para guardar los cambios seleccioanamos `[ Write ]` y escribimos `yes` para confirmar.

Con `[ Quit ]` salimos de la interfaz.

Comprobamos las particiones.

```shell
lsblk
```

## Formateo de las particiones

Formateamos la principal.

- Si quieres hacerlo en ext4

```shell
mkfs.ext4 -L <etiqueta> /dev/<nombre de la partición>
```

- Si quieres hacerlo en btrfs.

```shell
mkfs.btrfs -L <etiqueta> /dev/<nombre de la partición>
```

Formateamos la partición del sistema.

```shell
mkfs.fat -F 32 /dev/<nombre de la partición>
```

Formateamos swap si lo hemos creado anteriormente.

```shell
mkswap /dev/<partición de swap>
```

Comprobamos que los formatos sean correctos.

```shell
lsblk
```

## Montaje de las particiones

Montamos la partición principal.

```shell
mount /dev/<nombre de la partición> /mnt
```

Creamos el directorio boot y efi y montamos la partición del sistema.

```shell
mkdir -p /mnt/boot/efi
mount /dev/<nombre de la partición> /mnt/boot/efi
```

Montamos la partición de swap si la hemos creado antes.

```shell
swapon /dev/<nombre de la partición>
```

## Instalación del sistema base

Ahora vamos a instalar los paquetes esenciales para esta instalación.(`base`, `linux`, `linux-firmware`, `sof-firmware` y `efibootmgr`). También se recomienda instalar adicionalmente:
- Un editor de texto, en este caso se va a usar `neovim`.
- Herramientas de desarrollador con `base-devel`.
- Un bootloader en este caso se va a usar `grub`.
- Administrador de red con `networkmanager`.
- Cualquier otro paquete que se quiera usar.

Los administradores de ventanas vendrán mas adelante así que no se incluyen aquí.

```shell
pacstrap /mnt base linux linux-firmware sof-firmware efibootmgr \
neovim base-devel grub networkmanager
```

## Fstab

Generamos el archivo `fstab` en `/mnt/etc/fstab` y comprobamos que el contenido de `fstab` es correcto.
```shell
genfstab /mnt > /mnt/etc/fstab
cat /mnt/etc/fstab
```

## Chroot

Entramos en el sistema que estamos instalando.
```shell
arch-chroot /mnt
```

## Configuracion
### Timezone
Se establece el uso horario (en este caso de España) y comprobamos que sea correcta.
```shell
ln -sf /usr/share/zoneinfo/Europe/Madrid /etc/localtime
date
```
Ajustamos la hora del reloj hardware.
```shell
hwclock --systohc
```

### Localización
Abrimos `/etc/locale.gen` con un editor de texto.
```shell
neovim /etc/locale.gen
```
En este archivo se encuentran comentados todas las distintas localizaciones, hay que descomentar la que se desee, en este caso español de España `es_ES.UTF-8 UTF-8`.

Después se genera el archivo de localización,
```shell
locale-gen
```
También es necesario editar el archivo `/etc/locale.conf`, se añade la siguiente línea.
```shell
LANG=es_ES.UTF-8
```

### Keymap

Para configurar el keymap editando el archivo `/etc/vconsole.conf`, se añade la siguiente línea.
```shell
KEYMAP=es
```

### Hostname
Para configurar el keymap editando el archivo `/etc/hostname`, se añade el hostname que se desee.

## Root y usuario

Establecemos la contraseña de root
```shell
passwd
```
Añadimos un nuevo usuario y establecemos su contraseña.
  * `-m` para crear un directorio en `/home`.
  * `-G wheel` para añadir al usuario al grupo wheel (sudo).
  * `-s /bin/bash` para establecer su shell como `/bin/bash`.
```shell
useradd -m -G wheel -s /bin/bash <nombre de usuario>
passwd <nombre de usuario>
```

Configuramos el grupo wheel para que tenga capacidades de sudo. Para ello hay que editar el archivo sudoers con el siguiente comando
```shell
EDITOR=<el editor de elección> visudo
```
y descomentamos la línea:
```shell
# %wheel ALL=(ALL) ALL
%wheel ALL=(ALL) ALL
```

## Habilitar servicios necesarios (daemons)
Solo vamos a habilitar Network Manager.
```shell
systemctl enable NetworkManager
```

## Configurar el bootloader.
Ya que hemos instalado grub es necesario configurarlo antes de reiniciar el equipo
```shell
grub-install /dev/<nombre de la partición>
grub-mkconfig -o /boot/grub/grub.cfg
```

## Finalizar y reiniciar
Salimos de chroot
```shell
exit
```
Desmontamos todos los discos que no se estén usando.
```shell
umount -a
```
Reiniciamos el sistema.
```shell
reboot
```