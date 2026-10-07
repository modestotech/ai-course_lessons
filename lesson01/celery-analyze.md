# Lesson 1
Repo: https://github.com/celery/celery

## Prompts
- First prompt
    - Explain what this is. Output as markdown file.
- Second prompt
  - Explain this project, focus on architecture and layers and technical debt.

## Summary

### Vad och för vem
Celery är en öppen källkods-kö för distribuerade bakgrundsjobb i Python. Den används för att köa upp jobb som kan köras asynkront, utanför request/response-flödet.

## Arkitekturen i grova dragA
Applikationen skickar ett task-meddelande till en broker, till exempel RabbitMQ eller Redis. Worker-processer hämtar meddelandet, kör uppgiften och kan spara resultatet i en backend.

## De tre viktigaste beroendena
1. kombu: all kommunikation med brokers. Celery hanterar vad en task är och kombu hanterar hur meddelandet skickas, vilket gör att Celery kan stödja många brokers.
2. billiard: Celerys egen fork av multiprocessing. Standardpoolen (prefork) bygger på den, och den behövs för bättre kontroll över barnprocesser än standardbiblioteket ger.
3. vine: promises och barriers som används för asynkron resulthantering och callbacks.

## Överraskningar
Det finns några riktigt stora filer. celery/canvas.py är störst, därefter kommer app/base.py (1 736 rader) och concurrency/asynpool.py (1 489 rader).

## Var fick agenten rätt
Svårt att bedöma, men påståenderna känns relativt generella och således förmodligen korrekta. Jag bad den inte om hårda fakta, då det tenderar att bli mer fel.
