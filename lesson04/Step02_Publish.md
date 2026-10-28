# CI/CD

## Prompt

Jag ska sätta upp en CI/CD-pipeline med GitHub Actions för det här projektet. Analysera kodbasen och generera en pipeline som:

1. Körs vid push till main och vid pull requests mot main
2. Kör alla tester och avbryter om något test misslyckas
3. Kör en linter som avbryter vid fel
4. Bygger projektet (om det är relevant för stacken)
5. Deployar till Github Pages vid lyckad push till main

Förklara varje steg i pipelinen.

Add a step in the plan.md file in relation to this.
Make this in a separate commit.
Add a badge to the README.

## Answer

Solved it easily, I just activated GH Pages manually in repo settings.

# Verifiera hemlighetshantering

Not applicable, no secrets used.