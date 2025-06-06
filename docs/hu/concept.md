## OpenIntentSign – koncepció

Az **OpenIntentSign (OIS)** egy nyílt, tartalomfüggő érvényesítési modell, amelynek célja, hogy biztosítsa a rendszerek deklarált állapotának és a valós működésnek az összhangját.

Az OIS alap igénye, hogy a **szolgáltatást igénylő** és a **szolgáltatást nyújtó** entitás közötti adatáramlás:

* titkosított legyen,
* a szolgáltatást igénylő entitás jogosultsága hitelesen ellenőrzésre kerüljön,
* és mindeközben a közvetítő vagy irányító rendszerek **ne rejtsék el** a döntési logikát vagy az adatfolyamot.

Ez a modell nem egy protokoll, nem egy alkalmazás – hanem egy **filozófia és technikai keret**, amely lehetővé teszi a bizalomra épülő rendszerek működtetését, ahol minden komponens felelősséget vállal a saját állapotának igazolhatóságáért.

---

### Követelmények az OIS felé

#### Igény benyújtás:

* Minden igény egyediségét biztosítani kell (nem lehet újrafelhasználható vagy másolható formátum).
* Az aszinkron működés alapértelmezett és elvárt.
* Az útvonalválasztáshoz és szeparáláshoz szükséges csomagszintű metaadatok **publikusak**, ezek nem tartalmazhatnak bizalmas információt.
* A teljes megvalósítás **layer 7 (alkalmazási réteg)** szinten történik.
* Mind az **igény** (intent), mind az **adat** titkosított, és kizárólag az alkalmazások oldják fel azokat.
* Minden igényt külső kapcsolattól függetlenül kell elbírálni – azaz az érvényesítéshez nem szükséges online elérhetőség vagy külső rendszer.

#### Válasz/visszirány:

* Minden visszaadott adatcsomagnak egyedileg azonosíthatónak kell lennie.
* A válaszhoz tartozó titkosítási kulcsok **egyszer használatosak**, újrahasználat nem megengedett.

#### Haladó követelmények:

* **Auditálhatóság:** minden döntési folyamat rekonstruálható legyen; a validálási eredmények naplózottak és exportálhatók.
* **Verifikálhatóság:** harmadik fél számára is igazolható legyen egy igény vagy válasz érvényessége, a tartalom feltárása nélkül (pl. kriptográfiai lenyomatok).
* **Intentszintű naplózás:** minden igényhez külön, időbélyeggel és állapotjelzéssel ellátott napló tartozik.
* **Átlátható döntési logika:** a jogosultság-ellenőrzési szabályok verziózottak, auditálhatók, nem rejtettek.
* **Tartalomfüggő jogosultsági modell:** a hozzáférés elbírálása nemcsak az entitás, hanem az igényelt adatok jellege és környezete alapján történik.

---

### Lehetséges megvalósítási irány

Az alábbiakban egy javasolt, technológiától független implementációs vázlat szerepel, amely segíthet az OIS gyakorlati bevezetésében:

* **Csomagszerkezet:**

    * Az igény csomag három komponensből áll: `meta` (publikus routing adat), `intent` (titkosított kérés) és `payload` (titkosított adattartalom).
    * A `meta` komponens mindig nyilvános, és az útvonalválasztást segíti (pl. cél azonosító, érvényességi idősáv, csatorna jelleg, stb.).

* **Kulcskezelés:**

    * Minden igényhez aszimmetrikus kulcspárt generál a kliens.
    * A kliens által generált publikus kulcs nemcsak a titkosításhoz használható, hanem **a jogosultság-ellenőrzéshez szükséges metaadatokat is tartalmazza**.
    * A szolgáltató csak akkor fér hozzá a titkosított tartalomhoz, ha a jogosultsági érvényesítés sikeres.
    * A válaszkulcs mindig **egyszer használatos**, azaz session-szintű ephemeral kulcs.

* **Jogosultságkezelés:**

    * A jogosultsági információk közvetlenül a kulcs metaadataiból olvashatók ki, nem szükséges külön lekérdezni vagy betölteni őket.
    * A döntés logikája kriptográfiailag aláírt – vagyis a jogosultság-ellenőrzés később is visszaellenőrizhető.
    * A szabályverziók auditálhatók, exportálhatók, és nem rejtettek.

* **Offline döntési modell:**

    * A szolgáltató validációs komponense nem igényel központi kapcsolatot.
    * A szükséges érvényesítési adatok (pl. jogosultságlenyomatok, verifikációs táblák) előre teríthetők.

* **Adatvisszairány:**

    * A válasz szintén három komponensből áll: `meta`, `status` (pl. OK / DENY / REDIRECT), `payload` (titkosított válasz).
    * A visszairányban használt kulcs csak egyszer dekódolhat.

Ez az architektúra alkalmas akár gép-gép kommunikációra, akár emberek által triggerelt szolgáltatásigények feldolgozására is, és könnyen illeszthető decentralizált vagy edge környezetekbe.

---

### Konkrét példa

#### Példa: Egyszerű adatlekérés

(Ez a példa a kulcsgenerálás és a hitelesítés szempontjából is bemutatja a működést – feltételezzük, hogy a kliens rendelkezik egy alapvető gyökérkulccsal vagy tanúsítvánnyal, amelyből új, session-alapú kulcsokat tud hitelesen generálni vagy kérni.)

1. **Igénylő előkészítése:**

    * A kliens (DataRequester) generál egy új, egyszer használatos kulcspárt (pl. RSA),
      amelyet vagy:

        * a saját gyökérkulcsával ír alá, vagy
        * egy hitelesítő szolgáltatás (CA) által aláírat.
    * A publikus kulcsba beágyazza a jogosultságait leíró metaadatokat (pl. szerepkör, elérési szint, időbélyeg, cél).
    * A publikus kulcsba beágyazza a jogosultságait leíró metaadatokat (pl. szerepkör, elérési szint, időbélyeg, cél).

2. **Kérés összeállítása:**

    * A kérés három részből áll:

        * `meta`: nem titkosított adat, pl. célazonosító, időzóna, kérés típusa.
        * `data`: a tényleges kérés (pl. „Add vissza az XY felhasználó státuszát”), amely:

            * titkosítva van a szolgáltató nyilvános kulcsával,
            * és **alá van írva** az igénylő frissen generált privát kulcsával.
        * `pubkey`: az igénylő publikus kulcsa, jogosultságmeta-információkkal.

3. **Szolgáltató feldolgozása:**

    * A rendszer az igény `pubkey` mezőjéből kinyeri a jogosultságinformációkat, és összeveti az adott kérés kontextusával.
    * Amennyiben érvényes, dekódolja a `data` mezőt,

        * és **visszaellenőrzi az adatok integritását** az aláírással,
        * majd végrehajtja a lekérdezést. végrehajtja a lekérdezést.

4. **Válasz küldése:**

    * A szolgáltató az eredményt titkosítja az igénylő által megadott publikus kulccsal (azaz csak az igénylő tudja visszafejteni).
    * A válasz alá van írva a szolgáltató privát kulcsával, így az igénylő hitelesíteni tudja a forrást és az adatok integritását.
    * A válasz ismét három mezőből áll:

        * `meta`: válaszcímkék, állapot, stb.
        * `status`: például OK vagy DENY.
        * `payload`: titkosított válaszadat.

Ez a működésmód garantálja, hogy az igénylő és a szolgáltató kölcsönösen nem bíznak vakon egymásban, mégis működő, auditálható, kriptográfiailag biztosított kapcsolat jön létre.

### Felmerült, megválaszolandó kérdések 

1. **Kulcs visszavonási és érvényességi mechanizmusok**
   A rendszerben használt kulcsok érvényessége és biztonsága kiemelt kérdés. Ha egy privát kulcs kompromittálódik, annak használata biztonsági kockázatot jelenthet. Ezért szükséges:

    * Kulcsonként időbélyeget és lejárati időt tárolni.
    * Egy kulcs érvényességi idejét a metaadatok tartalmazzák, ezzel jelezve, hogy az adott kulcs milyen időintervallumban használható.
    * Szükség lehet egy **helyi CRL** (Certificate Revocation List) mechanizmusra vagy OCSP-szerű lekérdezési módszerre, még akkor is, ha a rendszer offline működésre is képes – ez utóbbit például **előre leszinkronizált revocation-listával** lehet megoldani.

2. **Több entitásos (delegált) működés**
   Ha a válaszadást nem maga a végső szolgáltató, hanem egy proxy végzi, biztosítani kell az eredeti entitás hitelességét:

    * A válasz aláírását a delegált szolgáltató végzi, de a proxy szerepét metaadatokban (pl. `delegator`, `acting-on-behalf-of`) fel kell tüntetni.
    * Lehetőség van **delegációs lánc** kiépítésére, ahol minden résztvevő fél aláírja a lekérdezést vagy választ, vagy a lánc első és utolsó tagja kriptográfiailag hitelesíti a kapcsolatot.

3. **Válasz tárolhatósága és cache-elhetősége**
   Az OIS alapvetően dinamikus, egyedi igényekre válaszoló modell, de vannak esetek, amikor a válasz cache-elése indokolt:

    * A válasz `meta` mezőjében szerepelhet egy TTL (Time To Live) érték, ami azt jelzi, meddig tekinthető érvényesnek a válasz.
    * A `signature` és a `timestamp` együtt biztosítja, hogy egy válasz újrafelhasználása ne járjon kockázattal – amennyiben a célrendszer validálja az aláírás frissességét és érvényességét.

4. **Integrációs interfészek**
   Az OIS célja, hogy alkalmazásrétegű és protokollfüggetlen legyen. Ezért:

    * REST: jól alkalmazható, különösen mikroszolgáltatásos architektúrákban.
    * gRPC: nagy teljesítményű, bináris-alapú megoldás, ami strukturált schema-támogatással bír.
    * MQTT: eseményvezérelt rendszerekben vagy edge-eszközöknél ajánlott.
    * Adapterek segítségével a titkosítás és az aláírás beilleszthető bármelyik fenti rétegbe, akár `middleware` vagy `gateway` szinten is.

5. **Tartalomfüggő szabálymotor lehetősége**
   Az egyszerű jogosultságellenőrzés helyett az OIS támogatja a **tartalomfüggő döntéshozatalt** is:

    * A metaadatok vagy a dekódolt `data` mező alapján futtatható egy logikai szabályrendszer, amely figyelembe veszi az **időpontot**, a **kérelmezett adattípust**, a **rendszerterheltséget**, vagy akár a **geolokációt** is.
    * Ezzel finomhangolható, hogy mikor, milyen feltételek mellett engedélyezhető egy lekérdezés – például: ugyanaz az entitás egyes napokon vagy műszakokban más jogosultságokat élvez.

---

### Nyílt modell

Az OIS nyílt szellemi termék, nem kereskedelmi célú újrafelhasználása engedélyezett.
A licenc: **Creative Commons BY-NC-SA 4.0**

> A modell szerzője: **Sinkó Gábor Zoltán**
> Kapcsolat: [GitHub - OpenIntentSign](https://github.com/OpenIntentSign)
