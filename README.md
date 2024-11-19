# OTUS PRO Homework 8 ZFS

## Домашняя работа 8: ZFS

### Домашнее задание:
1. Подготовка рабочего места   
2. Определить алгоритм с наилучшим сжатием:  
     - Определить какие алгоритмы сжатия поддерживает zfs (gzip, zle, lzjb, lz4);
     - создать 4 файловых системы на каждой применить свой алгоритм сжатия;
     - для сжатия использовать либо текстовый файл, либо группу файлов.
3. Определить настройки пула.    
     С помощью команды zfs import собрать pool ZFS.  
     Командами zfs определить настройки:
      - размер хранилища;
      - тип pool;
      - значение recordsize;
      - какое сжатие используется;
      - какая контрольная сумма используется.
4. Работа со снапшотами:
     - скопировать файл из удаленной директории;
     - восстановить файл локально. zfs receive;
     - найти зашифрованное сообщение в файле secret_message.
---
## Выполнение задания:
### 1. Подготовка рабочего места:
Выполнение домашнего задания предполагает, что на компьютере установлен Vagrant+VirtualBox   
**[Как установить Vagrant на Debian 12](https://github.com/avlikh/Install_Vagrant_Debian12/blob/main/README.md)**   

Развернем Vagrant-стенд:
  - Создайте папку с проектом и зайдите в нее (например: /opt/otus/ZFS):
```
mkdir -p /opt/otus/ZFS ; cd /opt/otus/ZFS
```
  - Клонируете проект с Github, набрав команду:
```
apt update -y && apt install git -y ; git clone https://github.com/avlikh/Otus_pro_08.git;
```
  - Запустите проект из папки, в которую склонировали проект (в нашем примере /opt/otus/ZFS):
```
vagrant up
```
Результатом выполнения команды vagrant up станет созданная виртуальная машина, с 8 дисками и уже установленным и готовым к работе ZFS.   
  - Зайдите в виртуальную машину (box):
```
vagrant ssh
```
  - Дальнейшие действия выполняются от пользователя root. Переходим в root пользователя:
```
sudo -i
```
---
### 2. Определить алгоритм с наилучшим сжатием

`lsblk`
<details>
<summary> результат выполнения команды: </summary>

```
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   40G  0 disk
└─sda1   8:1    0   40G  0 part /
sdb      8:16   0  512M  0 disk
sdc      8:32   0  512M  0 disk
sdd      8:48   0  512M  0 disk
sde      8:64   0  512M  0 disk
sdf      8:80   0  512M  0 disk
sdg      8:96   0  512M  0 disk
sdh      8:112  0  512M  0 disk
sdi      8:128  0  512M  0 disk
```
</details>

Создаём пул из двух дисков в режиме **RAID 1**:
```
zpool create otus1 mirror /dev/sdb /dev/sdc
```

**Создадим ещё 3 пула:**
```
zpool create otus2 mirror /dev/sdd /dev/sde
```
```
zpool create otus3 mirror /dev/sdf /dev/sdg
```
```
zpool create otus4 mirror /dev/sdh /dev/sdi
```

**Посмотрим информацию о пулах:**
```
zpool list
```
<details>
<summary> результат выполнения команды: </summary>

```
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
otus1   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus2   480M   106K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus3   480M   106K   480M        -         -     0%     0%  1.00x    ONLINE  -
otus4   480M   106K   480M        -         -     0%     0%  1.00x    ONLINE  -
```
</details>

**Добавим разные алгоритмы сжатия в каждую файловую систему:**   
Алгоритм lzjb: 
```
zfs set compression=lzjb otus1
```
Алгоритм lz4:
```
zfs set compression=lz4 otus2
```
Алгоритм gzip:
```
zfs set compression=gzip-9 otus3
```
Алгоритм zle:
```
zfs set compression=zle otus4
```

**Проверим, что все файловые системы имеют разные методы сжатия:**
```
zfs get all | grep compression
```
<details>
<summary> результат выполнения команды: </summary>

```
otus1  compression           lzjb                   local
otus2  compression           lz4                    local
otus3  compression           gzip-9                 local
otus4  compression           zle                    local
```
</details>

**Скачаем один и тот же текстовый файл во все пулы:**
```
for i in {1..4}; do wget -P /otus$i https://gutenberg.org/cache/epub/2600/pg2600.converter.log; done
```
<details>
<summary> результат выполнения команды: </summary>

```
--2024-11-18 16:43:59--  https://gutenberg.org/cache/epub/2600/pg2600.converter.log
Resolving gutenberg.org (gutenberg.org)... 152.19.134.47, 2610:28:3090:3000:0:bad:cafe:47
Connecting to gutenberg.org (gutenberg.org)|152.19.134.47|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 41098441 (39M) [text/plain]
Saving to: ‘/otus1/pg2600.converter.log’

100%[==================================================================================================================================================================================>] 41,098,441  1.23MB/s   in 22s

2024-11-18 16:44:23 (1.76 MB/s) - ‘/otus1/pg2600.converter.log’ saved [41098441/41098441]

--2024-11-18 16:44:23--  https://gutenberg.org/cache/epub/2600/pg2600.converter.log
Resolving gutenberg.org (gutenberg.org)... 152.19.134.47, 2610:28:3090:3000:0:bad:cafe:47
Connecting to gutenberg.org (gutenberg.org)|152.19.134.47|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 41098441 (39M) [text/plain]
Saving to: ‘/otus2/pg2600.converter.log’

100%[==================================================================================================================================================================================>] 41,098,441  2.66MB/s   in 22s

2024-11-18 16:44:45 (1.82 MB/s) - ‘/otus2/pg2600.converter.log’ saved [41098441/41098441]

--2024-11-18 16:44:45--  https://gutenberg.org/cache/epub/2600/pg2600.converter.log
Resolving gutenberg.org (gutenberg.org)... 152.19.134.47, 2610:28:3090:3000:0:bad:cafe:47
Connecting to gutenberg.org (gutenberg.org)|152.19.134.47|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 41098441 (39M) [text/plain]
Saving to: ‘/otus3/pg2600.converter.log’

100%[==================================================================================================================================================================================>] 41,098,441  1.48MB/s   in 32s

2024-11-18 16:45:17 (1.23 MB/s) - ‘/otus3/pg2600.converter.log’ saved [41098441/41098441]

--2024-11-18 16:45:17--  https://gutenberg.org/cache/epub/2600/pg2600.converter.log
Resolving gutenberg.org (gutenberg.org)... 152.19.134.47, 2610:28:3090:3000:0:bad:cafe:47
Connecting to gutenberg.org (gutenberg.org)|152.19.134.47|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 41098441 (39M) [text/plain]
Saving to: ‘/otus4/pg2600.converter.log’

100%[==================================================================================================================================================================================>] 41,098,441  2.00MB/s   in 21s

2024-11-18 16:45:39 (1.83 MB/s) - ‘/otus4/pg2600.converter.log’ saved [41098441/41098441]
```
</details>

**Проверим, что файл был скачан во все пулы:**
```
ls -l /otus*
```
<details>
<summary> результат выполнения команды: </summary>

```
/otus1:
total 22090
-rw-r--r--. 1 root root 41098441 Nov  2 07:57 pg2600.converter.log

/otus2:
total 18003
-rw-r--r--. 1 root root 41098441 Nov  2 07:57 pg2600.converter.log

/otus3:
total 10964
-rw-r--r--. 1 root root 41098441 Nov  2 07:57 pg2600.converter.log

/otus4:
total 40164
-rw-r--r--. 1 root root 41098441 Nov  2 07:57 pg2600.converter.log
```
</details>

Уже на этом этапе видно, что самый оптимальный метод сжатия у нас используется в пуле otus3.   
**Проверим, сколько места занимает один и тот же файл в разных пулах и проверим степень сжатия файлов:**
```
zfs list
```
<details>
<summary> результат выполнения команды: </summary>

```
NAME    USED  AVAIL     REFER  MOUNTPOINT
otus1  21.7M   330M     21.6M  /otus1
otus2  17.7M   334M     17.6M  /otus2
otus3  10.8M   341M     10.7M  /otus3
otus4  39.3M   313M     39.2M  /otus4
```
</details>

```
zfs get all | grep compressratio | grep -v ref
```

<details>
<summary> результат выполнения команды: </summary>

```
otus1  compressratio         1.82x                  -
otus2  compressratio         2.23x                  -
otus3  compressratio         3.66x                  -
otus4  compressratio         1.00x                  -
```
</details>

Таким образом, у нас получается, что алгоритм **gzip-9** (otus3) **самый эффективный** по сжатию.
---
### 3. Определить настройки пула.

**Скачиваем архив в домашний каталог:**
```
wget -O archive.tar.gz --no-check-certificate 'https://drive.usercontent.google.com/download?id=1MvrcEp-WgAQe57aDEzxSRalPAwbNN1Bb&export=download'
```

<details>
<summary> результат выполнения команды: </summary>

```
Resolving drive.usercontent.google.com (drive.usercontent.google.com)... 64.233.162.132, 2a00:1450:4010:c05::84
Connecting to drive.usercontent.google.com (drive.usercontent.google.com)|64.233.162.132|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 7275140 (6.9M) [application/octet-stream]
Saving to: ‘archive.tar.gz’

100%[==================================================================================================================================================================================>] 7,275,140   5.24MB/s   in 1.3s

2024-11-18 18:34:50 (5.24 MB/s) - ‘archive.tar.gz’ saved [7275140/7275140]
```
</details>

**Разархивируем его:**
```
tar -xzvf archive.tar.gz
```

<details>
<summary> результат выполнения команды: </summary>

```
zpoolexport/
zpoolexport/filea
zpoolexport/fileb
```
</details>

**Проверим, возможно ли импортировать данный каталог в пул:**
```
zpool import -d zpoolexport/
```

<details>
<summary> результат выполнения команды: </summary>

```
   pool: otus
     id: 6554193320433390805
  state: ONLINE
 action: The pool can be imported using its name or numeric identifier.
 config:

        otus                         ONLINE
          mirror-0                   ONLINE
            /root/zpoolexport/filea  ONLINE
            /root/zpoolexport/fileb  ONLINE
```
</details>

Данный вывод показывает нам имя пула, тип raid и его состав.   
**Сделаем импорт данного пула к нам в ОС:**
```
zpool import -d zpoolexport/ otus
```
**Посмотрим информацию о составе импортированного пула:**
```
zpool status
```
<details>
<summary> результат выполнения команды: </summary>

```
  pool: otus
 state: ONLINE
  scan: none requested
config:

        NAME                         STATE     READ WRITE CKSUM
        otus                         ONLINE       0     0     0
          mirror-0                   ONLINE       0     0     0
            /root/zpoolexport/filea  ONLINE       0     0     0
            /root/zpoolexport/fileb  ONLINE       0     0     0

errors: No known data errors

  pool: otus1
 state: ONLINE
  scan: none requested
config:

        NAME        STATE     READ WRITE CKSUM
        otus1       ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdb     ONLINE       0     0     0
            sdc     ONLINE       0     0     0

errors: No known data errors

  pool: otus2
 state: ONLINE
  scan: none requested
config:

        NAME        STATE     READ WRITE CKSUM
        otus2       ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdd     ONLINE       0     0     0
            sde     ONLINE       0     0     0

errors: No known data errors

  pool: otus3
 state: ONLINE
  scan: none requested
config:

        NAME        STATE     READ WRITE CKSUM
        otus3       ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdf     ONLINE       0     0     0
            sdg     ONLINE       0     0     0

errors: No known data errors

  pool: otus4
 state: ONLINE
  scan: none requested
config:

        NAME        STATE     READ WRITE CKSUM
        otus4       ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdh     ONLINE       0     0     0
            sdi     ONLINE       0     0     0

errors: No known data errors

```
</details>

  - Примечание: Если пул с именем otus уже есть, то можно поменять его имя во время импорта, введя
<details>
<summary> команду: </summary>

```
zpool import -d zpoolexport/ otus newotus
```
</details>

**Посмотрим настройки пула otus:**
```
zpool get all otus
```
<details>
<summary> результат выполнения команды: </summary>

```
NAME  PROPERTY                       VALUE                          SOURCE
otus  size                           480M                           -
otus  capacity                       0%                             -
otus  altroot                        -                              default
otus  health                         ONLINE                         -
otus  guid                           6554193320433390805            -
otus  version                        -                              default
otus  bootfs                         -                              default
otus  delegation                     on                             default
otus  autoreplace                    off                            default
otus  cachefile                      -                              default
otus  failmode                       wait                           default
otus  listsnapshots                  off                            default
otus  autoexpand                     off                            default
otus  dedupditto                     0                              default
otus  dedupratio                     1.00x                          -
otus  free                           478M                           -
otus  allocated                      2.09M                          -
otus  readonly                       off                            -
otus  ashift                         0                              default
otus  comment                        -                              default
otus  expandsize                     -                              -
otus  freeing                        0                              -
otus  fragmentation                  0%                             -
otus  leaked                         0                              -
otus  multihost                      off                            default
otus  checkpoint                     -                              -
otus  load_guid                      12539858150978043233           -
otus  autotrim                       off                            default
otus  feature@async_destroy          enabled                        local
otus  feature@empty_bpobj            active                         local
otus  feature@lz4_compress           active                         local
otus  feature@multi_vdev_crash_dump  enabled                        local
otus  feature@spacemap_histogram     active                         local
otus  feature@enabled_txg            active                         local
otus  feature@hole_birth             active                         local
otus  feature@extensible_dataset     active                         local
otus  feature@embedded_data          active                         local
otus  feature@bookmarks              enabled                        local
otus  feature@filesystem_limits      enabled                        local
otus  feature@large_blocks           enabled                        local
otus  feature@large_dnode            enabled                        local
otus  feature@sha512                 enabled                        local
otus  feature@skein                  enabled                        local
otus  feature@edonr                  enabled                        local
otus  feature@userobj_accounting     active                         local
otus  feature@encryption             enabled                        local
otus  feature@project_quota          active                         local
otus  feature@device_removal         enabled                        local
otus  feature@obsolete_counts        enabled                        local
otus  feature@zpool_checkpoint       enabled                        local
otus  feature@spacemap_v2            active                         local
otus  feature@allocation_classes     enabled                        local
otus  feature@resilver_defer         enabled                        local
otus  feature@bookmark_v2            enabled                        local
```
</details>

**Посмотрим все параметры файловой системы**
```
zfs get all otus
```
<details>
<summary> результат выполнения команды: </summary>

```
NAME  PROPERTY              VALUE                  SOURCE
otus  type                  filesystem             -
otus  creation              Fri May 15  4:00 2020  -
otus  used                  2.04M                  -
otus  available             350M                   -
otus  referenced            24K                    -
otus  compressratio         1.00x                  -
otus  mounted               yes                    -
otus  quota                 none                   default
otus  reservation           none                   default
otus  recordsize            128K                   local
otus  mountpoint            /otus                  default
otus  sharenfs              off                    default
otus  checksum              sha256                 local
otus  compression           zle                    local
otus  atime                 on                     default
otus  devices               on                     default
otus  exec                  on                     default
otus  setuid                on                     default
otus  readonly              off                    default
otus  zoned                 off                    default
otus  snapdir               hidden                 default
otus  aclinherit            restricted             default
otus  createtxg             1                      -
otus  canmount              on                     default
otus  xattr                 on                     default
otus  copies                1                      default
otus  version               5                      -
otus  utf8only              off                    -
otus  normalization         none                   -
otus  casesensitivity       sensitive              -
otus  vscan                 off                    default
otus  nbmand                off                    default
otus  sharesmb              off                    default
otus  refquota              none                   default
otus  refreservation        none                   default
otus  guid                  14592242904030363272   -
otus  primarycache          all                    default
otus  secondarycache        all                    default
otus  usedbysnapshots       0B                     -
otus  usedbydataset         24K                    -
otus  usedbychildren        2.01M                  -
otus  usedbyrefreservation  0B                     -
otus  logbias               latency                default
otus  objsetid              54                     -
otus  dedup                 off                    default
otus  mlslabel              none                   default
otus  sync                  standard               default
otus  dnodesize             legacy                 default
otus  refcompressratio      1.00x                  -
otus  written               24K                    -
otus  logicalused           1020K                  -
otus  logicalreferenced     12K                    -
otus  volmode               default                default
otus  filesystem_limit      none                   default
otus  snapshot_limit        none                   default
otus  filesystem_count      none                   default
otus  snapshot_count        none                   default
otus  snapdev               hidden                 default
otus  acltype               off                    default
otus  context               none                   default
otus  fscontext             none                   default
otus  defcontext            none                   default
otus  rootcontext           none                   default
otus  relatime              off                    default
otus  redundant_metadata    all                    default
otus  overlay               off                    default
otus  encryption            off                    default
otus  keylocation           none                   default
otus  keyformat             none                   default
otus  pbkdf2iters           0                      default
otus  special_small_blocks  0                      default
```
</details>

**Можно сделать выборку по параметрам:**

<details>
<summary> zfs get available otus </summary>

```
NAME  PROPERTY   VALUE  SOURCE
otus  available  350M   -
```
</details>

<details>
<summary> zfs get readonly otus </summary>

```
NAME  PROPERTY  VALUE   SOURCE
otus  readonly  off     default
```
</details>

<details>
<summary> zfs get recordsize otus </summary>

```
NAME  PROPERTY    VALUE    SOURCE
otus  recordsize  128K     local
```
</details>

<details>
<summary> zfs get compression otus </summary>

```
NAME  PROPERTY     VALUE     SOURCE
otus  compression  zle       local
```
</details>

<details>
<summary> zfs get checksum otus </summary>

```
NAME  PROPERTY  VALUE      SOURCE
otus  checksum  sha256     local
```
</details>

---
### 4. Работа со снапшотами.

**Скачаем файл с удаленного сервера:**
```
wget -O otus_task2.file --no-check-certificate https://drive.usercontent.google.com/download?id=1wgxjih8YZ-cqLqaZVa0lA3h3Y029c3oI&export=download
```

<details>
<summary> результат выполнения команды: </summary>

```
Resolving drive.usercontent.google.com (drive.usercontent.google.com)... 64.233.165.132, 2a00:1450:4010:c08::84
Connecting to drive.usercontent.google.com (drive.usercontent.google.com)|64.233.165.132|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 5432736 (5.2M) [application/octet-stream]
Saving to: ‘otus_task2.file’

100%[==================================================================================================================================================================================>] 5,432,736   5.00MB/s   in 1.0s

2024-11-18 19:11:32 (5.00 MB/s) - ‘otus_task2.file’ saved [5432736/5432736]


[1]+  Done                    wget -O otus_task2.file --no-check-certificate https://drive.usercontent.google.com/download?id=1wgxjih8YZ-cqLqaZVa0lA3h3Y029c3oI
```
</details>

**Восстановим файл из снапшота:**
```
zfs receive otus/test@today < otus_task2.file
```

**Найдем файл с зашифрованным сообщением:**
```
find /otus/test -name "secret_message"
```

<details>
<summary> результат выполнения команды: </summary>

```
/otus/test/task1/file_mess/secret_message
```
</details>

**Посмотрим содержимое найденного файла:**
```
cat /otus/test/task1/file_mess/secret_message
```

<details>
<summary> результат выполнения команды: </summary>

```
https://otus.ru/lessons/linux-hl/
```
</details>
