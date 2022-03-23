---
permalink: Application
---

# GOVSSO-ga liidestumiseks vajalikud tegevused

Teenust osutatakse kõigile valitsussektori asutustele vastavalt Rahandusministeeriumi [kodulehel](https://www.rahandusministeerium.ee/et/riigihaldus) toodud [tabelile](https://www.rahandusministeerium.ee/sites/default/files/riigihaldus/avaliku_sektori_asutused_asutuse_liikide_loikes_2021.xlsx) (XLSX) (v.a Muu avalik sektor).

Seega tuleb enne liidestumise planeerimist veenduda infosüsteemi vastavuses eeltoodud tabelile ning samuti veenduda, et muud liitumistingimused on sobilikud (v.t Lepingud ning tingimused).

## Analüüsida liidestava infosüsteemi tehnilist ülesehitust ja sobivust

GOVSSO liides põhineb OpenID Connect protokollil, kuid seda on kitsendatud täiendavate tingimustega. Juhul kui liidestatav infosüsteem põhineb nö "karbitarkvaral", ei pruugi lihtsalt seadistamisega olla võimalik GOVSSO teenusega liidestuda. 
Karbitarkvara muutmine ei pruugi aga olla võimalik või ka mõttekas. Seega tuleks enne liitumisotsuse tegemist veenduda, et liidestumine on tehniliselt võimalik.

## Lepingud ning tingimused

GOVSSO teenusega liitumise tingimused ning taotluste blanketid on leitavad Riigi Infosüsteemi Ameti [kodulehelt](https://www.ria.ee/et/riigi-infosusteem/eid/partnerile.html)

## Liidestumise protsess

1. Liituda GOVSSO demokeskkonnaga. Taotluse info on leitav siit: TODO
2. Teostada infosüsteemi arenduskeskkonna/testkeskkonna/demokeskkonna liidestus GOVSSO demokeskkonnaga. Info liidestumise tehnilistest tegevustest on leitav siit: [link](TechnicalSpecification) 
3. Testida liidestust kasutades GOVSSO demokeskkonda. Peamised testistenaariumid on leitavad siit: TODO
4. Liituda GOVSSO toodangukeskkonnaga. Taotluse info on leitav siit: TODO
5. Teostada infosüsteemi toodangukeskkonna liidestus GOVSSO toodangukeskkonnaga.
