# L2TP

Все команды далее стоит выполнять из под **root** пользователя или использовать ```sudo```.

Server - Ubuntu 18.04

Client - Ubuntu 18.04

[Шаг 1 - Установка L2TP](#шаг-1---установка-l2tp)

[Шаг 2 - Настройка L2TP](#шаг-2---настройка-l2tp)

[Шаг 3 - Настройка форвардинга](#шаг-3---настройка-форвардинга)

[Шаг 4 - Создание NAT-правил для iptables](#шаг-4---создание-nat-правил-для-iptables)

[Шаг 5 - Настройка клиентов](#шаг-5---настройка-клиентов)

## Шаг 1 - Установка L2TP

В качестве клиента и cервера мы будем использовать xl2tpd. Установим необходимые пакеты:
    
    apt install xl2tpd
    
## Шаг 2 - Настройка L2TP
    
Теперь необходимо отредактировать файл /etc/xl2tpd/xl2tpd.conf. Прежде чем внести какие-либо изменения, создадим его резервную копию: 

    mv /etc/xl2tpd/xl2tpd.conf{,.original}
    
Далее создадим новый конфигурационный файл:

    nano /etc/xl2tpd/xl2tpd.conf
    
Добавим следующие строки:

    [global]
    auth file = /etc/ppp/chap-secrets
    port = 1701
    access control = no
    
    [lns default]
    ip range = 10.102.10.100-254
    local ip = 10.102.10.1
    require chap = yes
    refuse pap = yes
    require authentication = no
    name = L2TPServer
    ppp debug = yes
    pppoptfile = /etc/ppp/options.xl2tpd
    
Создадим файл ```/etc/ppp/options.xl2tpd```, в котором можно указать необходимые опции ppp:

    ipcp-accept-local
    ipcp-accept-remote
    ms-dns 8.8.8.8
    mtu 1410
    mru 1410
    require-mschap-v2
    auth
    proxyarp
    connect-delay 5000
    debug

В файле ```/etc/ppp/chap-secrets``` указываются аутентификационные данные пользователей для CHAP аутентификации:

    # client server secret   IP addresses
    username L2TPServer password *
    
## Шаг 3 - Настройка форвардинга

Очень важно включить форвардинг IP на L2TP-сервере. Это позволит пересылать пакеты между публичным IP и приватными IP, которые мы настроили при помощи L2TP. Отредактируем /etc/sysctl.conf:
    
    net.ipv4.ip_forward = 1
    
Для применения изменений выполните команду:

    sysctl -p
    
# Шаг 4 - Создание NAT-правил для iptables

Перед тем, как изменить этот файл, мы должны найти публичный интерфейс сети. Для этого наберите команду:

    ip route | grep default
    
Публичный интерфейс должен следовать за словом **dev**. Например, в нашем случае этот интерфейс называется ```eth0```.

Далее перейдем к настройкам iptables.

    iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
    
Далее нужно сохранить проделанные нами изменения. Воспользуемся утилитой ```iptables-persistent```.

    apt install iptables-persistent
    
>Во время установки в открывшихся диалогах нужно выбрать два раза **Yes**.

Сохраним настройки iptables

    netfilter-persistent save
    netfilter-persistent reload
    
## Шаг 5 - Настройка клиентов

> Примечание. По умолчание графическое окно линукс интерфейса маленького размера. Далее при добавлении VPN мы столкнемся с тем, что не все настройки будут помещаться на экране. Для увеличения разрешения остановим ноду. В ее конфигурации(правой кнопкой -> Edit) в пункте QEMU custom options меняем -vga std на -vga qxl. Нажимаем Save.

Для начала установим необходимые пакеты:
    
    apt update
    apt install network-manager-l2tp
    apt install network-manager-l2tp-gnome
    
Далее заходим в Settings -> Network. 

![Настройка](https://sun1-86.userapi.com/t5VBHad3DOLRkw3K8OAfB6hzdb6gIfb2aMzs6w/dqIqedhAibE.jpg )

В разделе VPN нажимаем на "+". 

![Добавление VPN](https://sun1-30.userapi.com/95EqMIPesHVw5dMFv_VGvVbIIaCgI0IRJoRw4A/QAxCCH3JF9A.jpg )

В появившемся окне выбираем **Layer 2 Tunneling Protocol (L2TP).**

![Выбор L2TP](https://sun1-98.userapi.com/2pkLLfjPaC1FBXuClphXLB63u3QMvnxe006pIw/cTSGJGxTkOY.jpg )

В поле **Name** даем любое название VPN(к примеру "L2TP").

В поле **Gateway** указываем IP-адрес VPN сервера.

В поле **User name** и **Password** вводим логин и пароль для пользователя, который мы указывали на сервере в файле ```/etc/ppp/chap-secrets```. (Для ввода пароля нужно нажать на знак вопроса в поле Password и выбрать "Store the password for all users". В противном случае система будет запрашивать пароль при каждом VPN соединении).

![Ввод данных](https://sun1-20.userapi.com/SN2M2defeFCoQhzhANicLwwmbQr_5OSLKTpIyA/c_LaOyqWaBE.jpg )

Далее выбираем пункт **PPP Settings..**

Ставим галочку на пункте **Use Point-to-point encryption(MPPE).** И в поле **Security** выбираем **128-bit most secure.**

В разделе **Misc** устанавливаем значения **MTU** и **MRU** в 1410.

![PPP settings](https://sun1-22.userapi.com/ZfQxQMuUPweRYkpVB9WAngno9g03Re613OaFKA/ZhDcFjF7Kzc.jpg )

После чего нажимаем **Okey** и **Add**. Настройка клиента завершена. 

Кликаем на ползунок и при успешном подключении в верхней строке состояния должен появиться значок с замком.
