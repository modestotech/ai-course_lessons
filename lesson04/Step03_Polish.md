# Prompt

Gå igenom hela kodbasen systematiskt. Identifiera och lista enligt följande kategorier:
1. Saknad felhantering för externa anrop (API-anrop, databasanrop, filläsning)
2. Otydliga felmeddelanden som inte hjälper en användare förstå vad som gick fel
3. Edge cases som inte hanteras (tomma listor, null-värden, ogiltiga indata)
4. Saknad eller otillräcklig dokumentation i komplex kod
5. Hårdkodade värden som borde vara konfigurerbara

Begränsa dig till en kategori per fel, samt 10 fel.
För varje fynd: ange fil, rad och en konkret åtgärd.
Ge svaret som markdown.

# Answer

Jag har gått igenom src/, utom tester och de genererade shadcn-komponenterna. Koden är överlag noggrann: fel har typer, svar valideras och anrop har timeouts och omförsök. Därför ligger fynden mest i kanterna. Inga filer har ändrats.

#: 1
Kategori: 1. Felhantering
Fil:rad: src/views/access-graph/AccessGraph.tsx:57
Problem: void layoutGraph(graph).then(...) saknar catch. Om elkjs kastar ett fel, t.ex. vid en kant mot en nod som
saknas eller en mycket stor graf, uppstår en ohanterad rejection. Grafen ritas då aldrig ut.
Åtgärd: Lägg till .catch() som sparar felet i useLayout. Visa "Layout misslyckades" med en knapp för att försöka igen.
────────────────────────────────────────
#: 2
Kategori: 1. Felhantering
Fil:rad: src/components/CopyButton.tsx:9
Problem: navigator.clipboard.writeText saknar catch. Anropet nekas över http, i en iframe och när behörighet saknas,
och navigator.clipboard kan vara undefined. Knappen för att kopiera az-kommandot gör då ingenting, utan meddelande.
Åtgärd: Kontrollera att navigator.clipboard finns och fånga felet. Visa då "Kunde inte kopiera, markera texten
manuellt".
────────────────────────────────────────
#: 3
Kategori: 1. Felhantering
Fil:rad: src/lib/http/client.ts:647
Problem: Promise.all(chunks.map(batchChunk)): om en enda $batch-POST misslyckas (500 eller nätverksfel) avvisas hela
anropet. Resultaten från de andra chunkarna kastas bort fast de redan har hämtats.
Åtgärd: Använd Promise.allSettled. Ge varje id i den misslyckade chunken ett eget fel, så att övriga resultat behålls
(via loadBatch i services/graph/batch.ts:236).
────────────────────────────────────────
#: 4
Kategori: 2. Otydligt felmeddelande
Fil:rad: src/lib/http/client.ts:573
Problem: Stopped after 1000 pages. saknar URL och status, och säger inte varför det är ett problem. Dessutom är varje
sida redan loggad som ok, så felet syns inte i request-loggen.
Åtgärd: Skicka med den maskade URL:en och skriv t.ex. "Gav upp efter 1000 sidor från {url}: nextLink slutar aldrig.
Trolig loop eller för stor samling." Logga felet.
────────────────────────────────────────
#: 5
Kategori: 2. Otydligt felmeddelande
Fil:rad: src/services/graph/batch.ts:220
Problem: En 401 i ett $batch-delsvar blir Graph answered 401 (…) for …, utan azCommand. Ett vanligt 401-svar ger
däremot ett tydligt meddelande om utgången token och ett kommando att kopiera.
Åtgärd: När status är 401: bygg felet med unauthorizedError() från lib/http/errors.ts. Då får delsvaret samma
meddelande och samma az-kommando.
────────────────────────────────────────
#: 6
Kategori: 2. Otydligt felmeddelande
Fil:rad: src/app/layout/Scripts.tsx:38
Problem: Ett ogiltigt parametervärde (under min, tomt eller inte ett tal) ignoreras tyst. Fältet visar det nya värdet
medan resultatet fortfarande bygger på det gamla.
Åtgärd: Spara det inskrivna värdet, sätt aria-invalid och visa t.ex. "Måste vara ≥ {min}; visar resultat för {senaste
giltiga värde}".
────────────────────────────────────────
#: 7
Kategori: 3. Edge case
Fil:rad: src/domain/policy-parser/elements.ts:62 (används i src/domain/mapper/inbound.ts:41)
Problem: En <validate-jwt> utan <audiences> ger audiences: []. Då får operationen inga anropare och ingen finding alls.
I APIM betyder en saknad audience att fler släpps in, inte färre.
Åtgärd: Lägg till en finding no-audience (severity high) i findings.ts och skapa den när rule.audiences.length === 0.
Lägg till ett test med fixture.
────────────────────────────────────────
#: 8
Kategori: 3. Edge case
Fil:rad: src/domain/mapper/outbound.ts:75
Problem: authentication-managed-identity utan client-id använder APIM:s systemtilldelade identitet. Om tjänsten inte
har någon sådan blir principalId undefined utan finding, och vyn ser ut som om autentiseringen fungerar.
Åtgärd: Skapa en finding (identity-not-found eller en ny no-system-identity) när systemAssignedPrincipalId saknas.
Testa med en tjänst utan identity.
────────────────────────────────────────
#: 9
Kategori: 4. Dokumentation
Fil:rad: src/lib/http/limiter.ts:163-175
Problem: onAbort kör queue.splice(queue.indexOf(wake), 1). Det fungerar bara om wake avregistrerar abort-lyssnaren
innan den körs, och det står ingenstans. Bryts antagandet blir indexOf -1, och då tar splice(-1, 1) bort fel uppgift
i kön. Hur release() lämnar platsen direkt till nästa uppgift är inte heller förklarat.
Åtgärd: Skriv en kommentar som förklarar antagandet och hur platsen lämnas över. Lägg gärna till skyddet if (i !== -1).
────────────────────────────────────────
#: 10
Kategori: 5. Hårdkodat värde
Fil:rad: src/lib/token/resources.ts:19-43
Problem: Adresserna och audiences fungerar bara i Azures publika moln (management.azure.com, graph.microsoft.com).
Tenants i Azure Government eller Azure China kan inte analyseras. Samma värden finns också i CSP:n och i
tests/security/build.test.ts.
Åtgärd: Inför en molninställning (public / usgov / china) som väljs när appen byggs, t.ex. VITE_AZURE_CLOUD. Härled
API_RESOURCES, connect-src i security.config.ts och ALLOWED_ORIGINS från den.

Om du vill åtgärda fynden föreslår jag en commit per fynd. Du granskar dem innan något committas.

# PR
9 were fixed in this PR.
https://github.com/modestotech/azure-tools-interface/pull/3