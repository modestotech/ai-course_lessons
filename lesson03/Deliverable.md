# Step 1
## Prompt
Analysera det här felet och formulera tre hypoteser om rotorsaken, ordnade efter sannolikhet. För varje hypotes: förklara mekanismen och ange konkreta verifieringssteg. 

Problem description: A BizTalk sendport fails, apparently when the xml content contains an escpaed & (&amp;). Not sure what happens in what order. When the message succeeds, the Operation in BizTalk becomes the expected message type, when it fails, it sends just Send.

## Answer (summarized)
Den viktigaste ledtråden: ordningen. BTS.Operation sätts innan meddelandet når sändporten. Den skrivs i kontexten när meddelandet publiceras i MessageBox, till exempel av en orkestrerings logiska port, en pipeline-komponent eller ett resubmit-verktyg. Sändporten och adaptern läser bara värdet.

- Hypotes 1 (mest sannolik): &amp; av-escapas till rått & och XML:en blir ogiltig
- Hypotes 2: Felhanteringsvägen publicerar meddelandet via en annan port eller operation
- Hypotes 3: Mottagaren eller transporten hanterar & fel, och omsändningen tappar kontexten

# Step 2
## Prompt
Utifrån hypotes 1: vilka filer och funktioner ska jag undersöka? Vad ska jag leta efter?

## Answer (summarized)
1. Bekräfta en misstänkt plats
2 .Skicka ett testmeddelande där ett enda fält innehåller A &amp; B.
3. Aktivera tracking av meddelandekroppen (Track Message Bodies, före och efter port) på varje port och orkestrering i flödet.
4. Spara meddelandet efter varje steg. Det första steget där A & B står oescapat är skyldigt.

# Step 3
## Problemet du analyserade
Se ovan.

## Hur du beskrev det för agenten (vad gav du för kontext?)
Se ovan.

## Agentens hypoteser
Se ovan.

## Vilken hypotes du valde och varför
Hypotes 1, eftersom förklaring lät rimlig och stämmer överens med min känsla och erfarenhet.

## Vad som faktiskt var fel
Inte helt lyckats lokalisera var i kedjan felet finns ännu, men det är någonstans inne i den stora orkestreringen som encoding blir fel. AI:n gav inga direkta villospår, men samtidigt var det svårt att nå ett svar då det är flera repon inblandade, och en stor och rörig orkestrering, samt att kod skickas till AI.

## Vad agenten fick rätt och vad den missade
Så vitt jag förstår, har den inte fel i något den har sagt, men då den inte sett någon källkod så har den gett mer generella råd och gått igenom hur BizTalk fungerar.
