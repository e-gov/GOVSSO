---
permalink: Application
---

# GOVSSO-ga liidestumiseks vajalikud tegevused

Teenust osutatakse kõigile valitsussektori asutustele vastavalt Rahandusministeeriumi [kodulehel»](https://www.rahandusministeerium.ee/et/riigihaldus) toodud [tabelile»](https://www.rahandusministeerium.ee/sites/default/files/riigihaldus/avaliku_sektori_asutused_asutuse_liikide_loikes_2021.xlsx) (XLSX) (v.a Muu avalik sektor).

Seega tuleb enne liidestumise planeerimist veenduda infosüsteemi vastavuses eeltoodud tabelile ning samuti veenduda, et muud liitumistingimused on sobilikud (v.t Lepingud ning tingimused)

## Analüüsida liidestava infosüsteemi tehnilist ülesehitust ja sobivust

GOVSSO liides põhineb OpenID Connect protokollil, kuid seda on kitsendatud täiendavate tingimustega. Juhul kui liidestatav infosüsteem põhineb nö "karbitarkvaral" ei pruugi lihtsalt seadistamisega olla võimalik GOVSSO teenusega liidestuda. 
Karbitarkvara muutmine ei pruugi aga olla võimalik või ka mõtekas.  Seega tuleks enne liitumisotsuse tegemist veenduda, et liidestumine on tehniliselt võimalik.

## Lepingud ning tingimused

GOVSSO teenusega liitumise tingimused ning taotluste planketid on leitavad Riigi Infosüsteemi Ameti [kodulehelt](https://www.ria.ee/et/riigi-infosusteem/eid/partnerile.html)

## Liidestumise protssess

1) Liituda GOVSSO DEMO teenusega (täita taotlus ning sõlmida vajalikud lepingud: [link](https://www.ria.ee/et/riigi-infosusteem/eid/partnerile.html)
2) Teostada infosüsteemi liidestus. Info liidestumise tehnilistest tegevustest on leitav siit: [link](TechnicalSpecification#7-security-operations) 
3) Testida liidestust kasutades GOVSSO DEMO teenust. Peamised testistenaariumid on leitavad siit: [link](TechnicalSpecification)
4) Liituda GOVSSO toodang teenusega
5) Võtta GOVSSO kasutusele toodang teenuses