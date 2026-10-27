1.MOODLE Wireshark tehtävä

Osa 1. yle.fi

Käynnistin Wiresharkin ja valitsin Wi-Fi-verkkoliitännän. Aloitin liikenteen kaappauksen ja avasin 
selaimessa osoitteen https://www.yle.fi. Tämän jälkeen pysäytin kaappauksen, jotta pystyin tutkimaan kerättyjä paketteja.
Käytin Wiresharkin näyttösuodatinta: dns.
Tämän jälkeen rajasin tulokset koskemaan Ylen verkkotunnusta käyttämällä:
dns.qry.name contains "yle.fi"

Suodatin etsi DNS-kyselyitä, joiden kysytyssä verkkotunnuksessa esiintyi teksti yle.fi.
Valitsin DNS-vastauspaketin ja avasin paketin yksityiskohtaisesta näkymästä DNS vastauksen Answers osion. 
Siellä näkyi useita A-tyypin vastauksia eli IPv4-osoitteita. Yksi vastauksessa näkyneistä osoitteista oli:
18.165.122.126. 
DNS-vastauksen tiedoissa näkyi myös pyyntöön liittyvä viite sekä vasteaika, joka oli noin 118,7 millisekuntia.

DNS-vastauksesta löytyneen IP-osoitteen maantieteellistä sijaintia tarkistin ipinfo.io-palvelussa:
https://ipinfo.io/18.165.122.126

Palvelun tuloksissa näkyivät seuraavat tiedot:

IP-osoite: 18.165.122.126, 
Sijainti: Helsinki, Uusimaa, Suomi. 
Organisaatio: Amazon.com, Inc. 
ASN: AS16509. 
Isäntänimi: server-18-165-122-126.hel51.r.cloudfront.net. 

Osa 2. learn.hamk.fi

1. Kaappasin Wiresharkilla verkkoliikennettä samalla, kun avasin osoitteen https://learn.hamk.fi/ ja kirjauduin sisään. 
DNS-pakettien tarkasteluun käytin näyttösuodatinta:

dns.qry.name contains "learn.hamk.fi"

DNS-vastauksen Answers-osiosta löytyi: Verkkotunnus: learn.hamk.fi, IP-osoite: 195.148.239.84, TTL: 1195 sekuntia (19 min ja 55 s)

Yritin etsiä auktoritatiivisia DNS-palvelimia Wiresharkista. En löytänyt.
Auktoritatiivisten DNS-palvelimien nimet selvitin Windowsin komentokehotteessa komennolla: nslookup -type=NS hamk.fi. 
Komento palautti seuraavat nimipalvelimet:
ns1.hamk.fi, ns2.hamk.fi, ns3.hamk.fi, ns-secondary.funet.fi

2. TLS-kättelyn tarkasteluun käytin Wiresharkin näyttösuodatinta: ssl.handshake.ciphersuite.
Suodatuksen avulla löysin Client Hello- ja Server Hello -paketit. Client Hello sisältää selaimen tarjoamat 
salausalgoritmit, ja Server Hello kertoo palvelimen valitseman salausalgoritmin.

Server Hello -paketin supported_versions laajennus vahvisti, että yhteydessä käytettiin TLS 1.3 protokollaa. 
Client Hello -paketin Cipher Suites -osiossa näkyi 16 merkintää, joista yksi oli GREASE-varattu arvo. 
Server Hello -paketissa palvelimen valitsema salausalgoritmi oli: TLS_AES_128_GCM_SHA256 (0x1301). Tämä on TLS 1.3 salausalgoritmi. 

Osa 3. Docker basic authentication

Käynnistin tjarvenpaa/basic_auth-Docker-imagen uudeksi kontiksi. Määritin ympäristömuuttujilla käyttäjätunnuksen ja 
salasanan sekä liitin oman HTML-tiedoston kontin /var/www-hakemistoon. Julkaisin kontin portin 80 paikalliselle 
portille 80.
Kontti käynnistyi onnistuneesti ja verkkosivu avautui osoitteessa 
http://localhost/. 
Aloitin Wireshark-kaappauksen Adapter for loopback traffic capture -verkkoliitännästä. 
Avasin selaimessa osoitteen http://localhost/ ja kirjauduin sisään tunnuksilla. 
Pysäytin kaappauksen onnistuneen kirjautumisen jälkeen. Tallensin kaappaustiedoston nimellä basic_authentication_lab. 
Tiedosto on palautettu Moodleen. 
Käytin Wiresharkin näyttösuodatinta: http. Tämän jälkeen käytin suodatinta:http.authorization.
Suodatin näytti kaksi pakettia, joissa oli HTTP Authorization. Pakettien tiedoista löytyi Authorization: 
Basic, mikä osoitti, että Basic Authentication tunnistetiedot lähetettiin HTTP-pyynnön mukana.

2.VIIKKO 5 – Tietoturva, lokit ja verkkoliikenteen analysointi

JOHDANTO

Tietoturva on tärkeä osa verkonhallintaa. Sen avulla suojataan verkkoa, laitteita ja tietoja luvattomalta käytöltä. 
Tässä tehtävässä tutkitaan verkkoliikennettä, lokitietoja ja palvelimien tietoturvaa. Tavoitteena on oppia 
havaitsemaan mahdollisia uhkia ja parantamaan verkon turvallisuutta.

WIRESHARK-ANALYYSI

Avasin web1.pcap tiedoston Wiresharkissa ja tarkastelin TCP-liikennettä näyttösuodattimien avulla.

Kaappauksessa näkyi TCP-paketteja, joiden lähdeosoite oli 172.20.20.8 ja kohdeosoite 172.20.20.11. 
Lähdeosoite vastaa skannausta suorittavaa konetta ja kohdeosoite web1-palvelinta.
TCP SYN paketeissa havaitsin useita kohdeportteja, kuten esimerkiksi 53, 199, 1723 ja 110. SYN-lippu ilmaisee TCP-yhteyden 
aloitusyrityksen. Eri kohdeportteihin lähetetyt SYN-paketit ovat tyypillisiä porttiskannaukselle.

Tarkastelin palvelimelta lähteviä TCP-paketteja suodattimella ip.src == 172.20.20.11 && tcp.flags.reset == 1. 
Pakettien TCP lipuissa näkyi RST, ACK. Tämä tarkoittaa, että palvelin hylkäsi yhteysyrityksiä suljettuihin TCP-portteihin.
SYN-ACK-vastauksia ei löytynyt suodattimella ip.src == 172.20.20.11 && tcp.flags.syn == 1 && tcp.flags.ack == 1. 
Kaappauksessa ei siten havaittu tämän suodattimen mukaisia vastauksia, jotka tavallisesti viittaavat avoimiin TCP-portteihin.

DNS-liikenne:

Tarkastelin web1.pcap tiedoston DNS-liikennettä Wiresharkissa käyttämällä näyttösuodatinta dns. Suodatin ei 
palauttanut pakettia, joten kaappauksessa ei havaittu DNS-kyselyitä tai DNS-vastauksia.
Kaappauksen perusteella ei voitu tunnistaa käytettyjä DNS-palvelimia, koska varsinaista DNS-liikennettä ei havaittu.

HTTP-liikenne

Tarkastelin web1.pcap tiedoston HTTP-liikennettä Wiresharkissa käyttämällä näyttösuodatinta http. Suodatin ei palauttanut pakettia, 
joten kaappauksessa ei havaittu HTTP-pyyntöjä tai HTTP-vastauksia. 
Tarkistin lisäksi HTTP oletusporttiin 80 liittyvän liikenteen suodattimella tcp.port == 80. Suodatin palautti kaksi 
TCP-pakettia.
Ensimmäinen paketti oli 38322 → 80 [SYN], joka osoitti TCP-yhteyden muodostamisyrityksen porttiin 80. 
Toinen paketti oli 80 → 38322 [RST, ACK], joka osoitti yhteysyrityksen hylkäämisen. 
Tämä tarkoittaa, että portti 80 oli suljettu.
Kaappauksessa näkyi HTTP-porttiin kohdistuvaa liikennettä, mutta ei HTTP-protokollan 
mukaista tiedonsiirtoa. HTTP-palvelinta ei voitu tunnistaa toimivaksi tämän liikenteen perusteella.


LOKI_ANALYYSI

Tarkastelin web1-kontin lokitietoja ja etsin kirjautumistapahtumia, virheitä sekä palveluiden käynnistymiseen 
liittyviä merkintöjä.

Yritin tarkastella /var/log/syslog tiedostoa, mutta sitä ei löytynyt kontista. Kokeilin myös komentoa 
journalctl -n 20 --no-pager, joka palautti ilmoituksen No journal files were found.
Lisäksi tarkistin kontin lokit komennolla docker logs, mutta se ei palauttanut mitään.

Tarkistin kirjautumishistorian komennolla last. Komento palautti vain päivämäärä, milloin wtmp begins, 
eikä kirjautumistapahtumia näkynyt.

Tarkastelin Node Exporterin lokia tiedostosta /var/log/node_exporter.log.
Lokissa näkyi useita level=info merkintöjä, jotka liittyivät mittareiden kerääjien alustamiseen. 
Esimerkiksi meminfo, netdev ja netstat olivat käytössä.

Lokissa näkyi myös merkintä: msg="Listening on" address=[::]:9100.
Tämä osoitti, että Node Exporter käynnisti kuuntelun portissa 9100.
Lisäksi lokissa oli merkintä TLS is disabled.

Tarkastelin Nginxin access.log- ja error.log tiedostoja. access.log sisälsi kaksi HTTP GET -pyyntöä ja palauttivat 
HTTP-tilakoodin 200, joka tarkoittaa onnistunutta pyyntöä. 
Pyyntöjen lähdeosoite oli ::1, eli IPv6 localhost osoite. User-Agent oli ansible-httpget. 
Nginxin error.log oli tyhjä, eikä tarkastelluissa Node Exporterin lokiriveissä näkynyt virheilmoituksia. Niissä oli 
ainoastaan level=info-merkintöjä.

TURVALLISUUSARVIO

Suljetut TCP portit on turvallinen havainto.
Nmap-skannauksessa web1-palvelimen osoitteeseen 172.20.20.11 kaikki tarkistettua TCP-porttia ilmoitettiin suljetuiksi. 
Myös Wiresharkissa havaittiin RST, ACK -vastauksia, jotka viittasivat suljettuihin portteihin.
Suljetut portit vähentävät mahdollisia hyökkäyspintoja, koska kyseisissä porteissa ei havaittu yhteyksiä 
vastaanottavia palveluita.

Node Exporterin salaamaton yhteys on mahdollinen tietoturvariski.
Node Exporterin lokissa näkyi merkintä TLS is disabled. Tämä tarkoittaa, että Node Exporterin HTTP-rajapinta ei käytä 
TLS-salausta. Mittaustietoja voidaan tarkastella salaamattoman yhteyden kautta. Pitäisi rajoittaa Node Exporterin 
portin 9100 käyttöä sekä ottaa tarvittaessa TLS-salaus käyttöön.

web1-kontista ei löytynyt /var/log/syslog-tiedostoa, eikä journalctl palauttanut lokimerkintöjä. Myös docker logs oli tyhjä. 
Tämä vaikeuti mahdollisten tietoturvatapahtumien, virheiden ja kirjautumisyritysten tutkimista.

POHDINTA

1. Miten porttiskannaus näkyi liikenteessä?

Porttiskannaus näkyi Wiresharkissa useina TCP SYN -paketteina, jotka lähetettiin eri kohdeportteihin. Liikenteen lähdeosoite 
oli 172.20.20.8 ja kohdeosoite 172.20.20.11. Havaitsin esimerkiksi portteihin 53, 199, 1723 ja 110 yhteydenottoyrityksiä. 
Palvelin vastasi osaan yhteydenottoyrityksistä RST, ACK, mikä tarkoittaa, että kyseiset TCP portit olivat suljettuja.

2. Miten se näkyi lokeissa?

Loki-analyysissä ei löytynyt suoria merkintöjä porttiskannauksesta. Nginxin access.log sisälsi kaksi onnistunutta 
HTTP-pyyntöä, jotka oli tehty Ansiblella. Nginxin error.log oli tyhjä, eikä käytettävissä ollut /var/log/syslog-tiedostoa 
tai systemd-journalin merkintöjä. Porttiskannausta en voinut vahvistaa lokien perusteella.

3. Mitä hyötyä verkkoliikenteen analysoinnista on?

Verkkoliikenteen analysoinnin avulla voidaan tunnistaa yhteydenottoyrityksiä, selvittää liikenteen lähde- ja 
kohdeosoitteita sekä tarkastella käytettyjä protokollia ja portteja.
Wiresharkin avulla pystyin erottamaan TCP SYN -paketit palvelimen RST, ACK -vastauksista. Analysointi auttaa 
havaitsemaan epätavallista liikennettä, selvittämään verkkoyhteyksien ongelmia ja tutkimaan mahdollisia tietoturvapoikkeamia.

4. Miten monitorointia voisi hyödyntää hyökkäysten tunnistamisessa?

Monitorointia voidaan hyödyntää keräämällä jatkuvasti tietoa verkkoliikenteestä, palvelimien suorituskyvystä ja 
järjestelmien tapahtumista. Esimerkiksi suuri määrä lyhyessä ajassa eri porteille lähetettyjä SYN-paketteja voi viitata
porttiskannaukseen. rometheus- ja Grafana-työkaluilla voidaan seurata järjestelmien mittareita ja havaita poikkeavia 
muutoksia. Lisäksi keskitetty lokien keräys ja hälytykset auttavat tunnistamaan epäonnistuneita kirjautumisia, 
palveluvirheitä ja muita epäilyttäviä tapahtumia. Tehokas hyökkäysten tunnistaminen edellyttää verkkoliikenteen, 
lokien ja monitorointimittareiden yhdistämistä.
