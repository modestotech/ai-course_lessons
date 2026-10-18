Jag bad den att hålla respektive analys till max en halv sida för att få ner outputen till greppbar storlek.

Åtgärd fokuserade jag på säkerhet då dessa var enklare att greppa då projektet är nytt. 

För arkitektur och kodkvalitet bad jag claude om att lägga in det i planen och CLAUDE.md för att åtgärda senare. 

# Prompt 1 -- Arkitektur:
**Prompt**: Granska kodbasen mot de arkitekturprinciper jag definierat i CLAUDE.md. Var bryter koden mot principerna? Var är den i linje med dem? Ge konkreta filreferenser. 
Keep the scope so that the output gets about half page in size.
Save the output to a temp markdown file and pass it to clip.exe without messing up Swedish å ä ö.
Add your findings to plan md file and claude md file as you find appropriate.

## Commit
https://github.com/modestotech/azure-tools-interface/commit/26fe0eb3c6ff9d53f428047f72458b6b27a15879

## Svar
# Arkitekturgranskning mot CLAUDE.md (2026-10-18)

**Läge:** M1 klart. `lib/`, `app/session/` och delar av `services/` finns; `domain/`, `views/`, `scripts/` och `components/` finns inte än, så de principerna går inte att granska ännu.

## I linje med principerna
- **Allt nätverk i en klient:** `fetch` anropas bara i `src/lib/http/client.ts:129,212`. Spärrarna för read-only, värd och token körs före varje anrop (`client.ts:271-273`), och klienten följer inga redirects och skickar inga cookies (`client.ts:222`).
- **En klientinstans i `app/`:** `src/app/session/httpClient.ts`. `lib/` och `services/` importerar aldrig uppåt (inga träffar på `@/app|services|domain`).
- **Bara i minnet + Clear session:** `sessionStore.ts` har ingen persist-middleware och validerar ID:n (`setTenantId`/`setSubscriptionId`). `clearSession.ts` rensar tokens, klientens cache och loggen.
- **Säker export:** loggexporten går via allowlistan `EXPORTED_FIELDS` (`src/lib/http/requestLog.ts:77`), så felmeddelanden och omaskerade URL:er stannar kvar.
- **Ett ställe för nycklar, versioner och fel:** `services/queryKeys.ts`, `services/arm/apiVersion.ts` och `services/result.ts` (`settle()`, sorterar fel på `kind` via `isHttpError`).
- **Säkerhetstester:** `tests/security/` finns och går igenom.

## Brott och luckor
1. **Hårdkodade URL:er i test:** `src/app/session/sessionStore.test.ts:27,28,46,85,86` använder `https://management.azure.com`, trots regeln "use `ARM` / `GRAPH`". `src/test/helpers/fakeJwt.ts:16,26` hårdkodar också `aud` i stället för `API_RESOURCES[api].resource`.
2. **Lagerordningen kontrolleras inte automatiskt:** `eslint.config.js` saknar `no-restricted-imports`/import-boundaries. Att beroenden bara går `views → domain → services → lib` hålls i dag bara av disciplin. En regression syns inte förrän vid granskning.
3. **CSP saknas helt:** varken `index.html` eller `public/staticwebapp.config.json` har en CSP, fast CLAUDE.md kräver en gemensam definition för båda. Ingen milstolpe i planen äger den uppgiften.
4. **Globalt tillstånd på modulnivå i `lib/`:** `requestLogStore` (`requestLog.ts:199`) och maskcachen (`src/lib/http/mask.ts:18`) är singletons i `lib/`, medan principen säger att "the single instance lives in `app/`". Det fungerar eftersom `clearSession` rensar båda, men dokumenterat är det inte.
5. **`subscriptionId` i sessionstoren** (`sessionStore.ts`) är ett öppet beslut som ska fattas när M4:s selection-store byggs.

# Prompt 2 -- Säkerhet:

**Prompt 1**: Gå igenom kodbasen och leta efter: hårdkodade hemligheter eller uppgifter, saknad inputvalidering, injection-risker (SQL, kommando, path traversal), känsliga data i loggar, och osäkra standardvärden. Lista varje fynd med filnamn och radnummer. Keep the scope so that the output gets about half page in size.

**Prompt 2**: 

## PR

https://github.com/modestotech/azure-tools-interface/pull/2

## Svar

 Uppdatera PROJECT_PLAN.md så att säkerhetsfynden får en lösning

 Kontext

 Säkerhetsgranskningen gav sju fynd (se föregående svar). Fynd som PROJECT_PLAN.md redan löser lämnas orörda. Övriga får konkreta punkter i den milstolpe där koden byggs. Bara dokumentation ändras i det här steget, ingen kod.

 Redan täckt av planen (ingen ändring)

 - CSP (fynd 1, delvis): M4 rad 109–113 har redan connect-src/script-src, <meta> + globalHeaders och ett test.
 - Inklistrat subscription-ID (fynd 3, delvis): M4 rad 115 säger att ID:t bara väljs från GET /subscriptions och aldrig klistras in.

 Ändringar i PROJECT_PLAN.md

 M2: ny första punkt "lib/http-härdning före services"
 - encodePathSegment() i lib/http/odata.ts, eller en armPath()-byggare. Alla ID:n som interpoleras i sökvägar (sub, svc, apiId, opId, backendId från policy-XML, Graph-id) går genom den. .., /, ? och # avvisas, övrigt percent-kodas.Test: backend-id="../../x" kastar invalid-request. (fynd 3)
 - readOnly.ts: $batch-sub-URL:er måste vara relativa sökvägar utan .., //, \ eller schema. Test med /../beta/… och //evil. (fynd 5)
 - client.ts fetchOnce: sätt redirect: 'error' och credentials: 'omit'. Test: en 302 ger ett network-fel och följs inte. (fynd 6)

 M4: token-skärmen
 - Tenant-ID valideras som GUID eller verifierad domän (^[a-z0-9.-]+$) innan det används. Ett ogiltigt värde ger ett fältfel och används aldrig i azTokenCommand. azTokenCommand faller tillbaka på <TENANT_ID> om värdet är ogiltigt(försvar på djupet i lib/token/resources.ts). (fynd 2)
 - setSubscriptionId tar bara emot GUID:er. Pickern är den enda källan. (fynd 3)

 M4: CSP-punkten utökas med frame-ancestors 'none', base-uri 'none', object-src 'none', form-action 'none', plusX-Content-Type-Options: nosniff och Referrer-Policy: no-referrer i globalHeaders. Testet i tests/security/build.test.ts kontrollerar alla dessa. (fynd 1)

 M4: demoläget: main.tsx byter till onUnhandledRequest: 'error', och demoläget visar en banner "fiktiv data, klistrainte in riktiga tokens". (fynd 7)

 M5: export: fältet error exporteras bara som kind + status, inte Azures fritext. Alternativt maskeras GUID:er och e-post/UPN generellt ({guid}, {upn}) i exporten. Test: ett felmeddelande med UPN och GUID ger ingen träff i exporten.(fynd 4)

 Ändringar i CLAUDE.md (regler, enligt dokumentationsregeln)

 Under Conventions läggs två korta regler till:
 - ID:n i sökvägar byggs alltid med encodePathSegment(), aldrig med interpolering.
 - Tenant- och subscription-ID valideras som GUID innan de lagras.

 Under Authentication model ändras exportregeln: Azures fritext i felmeddelanden exporteras inte.

 Verifiering

 - Granska diffen för PROJECT_PLAN.md och CLAUDE.md. Ingen kod ändras, så inga tester behövs i det här steget.
 - Ingen commit förrän du har godkänt (CLAUDE.md › Git).

# Prompt 3 -- Kodkvalitet:

**Prompt**: Identifiera: funktioner som gör mer än en sak, otydlig namngivning (variabler, funktioner, klasser), saknad felhantering för externa anrop, och kod som är svår att läsa utan kommentarer. Lista de fem allvarligaste problemen.
Keep the scope so that the output gets about half page in size.
Save the output to a temp markdown file and pass it to clip.exe without messing up Swedish å ä ö.
Add your findings to plan md file and claude md file as you find appropriate.

## Commit
https://github.com/modestotech/azure-tools-interface/commit/2630cf3ea8e2810b51839c9cd130a4410b512650

## Svar
# Kodgranskning – de fem allvarligaste problemen

Omfång: `src/lib/` och `src/app/session/` (det som finns efter M1).

1. **Valideringsfel efter `send()` loggas som lyckade** (felhantering) – `lib/http/client.ts:366–369, 390–393, 408–414`. `getAllPages` och `batchChunk` kastar `invalid-response` *efter* att loggraden redan avslutats som `ok`. Matrix-loggen visar grönt fast analysen fallerar, och felet saknar `url`/`status`. Underanrop i `$batch` som svarar 403/404 syns inte heller i loggen. Åtgärd: skicka med schemat till `send()` så att valideringen går via `fail()`.

2. **Ingen timeout på externa anrop** (felhantering) – `client.ts:204–227`, `limiter.ts:24–34`. `fetchOnce` har ingen tidsgräns. Ett anrop som hänger blockerar en av limiterns åtta platser tills webbläsaren ger upp, och loggen visar `pending` under tiden. Köade uppgifter bryr sig inte heller om `signal`. Åtgärd: `AbortSignal.any([signal, AbortSignal.timeout(…)])` och ett eget felslag, `timeout`.

3. **`send()` gör för mycket** (ansvar) – `client.ts:263–348`. Funktionen är 85 rader och sköter skydd, loggning, retry-loop, läsning av svarskroppen, mappning av fel och JSON-parsning. Avbrott/nätverksfel fångas på två ställen med nästan identisk kod (306–312, 317–323). Åtgärd: bryt ut `fetchWithRetry()` och `readResponse()`.

4. **`batchChunk` är svår att läsa utan kommentarer** (läsbarhet) – `client.ts:376–426`. Retry-budgeten delas med `send()` över två loopar, och `attempt` styr backoff medan budgeten räknas ned på ett annat ställe. Variablerna heter bara `r`, `p`, `retry` och `wait`. Okända eller dubblerade svars-id:n ignoreras tyst (`continue`, rad 399). Åtgärd: bryt ut en ren `partitionBatchResponses()` och kommentera budgetregeln.

5. **Missvisande namn** (namngivning)
   - `HttpErrorKind 'http'` (`errors.ts:8`) säger ingenting, eftersom alla fel är HTTP-fel. Bättre: `unexpected-status`.
   - `TokenClaims.aud` (`jwt.ts:19`) är bara den första audiencen men heter som claimet, så den kan av misstag användas i stället för `audiences` vid kontroller.
   - `upn` innehåller i själva verket upn, unique_name eller preferred_username.
   - `TokenWarning`/`isBlockingWarning` (`checks.ts`) kallar blockerande fel för varningar.
   - En token som inte går att avkoda ger felet `audience-mismatch` (`client.ts:161`).

