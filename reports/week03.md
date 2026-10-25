1.Johdanto

Tämän viikon tavoitteena on tutustua verkon ja palvelimien monitorointiin Prometheuksen, Node Exporterin ja 
Grafanan avulla.
Prometheus kerää mittaustietoja määritellyistä kohteista. Node Exporter tarjoaa palvelimien järjestelmätietoja, 
kuten suorittimen, muistin ja levyn käyttöön liittyviä mittareita. Grafanan avulla mittaustietoja voidaan 
visualisoida.

2.Ympäristön rakentaminen

Prometheuksen asetustiedosto sijaitsee configs/prometheus/prometheus.yml. Tiedostossa on 
määritelty monitoroitavat kohteet ja mittaustietojen keräysväli.
Yritin asentaa Node Exporterin ansible kontissa seuraavalla komennolla:
ansible-playbook -i inventory.ini playbooks/install-node-exporter.yml
Ensimmäinen asennusyritys epäonnistui, koska Ansible ei saanut SSH-yhteyttä kohdekontteihin.
Tarkistin web1-kontin SSH-palvelun tilan komennolla service ssh status. Se ei ollut käynnissä.
Käynnistin SSH-palvelun konteissa web1, db1, client1 ja branch-client komennolla service ssh start. 
Sen jälkeen suoritin Ansible-asennuksen uudelleen. Toinen asennusyritys onnistui kaikissa neljässä kontissa.
Myös olen asentanut web-monitor komennolla: 
ansible-playbook -i inventory.ini playbooks/install-web-monitoring-demo.yml

Tarkistin Node Exporterin toiminnan web1-kontissa komennolla:
curl http://localhost:9100/metrics
Node Exporter vastasi HTTP-pyyntöön ja palautti suuren määrän järjestelmän mittareita (tehty kuvakaappaus). Tämä vahvisti, 
että Node Exporter on käynnissä.
Seuraavaksi tarkistin Prometheuksen Target Health:ssa, että Node Exporterin mittarit ovat saatavilla ja client1, 
branch-client, web1, db1 ovat up tilassa. Siitä myös tehty kuvakaappaus.

Grafanan Prometheus-tietolähde on jo ollut, yhteys toimii (tehty kuvakaappaus). 

PROMETHEUS KYSELYT

Seuraavaksi olen luonut dashboard Grafanassa nimellä Golden Topology Monitoring. Sinne lisäsin CPU Usage % paneeli. 
Paneelissa käytetään PromQL:

100 - (avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

Kysely laskee CPUn käyttöasteen prosentteina viimeisen viiden minuutin ajalla.
CPU käyttöaste oli melko matala, noin 1,3–1,8 %. Käyttöaste vaihteli hieman mittausjakson aikana. 
Eri kohteiden käyrät olivat hyvin lähellä toisiaan. Suuria kuormituspiikkejä ei havaittu.

Seuraavaksi lisäsin memory Usage % paneeli. PromQL-kysely:

(node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes)
/node_memory_MemTotal_bytes * 100
Kysely laskee käytössä olevan muistin osuuden prosentteina. Kokonaismuistista vähennetään käytettävissä oleva muisti, 
ja tulos jaetaan kokonaismuistilla. Näin saadaan arvio siitä, kuinka suuri osa muistista on käytössä.

Muistin käyttöaste vaihteli noin 34–37 % välillä. Käyttöaste nousi ajoittain ja laski uudelleen. Eri kohteiden käyrät olivat 
lähes samanlaisia. Muistin käyttöaste pysyi koko tarkastelujakson ajan alle 40 prosentissa.

Lisäsin Disk Usage % paneeli, PromQL-kysely:

100 - (node_filesystem_avail_bytes/node_filesystem_size_bytes * 100)

Kysely laskee järjestelmän käytetyn levytilan prosenttiosuuden. Paneelin avulla voidaan tarkkailla levytilan käyttöä. 
Kuvakaappauksessa levytilan käyttöaste oli noin 6,07 %. Käyttöaste nousi hitaasti tarkastelujakson aikana, mutta merkittävää 
muutosta ei tapahtunut.

Myös lisäsin Network Receive and Network Transmit paneelit. PromQL:

rate(node_network_receive_bytes_total[5m])

Kysely mittaa verkkoliitännän vastaanottaman datan keskimääräistä nopeutta viimeisen viiden minuutin aikana. Tulos 
ilmoitetaan tavuina sekunnissa (B/s).
Liikennemäärissä ei näkynyt suuria muutoksia. Verkkoliikenteen määrä vaihteli verkkoliitännän mukaan.

rate(node_network_transmit_bytes_total[5m])

Kysely mittaa verkkoliitännän lähettämän datan keskimääräistä nopeutta viimeisen viiden minuutin aikana. 
Tulos ilmoitetaan tavuina sekunnissa (B/s).
Mittarin avulla voidaan tarkastella lähtevää verkkoliikennettä eri verkkoliitännöissä.
Lähtevä verkkoliikenne pysyi suurimmaksi osaksi tasaisena. Suuria jatkuvia muutoksia ei havaittu.

CPU:n ja muistin käyttöä kuvaavat käyrät näyttävät lähes samoja arvoja web1, db1, client1 ja branch-client.

Minun luotujen Grafanan dashboardin avulla voidaan seurata keskitetysti CPU käyttöä, muistinkulutusta, levytilaa sekä saapuvaa ja 
lähtevää verkkoliikennettä.
PromQL-kyselyt mahdollistavat mittaustietojen käsittelyn ja visualisoinnin. Näiden mittareiden avulla voidaan 
tunnistaa suorituskyvyn muutoksia, vertailla monitoroitavia kohteita ja havaita mahdollisia ongelmia.

KUORMITUKSEN GENEROINTI

Levykuormitus
Suoritin web1:lla levykuormitustestin komennolla: dd if=/dev/zero of=testfile.img bs=1M count=500. 
Grafanan Disk Usage % paneelissa näkyi hyvin pieni ja tasainen nousu. Tämä johtuu siitä, että luotun 
tiedoston koko oli pieni verrattuna käytettävissä olevaan levytilaan.

CPU-kuormitus

Ensin asensin stress-ng komennolla: sudo apt install stress-ng
Suoritin web1-kontissa CPU-kuormitustestin komennolla stress-ng --cpu 4 --timeout 60s. kaikkien neljän 
monitoroitavien kontien CPU-käyrät nousivat lähes samanaikaisesti, vaikka kuormitusta 
aiheutettiin vain web1-kontissa.

Ennen kuormitustestiä CPU käyttöaste oli 1,42 %. Kuormituksen aikana käyttöaste nousi 
nopeasti ja korkeimmillaan oli 8,51 %. 

Memory Usage % paneelissa näkyi myös pieni nousu. Muistin käyttöaste nousi tarkastelujakson aikana 
noin 35,5 prosentista 38 prosenttiin. Muutos tapahtui suunnilleen samaan aikaan CPU-kuormituksen kanssa.

Verkkoliikenteessä ei näkynyt merkittäviä muutoksia. Molempien paneelien käyrät pysyivät 
vaakasuorina koko ajan.

SNMP vs PROMETHEUS

Tiedonkeruu

SNMP: tietoja haeataan SNMP-kyselyillä laitteiden MIB/OID arvoista. Prometheus hakee mitaustiedot 
esimerkiksi Node Exporterilta.

Käyttöönotto

SNMP: Laitteelle määritetään SNMP agentti ja tarvittavat asetukset. Prometheus: Asennetaan Prometheus
ja exporter, määritetään monitoroivat kohteet.

Mittarien määrä

SNMP: riippuu MIB-tiedoista. Prometheus: exporterin avulla on tarjottavissa suuri määrä erilaisia 
mittareita.

Visualisointi

SNMP: Vaatii erillisen monitorointiohjelmiston. Prometheus: Grafanalla voidaan luoda monipuolisia 
dashboardeja ja kuvaajia.

Hälytysmahdollisuudet

SNMP: SNMP trapit ja erilliset monitorointijärjestelmät mahdollistavat hälytykset.
Prometheus: Prometheuksen hälytyssäännöt ja Alertmanager mahdollistavat hälytysten käsittelyn.

Soveltuvuus pilviympäristöihin

SNMP: Sopii erityisesti perinteisten verkkolaitteiden monitorointiin. Prometheus: Sopii hyvin 
pilviympäristöihin, kontteihin ja dynaamisiin järjestelmiin.


POHDINTA

1.Mitä hyötyä Prometheuksesta on verrattuna SNMP?

Prometheuksen avulla voidaan kerätä monipuolisesti tietoja palvelimista, sovelluksista ja konteista. 
Prometheus tallentaa mittaustiedot aikasarjoina, joten niiden muutoksia voidaan seurata ajan kuluessa. 
PromQL-kyselykielen avulla tietoja voidaan myös käsitellä ja analysoida. 
SNMP on verkonvalvontaprotokolla, kun taas Prometheus on kokonaisvaltainen mittaustietojen keruu- 
ja tallennusjärjestelmä. 

2.Millaisia mittareita ylläpitäjän kannattaa seurata jatkuvasti?

Ylläpitäjän kannattaa seurata erityisesti CPUn käyttöastetta, muistin kulutusta, 
levytilan käyttöä ja verkkoliikennettä.

Lisäksi on tärkeää seurata palveluiden saatavuutta ja mahdollisia virheitä. Näiden mittareiden 
avulla voidaan huomata ongelmia ennen kuin ne vaikuttavat järjestelmän toimintaan.

3.Mitä tietoa dashboardisi tarjoaa ylläpitäjälle?

Olen luonut Grafanaan Golden Topology Monitoring dashboardin, jossa näkyvät CPU käyttöaste, muistin 
käyttö, levytilan käyttö sekä saapuva ja lähtevä verkkoliikenne.
Dashboardin avulla ylläpitäjä voi seurata järjestelmän suorituskykyä ja havaita muutoksia eri 
mittareissa.
Esimerkiksi CPU-kuormitustestissä käyttöaste nousi 1,42 prosentista 8,51 prosenttiin. 
Muutos näkyi Grafanan kuvaajassa heti.

4.Mitä uusia mittareita lisäisit dashboardiin?

Voisin lisätä esimerkiksi levyjen luku- ja kirjoitusnopeudet (PromQL: rate(node_disk_read_bytes_total[5m] rate(node_disk_written_bytes_total[5m]) ). 
Myös Verkkoyhteyksien virheiden ja pudonneiden pakettien seuranta olisi hyödyllistä, koska niiden avulla voidaan havaita verkon toimintahäiriöitä ja yhteysongelmia. 
Jos virheiden tai pudonneiden pakettien määrä kasvaa, se voi viitata esimerkiksi verkon ruuhkautumiseen tai yhteyden huonoon laatuun. 
Näiden mittareiden avulla ylläpitäjä voisi tunnistaa ongelmat ajoissa. (PromQl Vastaanotetun verkkoliikenteen virheet: rate(node_network_receive_errs_total[5m]),
Lähetetyn verkkoliikenteen virheet: rate(node_network_transmit_errs_total[5m]), 
Vastaanotetut pudonneet paketit: rate(node_network_receive_drop_total[5m]), 
Lähetetyt pudonneet paketit: rate(node_network_transmit_drop_total[5m])

5.Miten monitorointitiedosta voisi olla hyötyä vianetsinnässä?

Monitorointitietojen avulla voidaan selvittää, milloin ongelma on alkanut ja mitä järjestelmässä tapahtui samaan aikaan.
Jos esimerkiksi palvelin toimii hitaasti, ylläpitäjä voi tarkistaa CPU ja muistin käyttöasteen sekä levytilan tilanteen. 
Jos verkkoyhteydessä on ongelmia, voidaan tarkastella verkkoliikenteen määrää ja mahdollisia virheitä.
Talennettuja mittaustietoja voidaan vertailla ja huomata poikkeamia järjestelmässä.


6.Mitä opin tehtävän aikana?

Tehtävän aikana opin, miten Prometheus, Node exporter ja Grafana toimivat yhdessä. 
Asensin Node Exporterin Ansible-playbookin avulla ja tarkistin sen toiminnan. Aluksi asennus epäonnistui, koska SSH-palvelut eivät olleet käynnissä kohdekonteissa. 
Kun käynnistin SSH-palvelut, Ansible-asennus onnistui.
Tehtävä auttoi ymmärtämään, miten järjestelmien monitorointi toimii käytännössä. Opin asentamaan ja tarkistamaan Node Exporterin toiminnan, 
käyttämään Prometheusta ja rakentamaan Grafanaan mittareita näyttäviä dashboardeja.
Erityisen hyödyllistä oli kuormitustesti, koska sen avulla näin konkreettisesti, miten järjestelmän suorituskyvyn muutokset näkyvät monitorointitiedoissa.
Opin myös käyttämään PromQl kyselyitä.
Prometheuksen vahvuuksia ovat monipuolinen mittaustietojen kerääminen, aikasarjatietojen tallentaminen ja PromQL-kyselykieli.
Grafanan avulla mittaustiedot voidaan esittää selkeinä kuvaajina. Prometheus tukee myös automaattista mittareiden keräämistä ja hälytysten määrittämistä.
Prometheus ja Grafana muodostavat monipuolisen ratkaisun järjestelmien valvontaan, mutta niiden tehokas käyttö 
edellyttää oikeiden mittareiden valintaa ja tulosten ymmärtämistä.
