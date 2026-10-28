# Instructions

README: Skriv en komplett README med README-mallen från fliken Tips. Kontrollera att den täcker: vad projektet gör, hur man installerar, hur man konfigurerar, hur man kör, vilka externa beroenden som krävs.

CV-punkt: Skriv din CV-punkt. Formatet:

Designade och utvecklade [vad] med [teknikstack] med hjälp av agentdriven utveckling (Claude Code). Projektet [löser vilket problem] och är tillgängligt på [URL].

Se avsnittet "CV-mall" i fliken Tips för exempel på bra och dåliga formuleringar.

Repot:

Publikt
README på plats
Inga känsliga filer committade (kontrollera med agenten)
CI/CD-badge i README om du vill visa pipeline-status

## README template

README-mall
Det här är strukturen varje deltagares README bör följa. Kopiera och fyll i.

[Projektnamn]
En mening om vad projektet gör och vilket problem det löser.

Live: [länk till deployad version om det inte är ett repo man drar ner och använder]

Vad det gör
2-3 meningar om funktionaliteten. Inga buzzwords. Beskriv vad en användare faktiskt kan göra.

Exempel:

Analyserar pull requests i ett GitHub-repo och sammanfattar vad som ändrats, vilka filer som påverkats och om det finns uppenbara risker. Körs som ett CLI-kommando och skriver resultatet till stdout eller en fil.

Kom igång
Krav
[Lista runtime-krav, t.ex. Python 3.11+, Node 20+]
[Lista externa tjänster som krävs, t.ex. OpenAI API-nyckel, GitHub token]
Installation
git clone https://github.com/[org]/[repo]
cd [repo]
[installationskommando, t.ex. pip install -r requirements.txt eller npm install]
Konfiguration
Skapa en .env-fil i projektets rot:

[VARIABELNAMN]=ditt-värde
[ANNAN_VARIABEL]=ditt-värde
Alla nödvändiga variabler finns listade i .env.example.

Kör projektet
[kommando för att starta, t.ex. python main.py eller npm run dev]
Arkitektur
Kort beskrivning (3-5 meningar) av hur projektet är strukturerat. Vilka lager finns? Var finns logiken?

[repo-rot]/
├── [mapp]/        # [vad den innehåller]
├── [mapp]/        # [vad den innehåller]
└── [fil]          # [vad den gör]
Teknisk stack
Komponent	Val	Anledning
[t.ex. Backend]	[t.ex. FastAPI]	[t.ex. Snabbt att prototypa, inbyggd validering]
[t.ex. Databas]	[t.ex. SQLite]	[t.ex. Ingen infrastruktur krävs för MVP]
Begränsningar
Var ärlig om vad projektet inte gör eller inte gör bra. Det är ett tecken på professionalism, inte svaghet.

Utvecklat med
Claude Code (agentdriven utveckling) som del av kursen Next-Generation Software Development with AI.

CV-punkt
CV-punkt: Exempel på bra och dåliga formuleringar
Använd den här filen som referens när du skriver din CV-punkt och när du ger feedback på andras.

Formatet
Designade och utvecklade [vad] med [teknikstack] med hjälp av agentdriven utveckling (Claude Code). Projektet [löser vilket problem] och är tillgängligt på [URL].

Dåliga formuleringar: varför de inte håller
"Erfarenhet av AI-verktyg och moderna utvecklingsmetoder" Problem: Säger ingenting konkret. Vad är "AI-verktyg"? Vad är "moderna metoder"? Kan inte verifieras.

"Använde Claude Code för att bygga ett projekt" Problem: Inget om vad projektet är, vilken stack, vilket problem det löser. Ingen URL. Kan inte granskas.

"Agentdriven AI-assisterad full-stack-applikation med cutting-edge-teknologi" Problem: Buzzwords utan substans. "Full-stack" och "cutting-edge" säger inget om vad du faktiskt kan.

"Bidrog till ett open source-projekt med hjälp av AI" Problem: "Bidrog" är vagt. Hur? Vad? Hur stort bidrag?

Bra formuleringar: varför de håller
"Designade och utvecklade ett CLI-verktyg i Python (Click, PyGithub) som analyserar GitHub pull requests och identifierar riskfyllda ändringar. Projektet är tillgängligt på github.com/[namn]/pr-analyzer." Varför det håller: Konkret vad (CLI-verktyg), konkret stack (Python, Click, PyGithub), konkret funktion (analyserar PR, identifierar risker), verifierbar URL.

"Byggde och deployade en REST API i FastAPI och PostgreSQL som hanterar personalschemaläggning för ett mindre företag. Live på [URL], med CI/CD via GitHub Actions." Varför det håller: Stack specificerad, problemdomän tydlig, deployment bekräftad med URL, CI/CD nämnt som konkret leverabel.

"Refaktorerade en befintlig Python-tjänst (3 000 rader) med hjälp av agentdriven utveckling: extraherade affärslogik från I/O-lager, lade till saknade tester (täckning 0% -> 74%), identifierade och åtgärdade tre säkerhetssårbarheter. Koden finns i [länk]." Varför det håller: Kvantifierat (3 000 rader, 0% -> 74%), konkreta åtgärder (tre kategorier), verifierbar länk.

Att tänka på
Om du inte har en URL fungerar projektet inte som CV-bevis. Deploya såfall alternativt ha en githublänk.
Teknikstacken ska vara specifik: inte "Python" utan "FastAPI + PostgreSQL + Redis".
Undvik "hjälpte", "bidrog", "var delaktig i". Vad gjorde du specifikt?
Skriv en mening om vilket problem projektet löser. Det ger kontexten som gör resten trovärdigt.
CI/CD-prompt
CI/CD-prompt: Träff 4 Demo
Kör prompten i Claude Code med projektets rot som arbetskatalog.

Prompt: Generera GitHub Actions-pipeline
Jag ska sätta upp en CI/CD-pipeline med GitHub Actions för det här projektet. Analysera kodbasen och generera en pipeline som:

1. Körs vid push till main och vid pull requests mot main
2. Kör alla tester och avbryter om något test misslyckas
3. Kör en linter som avbryter vid fel
4. Bygger projektet (om det är relevant för stacken)
5. Deployar till [Vercel, Railway eller Fly.io (välj det som passar projektet)] vid lyckad push till main
6. Använder GitHub Secrets för alla känsliga värden, inga hemligheter i koden

Förklara varje steg i pipelinen och vilka GitHub Secrets jag behöver sätta upp.
Anpassa: Byt ut deployment-tjänst mot det som är relevant för demoprojektet. Vercel passar för frontend/fullstack, Railway eller Fly.io för backend-tjänster.

Förväntat resultat
En .github/workflows/ci.yml-fil med:

on: push: branches: [main] och on: pull_request: branches: [main]
Separata jobb för test och deploy, där deploy beror på att test lyckas
secrets. för API-nycklar och deployment-tokens
Tydliga steg med name: på varje
Prompt 2: Verifiera hemlighetshantering
Gå igenom hela kodbasen och kontrollera att inga hemligheter, API-nycklar, lösenord eller connection strings är hårdkodade. Lista varje fynd med filnamn och radnummer. Lista även alla platser där miljövariabler läses in och kontrollera att de hanteras korrekt om de saknas.
Poäng att lyfta: Kör alltid den här prompten innan en deployment. Det är tio sekunder som kan förhindra en incident.

## Prompt

Analyze the project and the current README, PLAN and CLAUDE markdown files and rewrite the README according to this template in English.
Try to not be too chatty. Keep it short and sweet according to the template.
Be sure to emphasize the security aspects, and that the DEMO mode is great for visualizing how it works.
Ask me if any clarifications are needed.

## Result

Used AI but did several manual changes as well.

PR: https://github.com/modestotech/azure-tools-interface/pull/5

### CV Point

Designade och utvecklade med hjälp av agentdriven utveckling (Claude Code) en webbapp som visualiserar åtkomst (genom Azure Entra) till Azure API Management-API:er, samt hur API:erna är uppsatta. Appen visar samband som är svåra att överblicka genom portalen, eller enstaka CLI-anrop. Bland annat följande tekniker användes: React, TypeScript, TanStack Query, React Flow och Tailwind CSS. Den är tillgänglig på https://modestotech.github.io/azure-tools-interface/.