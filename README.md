U projektu su korišteni vatrozidi [Wallarm API Firewall](https://github.com/wallarm/api-firewall) i [Open-appsec](https://github.com/openappsec/open-appsec-npm). Za XS2A API poslužila je implementacija [Adorsys Core XS2A](https://github.com/adorsys/xs2a).

# Instalacija Adorsys Core XS2A

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

## Wallarm API Firewall sa modulom za ModSecurity pravila

U projektu korišten je [OWASP CoreRuleSet](https://github.com/coreruleset/coreruleset) skup ModSecurity pravila. Za korištenje modula potrebno je prethodno navedenoj konfiguraciji vatrozida dodati još dvije varijable okruženja.

- U varijabli `APIFW_MODSEC_CONF_FILES` navode se putanje konfiguracijskih datoteka za sam modul, odvojene znakom `;`
- Korištene su dvije konfiguracijske datoteke:
	- [Preporučena konfiguracijska datoteka Coraza biblioteke](https://github.com/corazawaf/coraza/blob/main/coraza.conf-recommended) u kojoj je potrebno izvršiti promjenu u `SecRuleEngine` direktivi koja je početno postavljena na `DetectionOnly`, a potrebno ju je postaviti na `On` kako bi modul blokirao zahtjeve
	- [Demonstracijska konfiguracijska datoteka CRS projekta](https://github.com/coreruleset/coreruleset/blob/main/crs-setup.conf.example) unutar koje je potrebno dopustiti inicijalno blokirane HTTP metode PUT i DELETE korištene od PSD2 API-ja; potrebno je u dokumentu potražiti liniju `setvar:'tx.allowed_methods=GET HEAD POST OPTIONS'`, dodati `PUT` i `DELETE` te otkomentirati pravilo
	-
- `APIFW_MODSEC_RULES_DIR` sadrži putanju na direktorij sa ModSec pravilima; korištena su [OWASP CRS pravila](https://github.com/coreruleset/coreruleset/tree/main/rules)


