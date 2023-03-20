---
permalink: Application
---

<img src='img/el_regionaalarengu_fond_horisontaalne.jpg' width="350" height="200" alt="Euroopa Liit Euroopa Regionaalarengu Fond"/>

# GovSSO-ga liidestumiseks vajalikud tegevused

Teenust osutatakse kõigile valitsussektori asutustele vastavalt Rahandusministeeriumi [kodulehel](https://www.rahandusministeerium.ee/et/riigihaldus) toodud tabelile (XLSX) (v.a Muu avalik sektor).

Seega tuleb enne liidestumise planeerimist veenduda infosüsteemi vastavuses eeltoodud tabelile ning samuti veenduda, et muud liitumistingimused on sobilikud (v.t Lepingud ning tingimused).

## Analüüsida liidestava infosüsteemi tehnilist ülesehitust ja sobivust

GovSSO liides põhineb OpenID Connect protokollil, kuid seda on kitsendatud täiendavate tingimustega. Juhul kui liidestatav infosüsteem põhineb nö "karbitarkvaral", ei pruugi lihtsalt seadistamisega olla võimalik GovSSO teenusega liidestuda. 
Karbitarkvara muutmine ei pruugi aga olla võimalik või ka mõttekas. Seega tuleks enne liitumisotsuse tegemist veenduda, et liidestumine on tehniliselt võimalik.

## Lepingud ning tingimused

GovSSO teenusega liitumise tingimused ning taotluste blanketid on leitavad Riigi Infosüsteemi Ameti [kodulehelt](https://www.ria.ee/riigi-infosusteem/elektrooniline-identiteet-ja-usaldusteenused/kesksed-autentimisteenused#govsso).

## Liidestumise protsess

1. Liituda GovSSO demokeskkonnaga. Taotluse info on leitav [siit](https://www.ria.ee/riigi-infosusteem/elektrooniline-identiteet-ja-usaldusteenused/kesksed-autentimisteenused#govsso).
2. Teostada infosüsteemi arenduskeskkonna/testkeskkonna/demokeskkonna liidestus GovSSO demokeskkonnaga. Info liidestumise tehnilistest tegevustest on leitav [siit](TechnicalSpecification).
3. Testida liidestust kasutades GovSSO demokeskkonda. Peamised testistenaariumid on leitavad [siit](https://github.com/e-gov/GOVSSO-Test).
4. Liituda GovSSO toodangukeskkonnaga. Taotluse info on leitav [siit](https://www.ria.ee/riigi-infosusteem/elektrooniline-identiteet-ja-usaldusteenused/kesksed-autentimisteenused#govsso).
5. Teostada infosüsteemi toodangukeskkonna liidestus GovSSO toodangukeskkonnaga.
