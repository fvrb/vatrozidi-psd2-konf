U projektu su korišteni vatrozidi [Wallarm API Firewall](https://github.com/wallarm/api-firewall) i [Open-appsec](https://github.com/openappsec/open-appsec-npm). Za XS2A API poslužila je implementacija [Adorsys Core XS2A](https://github.com/adorsys/xs2a).

## Instalacija Adorsys Core XS2A

Za instalaciju Adorsys Core XS2A API-ja potrebno je upisati sljedeće naredbe:
```bash
$ git clone https://github.com/adorsys/xs2a.git
$ cd xs2a
$ mvn clean install
```
XS2A se pokreće sa naredbom `docker-compose up`. U repozitoriju su dostupne datoteke koje originalnoj Docker Compose konfiguraciji XS2A API-ja dodaju konfiguracijske podatke vatrozida za istovremeno podizanje API-ja i vatrozida.
> **Note:** Za instalaciju je potrebna Java 11.

## Wallarm API Firewall

Za pokretanje Wallarm API Firewalla koristi se datoteka `docker-compose_wallarm.yml`.
Vatrozid se konfigurira preko Docker varijabli okruženja unutar navedene datoteke. 
Slijedi objašnjenje i upute za samostalno uređivanje Docker Compose konfiguracije.

- Vatrozid i API postavljeni su na istu Docker mrežu `xs2a-net` definiranu u `networks` polju svih kontejnera.
- Vrijednost `8081:8080` u polju `ports` označava da je kontejner na računalu domaćin dostupan na vratima `8081` koja se preslikavaju na vrata `8080` kontejnera
- OpenAPI specifikaciju PSD2 koja će biti korištena moguće je preuzeti sa stranica [Berlin Grupe](https://www.berlin-group.org/nextgenpsd2-downloads)
- Konfiguraciju je na računalu domaćinu unutar mape XS2A API-ja potrebno postaviti na putanju koja se dijeli sa kontejnerom; u konfiguraciji preslikavanje mape sa računala domaćina u datotečni sustav kontejnera definira se u `volumes` polju (prema trenutnoj vrijednosti `/volumes/api-firewall` se preslikava na `/opt/resources/api-firewall`.
- Putanja OpenAPI specifikacije iz datotečnog sustava kontejnera pridružuje se `APIFW_API_SPECS` varijabli okruženja kontejnera, koje su navedene pod `environment` poljem
- Varijabla `APIFW_SERVER_URL` sadrži adresu i vrata API-ja (http://xs2a-standalone-starter:8080); za adresu je moguće koristiti naziv kontejnera (xs2a-standalone-starter) jer se kontejneri nalaze na istoj Docker mreži 
- Vrijednost varijable `APIFW_SERVER_MAX_CONNS_PER_HOST` označava maksimalan broj veza koje će vatrozid istovremeno držati otvorenima prema API-ju; postavljanje ove vrijednosti može biti važno u spriječavanju gubitka resursa; početna vrijednost je 512
- Varijable `APIFW_READ_TIMEOUT` i `APIFW_WRITE_TIMEOUT` postavljaju maksimalno dopušteno vrijeme potrebno vatrozidu za čitanje klijentovog zahtjeva i vrijeme za slanje odgovora natrag klijentu nakon što je vatrozid zaprimio odgovor API-ja; obje vrijednosti početno su postavljene na 10 sekundi
- `APIFW_SERVER_DIAL_TIMEOUT` postavlja maksimalno dopušteno vrijeme za spajanje vatrozida na API
- Vrijednost varijable `APIFW_REQUEST_VALIDATION` određuje hoće li vatrozid blokirati maliciozne dolazne zahtjeve (vrijednost `BLOCK`), bilježiti ih bez blokiranja (`LOG_ONLY`) ili ne provoditi provjeru dolaznih zahtjeva (`DISABLE`)
- Analogno vrijedi za varijablu `APIFW_RESPONSE_VALIDATION` koja određuje način rada za zahtjeve koji pristižu vatrozidu kao API-jev odgovor

Zahtjevima koji su podložni ranjivosti Mass Assignment predlaže se dopunjavanje odgovarajućeg zapisa u OpenAPI specifikaciji sa parametrom `additionalProperties` postavljenim na `false`.

## Wallarm API Firewall sa modulom za ModSecurity pravila

U projektu korišten je [OWASP CoreRuleSet](https://github.com/coreruleset/coreruleset) skup ModSecurity pravila. Za korištenje modula potrebno je prethodno navedenoj konfiguraciji vatrozida dodati još dvije varijable okruženja. Gotova konfiguracija jedostupna u repozitoriju pod imenom `docker-compose_appsec_local.yml`.

- U varijabli `APIFW_MODSEC_CONF_FILES` navode se putanje konfiguracijskih datoteka za sam modul, odvojene znakom `;`
- Korištene su dvije konfiguracijske datoteke:
	- [Preporučena konfiguracijska datoteka Coraza biblioteke](https://github.com/corazawaf/coraza/blob/main/coraza.conf-recommended) u kojoj je potrebno izvršiti promjenu u `SecRuleEngine` direktivi koja je početno postavljena na `DetectionOnly`, a potrebno ju je postaviti na `On` kako bi modul blokirao zahtjeve
	- [Demonstracijska konfiguracijska datoteka CRS projekta](https://github.com/coreruleset/coreruleset/blob/main/crs-setup.conf.example) unutar koje je potrebno dopustiti inicijalno blokirane HTTP metode PUT i DELETE korištene od PSD2 API-ja; potrebno je u dokumentu potražiti liniju `setvar:'tx.allowed_methods=GET HEAD POST OPTIONS'`, dodati `PUT` i `DELETE` te otkomentirati pravilo
- `APIFW_MODSEC_RULES_DIR` sadrži putanju na direktorij sa ModSec pravilima; korištena su [OWASP CRS pravila](https://github.com/coreruleset/coreruleset/tree/main/rules)

## Open-appsec sa NPM-om, lokalno upravljanje
Za podizanje Open-appsec vatrozida koristit će se posebno dizajniran [kontejner Nginx Proxy Managera](https://github.com/openappsec/open-appsec-npm) (NPM) koji sadrži Open-appsec dodatak.

U nastavku je opisan proces podizanja Open-appsec vatrozida uključujući podešavanje Nginx posrednika kroz NPM sučelje.

- Unutar XS2A direktorija naredbom `mkdir ./appsec-localconfig` potrebno je kreirati direktorij u koji je potrebno smjestiti [inicijalnu lokalnu konfiguraciju](https://github.com/openappsec/open-appsec-npm/blob/main/deployment/local_policy.yaml)
- U XS2A direktorij postaviti Docker Compose konfiguraciju dostupnu u ovom repozitoriju pod imenom `docker-compose_appsec_local.yml` i pokrenuti
- NPM sučelje za upravljanje posrednicima dostupno je na adresi `http://[hostname or IP of your host]:81`
- Za inicijalnu prijavu u sustav potrebni su sljedeći podaci
	- E-mail address: admin@example.com
	- Password: changeme
- Potrebno je kreirati novi Proxy Host i u kreacijski obrazac unijeti sljedeće podatke:
	- Domain Names: localhost
	- Scheme: http
	- Forward Hostname / IP: xs2a-standalone-starter
	- Forward Port: 8080
- U ovako postavljenom sustavu, konfiguraciju vatrozida je moguće izmijenjivati samo uređivanjem lokalne deklarativne konfiguracije na putanji `./appsec-localconfig/local_policy.yaml`

## Open-appsec sa NPM-om, središnje upravljanje
Ovako konfiguriran sustav koristit će Web UI za nadgledanje i upravljanje vatrozidom. 

- Za podizanje koristi se Docker Compose konfiguracija `docker-compose_appsec_webui.yml` iz ovog repozitorija
- Na kraju konfiguracije nalazi se mjesto za upis tokena preuzetog sa web sučelja koje služi za prijavu agenta u središnji sustav
- Za dobivanje tokena potrebno je prijaviti se u [sustav](https://my.openappsec.io/), otići na Profiles karticu, kreirati novi profil i u polju Sub-type odabrati: Dual Container - NGINX Proxy Manager + open-appsec
- Sada je potrebno kreirati Nginx posrednika kroz NPM sučelje na jednak način kao što je opisano u prethodnoj konfiguraciji
-  Nakon aktivacije posrednika potrebno je aktivirati zaštitu nad njime, na kartici Assets u web sučelju potrebno je kreirati novi resurs unutar kojeg je u polju Profiles potrebno odabrati prethodno kreirani profil te u API URLs polju dodati vrijednost `http://localhost:8081`.

## Konfiguracija zaštite Open-appsec vatrozidom
Za izvršavanje sljedećih izmjena potrebno je odabrati odgovarajući resurs u Assets kartici. U novootvorenom sučelju potrebno je podesiti sljedeće stavke:
- U Threat Protection kartici, pod Advanced dostupna je opcija `Non-valid HTTP Methods` koja postavljena na `Yes` spriječava zahtjeve sa HTTP metodama van sljedećeg skupa: `GET, POST, DELETE, PATCH, PUT, CONNECT, OPTIONS, HEAD, TRACE`
- Na kartici Rate Limit postaviti `Mode` na `Prevent` te kreirati novo pravilo za frekvenciju zahtjeva:
	- URI postaviti na `/` kako bi pravilo vrijedilo za sve krajnje točke API-ja
	- Frekvenciju postaviti na 100 zahtjeva po sekundi
- Nakon što model strojnog učenja u sklopu vatrozida dostigne dovoljnu razinu utreniranosti, na Threat Protection kartici potrebno je `Mode` postaviti na `Prevent` kako bi detektirani maliciozni zahtjevi bili blokirani
