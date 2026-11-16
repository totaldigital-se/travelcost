# travelcost

Resekostnader → Timekeeper.

En webbsida där man rapporterar resekostnader, alltså traktamente och milersättning. Traktamentet fylls i i en kalendervy. Sidan gör om resan till rader som skapas som utlägg i Timekeeper via deras API.

I stället för att lägga in varje post för hand (hel- och halvdag, måltidsavdrag, tider) gör man så här:

1. Logga in med sitt vanliga Timekeeper-konto.
2. Klicka på första och sista resdagen i kalendern.
3. Fyll i avresetid, hemkomsttid, projekt och ort.
4. Klicka i **F / L / M** för de dagar då frukost, lunch eller middag ingått.
5. Lägg till eventuella körningar under **Milersättning**: datum, biltyp (bensin/diesel eller el) och antal mil, med decimaler.
6. Kontrollera raderna och klicka på **Skicka**. Första klicket ändrar knappen till "Bekräfta", och andra klicket skickar raderna.

## Regler

| Dag | Regel |
|---|---|
| Avresedag | Hel dag om avresan sker före 12.00, annars halvdag |
| Hemresedag | Hel dag om hemkomsten sker efter 19.00, annars halvdag |
| Dagar däremellan | Hel dag |
| Endagsresa | Inget traktamente, eftersom resan saknar övernattning |
| Måltidsavdrag | En rad per måltid med lönearten för respektive måltid. På halvdagar halveras avdraget (kan stängas av under Inställningar) |
| Milersättning | En rad per körning: antal mil × företagets sats inkl. tillägg, 45 kr/mil för bensin/diesel och 34 kr/mil för elbil (`MILEAGE_RATES`). Kommentaren visar sträckan och uträkningen |

Beloppen hämtas alltid från lönearterna i Timekeeper. När Skatteverkets belopp ändras uppdaterar man lönearterna i Timekeeper. Sidan behöver inte ändras.

Sidan hanterar **inte**: nattraktamente, utlandstraktamente, sänkt traktamente efter tre månader på samma ort eller kravet på att resmålet ligger mer än 50 km bort. Den anställda ansvarar själv för att resan ger rätt till traktamente.

## Konfiguration

Överst i `<script>`-delen av `index.html`:

```js
const DEFAULT_WAGE_CODES = { full: 11050, half: 12556, breakfast: 12543, lunch: 12544, dinner: 12545,
                             mileFuel: null, mileEl: null };
const MILEAGE_RATES = { mileFuel: 45, mileEl: 34 };   // kr/mil inkl. tillägg
const DEFAULT_PROJECT_NR = '998';                     // Internt (Total Digital)
const SUPPORT_CONTACT = 'ekonomiavdelningen';
```

- `DEFAULT_PROJECT_NR` är projektnumret som nya resor och körningar får som standard. Den som vill kan välja ett eget standardprojekt under *Inställningar*.
- `MILEAGE_RATES` innehåller företagets milersättning per mil inklusive tillägget. Den ersätter lönearternas standardbelopp i Timekeeper.

- `DEFAULT_WAGE_CODES` innehåller ID:n för företagets lönearter i Timekeeper. Alla anställda får dem automatiskt. Den som vill kan byta löneart under *Inställningar*. Valet sparas då i den egna webbläsaren. Står det `null` letar sidan upp lönearten via namnet. För milersättning letar den efter "Milersättning" + "bensin/diesel" eller "el bil".
- `SUPPORT_CONTACT` visas i sidfoten och i felmeddelanden.

## Säkerhet

- Sidan är en enda statisk HTML-fil utan egen server och utan några hemligheter i koden.
- Inloggningen går **direkt från webbläsaren till `api2.timekeeper.se`** med Basic Auth. Lösenordet sparas inte.
- I webbläsarens `localStorage` sparas bara resor, måltidsval och val av löneart, aldrig lösenordet.
- **Det viktiga är att skydda koden.** Den som kan ändra `index.html` kan få sidan att skicka iväg lösenord. Gör därför så här:
  - Ge skrivrättighet till repot bara till ett fåtal personer.
  - Slå på branch protection för `main`, så att ändringar kräver en pull request med godkänd granskning.
  - Lägg aldrig in skript från andra webbplatser i sidan.

## Driftsättning på GitHub Pages

1. Skapa ett repo på GitHub, till exempel `travelcost`, och pusha koden:
   ```bash
   git remote add origin https://github.com/totaldigital-se/travelcost.git
   git push -u origin main
   ```
2. Gå till **Settings → Pages**. Välj *Deploy from a branch*, branch `main` och mappen `/ (root)`.
3. Efter ungefär en minut finns sidan på `https://totaldigital-se.github.io/travelcost/`.
4. Vill ni ha en egen adress, t.ex. `trakt.totaldigital.se`, anger ni den under *Custom domain* och lägger till en CNAME-post i DNS.
5. Slå på branch protection under **Settings → Branches**, enligt avsnittet om säkerhet ovan.

Om sidan bara ska synas för inloggade anställda kan den i stället driftsättas på Azure Static Web Apps eller Cloudflare Pages, med inloggning via företagskontot. Koden behöver inte ändras.

## Testa lokalt

Dubbelklicka på `index.html` så öppnas den i webbläsaren. Det fungerar eftersom Timekeepers API tillåter anrop från alla adresser. Knappen **Prova i demoläge** visar sidan med påhittade lönearter och projekt, och ingenting skickas till Timekeeper.

## API-anrop som används

| Anrop | Syfte |
|---|---|
| `GET /api/v1/Login/me` | Hämta den inloggade användaren |
| `GET /api/v1/WageCode` | Hämta lönearter och belopp |
| `GET /api/v1/Project` | Hämta projekt |
| `GET /api/v1/ExpenseRegistration` | Hitta dubbletter. Svarar 404 när det inte finns några utlägg, vilket sidan tolkar som en tom lista |
| `POST /api/v1/ExpenseRegistration` | Skapa en rad |

API-dokumentation: <https://api2.timekeeper.se/swagger/v1/swagger.json>
