# Настройка установки UBUNTU 20.04 по сети

## Условия:

1. Хост-сервер (Уже установлена Ubuntu 20.04, впрочем дистрибутив и версия мало на что влияют)
2. Mikrotik в качестве DHCP-сервера (не думаю, что от модели к модели имеются значительные изменения в плане настроек)
3. Хост-клиент (куда планируем устанавливать систему)

## Подготовка:

1. Скачиваем образ Ubuntu 20.04.6 с [официального сайта](http://releases.ubuntu.com/20.04.6/)
2. Будем использовать образ системы Syslinux, скачанный с [www.kernel.org](https://www.kernel.org/pub/linux/utils/boot/syslinux/). Из него нам понядобятся только некоторые файлы:
    - chain.c32
    - ldlinux.c32
    - libcom32.c32
    - libutil.c32
    - memdisk
    - pxechn.c32
    - pxelinux.0
    - vesamenu.c32

## Поднимаем TFTP-сервер. 
### Устанавливаем:
- atftpd
- tftpd-hpa

Для установки некоторых пакетов нам понадобится пакет aptitude
```
sudo apt install -y aptitude
```

Далее, с его помощью устанавливаем необходимые пакеты:

```
sudo aptitude -R install atftpd tftpd-hpa
```
(в некоторых случаях дополнительно устанавливают apache2, но в нашем случае он не нужен)

### Настраиваем TFTP-сервер:
В файле настроек `/etc/default/atftpd`:

```
USE_INETD=true
```
меняем на:
```
USE_INETD=false
```
(нам эта опция не нужна)

Так же запоминаем каталог, прописанный в конце строки `OPTIONS`, вероятнее всего это будет `/srv/tftp`


В файле настроек `/etc/default/tftpd-hpa` прописываем папку, запомненную в предыдущем шаге:
```
TFTP_DIRECTORY="/srv/tftp"
```

Теперь просто запускаем сервер:
```
sudo /etc/init.d/atftpd start
```


Создаем папку `ubuntu` в нашей папке TFTP-сервера
```
mkdir /srv/tftp/ubuntu
```

И смонтируем в нее нас скачанный образ Ubuntu:
```
sudo mount -o loop /home/myusername/ubuntu-20.04.6-desktop-amd64.iso /srv/tftp/ubuntu/
```

### Настраиваем загрузочное меню

Закидываем файлы образа Syslinux (те, что указал в начале) в папку `/srv/tftp`

Создаем в ней еще одну папку `pxelinux.cfg`:
```
mkdir /srv/tftp/pxelinux.cfg
```

И содаем в ней файл `default`, в нем как раз и будет сконфигурировано наше загрузочное меню:
```
nano /srv/tftp/pxelinux.cfg/default
```
И прописываем в нем следующее:
```
DEFAULT vesamenu.c32
PROMPT 0
timeout 80
TOTALTIMEOUT 9000

MENU TITLE XANDER UBUNTU INSTALL
MENU INCLUDE pxelinux.cfg/graphics.conf
MENU AUTOBOOT Starting Local System in 8 seconds

LABEL bootlocal
  MENU label Boot Local
  MENU default
  localboot 0x80

LABEL UBUNTU live-install
  MENU label ^Install Ubuntu
  KERNEL ubuntu/casper/vmlinuz
  INITRD ubuntu/casper/initrd
  SYSAPPEND 3
  APPEND root=/dev/nfs boot=casper persistent  netboot=nfs nfsroot=IP_АДРЕС_НАШЕГО_СЕРВЕРА:/srv/tftp/ubuntu/ file=/ubuntu/preseed/ubuntu.seed only-ubiquity quiet splash fsck.mode=skip ---

LABEL UBUNTU
  MENU label ^Live UBUNTU
  KERNEL ubuntu/casper/vmlinuz
  INITRD ubuntu/casper/initrd
  SYSAPPEND 3
  APPEND root=/dev/nfs boot=casper fsck.mode=skip netboot=nfs nfsroot=IP_АДРЕС_НАШЕГО_СЕРВЕРА:/srv/tftp/ubuntu/ ---
```

Немного объяснений:
- `MENU TITLE` - Заголовок нашего меню
- `MENU INCLUDE` - файл настроек графики (о нем немного ниже)
- `LABEL bootlocal` - указание на загрузку с локального диска
- `LABEL UBUNTU live-install` - указание на запуск установки системы
    обратите внимание: 
    - указание путей к файлам KERNEL, INITRD и в параметре `file` идет относительно нашей папки /srv/tftp
    - параметр `fsck.mode=skip` отвечает за то, чтобы не производилась проверка установочного диска (я ее отключаю для ускорения процесса, всё таки по сети это занимает довольно много времени)
- `LABEL UBUNTU` - указание на запуск LIVE-CD Ubuntu с нашего образа

- Подмонтироваться на клиент наша папка будет по NFS, инструкции ниже.

Содержимое `pxelinux.cfg/graphics.conf`:
```
MENU MARGIN 10
MENU ROWS 16
MENU TABMSGROW 21
MENU TIMEOUTROW 26
MENU COLOR BORDER 30;44 #00000000 #00000000 none
MENU COLOR SCROLLBAR 30;44 #00000000 #00000000 none
MENU COLOR TITLE 0 #ffffffff #00000000 none
MENU COLOR SEL 30;47 #40000000 #20ffffff
MENU BACKGROUND background.png
NOESCAPE 0
ALLOWOPTIONS 0
```
Здесь можно поиграть с настройками текста, фона и установить своё фоновое изображение.

### Настраиваем сервер NFS

Устанавливаем `nfs-kernel-server`:
```
sudo apt install -y nfs-kernel-server
```

Прописываем в `/etc/exports` нашу папку:
```
/srv/tftp/ubuntu/ *(ro,sync,no_wdelay,insecure_locks,no_root_squash,insecure)
```
и перезапускаем nfs:
```
sudo systemctl restart nfs-kernel-server
```

**С настройкой сервера закончили**

### Настройка Mikrotik

Берем за факт, что наш микротик выолняет роль DHCP-сервера и от этого работаем:

1. Создаем Опции:

    1.1. **option66** - Код 66 и IP-адрес нашего сервера (обратите внимание на написание: в поле следует прописать `s'123.123.123.123'`, в начале s и IP-адрес взять в кавычки)
    
    1.2. **option67** - Код 67 и путь до файла-образа (относительно папки /srv/tftp на сервере, синтаксис аналогичный)

2. Создаем Option Set (для  удобства), и добавляем в него поочередно option66 и option67

3. Настраиваем Сети (Networks):

    3.1. `Next Server` - IP-адрес нашего сервера

    3.2. `DHCP Option Set` - выбираем созданныq Option Set


## Настраиваем компьютер-клиент на загрузку по сети
В данном случае всё зависит от конкретной модели материаской платы и установленного на нее БИОСа.

Важно включить возможность загружаться по сети, и настроить на загрузку в режиме Legacy.

Вот и всё, можно перезапускть клиент, выбирать загрузку по сети и пробовать загрузиться.