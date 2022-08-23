---
permalink: BusinessDescription
---

# Ärikirjeldus

Riigi SSO teenus (GOVSSO) on Riigi Infosüsteemi Ameti poolt pakutav teenus, millega asutus saab oma e-teenusesse lisada nii siseriiklike kui ka Euroopa Liidu piiriüleste autentimismeetodite toe koos seansihaldusega.

Teenust osutatakse kõigile valitsussektori asutustele vastavalt Rahandusministeeriumi [kodulehel»](https://www.fin.ee/riik-ja-omavalitsused-planeeringud/riigihaldus) toodud [tabelile»](https://www.fin.ee/media/6840/download)(XLSX) (v.a Muu avalik sektor).

Siseriiklikest autentimismeetoditest on toetatud:

- mobiil-ID (ainult Eesti isikukoodiga kasutajad)
- ID-kaart
- smart-ID (ainult Eesti isikukoodiga kasutajad) *Palume pöörata e-teenuste osutajatel tähelepanu asjaolule, et Smart-IDga isiku tuvastamisel ei ole võimalik eristada Eesti residente mitte residentidest (näiteks nagu seda võimaldab e-residentide sertifikaat).*

Samuti on toetatud piiriülene autentimine [Euroopa Liidu teavitatud eID vahenditega](https://ec.europa.eu/cefdigital/wiki/display/EIDCOMMUNITY/Overview+of+pre-notified+and+notified+eID+schemes+under+eIDAS) läbi eIDAS-Node taristu.

## Kellele?

Valitsussektori asutustele, kes soovivad:
- oma e-teenustes pakkuda kasutajatele laia valikut autentimismeetodeid, ise neid meetodeid teostamata.
- lisada oma e-teenusele SSO toe.

## Tehnilised tingimused?

E-teenus liidestatakse autentimisteenusega OpenID Connect protokolli kohaselt. Vt lähemalt: [TechnicalSpecification](TechnicalSpecification).

## Kuidas liituda?

Asutusel tuleb:

1 välja selgitada, kas ja millistes e-teenustes soovitakse GOVSSO teenus kasutusele võtta<br>

2 kavandada ja tellida liidestamistöö<br>

3 teostada arendus<br>

4 esitada RIA-le taotlus teenusega liitumiseks<br>

- RIA registreerib teie rakenduse GOVSSO teenuse kliendiks ja avab juurdepääsu GOVSSO testkeskkonda.

5 testida liidest RIA testteenuse vastu

- RIA abistab võimalike probleemide lahendamisel

6 eduka testimise järel taodelda registreerimist toodanguteenusega

- RIA avab teie rakendusele juurdepääsu GOVSSO toodangukeskkonda.

## Millal?

Testteenus on avatud 2022. a märtsist

Teenus avatakse on toodangukeskkonnas augustist 2022.

## Rohkem teavet?

Kontakt: `help@ria.ee`.

Kui pöördute liidestamisel või liidestatud klientrakenduses TARA kasutamise tehnilise probleemiga, siis palume valmis panna väljavõte klientrakenduse logist. Tõrkepõhjuse väljaselgitamiseks vajame teavet, mis päring(ud) TARAsse saadeti ja mis vastuseks saadi.

Samuti tasub heita pilk [korduma kippuvate küsimuste rubriiki](Faq).

Liitumisega seotud info leiab [siit](Application).

[TechnicalSpecification](TechnicalSpecification) (liidese arendajale).