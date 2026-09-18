JOHDANTO

Tässä työssä tutustutaan verkonhallinnan harjoitteluympäristöön ja sen rakenteeseen. Ympäristö 
koostuu reitittimistä, palvelimista,työasemista sekä verkonhallinta- ja valvontajärjestelmistä. Sen
tarkoituksena on harjoitella verkkojen dokumentointia, reitityksen tutkimista, valvontaa ja 
vianmääritystä käytännössä.

Tehtävä 1.1 - Topologian kartoitus

r1 - Edge router. Välittää liikennettä käyttäjäverkon ja muun verkon vällillä.
r2 - Core router. Välittää liikennettä verkon eri osien välillä.
r3 - Branch router. Yhdistää branchtoimisto muun verkon yhteyteen.
client1 - Käyttäjäverkon asiakaskone.
attacker - Tietoturvatestaukseen ja hyökkäysten simulointiin tarkoitettu kone. 
web1 - web server
db1 - database server
branch-client - branchtoimiston asiakaskone
ansible - Hallintakone, jota käytetään Ansible-automaation ja laitteiden hallintaan
prometheus - monitorointipalvelu, joka kerää ja tallentaa metrics järjestelmistä ja palveluista
grafana - visualisointipalvelu, jolla monitorointitietoja esitetään dashboardeilla.
zabbix - verkon ja palveluiden monitorointijärjestelmä.
cadvisor - konttien suorituskykyä ja resurssien käyttöä seuraava monitorointipalvelu. 

Tehtävä 1.3 - IP osoitteiden dokumentointi

Verkko	              Tarkoitus	              				   Yhdyskäytävä
10.10.10.0/24	      User/client network (laitteet:client1, attacker,r1)  10.10.10.1 r1 	
10.10.20.0/24	      Server network (laitteet:web1, db1,r2)      	   10.10.20.1 r2
10.10.30.0/24	      Branch office network (laitteet:branch-client,r3)    10.10.30.1 r3
10.10.99.0/24         Management network (ansible,prometheus,grafana,      10.10.99.1 r2	      	
                      zabbix,cadvisor,syslog,mgmt-bp,r2)

10.255.12.0/30	      point-to-poin yhteys r1 ja r2 välillä.                -
                      r1:10.255.12.1,r2:10.255.12.2.
 
10.255.23.0/30        point-to-point yhteys r2 ja r3 välillä.               -
                      r2:10.255.23.1,r3:10.255.23.2. 


Laitteiden IP osoitteet:

client1: 10.10.10.101
attacker: 10.10.10.200
web1: 10.10.20.101
db1: 10.10.20.102
branch-client: 10.10.30.101



Tehtävä 1.4 - Reitityksen tutkiminen

YHTEYDET VERKKOIHIN: Yhteyttä ei löytynyt kaikkiin testattuihin verkkoihin. Yhteys Web1 verkkoon 10.10.20.101 
epäonnistui, kaikki 4 pakettia menetettiin. Myös yhteys branch-client 10.10.30.101 osoitteeseen 
epäonnistui, kaikki 4 pakettia menetettiin.

LIIKENTEEN REITTI: Liikenne ei kulkenut odotetusti kurssin reitittimien kautta. Liikene kulki 172.20.20.1, 
172.22.80.1, 172.20.10.1 osoitteiden kautta. 

REITITTIMET REITILLÄ: Traceroute ei näyttänyt reitittimiä r1, r2 tai r3. Likenne kulki Dockerin ja WSL 
hallintaverkon kautta, koska client1 laitteella näkyi vain hallintaverkon IP-osoite 172.20.20.8/24, 
eikä 10.10.10.101.

KOMENTOJEN TULOSTEET:

~/Verkonhallinta$ docker exec -it clab-hamk-verkonhallinta-golden-client1 bash
root@client1:/# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
24: eth0@if25: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:14:14:08 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.20.20.8/24 brd 172.20.20.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe14:1408/64 scope link
       valid_lft forever preferred_lft forever
root@client1:/# ip route
default via 172.20.20.1 dev eth0
172.20.20.0/24 dev eth0 proto kernel scope link src 172.20.20.8
root@client1:/# ping -c 4 10.10.20.101
PING 10.10.20.101 (10.10.20.101) 56(84) bytes of data.


--- 10.10.20.101 ping statistics ---
4 packets transmitted, 0 received, 100% packet loss, time 3063ms

root@client1:/#
root@client1:/# ping -c 4 10.10.30.101
PING 10.10.30.101 (10.10.30.101) 56(84) bytes of data.

--- 10.10.30.101 ping statistics ---
4 packets transmitted, 0 received, 100% packet loss, time 3058ms

root@client1:/# traceroute 10.10.30.101
traceroute to 10.10.30.101 (10.10.30.101), 30 hops max, 60 byte packets
 1  172.20.20.1 (172.20.20.1)  1.452 ms  1.178 ms  1.107 ms
 2  LAPTOP-EQ4N32UL.mshome.net (172.22.80.1)  1.081 ms  1.012 ms  0.950 ms
 3  172.20.10.1 (172.20.10.1)  6.318 ms  6.277 ms  6.231 ms
 4  * * *
 5  * * *
 6  * * *
 7  * * *
 8  * * *
 9  * * *
10  * * *
11  * * *
12  * * *
13  * * *
14  * * *
15  * * *
16  * * *
17  * * *
18  * * *
19  * * *
20  * * *
21  * * *
22  * * *
23  * * *
24  * * *
25  * * *
26  * * *
27  * * *
28  * * *
29  * * *
30  * * * 

YHTEENVETO

Eniten aikaa kului verkkokaavion tekemiseen, koska Ip-osoitteiden, reitittimien, verkkojen ja 
laitteiden yhteyksien selvittäminen vaati useiden eri tiedostojen ja komentojen tutkimista. 
Hyvä dokumentaatio auttaa IT-asiantuntijaa ymmärtämään verkon rakenteen nopeasti. Sen avulla laitteiden, 
IP-osoitteiden ja yhteyksien löytäminen on helpompaa ja se nopeuttaa ongelmien selvittämistä.
