Настройка MPLS
Дана топология:

![p1](https://sun9-40.userapi.com/c855120/v855120376/1e463d/7xlrYsn1_Pg.jpg)

Рис. 1.1. - Топология сети
MPLS это метод маркировки пакетов, который устанавливает приоритетность данных. Большинство соединений сети должны анализировать каждый пакет данных на каждом маршрутизаторе, чтобы точно понимать его маршрут следования.
Технология MPLS должна «подниматься» на уже существующей и нормально функционирующей IP сети, так как она использует таблицу маршрутизации (FIB - Forwarding Information Base). В данной работе выбранный протокол маршрутизации OSPF. На каждом роутере начиная с MPLS-1 и заканчивая MPLS-2 зададим ip-адреса на интерфейсах и включим их:

```
MPLS-1>en
MPLS-1#conf t         
MPLS-1(config)# int e0/0                            #e0/0 - тот интерфейс, с которым Вы работаете
MPLS-1(config-if)# ip address 12.34.57.2 255.255.255.252   #Задаём ip-адрес и маску сети
MPLS-1(config-if)# no sh                                                                    # Включаем интерфейс
MPLS-1(config-if)# int Loopback0                                      #Создадим Loopback-интерфейс
MPLS-1(config-if)# ip address 10.10.10.1 255.255.255.255 
```

Loopback0 - это логический интерфейс, он не привязан к физической среде, следовательно он будет поднят до тех пор, пока само устройство находится в рабочем состоянии или до команды "shutdown" на интерфейсе. Адресс интерфейса может быть абсолютно любой.
Туннели - TE, GRE/DMVPN, IPSec - если используют адреса лупбэков для пиринга, то остаются в подняты, пока соответствующие адреса анонсируются. В случае же физических интерфейсов, они бы упали вслед за интерфейсом, адрес которого используется.

```
MPLS-1(config-if)# do wr mem                            #ОБЯЗАТЕЛЬНО сохраняйте настройки!
```

И так на каждом интерфейсе каждого роутера. И везде по одному  Loopback-интерфейсу.
На каждом маршрутизаторе необходимо создать процесс OSPF командой router ospf номер-процесса, не имеет значения сам номер. Затем описываем все сети, входящие в процесс маршрутизации с помощью команды network. Когда мы указываем некоторую сеть, это приводит к двум последствиям:
а. Информация об этой сети начинает передаваться другим маршрутизаторам (при условии, что на маршрутизаторе есть рабочий интерфейс в данной сети)
б. Через интерфейс, находящийся в этой сети маршрутизатор начинает общаться с соседями.
Необходимо указывать на каждом маршрутизаторе те сети, которые непосредственно подключенные к нему. Исключением является MPLS-6 – на нём не надо указывать сеть 12.34.59.0, так как, с последующим роутером нет необходимости устанавливать соседские отношения роутерам во всей зоне MPLS и туда без этого пойдет маршрут по умолчанию, поэтому именно про сеть 12.34.59.0 никому из внутренних маршрутизаторов знать не нужно. Настроим маршрутизацию:

```
MPLS-1(config)# router ospf 1   #
MPLS-1(config)# network 0.0.0.0 255.255.255.255 area 0
MPLS-1(config)# do wr mem
```

При добавлении сетей используется  wildcard(обратная) маска. Обратная маска получается инверсией к маске подсети. Например, если маска подсети 255.255.252.0, то обратная маска будет 0.0.3.255. В нашем случае, чтобы упростить работу мы разрешили всё и адрессацию на loopback-интерфейсах.
Проделаем это со всеми роутерами в области MPLS!
Настройка непосредственно самого MPLS.

```
MPLS-1(config)#ip cef            # Включаем Cisco Express Forwarding, технология быстрой коммутации пакетов на 3-ем уровне (если не включено)
MPLS-1(config)#mpls ip                           #включаем коммутацию по меткам (MPLS)
MPLS-1(config)#mpls label protocol ldp # Протокол, по которому будут обмениваться метками между собой роутеры
MPLS-1(config)#mpls ldp router-id l0  #определяем, какой интерфейс (IP-адрес) берется в качестве ID роутера в процессе MPLS, l0 - сокращение от “Loopback0”
MPLS-1(config)#int e0/0
MPLS-1(config-if)#mpls ip – включаем MPLS на интерфейсе;
MPLS-1(config-if)#mpls mtu 1500 #задаём размер mtu
MPLS-1(config-if)#int e0/1
MPLS-1(config-if)#mpls ip
MPLS-1(config-if)#mpls mtu 1500
MPLS-1(config-if)#do wr mem
```

Проделываем это на каждом роутере и на каждом рабочем интерфейсе.
Настройка MPLS выполнена, теперь Вы можете просмотреть таблицу командой:

```
MPLS-1(config)#do sh mpls forwarding-table
MPLS-1(config)#do sh mpls ldp neighbor                    #таблица с соседями
```

Проверим работу на MPLS-1:

Как и положено, Вы видите все сети, которые известны роутеру и интерфейсы через которые в эти самые сети можно попасть. Также у роутера, как и на схеме, 2 соседа.
