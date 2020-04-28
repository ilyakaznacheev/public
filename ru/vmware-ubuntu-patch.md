Сегодня столкнулся со следующей проблемой: при установке VMware Workstation 8 на Linux Mint 13 (ядро 3.2, как и в ubuntu 12.04 LTS или других новых дистрибутивах) и последующем запуске возникает требование пропатчить ядро системы. Как я узнал из интернета, ситуация достаточно распространенная, однако, ни одной подробной инструкции "от начала до конца" я не нашел. Проблема была мной решена, в связи с чем я решил написать небольшой гайд. Итак, последовательность действий такова:

1.  Скачать и установить VMware Workstation 8 (я использовал Workstation 8.0.4, но с другими тоже должно работать)
<habracut>*   Скачать патч ядра. Я видел как для ядра 3.2, так и для 3.4\. Я скачал вместе с установочником VMware, но можно и иначе сделать: (см *)*   Установить VMware <source lang="bash"> $ sudo sh VMware-Workstation-Full-8.0.4-744019.i386.bundle*   Запустить программу. Программа попросит установить дополнения. Далее возможны два варианта развития событий: а) при установке первого компонента выдаст сообщение, что необходимо пропатчить ядро. Тогда пункт 5 пропускаем б) после установки дополнений выдаст ошибку о том, что один из компонентов не установлен. Тогда выполняем пункт 5*   Необходимо открыть в текстовом редакторе файл patch-modules_3.2.0.sh (или аналогичный**) и заменить строки

    > [ "$vmver" == "workstation$vmreqver" ] && product="VMWare WorkStation" [ "$vmver" == "player$plreqver" ] && product="VMWare Player"

    на одну строку

    > product="VMWare WorkStation"

    *   Проверяем, установлен ли пакет "patch" <source lang="bash"> $ sudo apt-get install patch*   Установить патч <source lang="bash"> $ sudo -s # cd адрес_папки_с_патчем # sh patch-modules_3.2.0.sh*   В некоторых случаях при установке патча выпадает ошибка вида:

    > /home/the23/Загрузки/VMware Workstation 8.0.4 build 744019 for Linux/patch_for_kernel_3.2.0/patch-modules_3.2.0.sh: 27: [: workstation8.0.4: unexpected operator /home/the23/Загрузки/VMware Workstation 8.0.4 build 744019 for Linux/patch_for_kernel_3.2.0/patch-modules_3.2.0.sh: 28: [: workstation8.0.4: unexpected operator Sorry, this script is only for VMWare WorkStation 8.0.4 or VMWare Player 4.0.4\. Exiting

    чтобы этого не происходило, тоже нужно перейти к пункту 5 и выполнить все действия по-новой*   После того как патч установится, запускаем программу</habracut>

* можно скачать и установить патч следующим образом <source lang="bash"> $ sudo apt-get install patch $ cd $ wget http://webupd8.googlecode.com/files/vmware802fixlinux320.tar.gz $ tar -xvf vmware802fixlinux320.tar.gz $ sudo ~/vmware802fixlinux320/patch-modules_3.2.0.sh **если система на ядре Linux 3.4, тогда указываете _patch-modules_3.4.0.sh_ соответственно P.S. Я в линуксе пока новичок, так что предполагаю наличие ошибок и неточностей в тексте. Буду рад поправить и дополнить статью. Надеюсь, информация окажется полезной