<h5>Rapport skriven av Johanna Szepanski js223kz</h5>


<h1>Optimeringsförslag i applikationen https://github.com/thajo/messy_labbage</h1>

<p>Det finns en del utrymme för optimering av applikationens prestanda.
Jag har i första hand tittat på hur applikationen kan optimeras frontend
och det med relativt enkla åtgärder. Säkerhetsapekten är såklart den viktigaste,
men för användarupplevelsen rekommenderar jag att man genomför så mycket som
möjligt av nedanstående åtgärder. Primärt rör det sig om laddningstider [1]-[2].
Då jag inte har någon uppfattning om antalet användare av applikationen
kan jag inte uttala mig om miskade kostnader i utnyttjandet av bandbredd,
men vid ett stort antal användare blir det en trevlig bonus.</p>


<h3>Minska antalet Http-anrop</h3>
Ju fler förfrågningar som skickas från browsern till webbservern ju längre tid ta sidan att laddas.
Faktum är att enligt undersökningar står HTML-dokumentet endast av 10-20% av laddningstiden för slutanvändaren.
Alla andra komponenter som t ex bilder, css- och skriptfiler, som det refereras till i HTML-dokumentet står
för resterande 80-90%, [1, s.1], [3]. Längre laddningstider innebär sämre användarupplevelse.
Det finns flera sätt att minska på antalet Http-anrop. Nedan listar jag några som rör just den här applikationen.


<h5>Ta bort oanvända resurser</h5>
En bild som laddas, men aldrig syns, b.jpeg, skapar onödig laddningstid.<br>
Browsern försöker ladda, men servern skickar tillbaka en 404:a, materialize.js, skapar onödig laddningstid.<br>
Filer som inte laddas, men som heller inte används, clock.png, delete.png, favicon.png, dump.html.<br>

<b>Åtgärd:</b>
Rensa bort outnyttjade resurser som förlänger laddningstiderna och "döda filer" som inte används.


<h5>Cacha komponenter [4]</h5>
Första gången en användare besöker en sida laddas alla komponenter in och laddningstiden blir längre. Hur lång den
sedan blir vid efterföljande besök på samma sida beror på huruvida komponenterna cachas eller inte. Dynamiska HTML-sidor
cachar man inte, men bilder, css- och skriptfiler går alldeles utmärkt att cacha om de inte ändras stup i kvarten.
Gör en utvärdering av komponenterna som laddas, har de inte ett närliggande "last modified date" är de säkert kandidater för
cachning [4, s.22-27].

Som utvecklare kan man säkerställa att komponenter sparas i browserns cache genom att lägga till en Future Expires Header.
Future Expires Header använder ett specifikt datum för att hålla koll på när en komponent måste laddas om. Det innebär
att när det datumet löper ut måste servern tillhandahålla ett nytt datum [4, s.22-23].<br>
Ett smidigare sätt är då att använda Cache Control och Max Age, som bestämmer hur länge en komponent får cachas i sekunder.
De browsers som inte stödjer HTTP/1.1 kan inte använda Cache Control, men då det är väldigt få bör det inte ställa till
några problem [4, s.22-24].

Använder man inte Future Expires kommer browsern ändå att spara resursen i sin cache och sedan när resursen efterfrågas
skicka en sk Conditional GET till servern. Om komponenten inte har ändrats skickar servern tillbaka ett svar till browsern
att använda den komponent den har i sin cache. Med andra ord en onödig tripp till servern som förlänger sidans laddningstid, [2], [4, s.28].

I denna applikation är alla komponenters Future Expire satt till -1, vilket säger till browsern att inte cacha något, [5, s.39].

<b>Åtgärd:</b>
Eftersom applikationens användare kommer att vara återkommande besökare är cachning en viktigt åtgärd för
användarupplevelsen och jag rekommenderar verkligen att det implementeras [4, s.24-25]. I min bedömning kan alla bilder,
css- och skriptfiler i applikationen cachas. Något man dock bör tänka på när man låter browsers och proxies cacha
komponenter är hur man sköter versionshantering och uppdatering av filer. Läs mer om Revving Filenames [4, s.27].


<h5>Slå ihop filer</h5>
Flera filer betyder flera HTTP-anrop och längre laddningstid. Det betyder inte att man ska trycka in all kod i en fil,
men i den här applikationen hade jag integrerat head.html i Default.html.

<b>Åtgärd:</b>
Integrera head.html i Default.html.


<h3>Minska laddningstider på Http-anrop</h3>


<h5>Komprimera filer, [5]</h5>
Svarstiden från servern är kortare ju mindre storleken är på svaret. Att komprimera HTTP-svaren är ett kraftfullt sätt att
minska laddningstiderna. Det är den lättaste och mest effektiva åtgärden att genomföra, [5, s.29].

Om klienten stödjer kompression av filer visar den det genom att skicka en Accept Encoding header. När servern ser detta
kan den komprimera filerna enligt de metoder klienten stödjer. Mest vanligt och bäst är att använda gzip.
Det går att komprimera alla textsvar inklusive JSON och XML. Bilder och PDF:er bör inte komprimeras då detta redan är
gjort och bara kommer att slösa processorkraft. Då det tar kraft att både komprimera och dekomprimera filer bör man ha som
huvudregel att inte komprimera filer på mindre än 1-2K, [5, s.29-31].

Man bör vara medveten om att komprimeringsprocessen kan bli något mer komplicerad om klienten går via en proxyserver.
Proxyservern cachar komprimerade filer om användare nr 1 stödjer detta. När användare nr 2, som inte stödjer
komprimering skickar en förfrågan via proxyn som svarar den tillbaka med samma komprimerade filer. Inte bra, men går att komma runt
genom att sätta en Vary Header, eller Cache Control Private [5, s.33-24].

<b>Åtgärd:</b>
Man måste som med alla dessa optimeringsåtgärder väga för- och nackdelar mot varandra. Applikationen, som den är nu,
komprimerar inga filer. Jag tycker det är värt ett försök då vinningen kan bli väldigt stor.


<h5>Referenser till CSS-filer ska ligga i HEAD-taggen, [6]</h5>
En browser laddar ett HTML-dokument uppifrån och ner. När css-filer placeras i slutet av Html-dokumentet blockerar
browsern rendering av alla element tills dess att css-filen är laddad. Detta för att slippa rendera om element ifall
css-koden kommer att förändra något av dessa element.Det är inte det att laddningstiden faktiskt är kortare om man placerar
css-filen högst upp, tvärtom, men eftersom inget renderar ut förrän css-filen har laddats upplevs laddningstiden som längre.
Psykologiskt spelar detta roll då en helt, blank, vit sida som inte visar några synliga tecken på att ladda gör användaren frusterad.
I vissa browsers renderas elementen ut ostylade till att börja med, vilket också det ger en dålig användarupplevelse och
badwill för sidägaren, [5, s.37-38].

<b>Åtgärd:</b>
Se filerna message/views/admin.html, message/views/index.Html


<h5>Referenser till skriptfiler ska ligga långt ner i body-taggen, [7]</h5>
Som regel använder sig en browser av det som kallas parallell nerladdning, dvs att den laddar två komponenter i taget.
Är applikationen fördelad över två "hosts" betyder det att fyra komponenter laddas parallellt. När ett skript laddas blockerar
det renderingen av alla komponenter som ligger efter skriptet även om komponenterna ligger på en annan host. Blockeringen sker
därför att skriptet kanske använder sig av document.write för att förändra hur sidan ska renderas. Alltså väntar browsern hellre
på att skriptet laddats klart. Ett annat skäl är att browsern vill försäkra sig om att skripten laddas i rätt ordning med hänsyn
till dependecies och liknande. Det går inte att förlita sig på sk Deferred Scripts.

<b>Åtgärd:</b>
Se filen siteViews/partials/head.html där script-taggar ligger i Head-taggen.


<h5>Lägg styling i externa filer, [8]</h5>
Även om inline styling vanligtvis ger kortare laddningstid än att använda externa filer skulle jag i den här applikationen
rekommendera att bryta ut all styling i en egen fil. På så sätt får man bort all duplicerad kod och gör stylingen lättare
att underhålla. Det ger även den fördelen att css-filerna kan cachas på klienten och behöver således inte laddas in varje
gång en sida efterfrågas efter den första förfrågan [8, 55-57]. Referera till css-filen i HTML-filen som renderar den övergripande
layouten, Default.html

<b>Åtgärd:</b>
Bryt ut css-koden från message/views/admin.html, message/views/index.Html och lägg i en egen css-fil. Jag skulle
också tagit all css från siteViews/css/signin.css och lagt i samma fil. Det är så lite kod att jag inte kan försvara två
HTTP-förfrågningar istället för en. Läs in filen en gång i Default.html i head-taggen.


<h5>Minifiera Javascriptfiler, [9]</h5>
Minifiera alla javascriptfiler med JSMIn och få bort alla kommentarer och onödiga mellanslag.
Är filerna redan komprimerade kommer en minifiering ge ytterligare 20% kortare laddningstid. Bry er inte om att
minimera css-filerna då det inte ger något större resultat. Se hellre till att ta bort död kod som jag har obeserverat
finns lite här och där.


<h3>Referenser</h3>

<P>[1] Steve Souders, The importance of web performance i <i>High Performance Web Sites: Essential Knowledge for Frontend Engineers</i>,
Redaktör, Andy Oram. Sebastopol, California: O'Reilly Media, 2007, 1-5.</p>

<P>[2] Steve Souders, HTTP overview i <i>High Performance Web Sites: Essential Knowledge for Frontend Engineers</i>,
Redaktör, Andy Oram. Sebastopol, California: O'Reilly Media, 2007, 6-9.</p>

<P>[3] Steve Souders, Rule 1: make fewer http requests i <i>High Performance Web Sites: Essential Knowledge for Frontend Engineers</i>,
Redaktör, Andy Oram. Sebastopol, California: O'Reilly Media, 2007, 10.</p>

<P>[4] Steve Souders, Rule 3: add an expires header i <i>High Performance Web Sites: Essential Knowledge for Frontend Engineers</i>,
Redaktör, Andy Oram. Sebastopol, California: O'Reilly Media, 2007, 22-28.</p>

<P>[5] Steve Souders, Rule 4: gzip components i <i>High Performance Web Sites: Essential Knowledge for Frontend Engineers</i>,
Redaktör, Andy Oram. Sebastopol, California: O'Reilly Media, 2007, 29-36.</p>

<P>[6] Steve Souders, Rule 5: put stylesheets at the top i <i>High Performance Web Sites: Essential Knowledge for Frontend Engineers</i>,
Redaktör, Andy Oram. Sebastopol, California: O'Reilly Media, 2007, 37-44.</p>

<P>[7] Steve Souders, Rule 6: put scripts at the bottom i <i>High Performance Web Sites: Essential Knowledge for Frontend Engineers</i>,
Redaktör, Andy Oram. Sebastopol, California: O'Reilly Media, 2007, 45-50.</p>

<P>[8] Steve Souders, Rule 8: make javascript and css external i <i>High Performance Web Sites: Essential Knowledge for Frontend Engineers</i>,
Redaktör, Andy Oram. Sebastopol, California: O'Reilly Media, 2007, 55-62.</p>

<P>[9] Steve Souders, Rule 10: minify javascript i <i>High Performance Web Sites: Essential Knowledge for Frontend Engineers</i>,
Redaktör, Andy Oram. Sebastopol, California: O'Reilly Media, 2007, 69-75.</p>

