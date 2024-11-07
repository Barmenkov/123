# HQ-SRV
```
TYPE=eth
DISABLED=no
NM_CONTROLLED=no
BOOTPROTO=static
CONFIG_IPv4=yes" > /etc/net/ifaces/ens192/options
```

```
192.168.0.2/26 > /etc/net/ifaces/ens192/ipv4address
```

```
default via 192.168.0.1 > /etc/net/ifaces/ens192/ipv4route
```

```
systemctl restart network
```
Устанавливаем bind:

```
apt-get install bind bind-utils
```

Редактируем конфиг:

```
nano  /var/lib/bind/etc/options.conf
```
![image](https://github.com/user-attachments/assets/8e78f3be-959d-41a1-9d66-83f2e1b393bb)
![image](https://github.com/user-attachments/assets/8648656b-78eb-4012-9bf3-e89db10bf09c)
Проверяем на ошибки

```
named-checkconf
```
Если ошибок нет, то запускаем `bind`

```
systemctl enable --now bind
systemctl status bind
```


Редактируем `resolv.conf`

```
nano /etc/net/ifaces/ens192/resolv.conf 
```

```
search au-team.irpo
nameserver 127.0.0.1
nameserver 192.168.0.2
nameserver 77.88.8.8
```

Перезагружаем сеть

```
systemctl restart network
dig ya.ru
```
### Создаем зону прямого просмотра

```
nano  /var/lib/bind/etc/local.conf

zone "au-team.irpo" {
        type master;
        file "au-team.irpo.db";
};
```
оздаем копию файла-шаблона прямой зоны `/var/lib/bind/etc/zone/localdomain`

```
# cp /var/lib/bind/etc/zone/localdomain  /var/lib/bind/etc/zone/au-team.irpo.db
```

Задаем права на файл
```
chown named. /var/lib/bind/etc/zone/au-team.irpo.db

chmod 600 /var/lib/bind/etc/zone/au-team.irpo.db
```

Открываем для редактирования

```
nano /var/lib/bind/etc/zone/au-team.irpo.db
```

```
$TTL    1D
@       IN      SOA     au-team.irpo. root.au-team.irpo. (
                                2024102200      ; serial
                                12H             ; refresh
                                1H              ; retry
                                1W              ; expire
                                1H              ; ncache
                        )
        IN      NS      au-team.irpo.
        IN      A       192.168.0.2
hq-rtr  IN      A       192.168.0.1
br-rtr  IN      A       192.168.1.1
hq-srv  IN      A       192.168.0.2
hq-cli  IN      A       192.168.0.66
br-srv  IN      A       192.168.1.2
moodle  IN      CNAME   hq-rtr
wiki    IN      CNAME   hq-rtr
```

Проверяем, что зона настроена Предварительно

```
named-checkconf -z
```
Перезагружаем `bind`

```
systemctl restart bind
```

Проверяем

```
dig hq-srv.au-team.irpo
```
### Создаем зону обратного просмотра и PTR записи

```
nano  /var/lib/bind/etc/local.conf
```

```
zone "0.168.192.in-addr.arpa" {
        type master;
        file "au-team.irpo_rev.db";
};
```

Копируем шаблон файла

```
cp /var/lib/bind/etc/zone/{127.in-addr.arpa,au-team.irpo_rev.db}
```

Задаем права на файл
```
chown named. /var/lib/bind/etc/zone/au-team.irpo_rev.db

chmod 600 /var/lib/bind/etc/zone/au-team.irpo_rev.db
```

Открываем для редактирования

```
nano /var/lib/bind/etc/zone/au-team.irpo_rev.db
```

Вставляем в него следующее содержимое.

```
$TTL    1D
@       IN      SOA     au-team.irpo. root.au-team.irpo. (
                                2024102200      ; serial
                                12H             ; refresh
                                1H              ; retry
                                1W              ; expire
                                1H              ; ncache
                        )
        IN      NS      au-team.irpo.
1       IN      PTR     hq-rtr.au-team.irpo.
2       IN      PTR     hq-srv.au-team.irpo.
66      IN      PTR     hq-cli.au-team.irpo.
```

Проверяем

```
named-checkconf -z
```
Перезагружаем `bind`

```
systemctl restart bind
```

Проверяем

```
dig -x 192.168.0.2
```
### Комплекская проверка с HQ-CLI
![image](https://github.com/user-attachments/assets/4fab3d75-567a-406d-a60b-de85ac40ff3b)
Делаем на всякий случай бэкап конфига:

```
cd /etc/openssh
cp -a sshd_config sshd_config.bak
```

Редактируем файл конфигурации SSH сервера

```
nano sshd_config
```

Изменяем следующие параметры. Не забываем их раскоментировать. Если какой-то параметр не находиться, то просто добавьте его сами

```
Port 2024
MaxAuthTries 3
Banner /etc/openssh/banner
AllowUsers sshuser
```

Чтобы зайти через PUTTY нужно ввести команды на обоих машинах Linux

```
systemctl start serial-getty@ttyS0.service
systemctl enable serial-getty@ttyS0.service
systemctl status serial-getty@ttyS0.service
```

Создаем файл с баннером

```
nano /etc/openssh/banner
```

Вставляем в него следующие содержимое

```
******************************************************
*                     WARNING!                       *
*                                                    *
*           Access only authorized users!            *
*                                                    *
*  If you not authorized user, close this session!   *
*                                                    *
******************************************************
```

## HQ-SRV и BR-SRV

```
useradd -m -u 1010 sshuser
passwd sshuser
```

```
nano /etc/sudoers.d/sshuser
```

```
sshuser ALL=(ALL) NOPASSWD:ALL
```

```
su - sshuser
sudo whoami
```

Перезагружаем `SSH`

```
systemctl restart sshd
```

## Проверка

```
На машине BR-SRV вводим: ssh sshuser@hq-srv.au-team.irpo -p 2024
```
Проверяем какой часовой пояс установлен

```
timedatectl status
```
Если отличается, то устанавливаем

```
timedatectl set-timezone Asia/Yekaterinburg
```
