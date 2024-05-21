	a. Присвоить имена в соответствии с топологией
nano /etc/hostname - меняем, сохраняем, перезагружаем
	
	b. Рассчитайте IP-адресацию IPv4 и IPv6. 
	c. Пул адресов для сети офиса BRANCH - не более 16
	d. Пул адресов для сети офиса HQ - не более 64
для расчета формула 2^(32-n), где n - кол-во используемых бит. (или используйте свой более удобный метод)
Нужно не более 64 устройств? проверяем маску /26
11111111.11111111.11111111.11000000. используется 26 бит, подставляем в формулу 2^(32-26)=2^6=64-2 (не забываем  что 2 ип системные широковещ и номер сети)
и так делаем по своему заданию.

	Задаем IP на устройства (ип у каждого свои)
nano /etc/network/interfaces 
auto ensXX
iface ensXX inet static (если порт получает через dhcp то пишем dhcp)
address x.x.x.x/xx
gateway x.x.x.x (шлюз пишем на всех кроме ISP, и на устройствах которые к ним подключены)

	Включаем маршрутизацию на всех роутерах (ISP,BR-R,HQ-R)
nano /etc/sysctl.conf
раскоментируем строку net.ipv4.ip_forward=1 - сохраняем, выходим.

	Раздаем инет по сети через iptables на роутерах (ISP,BR-R,HQ-R)
nano/etc/network/interfaces
пишем ниже строку - post-up iptables -A POSTROUTING -t nat -j MASQUERADE - сохраняем, выходим, проверяем.
	
	Настраиваем DNS (если не качаются пакеты и не пингуются сайты по доменному имени)
nano /etc/resolv.conf
domain localdomain
search localdomain
nameserver 8.8.8.8

	2. Настройте внутреннюю динамическую маршрутизацию по 
	средствам FRR. Выберите и обоснуйте выбор протокола 
	динамической маршрутизации из расчёта, что в 
	дальнейшем сеть будет масштабироваться. (ISP,BR-R,HQ-R)
apt install frr 
nano /etc/frr/daemons - меняем строку ospfd=no на ospfd=yes - выходим, сохраняем.
systemctl restart frr
systemctl enable frr (можно прописать на всякий случай, чтоб запускался при запуске устройства. Можно и с другими службами также)
Далее настраиваем динамическую маршрутизацию трафика через OSPF
--ISP--
vtysh (попадаем в оболочку cisco)
conf t
router-id 1.1.1.1 (или другой вам удобный id) 
router ospf
network 1.1.1.0/30 area 0 (АДРЕС СЕТИ И МАСКА У ВАС МБ БУДЕТ ДРУГОЙ, а area 0 можете оставить)
network 2.2.2.0/30 area 0 (АДРЕС СЕТИ И МАСКА У ВАС МБ БУДЕТ ДРУГОЙ, а area 0 можете оставить)
network 3.3.3.0/30 area 0 (АДРЕС СЕТИ И МАСКА У ВАС МБ БУДЕТ ДРУГОЙ, а area 0 можете оставить)
end
write mem (или do wr m)
--HQ-R--
vtysh
conf t
router-id 2.2.2.2
router ospf
network 2.2.2.0/30 area 0
network 172.16.0.0/26 area 0
end
write mem
--BR-R-
vtysh
conf t
router-id 3.3.3.3
router ospf
network 3.3.3.0/30 area 0
network 192.168.0.0/26 area 0
end 
write mem
проверяем через ip route

	a. Составьте топологию сети L3.

просто сделай логическую схему своей сети

	3. Настройте автоматическое распределение IP-адресов на роутере HQ-R.
	a. Учтите, что у сервера должен быть зарезервирован адрес.

--HQ-R--
apt install isc-dhcp-server
nano /etc/default/isc-dhcp-server - вставляем в INTERFACESv4=ensXX свой порт, который смотрит в сторону HQ-SRV
nano /etc/dhcp/dhcpd.conf - пишем ниже:
subnet 172.16.0.0 netmask 255.255.255.192 {     (подсеть и маска у вас другие будут)
range 172.16.0.2 172.16.0.10;      (range тоже другой будет)
option routers 172.16.0.1; (указываем IP порта смотрящего в HQ-SRV)
}
host HQ-SRV {
hardware ethernet xx:xx:xx:xx:xx:xx;   (xx - это мак адрес)
fixed-address 172.16.0.5;   (ип может быть у вас другой)
}
#выходим
systemctl restart isc-dhcp-service
#проверяем далось ли.

	4. Настройте локальные учётные записи на всех устройствах в соответствии с таблицей 2.
	учетная запись		пароль		примечание
	Admin 			P@ssw0rd	CLI HQ-SRV HQ-R
	Branch admin		P@ssw0rd	BR-SRV BR-R
	Network admin		P@ssw0rd 	HQ-R BR-R BR-SRV
su -l
useradd Admin
passwd Admin (и так с остальными)
usermod -aG sudo Admin (если пользователя нужно добавить в sudo)
nano /etc/passwd (для проверки создания пользователя)

	5. Измерьте пропускную способность сети между двумя 
	узлами HQ-R-ISP по средствам утилиты iperf 3. 
	Предоставьте описание пропускной способности канала со скриншотами.

устанавливаем на HQ-R и ISP
apt install iperf3 
--ISP--
iperf3 -s (если не запускается, пока забейте)
--HQ-R--
iperf3 -c 2.2.2.1   (2.2.2.1 это ип порта ISP который на нас смотрит, у вас может быть другой)
происходит тест, скриним, фиксируем.
так же можем сделать это в обратную сторону
iperf3 -c 2.2.2.1 8 -R

	6. Составьте backup скрипты для сохранения конфигурации 
	сетевых устройств, а именно HQ-R BR-R. 
	Продемонстрируйте их работу.

mkdir /etc/networkbackup (путь можешь свой сделать)
crontab -e
30 15 * * * cp /etc/frr/frr.conf /etc/networkbackup (перевод звездочек - минута | час | день | месяц | день недели |)
(по примеру выше скрипт сохранения будет срабатывать каждый день в 15:30. выбирайте свое время.)

	7. Настройте подключение по SSH для удалённого 
	конфигурирования устройства HQ-SRV по порту 2222. 
	Учтите, что вам необходимо перенаправить трафик на этот 
	порт по средствам контролирования трафика.

--HQ-SRV--
apt install openssh-server
systemctl enable ssh
nano /etc/ssh/sshd_config
Port 2222
PermitRootLogin no
PasswordAuthentication yes
закрываем файл.
systemctl restart ssh
ssh student@192.168.1.2 -p 2222 (пробуем подключаться)

	8. Настройте контроль доступа до HQ-SRV по SSH со всех 
	устройств, кроме CLI.

--HQ-SRV--
nano /etc/ssh/sshd_config
AllowUsers student@192.168.0.1 student@192.168.0.140 student@192.168.0.129 (вводите ip и пользователя всех устройств кроме CLI)
systemctl restart ssh

	
