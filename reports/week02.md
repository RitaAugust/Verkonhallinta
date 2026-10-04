1.Johdanto

SNMP (Simple Network Management Protocol) on protokolla, jota käyetään verkkolaitteiden ja palvelimien valvontaan
sekä niiden tietojen keräämiseen. 
SNMP avulla voidaan lukea verkkolaitteiden ja palvelimien tilatietoja, kuten: CPU-kuormaa, muistin käyttö 
rajapintojen liikennemäärät, virhelaskurit, laitteiden lämpötilat. Tietoja voidaan käyttää verkon toiminnan 
seuraamiseen ja ongelmien selvittämiseen. 

SNMP toiminnassa on kaksi keskeistä osaa: SNMP-agentti ja SNMP-manager. Agentti toimii valvottavassa laitteessa ja
tarjoaa tietoja kyselyitä varten. Manager lähettää kyselyitä agentille ja kerää vastaukset valvontaa varten.  

2.Asennus 

Miten SNMP-agentti asennettiin?

SNMP-agentti asennettiin web1, db1 ja branch-client palvelimille. Asennuksen esimerkinä käytän web1. 
Siirryttiin palvelimen konttiin komennolla docker exec -it clab-hamk-verkonhallinta-golden-web1 bash. 
Olessaan root@web1 päivitettiin pakettitiedot ja 
asennettiin SNMP komennoilla: apt update ja apt install snmp snmpd -y.
snmp sisältää SNMP-kyselyiden suorittamiseen tarvittavia komentorivityökaluja. snmpd on SNMP-agenttipalvelu,
joka tarjoaa laitteen tietoja kyselyitä varten.  

Mitä konfiguraatiomuutoksia tehtiin? 

Konfiguraatiomuutoksia tehttiin tiedostossa snmpd.conf (agentin konfiguraatiorakenne), pääsy sinne: 
cd etc/snmp/. 
Ennen kun avattiin konfiguraatio komennolla nano snmpd.conf, piti asenttaa nano komennolla apt install nano -y.

rocommunity public oli olemassa, ei ollut tarvetta tehdä muutoksia. Konfiguraatioon lisättiin OID-haaran .1.3.6.1.2 
system + hrSystem groups only alle: 
view   systemonly  included   .1.3.6.1.2 
Konfiguraation jälkeen SNMP-palvelu käynnistettiin uudelleen komennolla: service snmpd restart.
Tarkistettu onko SNMPD päällä komennolla: service snmpd status. Komento ps aux näyttää prosessilistauksen. 
Komento ps aux |grep snmpd näyttää, että palvelu on käynnistynyt. 

3.Kerätyt tiedot 

Asennetaan Ansible-palvelimelle SNMP, jotta olisi mahdollista tehdä kyselyitä SNMP-agentilta.
Otetaan Ansible-palvelun käyttöön komennolla: docker exec -it clab-hamk-verkonhallinta-golden-ansible bash
Päivitykset komennolla apt update, ja sen jälkeen SNMP asennus: apt install snmp -y.
Testattiin, onko web1 pingattavissa: ping web1. Ping käskyä ei ollut, piti asentaa sen: 
apt install -y iputils-ping
Sen jälkeen ping web1 onnistui:
root@ansible:/# ping web1
PING web1 (172.20.20.7) 56(84) bytes of data.
64 bytes from clab-hamk-verkonhallinta-golden-web1.clab-mgmt (172.20.20.7): icmp_seq=1 ttl=64 time=10.7 ms
Web1 vastaa management-verkosta.
Kokeiltiin ping 10.10.20.101, muttei saatu yhteyttä. 

Asennettiin apt install net-tools, apt install iputils ja apt install iproute2.
net-tools-paketti sisältää verkkotyökaluja verkon asetusten ja yhteyksien tarkasteluun.
iputils sisältää työkaluja verkkoyhteyksien testaamiseen ja diagnosointiin. 
iproute2 sisältää työkalut verkkoliitäntöjen ja reitityksen hallintaan.

 
Suoritetut SNMP-kyselyt

Web1, db1, branch-client ja Ansible palvelimilla asennettiin snmp-mibs-downloader. Se lataa 
SNMP MIB tietokantoja, joiden avulla laitteiden
SNMP-tiedot näkyvät ymmärrettävässä muodossa numeroiden sijaan. Komento: apt install snmp-mibs-downloader
Jotta saada mibs käyttöön, /etc/snmp avataan snmp.conf konfiguraatiotiedoston. Siellä muokataan
ohjeen mukaisesti, eli komentoidaan mibs pois lisäämällä # edessä. 

Ensimmäinen SNMP-kysely tehtiin Ansible-palvelimelta komennolla: snmpwalk -v2c -c public web1 system
Kysely ei onnistunut. "System" vaihdettiin numeeriseen OID-tunnisteeseen: 
snmpwalk -v2c -c public web1 .1.3.6.1.2.1.1
Kysely ei onnistunut ja palautti: Timeout: No Response from web1. Tarkistettiin SNMP-agentin kuunteluosoitetta 
ss -lunp | grep ':161' komennolla. Tuli ilmi, että SNMP-agentti kuunteli paikallista loopback osoitetta ja tämän 
vuoksi kyselyitä ei voitu tehdä Ansible-palvelimesta. Tehtiin konfiguraatio web1-palvelimen snmpd.conf tiedostossa. 
Snmpd.conf tiedotossa muutettiin agentaddress, eli 127.0.0.1,[::1] sijaan laitettiin 172.20.20.7:161 ja 
käynnistettiin SNMP uudelleen komennolla service snmpd restart.  
Asennettiin SNMP-agentti myös db1:lle ja branch-client:lle samalla tavalla kuin web1:lle. Myös tehtiin samanlaiset
konfiguraatiot kuin web1:lle. 
 
Keskeiset komentotulosteet ja havainnot

Tiedot haettiin Ansible-palvelimesta SNMP-kysellyllä: snmpwalk -v2c -c public web1 .1.3.6.1.2.1.1 

järjestelmän nimi
web1
SNMPv2-MIB::sysName.0 = STRING: web1

Myös voi hakea tiedot komennolla: snmpget -v2c -c public web1 sysName.0

käyttöjärjestelmä
Linux
SNMPv2-MIB::sysDescr.0 = STRING: Linux web1 6.18.33.2-microsoft-standard-WSL2 #1 SMP 
PREEMPT_DYNAMIC Thu Jun 18 21:54:43 UTC 2026 x86_64

Myös voi hakea tiedot komennolla: snmpget -v2c -c public web1 sysDescr.0

uptime
0:59:08.13
DISMAN-EVENT-MIB::sysUpTimeInstance = Timeticks: (354813) 0:59:08.13

Myös voi hakea tiedot komennolla: snmpget -v2c -c public web1 sysUpTime.0

Samalla tavalla tehtiin kyselyt db1:lle ja branch-client:lle:

järjestelmän nimi
db1
SNMPv2-MIB::sysName.0 = STRING: db1

käyttöjärjestelmä
Linux
SNMPv2-MIB::sysDescr.0 = STRING: Linux db1 6.18.33.2-microsoft-standard-WSL2 #1 
SMP PREEMPT_DYNAMIC Thu Jun 18 21:54:43 UTC 2026 x86_64

uptime
0:13:06:23
DISMAN-EVENT-MIB::sysUpTimeInstance = Timeticks: (78623) 0:13:06.23

järjestelmän nimi 
branch-client
SNMPv2-MIB::sysName.0 = STRING: branch-client

käyttöjärjestelmä
Linux
SNMPv2-MIB::sysDescr.0 = STRING: Linux branch-client 6.18.33.2-microsoft-standard-WSL2 #1 
SMP PREEMPT_DYNAMIC Thu Jun 18 21:54:43 UTC 2026 x86_64

uptime
0:18:15.40
DISMAN-EVENT-MIB::sysUpTimeInstance = Timeticks: (109540) 0:18:15.40

4.Verkkorajapinnat 

SNMP:n avulla kerätyt verkkorajapintatiedot 

web1:

root@ansible:/# snmpwalk -v2c -c public web1 ifDescr
IF-MIB::ifDescr.1 = STRING: lo
IF-MIB::ifDescr.22 = STRING: eth0

db1:

root@ansible:/# snmpwalk -v2c -c public db1 ifDescr
IF-MIB::ifDescr.1 = STRING: lo
IF-MIB::ifDescr.18 = STRING: eth0

branch-client:
root@ansible:/# snmpwalk -v2c -c public branch-client ifDescr
IF-MIB::ifDescr.1 = STRING: lo
IF-MIB::ifDescr.40 = STRING: eth0

Rajapintojen tunnistaminen ja tulkinta 

Kyselyn tuloksena jokaisella web1, db1 ja branch-client löytyi kaksi verkkorajapintaa:
web1:
-lo (loopback): paikallinen rajapinta, jota käytetään laitteen sisäiseen verkkolliikenteeseen.
-eth0 (ethernet): rajapinta, joka yhdistää palvelimet verkkoon. 


5.OID-analyysi 

Käytetyt OID-objektit ja mitä tietoa ne tarjoavat ja mihin niitä käytetään? 

OID	
sysName.0: järjestelmän nimi, esimerkiksi web1, db1 tai branch-client
sysDescr.0: järjestelmän kuvaus, joka sisältää tietoja käyttöjärjestelmästä, ohjelmistosta ja laitteistosta  	
sysUpTime.0: aika, jonka SNMPn hallintaosa on ollut käynnissä. Se mittaa aikaa sadasosasekunteina (timeticks)	
ifDescr: Verkkorajapintojen nimet ja kuvaus	
ifOperStatus: Verkkorajapintojen tämänhetkinen toimintatila, eli up tai down. 


7.Pohdinta 

SNMP avulla seurataan laitteiden ja palvelimien toimintaa. Sen avulla voidaan kerätä tietoa laitteiden tilasta
ilman, että jokaiseen laitteeseen tarvitsee kirjautua erikseen. Tämä helpottaa verkon valvontaa ja ongelmien 
havaitsemista. SNMP avulla voidaan kerätä monenlaisia tietoa laitteista ja palvelimista. SNMP tärkeimpiä 
hyötyjä ovat keskitetty valvonta, tiedon keräämisen automatisointi ja laitteiden tilan nopea tarkistaminen. 
SNMP avulla voidaan kerätä esimerkiksi: verkkorajapintojen toimintatila, verkkoliikenteen määrät, virheet ja
hylätyt paketit, laitteen suorituskyky eli prosessorin kuormitus ja muistin käyttö), laitteen laitteistotiedot jne. 
SNMP rajoituksia ja haasteita ovat käyttöoikeuksien hallinta ja eri laitteiden tietojen tulkitseminen.
MIB puuttuminen voi vaikeuttaa tietojen lukemista. Myös SNMPv2c tietoturva on rajallinen, koska se 
ei salaa liikennettä. 
En ole käyttänyt SNMP aiemmin, joten minulla ei ole vielä paljon käytännön kokemusta eri SNMP-versioista.
Teorian mukaan SNMPv3 on turvallisempi kuin SNMPv2c, koska se tukee käyttäjien tunnistamista ja 
viestien salaamista. SNMPv3 suositellaan käyttää ympäristössä, joissa käsitellään arkaluonteista tietoja.
  
Tehtävän aikana opin SNMP perusperiaatteitta ja miten sitä voidaan käyttää laitteiden valvontaan. Asensin 
SNMP-agentin web1:lle, db1:lle ja branch-client:lle. Tutkin ja tein muutoksia niiden konfiguraatio-tiedostossa 
snmpd.conf. Harjoittelin web1, db1 ja branch-client tietojen hakemista Ansible-palvelimesta käyttäessä
snmpwalk ja snmpget. Tutkin myös palvelimien verkkorajapintoja ja ip osoitteet. 
Tutustuin MIB:siin. Opin, että OID avulla voidaan hakea tiettyjä tietoja ja että MIB auttaa muuttamaan 
numeeriset OID luettavaksi nimiksi. Opin, että onnistunut verkkoyhteys ei yksin takaa SNMP-kyselyn onnistumista. 
 

