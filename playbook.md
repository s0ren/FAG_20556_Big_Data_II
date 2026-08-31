---
draft: true
---

## Website oprettet med Quarto

ctrl-p, `quarto: Create New Project`.

## Tilføj git og put på github

Vælg _Source Control_ ikonet længst til venstre, og tryk __Publish to GitHub__.

## For at publicere på github pages

kørete alle disse:

    git checkout --orphan gh-pages
    git reset --hard # make sure all changes are committed before running this!
    git commit --allow-empty -m "Initialising gh-pages branch"
    git push origin gh-pages

og skiftede tilbage til main branch

Herfra kan man publisere med kommandoen

    quarto publish gh-pages

Svar __`Yes`__