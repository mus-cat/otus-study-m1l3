# Домашнее задание по LVM
### Важно: Для выполнения использовался образ на базе debian 10
- Оцениваем объем места занимаемого ФС на устройстве **/dev/VolGroup00/LogVol00** ("корень" ОС)
```
lsblk
df -h
```
!["Исходно занимаемое место ФС и LVM"](https://github.com/mus-cat/otus-study-m1l3/blob/main/01.VIewInitBlockDevLayout(lsblk).png)

- Выполняем некторые действия (например осуществив загрузку с Live-CD или действуя по аналогии с методичой, но с поправкой на ext4) и загружаем систему с другим "корнем" \
!["Загрузка под другим корнем"](https://github.com/mus-cat/otus-study-m1l3/blob/main/02.UnderOtheRoot.png)

- Это позволит уменьшить исходную корневую файловую систему расположенную на устройстве **/dev/VolGroup00/LogVol00** до ``7GB``, а затем и размер самого LV. В данном случае это можно сделать, т.к. используемая  в корне ФС EXT4 поддерживает возможность уменьшения в размере. В случае ФС которые не умеют уменьшатся, например XFS, необходимо пересоздать ФС предварительно сохранив ее содержимое в другом месте и уменьшив размер LV:
```
resize2fs /dev/VolGroup00/LogVol00 7G
lvresize -L 8G VolGroup00/LogVol00
```
!["Уменьшаем место занимаемое ФС и LV"](https://github.com/mus-cat/otus-study-m1l3/blob/main/03.resizeFSandLV.png)

- На диске sdb переименовываем имеющийся LV в более логичное имя для монтрования его в /home. Также уменьшим размер LV, чтобы в VG осталось место для создания snapshot-тома. Перенесим на данный LV домашние папки пользователей.
```
lvrename 4test01 root01 home
lvreduce 4test01/home -L -1G
mkfs.ext4 /dev/mapper/4test01-home
mount /dev/mapper/4test01-home /mnt
mount /dev/VolGroup00/LogVol00 /opt
cp -ar /opt/home/* /mnt/
umount /mnt /opt
```
  !["Подготавливаем LV для /home и переносим на него имеющиеся данные пользователей"](https://github.com/mus-cat/otus-study-m1l3/blob/main/04.CreateVolForHome.png)
  
- Создадим новую VG в которую входят два диска, а на ней создадим LV с функцией заркала. Скопируем туда содержимое директории **/var**
```
vgcreate 4var /dev/sdd /dev/sde
lvcreate 4var -l 100%FREE -m 1 -n var
mkfs.ext4 /dev/mapper/4var-var
```
!["СОздаем LV для /var с функцией mirror"](https://github.com/mus-cat/otus-study-m1l3/blob/main/05.CreateVolForVar.png)

- Вносим изменения в файл **fstab** на "оригинальном" корневом томе (в моём случае файл **/opt/etc/fstab**). Приписываем монтирование ``/dev/mapper/4test01-home -> /home`` и ``/dev/mapper/4var-var -> /var``. 
```
mount /dev/mapper/VolGroup00-LogVol00 /opt
vi /opt/etc/fstab
```
!["Изменение в файле fstab"](https://github.com/mus-cat/otus-study-m1l3/blob/main/06.ModifyFstab.png)

- Перезагружаем сиситему для монтровани "оригинального" корня. Проверяме какие устройства куда смонтированы.
```
mount
ls /home
ls /var
```
!["После перезагрузки"](https://github.com/mus-cat/otus-study-m1l3/blob/main/07.reboot.png)

- Создаем несколько файлов в папке **/home** и вычисляем их md5 суммы сохранив их в файл **fSum.md5** (дальше с их помощью проверим, что файлы из 
snapshot-тома восстановились корректно)
```
cd /home
for i in 1 2 3; do dd if=/dev/urandom of="$i.img" bs=1M count=100; done
md5sum *.img > fSum.md5
```
!["Создаем тестовые файлы"](https://github.com/mus-cat/otus-study-m1l3/blob/main/08.makeFilesInHome.png)

- Создаём snapshot текущего состояния тома (**/dev/mapper/4test01-home**) смонтированного в **/home**.
```
lvcreate -s -n home-snap -L 1G /dev/mapper/4test01-home
lvs
lsblk
```
!["Создаем snapshot"](https://github.com/mus-cat/otus-study-m1l3/blob/main/09.makeSnap.png)

- Затем изменяем файлы созданные ранее
```
rm 1.img
echo "Additional text" >> 2.img
truncate -s 10M 3.img
dd if=/dev/urandom count=90 bs=10M >> 3.img
md5sum -c fSum.md5
```
!["Изменяем файлы"](https://github.com/mus-cat/otus-study-m1l3/blob/main/10.corruptFileInHome.png)

- Монтируем snapshot и проверяем, что в нем файлы видны неизмененными
```
mount /dev/4test01/home-snap /mnt
cd /mnt
md5sum -c fSum.md5
```
!["Проверяем как данные видны при монтировании snapshot"](https://github.com/mus-cat/otus-study-m1l3/blob/main/11.mountSnapshot.png)

- Восстанавливаем том из snapshot и проверяем, что все хорошо. Т.к. исходный LV был смонтирован и использовался, то процесс сразу не запустился, он был отложен до момент, когда LV будут повторно активирован и не смонтрованны. Для запуска процесса слияния, необходимо было отмонтировать устройства **/dev/4test01/home-snap** и **/dev/4test01/home**, а затем последовательно деактивировать (``lvchange -an ...``) и активировать (``lvchange -ay ...``) одно из них. В заключении смонтировали устройство **/dev/4test01/home** обратно в **/home**. Для проверки использовали сохраненные md5 суммы файлов. В результате слияния snapshota с исходным LV, первый исчезает, что видно по выводу команды **lsblk**.
```
umount /mnt
lvconvert --merge /dev/4test01/home-snap
umount /home
lvchange -an /dev/4test01/home
lvchange -ay /dev/4test01/home
mount /home
cd /home
md5sum -c fSum.md5
lsblk
```
!["Слияние и проверка"](https://github.com/mus-cat/otus-study-m1l3/blob/main/12.mergeSnapshot.png)

