# OTUS PRO Homework 8 ZFS

## Домашняя работа 8: ZFS

### Домашнее задание:
   - Определить алгоритм с наилучшим сжатием:
     - Определить какие алгоритмы сжатия поддерживает zfs (gzip, zle, lzjb, lz4);
     - создать 4 файловых системы на каждой применить свой алгоритм сжатия;
     - для сжатия использовать либо текстовый файл, либо группу файлов.
   - Определить настройки пула.  
     С помощью команды zfs import собрать pool ZFS.  
     Командами zfs определить настройки:
      - размер хранилища;
      - тип pool;
      - значение recordsize;
      - какое сжатие используется;
      - какая контрольная сумма используется.
   - Работа со снапшотами:
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

Таким образом, у нас получается, что алгоритм **gzip-9 самый эффективный** по сжатию.
