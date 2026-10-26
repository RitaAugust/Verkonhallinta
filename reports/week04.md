1.Johdanto

Infrastructure as Code (IaC) tarkoittaa IT-infrastruktuurin hallintaa ja konfigurointia koodin ja määritystiedostojen avulla 
manuaalisen työskentelyn sijaan. Sen avulla palvelimien asetukset ja ohjelmistojen asennukset voidaan automatisoida ja 
toistaa luotettavasti.

Tässä työssä käytetään Ansiblea infrastruktuurin automatisointiin. Ansible-playbookit määrittelevät suoritettavat tehtävät, 
kuten ohjelmistojen asentamisen ja palvelimien konfiguroinnin. Automaatio vähentää manuaalista työtä, parantaa ympäristön 
yhdenmukaisuutta ja helpottaa muutosten hallintaa Git-versionhallinnan avulla.

2.Tutustuminen inventoryyn

Ansible-palvelimella tarkistin inventory cat /ansible/inventory.ini komennolla. 

Inventorysta löytyvät ympäristön laitteet ja niiden tiedot, kuten . Inventory sisältää reitittimiä, asiakastyöasemia, 
palvelimia, valvontajärjestelmiä ja hallinta. Sen avulla Ansible tietää, mihin laitteisiin tehtävät suoritetaan 
ja miten niihin muodostetaan yhteys.

Inventory on jaettu seuraaviin ryhmiin:
routers: reitittimet r1, r2 ja r3. Ryhmälle on määritelty verkkolaitteiden hallintaan käytettävä network_cli-yhteys.
clients: asiakastyöasemat client1, attacker ja branch-client. Ryhmän yhteystapana käytetään SSH.
servers: web1 ja db1. Molempiin muodostetaan SSH-yhteys.
monitoring infrastructure: Prometheus-, Grafana-, Zabbix- ja cAdvisor-valvontapalvelut.
management tools: Ansible.
Laitteille on määritelty ansible_host muuttuja, jonka avulla Ansible löytää oikean Containerlab kontin. 
Osalla laitteista on myös data_ip muuttuja, joka kuvaa laitteen dataverkon IP-osoitetta. 
Lisäksi inventory sisältää loogisia ryhmiä, kuten user_network, server_network ja branch_office. 
Infrastucture ryhmät network_devices ja linux_hosts muodostetaan aliryhmissä. Ryhmä Node_exporter targets
sisältää reitittimet, asiakastyöasemat ja palvelimet. 
 
Ryhmittely helpottaa automaatiota, koska playbook voidaan kohdistaa tiettyyn laiteluokkaan yksittäisten laitteiden sijaan. 
Esimerkiksi tehtävä voidaan suorittaa vain palvelimille käyttämällä servers ryhmää tai valvonnan kohteille käyttämällä 
node_exporter ryhmää. Ryhmäkohtaiset muuttujat vähentävät toistuvia asetuksia ja helpottavat inventoryn ylläpitoa.

Tarkistin yhteydet komennolla ansible all -i /ansible/inventory.ini -m ping. 

Reitittimet r1, r2 ja r3 vastasivat onnistuneesti. Muut testatut kohteet palauttivat UNREACHABLE-tuloksen, koska 
SSH-yhteys kohdepalvelimien porttiin 22 hylättiin.


3.Suorita ensimmäinen playbook

Suoritin ansible-playbook -i ../inventory.ini ping.yml komento hekemistossa ansible/playbooks. 

Playbookin ensimmäinen osuus kohdistuu linux_hosts ryhmään ja testaa yhteydet. Asetus gather_facts: false 
estää järjestelmätietojen automaattisen keräämisen ennen tehtävien suorittamista.

Toinen osuus kohdistuu routers-ryhmään. Se suorittaa reitittimillä komennon show version ja tallentaa tuloksen 
result-muuttujaan. Tämän jälkeen debug-tehtävä tulostaa reitittimen nimen ja ilmoituksen yhteyden toimivuudesta.

Suorituksessa kaikki kymmenen Linux-ryhmän kohdetta palauttivat UNREACHABLE-tuloksen. 
Virheilmoitusten mukaan yhteys kohteiden SSH-porttiin 22 hylättiin. Tulosteessa ei näy reitittimiä käsittelevän 
paly-osuuden suoritusta.

4.Tutustu valmiisiin esimerkkeihin

install-snmp playbook

SNMP-playbook asentaa SNMP-paketit, määrittää SNMP-agentin asetukset ja käynnistää palvelun. 
Playbook käyttää apt-moduulia pakettivaraston päivittämiseen ja pakettien asentamiseen, 
copy-moduulia asetustiedoston luomiseen, service-moduulia palvelun hallintaan, shell-moduulia prosessin tarkistamiseen 
sekä debug-moduulia tuloksen tulostamiseen.
Muuttuja snmp_community määritellään vars-osiossa ja sitä käytetään SNMP-asetustiedoston sisällössä. 
Näin arvoa voidaan muuttaa yhdestä paikasta.
copy-tehtävä ilmoittaa notify: restart snmpd -määrityksellä handlerille, jos asetustiedosto muuttuu. 
handlers-osiossa oleva restart snmpd käynnistää SNMP-palvelun uudelleen, jotta uudet asetukset tulevat käyttöön. 

SNMP-pakettien asennus ja konfigurointi onnistuivat kaikilla neljällä kohteella, mutta palvelun käynnistäminen 
automaattisesti ei onnistunut. Manuaalinen testaus vahvisti SNMP toiminnan web1:lla.

install-node-exporter playbook

Node Exporter -playbook asentaa Prometheus Node Exporterin, käynnistää sen ja tarkistaa, että mittarit 
ovat saatavilla HTTP kautta.
Playbook käyttää apt-moduulia riippuvuuksien asentamiseen, file-moduulia asennushakemiston luomiseen, 
get_url-moduulia ohjelmistopaketin lataamiseen, unarchive-moduulia paketin purkamiseen, copy-moduulia ohjelmatiedoston 
siirtämiseen, shell-moduulia prosessin käynnistämiseen ja uri-moduulia HTTP testaamiseen. 
debug-moduuli tulostaa onnistumisilmoituksen.
node_exporter_version määrittää asennettavan version.
Node Exporter playbookissa ei ole handleria, koska käynnistys ja tarkistus suoritetaan suoraan. 
Playbook ei käytä notify-määritystä eikä erillistä handleria palvelun uudelleenkäynnistykseen. 
Sen sijaan se pysäyttää mahdollisen aiemman prosessin ja käynnistää Node Exporterin uudelleen 

Node Exporter -playbook suoritettiin onnistuneesti kaikilla neljällä kohteella: web1, db1, client1 ja branch-client. 
Kaikki tehtävät valmistuivat ilman virheitä (failed=0, unreachable=0). 



5.Oma playbook: install-webserver.yml 

Tehtävän tavoitteena on luoda oma Ansible-playbook, joka asentaa ja konfiguroi Nginx web palvelimen web1:lle.
Loin install-webserver.yml-playbookin, joka päivittää pakettilistan, asentaa Nginxin ja luo /var/www/html/index.html-sivun. 
Sivulla näkyy palvelimen nimi web1.
Playbookissa käytetään apt-, copy-, command-, shell-, uri- ja debug-moduuleja.
Nginxin käynnistämiseen kokeiltiin ensin Ansible service moduulia asetuksilla state: started ja enabled: 
yes, mutta suoritus epäonnistui virheeseen Service is in unknown state. En löytänyt syytä. 
Sen takia Nginxin käynnistys toteutettiin shell-moduulilla ja perinteisen käynnistysjärjestelmän asetukset 
määritettiin komennolla update-rc.d nginx defaults.
Playbook suoritettiin onnistuneesti web1-kohteella. Kaikki seitsemän tehtävää valmistuivat ilman virheitä.
web1 : ok=7 changed=0 unreachable=0 failed=0 (tehty kuvakaappaus)

6.Kerää järjestelmätietoja

Keräsin järjestelmän tietoja komennon ansible all -i ../inventory.ini -m setup tulosteesta. 

web1: 		 käyttöjärjestelmä: Ubuntu 24.04, 	 IP-osoite: 172.20.20.6,	 Prosessorit: 12,	 Muisti (MB): 7608.
db1:     	 käyttöjärjestelmä: Ubuntu 24.04,         IP-osoite: 172.20.20.7,          Prosessorit: 12,         Muisti (MB): 7608.
client1: 	 käyttöjärjestelmä: Ubuntu 24.04,         IP-osoite: 172.20.20.3,          Prosessorit: 12,         Muisti (MB): 7608.
branch-client:    käyttöjärjestelmä: Ubuntu 24.04,         IP-osoite: 172.20.20.5,          Prosessorit: 12,         Muisti (MB): 7608.
r1:               käyttöjärjestelmä: Ubuntu 24.04,         IP-osoite: 172.20.20.15,          Prosessorit: 12,         Muisti (MB): 7608.
r2:            käyttöjärjestelmä: Ubuntu 24.04,         IP-osoite: 172.20.20.16,         Prosessorit: 12,         Muisti (MB): 7608.
r3:            käyttöjärjestelmä: Ubuntu 24.04,         IP-osoite: 172.20.20.14,         Prosessorit: 12,         Muisti (MB): 7608.

Reitittimien IP-osoitteet tarkistettiin erikseen suoraan konteista, koska Ansible setup -moduulin tuloksia ei voitu pitää niiden osalta luotettavina.
Ansible setup -moduuli ilmoitti kaikille kolmelle reitittimelle saman IP-osoitteen 172.20.20.8. Tulos ei vastannut suoraan konteista tehtyä tarkistusta. 

6.Analyysi

Käsin tehtävässä asennuksessa ohjelmat ja asetukset määritetään palvelimelle yksitellen. Tämä voi olla toimiva ratkaisu pienessä ympäristössä, 
mutta useiden palvelimien hallinta vie aikaa ja lisää virheiden mahdollisuutta.
Lisäksi samojen asetusten tekeminen eri palvelimille voi johtaa eroihin. 
Ansiblella asennukset ja konfiguroinnit määritellään playbookeihin, joiden avulla tehtävät voidaan suorittaa automaattisesti useille palvelimille. 
Tässä tehtävässä Ansiblea käytettiin esimerkiksi Node Exporterin asentamiseen ja Nginx-web-palvelimen määrittämiseen. 
Nginx-playbookilla asennettiin web-palvelin web1-palvelimelle, luotiin HTML-sivu ja varmistettiin, että palvelin palautti HTTP 200 -vastauksen.

Automaation hyödyt:
Nopeus: Samat tehtävät voidaan suorittaa useilla palvelimilla ilman, että jokainen asennus tehdään erikseen.
Yhdenmukaisuus: Playbookit auttavat pitämään palvelimien asetukset samanlaisina.
Vähemmän virheitä: Toistuvat työvaiheet suoritetaan määritellyllä tavalla, mikä vähentää manuaalisten virheiden mahdollisuutta.
Toistettavuus: Sama playbook voidaan suorittaa uudelleen, kun ympäristö pitää asentaa tai määrittää uudelleen.
Dokumentointi ja versionhallinta: Playbookien sisältöä voidaan tarkastella ja muutoksia seurata Git-versionhallinnan avulla.

Automaatio on erityisen tärkeää silloin, kun hallittavia palvelimia on paljon, samoja asennuksia tehdään toistuvasti tai ympäristö pitää pystyä 
rakentamaan uudelleen luotettavasti. Se on hyödyllistä myös tuotantoympäristöissä, joissa palvelimien asetusten yhdenmukaisuus 
ja muutosten jäljitettävyys ovat tärkeitä.
Suurissa ympäristöissä manuaalinen asennus olisi hidasta ja altistaisi järjestelmän erilaisille konfiguraatiovirheille. 
Ansible mahdollistaa samojen tehtävien suorittamisen hallitusti useille kohteille. 
Käsin tehtävä asennus sopii yksittäisiin ja yksinkertaisiin tehtäviin, kun taas Ansible on tehokkaampi ratkaisu toistuviin 
ja useita palvelimia koskeviin asennuksiin. 
