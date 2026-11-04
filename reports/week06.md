JOHDANTO

Zabbix on open open-source valvontajärjestelmä, jota käytetään palvelimien, verkkolaitteiden ja palveluiden 
toiminnan seuraamiseen. Sen avulla voidaan tarkkailla esimerkiksi suorittimen käyttöastetta, muistin kulutusta, 
levytilaa ja verkkoliikennettä.
Zabbix kerää tietoja valvottavista laitteista ja näyttää ne graafisesti dashboardissa. Järjestelmään voidaan myös 
määrittää triggereitä, jotka ilmoittavat mahdollisista ongelmista.
Zabbix tukee myös SNMP-protokollaa, jonka avulla voidaan valvoa verkkolaitteita ilman Zabbix Agentin asentamista. 
Tässä tehtävässä tutustutaan Zabbixin käyttöön, lisätään valvottavia laitteita, tarkastellaan mittareita ja 
testataan hälytysten toimintaa.

6.1 - Tutustu Zabbixiin

Käyttöliittymän osat:

Hosts

Näyttää Zabbixiin lisätyt valvottavat laitteet, kuten palvelimet ja verkkolaitteet. 
Täällä voidaan lisätä uusia laitteita ja muokata niiden valvonta-asetuksia.
Configuration -> Hosts näkymässä näkyi Zabbix server niminen valvottava laite. Sille oli määritetty 12 sovellusryhmää 
(Applications), 88 mittaria (Items), 48 triggeriä, 14 kuvaajaa (Graphs) ja 3 automaattisen tunnistuksen sääntöä (Discovery).
Laitteen tila oli Enabled, mutta saatavuuden kohdalla ZBX näkyi punaisena. 
Tämä tarkoittaa, että Zabbix Agent ei ollut saatavilla.
Agent encryption asetuksena oli None.
Hosts-näkymän avulla voidaan tarkastella valvottavia laitteita, niiden saatavuutta ja valvonta-asetuksia.

Templates

Sisältää valmiita valvontamalleja, joita voidaan liittää laitteisiin. Mallit sisältävät esimerkiksi mittareita ja 
triggereitä, joten samoja asetuksia ei tarvitse tehdä jokaiselle laitteelle erikseen. 
Configuration -> Templates näkymässä oli paljon valmiita valvontamalleja eri palveluille ja sovelluksille. 
Esimerkiksi Apachelle, HAProxylle, IMAPille, LDAPille ja Nginxille oli omia malleja. 
Osassa malleista käytettiin HTTP-yhteyttä ja osassa Zabbix Agentia tietojen keräämiseen.

Monitoring

Näyttää valvottavien laitteiden tilan, kerätyt mittaustiedot ja havaitut ongelmat. Sen avulla voidaan seurata verkon 
ja palvelimien toimintaa. 
Avasin Latest data, mutta siellä ei näkynyt mitään tietoja. Problems-näkymässä näkyi hälytys “Zabbix agent is not available 
(for 3m)”. Tämä tarkoittaa, että Zabbix ei saanut yhteyttä valvottavan laitteen agenttiin.

Dashboards

Näyttää valvontatiedot yhdessä näkymässä esimerkiksi kuvaajina, taulukoina ja tilatietoina. Dashboardia voidaan muokata 
tarpeen mukaan. 

Dashboard -näkymässä näkyi yhteenveto Zabbixin valvontatiedoista. Dashboardissa oli tietoja valvottavista laitteista, 
mittareista ja triggereistä. Dashboardissa näkyi myös oranssi hälytys “Zabbix agent is not available (for 3m)”. 

Alerts

Sisältää hälytysten hallintaan liittyviä asetuksia ja tietoja. Hälytysten avulla voidaan ilmoittaa esimerkiksi 
palvelimen ongelmista tai liian korkeasta CPU-käyttöasteesta.

Reports

Sisältää raportointiin liittyviä toimintoja, joiden avulla voidaan tarkastella esimerkiksi järjestelmän toimintaa, 
saatavuutta ja hälytystietoja.

6.2, 6.3. ja 6.4 - Lisää web1, db1 ja branch-client valvontaan

Tehtävän tavoitteena oli asentaa Zabbix Agent 2 web1, db1 ja branch-client palvelimille ja määrittää se kommunikoimaan Zabbix Serverin kanssa.
 
Aloitin lataamalla kaikille kolmille (web1, db1 ja branch-client) Zabbix 7.4 -version Ubuntu 24.04 -käyttöjärjestelmälle tarkoitetun repository-paketin wget-komennolla:
wget https://repo.zabbix.com/zabbix/7.4/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.4+ubuntu24.04_all.deb

Komento latasi Zabbixin repository-paketin.
Seuraavaksi asensin ladatun paketin:
dpkg -i zabbix-release_latest_7.4+ubuntu24.04_all.deb

Tämä lisäsi Zabbixin ohjelmistolähteen Ubuntuun, jolloin Zabbix-paketit voitiin asentaa järjestelmän paketinhallinnan 
kautta.
Päivitin tämän jälkeen pakettiluettelon komennolla: apt update. 
Asensin Zabbix Agent 2 seuraavalla komennolla: apt install zabbix-agent2.
Zabbix Agent 2 kerää palvelimen suorituskykyyn ja toimintaan liittyviä mittaustietoja, kuten suorittimen käyttöastetta, 
muistin käyttöä, levytilaa ja verkkoliikennettä.

Lisäsin web1, db1 ja branch-client palvelimet Zabbixin valvontaan. Tein seuraavia muutoksia zabbix_agent2.conf tiedostoissa Visual Studio Codella
(Asetustiedosto sijaitsee polussa: /etc/zabbix/zabbix_agent2.conf):
Passive checks related: Server=127.0.0.1,172.20.20.0/24
Määritys sallii Zabbix-palvelimen ja muiden määritellyn verkon laitteiden yhteydet agenttiin passiivisia tarkistuksia 
varten. Lisäksi paikallinen osoite 127.0.0.1 sallii yhteydet saman laitteen sisältä.

Active checks related: ServerActive=172.20.20.10
Tämä määrittää Zabbix Serverin IP-osoitteen, johon Zabbix Agent 2 ottaa yhteyttä aktiivisia tarkistuksia varten. 

Hostname: Hostname=web1, Hostname=db1 ja Hostname=branch-client
Hostname kertoo Zabbix Serverille, minkä valvottavan laitteen tietoja agentti lähettää. Nimen tulee vastata Zabbixin 
käyttöliittymässä määritettyä hostin nimeä.

6.5 Valvontamittarien analyysi

db1

CPU usage: 3,0038 %. Prosessorin käyttöaste on 3,0038 %, eli palvelin käyttää tällä hetkellä vain pienen osan prosessorin kapasiteetista. 
Tämä kertoo, että db1 prosessorikuormitus on matala. Mittaria kannattaa seurata, jotta voidaan havaita mahdollinen ylikuormitus ja 
suorituskykyongelmat ajoissa.

Memory usage: 49,6754 %. db1:lla on käytettävissä noin 49,68 % muistista. Tämä tarkoittaa, että muistia on vielä hyvin saatavilla. 
Mittaria kannattaa seurata, jotta mahdollinen muistin loppuminen voidaan havaita ajoissa ja välttää palvelimen hidastumista.

Load average (1m avg): 	0.4663. Palvelimen keskimääräinen kuormitus viimeisen minuutin aikana. Arvo on matala, mikä tarkoittaa, että palvelimella 
ei ole tällä hetkellä suurta työkuormaa. Mittaria seurataan, jotta voidaan havaita prosessorin ylikuormitus ja mahdolliset suorituskykyongelmat.

Disk write rate (sdd): 20,1476 w/s. Levyasema sdd suorittaa noin 20 kirjoitusoperaatiota sekunnissa. Mittari kertoo, kuinka paljon levylle 
kirjoitetaan tietoa. Sitä kannattaa seurata, jotta voidaan havaita poikkeuksellisen suuri levyn kirjoituskuormitus ja mahdolliset 
suorituskykyongelmat.

Network traffic (eth0): 2,47 Kbps vastaanotettu ja 1,54 Kbps lähetetty. Mittari kertoo verkkoliikenteen määrän eli kuinka paljon palvelin 
vastaanottaa ja lähettää tietoa verkon kautta sekunnissa. Mittaria seurataan, jotta voidaan havaita poikkeava verkkoliikenne, 
yhteysongelmat ja mahdolliset verkkoyhteyden pullonkaulat. 

web1 

CPU usage: 3.1961 %.

Memory usage: 49.9592 %.

Load average (1m avg): 0.4756.

Disk utilization (sdd): 5.0175 %. Tämä kertoo, kuinka suuren osan ajasta levy on aktiivisessa käytössä eli käsittelee luku- ja 
kirjoitusoperaatioita. Arvo on matala, joten levy ei ole tällä hetkellä kovassa kuormituksessa. Mittaria kannattaa seurata, jotta mahdollinen 
levyn ylikuormitus ja suorituskykyongelmat voidaan havaita ajoissa.

Network traffic (eth0): 3.08 Kbps vastaanotettu ja 2.22 Kbps lähetetty.  

branch-client

CPU usage: 3.1194 %.

Memory usage: 49.6492 %.

Load average (1m avg): 0.2988.

Disk utilization (sdd): 3.9358 %. 

Netwok traffic (eth0): 2.63 Kbps vastaanotettu ja 1.8 Kbps lähetetty.

6.6 – Dashboardin luominen

Zabbixiin luotiin dashboard nimeltä Golden Topology Status. Dashboardin tarkoituksena on tarjota yhdellä silmäyksellä yleiskuva 
kaikkien kolmen koneen tilasta.

Dashboardiin lisättiin seuraavat widgetit:
1. CPU Utilization – näyttää prosessorin käyttöasteen.
2. Memory Utilization – näyttää muistin käyttöön liittyvät arvot.
3. Disk Utilization – tarkoitettu levytilan käytön seurantaan.
4. Network Traffic – näyttää saapuvan ja lähtevän verkkoliikenteen.
5. Host Status – näyttää koneiden Zabbix-agenttien tilan.
Verkkoliikenteen seurannassa käytettiin verkkoliittymän eth0 mittareita Bits received ja Bits sent.

Tehty dashboardin kuvakaapaukset. Ne löytyy tiedostossa reports/images/week06_kuvat.

Tehtävä 6.7 – Triggerien määrittäminen

Zabbixilla web1:lle, db1:lle ja branch-client:lle määritetään kaksi triggeriä, joiden tarkoituksena on ilmoittaa palvelimen liian korkeasta 
CPU-kuormituksesta ja vähäisestä vapaasta levytilasta.

Triggeri 1: CPU-kuormitus 
- Triggerin nimi: High CPU utilization. 
- Item: CPU utilization
- Ehto: {branch-client:system.cpu.util.last()}>80.
- Vakavuusluokka: Warning (Varoitus).
Merkitys: Triggeri aktivoituu, kun prosessorin käyttöaste ylittää 80 %. Tämä ilmoittaa mahdollisesta liian suuresta järjestelmäkuormituksesta.

Triggeri 2: Vapaa levytila
- Triggerin nimi: Low free disk space.
- Item: /configs: Space utilization.
- Ehto: {branch-client:vfs.fs.size[/configs,pused].last()}>80.
- Vakavuusluokka: Warning (Varoitus).
Merkitys: Triggeri aktivoituu, kun tiedostojärjestelmän vapaa levytila laskee alle 20 prosentin. Tämä varoittaa levytilan loppumisen riskistä.

6.8 – Hälytyksen simulointi

Tehtävän tavoitteena oli testata Zabbix-järjestelmän hälytysten toimintaa aiheuttamalla keinotekoisesti korkea CPU-kuormitus 
branch-client-palvelimella.

CPU-kuormitus aiheutettiin komennolla:

yes > /dev/null

CPU-kuormitusta seurattiin Zabbixin kohdasta Monitoring -> Latest data, valitsemalla palvelimeksi branch-client ja mittariksi CPU utilization. 
CPU-kuormitus nousi lopulta noin 99 prosenttiin, mikä ylitti aikaisemmin määritetyn 80 prosentin hälytysrajan. 
Hälytyksen tila tarkistettiin kohdasta Monitoring -> Problems. Hälytyksen nimi oli High CPU utilization ja vakavuustaso Warning.
Testin jälkeen CPU-kuormitusta pysäytettiin komennolla: pkill yes.
CPU-kuormituksen simulointi onnistui. Hälytys ilmestyi noin 6 minuutin päästä. CPU-käyttö nousi 99 prosenttiin, eli määritetty 80 prosentin hälytysraja ylitettiin. 
Testillä voitiin tarkistaa, että Zabbixin CPU-hälytys reagoi korkeaan CPU kuormitukseen.

6.9 – Häiriötilanne

Tehtävän tavoitteena oli simuloida palveluhäiriö pysäyttämällä Apache-verkkopalvelu web1:lla ja tarkkailla, havaitseeko Zabbix 
häiriön sekä kuinka nopeasti valvontajärjestelmä reagoi.

Ensin tarkistettiin, että Apache-palvelu oli käynnissä: service apache2 status. Tämän jälkeen Apache-palvelu pysäytettiin komennolla: service apache2 stop.
Zabbix havaitsi Apache-palvelun pysähtymisen. Hälytys ilmestyi kohtaan Monitoring -> Problems. 
Severity oli high, status: problem, host: web1, problem: Apache:Process is not running, duration: 2m. 

6.10 – Kurssin työkalujen vertailu

- Tiedonkeruu
SNMP: Kerää verkkolaitteiden ja palvelimien tietoja SNMP-protokollalla. 
Prometheus: Kerää mittaustietoja yleensä pull-menetelmällä HTTP-rajapintojen kautta.
Zabbix: Kerää tietoja Zabbix-agentilla, SNMP:llä, IPMI:llä ja muilla menetelmillä.

- Dashboardit
SNMP: erillisen valvontatyökalun avulla.
Prometheus: usein käytetään Grafanaa.
Zabbix: sisäänrakennetut dashboardit, graafit ja näkymät.

- Hälytykset
SNMP: erillisen järjestelmän avulla.
Prometheus: Alertmanager hoitaa hälytysten käsittelyn ja lähettämisen.
Zabbix: sisäänrakennetut triggerit ja hälytykset.

- Käyttöönotto
SNMP: Perusvalvonta on melko helppo ottaa käyttöön, koska useimmat verkkolaitteet tukevat sitä valmiiksi. Käyttöönotossa SNMP täytyy ottaa 
käyttöön valvottavassa laitteessa, määrittää SNMP-versio (v1, v2c tai v3), yhteisönimi tai käyttäjätunnukset sekä sallia valvontapalvelimen 
pääsy laitteen SNMP-porttiin. Lisäksi valvontajärjestelmään määritetään laitteen IP-osoite ja tarvittavat OID-tunnisteet, joiden avulla 
kerätään esimerkiksi prosessorin, muistin ja verkkoliikenteen tietoja.
Prometheus: Vaatii yleensä Prometheuksen, exporterien ja Grafanan asentamisen ja määrittämisen.
Zabbix: Zabbixin käyttöönotto vaatii Zabbix-palvelimen asentamisen ja määrittämisen, tietokannan luomisen sekä web-käyttöliittymän käyttöönoton.
Lisäksi valvottaville palvelimille asennetaan ja määritetään Zabbix-agentit. Määritetään lisäksi hostit, template-mallit, mittarit ja hälytykset.

- Skaalautuvuus
SNMP: Soveltuu hyvin suurten verkkolaitemäärien valvontaan.
Prometheus: Skaalautuu hyvin, mutta suuret ympäristöt voivat vaatia erillistä tallennusratkaisua.
Zabbix: Soveltuu pienistä ympäristöistä suuriin yritysverkkoihin. 

- Yrityskäyttö
SNMP: Yleinen verkkolaitteiden valvonnassa yrityksissä.
Prometheus: Käytetään paljon pilviympäristöissä, konttiympäristöissä ja moderneissa IT-järjestelmissä.
Zabbix: Soveltuu yritysten palvelimien, verkkolaitteiden ja sovellusten keskitettyyn valvontaan.

6.11 – Pohdinta

Keskitetyn valvonnan avulla kaikkien palvelimien ja verkkolaitteiden toimintaa voidaan seurata yhdestä paikasta. 
Ylläpitäjän ei tarvitse tarkistaa jokaista laitetta erikseen. Valvonta auttaa havaitsemaan ongelmat ajoissa, vähentää käyttökatkoja ja 
helpottaa vikatilanteiden selvittämistä. Lisäksi historiatietojen avulla voidaan seurata järjestelmän toimintaa ja ennakoida mahdollisia ongelmia.
Tärkeimpiä mittareita ovat CPU-kuormitus, muistin käyttö, levytilan käyttö, verkkoliikenne ja palveluiden tila. Näiden avulla voidaan 
havaita esimerkiksi palvelimen ylikuormittuminen, muistin loppuminen, levytilan täyttyminen tai verkkoyhteyden ongelmat. 
Myös palvelimien käytettävyyden seuranta on tärkeää, jotta mahdolliset käyttökatkot havaitaan nopeasti.
Ylläpitäjän pitäisi saada hälytys esimerkiksi silloin, kun CPU kuormituu liikkaa, levytilaa on jäljellä vähän tai muistin käyttö kasvaa liian 
suureksi. Hälytys pitäisi tulla myös silloin, kun palvelin tai verkkolaite ei vastaa, tärkeä palvelu pysähtyy, verkkoyhteydessä havaitaan 
ongelmia tai järjestelmässä tapahtuu odottamaton käyttökatko. Hälytysten avulla ylläpitäjä voi reagoida ongelmiin ennen kuin ne 
aiheuttavat suurempia häiriöitä käyttäjille.
Käyttäisin Prometheusta erityisesti kontti- ja pilviympäristöjen sekä sovellusten suorituskyvyn valvontaan. 
Zabbixia käyttäisin puolestaan palvelimien, verkkolaitteiden ja palveluiden kokonaisvaltaiseen valvontaan. 
Tässä ympäristössä lisäisin vielä esimerkiksi tietokannan, varmuuskopioiden ja verkkoyhteyksien valvonnan.
Hyvin toteutettu valvonta parantaa järjestelmän luotettavuutta, auttaa ehkäisemään ongelmia ja nopeuttaa häiriötilanteisiin reagoimista.
 




