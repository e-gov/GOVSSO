---
permalink: Faq
---

<img src='img/el_regionaalarengu_fond_horisontaalne.jpg' width="350" height="200" alt="Euroopa Liit Euroopa Regionaalarengu Fond"/>

# Korduma kippuvad küsimused
{: .no_toc}

Hea GovSSO-ga liidestuja! Siit leiad valiku küsimusi, mis teistel on tekkinud - ja vastused.

- TOC
{:toc}

## Mida arvestada liitumistaotluses toodud klientrakenduse lühinimetuse valimisel?
  
Klientrakenduse lühinimetus on mõeldud kasutajale kuvamiseks mobiilseadmes. Lühinimetuse mobiilseadmes kuvamise eesmärgiks on autentimisprotsessi käigus lõppkasutaja täiendav informeerimine autentimise algatanud infosüsteemist, et muuta autentimise protsess läbipaistvamaks. 

Lühinimetuse valimisel tuleks arvestada järgmiste asjaoludega:

1. Klientrakenduse lühinimetuse pikkus on piiratud. <br/><br/>Lühinimetuse maksimaalne lubatud pikkus sõltub nimetuses kasutatavatest tähemärkidest. Kui piirdutakse standardsete [GSM märgistiku](https://en.wikipedia.org/wiki/GSM_03.38) sümbolitega võib lühinimetus olla maksimaalselt 40 tähemärki. Juhul kui kasutatakse GSM-7 väliseid sümboleid, võib lühinimetus olla maksimaalselt 20 tähemärki.

2. Klientrakenduse lühinimetus peaks olema kasutajale arusaadav ja seostatav autentimist algatava infosüsteemiga. <br/><br/>Kui klientrakenduse nimetuse maksimaalne pikkus ei ületa ülaltoodud piirangut võib sama nimetust kasutada ka lühinimetuse puhul. Muul juhul võib klientrakenduse lühinimetuse tuletada infosüsteemi täispikast nimetusest. Näiteid klientrakenduse lühinimetustest:`Eesti.ee`, `E-MTA`, `eKool`.

## Kas GovSSO toetab ka teesklemist (impersonation)?

TARA ega GovSSO seda praegu ei toeta.

## Kui olete loginud sisse rohkem kui ühte süsteemi, nt. Süsteem B, Süsteem C ja Süsteem D, siis kas saab valida välja logida ühest või kahest ainult (Süsteem B ja Süsteem C), ja ühes (Süsteem D) jätkata sessiooni või saab ainult kõikidest välja logida, mitte valikuliselt?

See on GovSSO-s hoitud kasutaja jaoks lihtsana, et saab valida ainult, kas logida kõikidest välja või jätkata kõikides.

## Mis on ID tokeni kehtivusaeg?

Hetkel on selleks 15 minutit. Infosüsteem peab kehtivusaja välja lugema ID Tokeni "exp" väärtusest, mitte hardcode'ima 15 minutit, sest see võib GovSSO teenuse poolel tulevikus muutuda.

## Kui pikalt sisselogitud sessioone GovSSO ja kliendi rakenduse vahel hoitakse?

Nii kaua kuni klientrakendus seanssi perioodiliselt pikendab, aga mitte kauem kui 12 tundi. Seejärel peab kasutaja uuesti autentima.

## Mida tehakse "küpsiste varguse" vastu? Nt kui kasutaja kasutab avalikku (nt raamatukogu) arvutit ja unustab väljuda SSO kaudu?

Infosüsteem peab iga 15 minuti tagant seanssi pikendama (sooritama pikendamise päringu GovSSO-sse). Kui kasutaja on veebibrauseris infosüsteemi saki sulgenud, siis seanssi enam ei pikendata ja seanss (s.h. küpsised) aegub maksimaalselt 15 minuti jooksul.

## Kas saki sulgemisest piisab ja brauserit ennast ei tule sulgeda? St tavaliselt brauser jagab seanssi sakkide vahel ja saki sulgemisest üksi ei piisa.

Kui kasutaja unustas välja logimise nuppe klikkida, aga sulgeb veebibrauseri kõik sakid, kus infosüsteem oli GovSSO kaudu sisse logitud, siis ükski infosüsteem enam seansi pikendamise päringuid ei soorita. Selle tulemusena GovSSO seanss aegub maksimaalselt 15 minuti jooksul.

## Kui kasutaja pani selle saki kinni aga brauserit mitte ja siis tuleb uus kasutaja ja avab sama rakenduse (või teeb "ava suletud kaart uuesti"), siis peaks OIDC järgi küll token olema aegunud, kuid GovSSO RT ei pruugi olla ju aegunud (seega uuendatakse token taustal ära) ja isegi kui RT on aegunud ja kasutaja suunatakse GovSSO-sse autentima, ei pruugi GovSSO seanss (nt küpsis) olla aegunud. Ehk siis oluline on, mida kasutab GovSSO oma seansi tuvastamiseks ja mis aja peale sunnitakse kasutajat uuesti oma "füüsilist tokenit" näitama (nt mobiili-ID või Smart-ID või eIDAS eID või ID-kaarti soovitakse uuesti näha soovitatavalt koos PIN-i küsimisega).

Kui avatakse suletud sakk ja kui klientrakenduse enda seanss on veel elus (oleneb klientrakenduse teostuse nüanssidest), nii et klientrakendus suudab teha valiidse pikendamise päringu GovSSO-sse ja eelmisest pikendamise päringust ei ole möödas kauem kui 15 minutit, siis GovSSO seanss pikeneb.
Kui eelmisest pikendamise päringust GovSSO-sse on möödas kauem kui 15 minutit, siis GovSSO enam seanssi ei pikenda, isegi kui klientrakenduse enda kohalik seanss on veel elus. Klientrakendus saab pikendamise päringu vastuseks vea ja on kohustatud enda kohaliku seansi ka lõpetama.
Kui GovSSO seanss on aegunud/lõppenud, siis ainus variant seanssi uuesti püsti saada on, et GovSSO järgmisel autentimispäringul kuvab kasutajale uuesti autentimise kuva (n-ö sunnitakse kasutajat uuesti oma "füüsilist tokenit" näitama (nt mobiili-ID või Smart-ID või eIDAS eID või ID-kaarti).
Seda, kas ID-kaardi puhul uuesti PIN koodi küsitakse, sõltub kas veebibrauseris puhverdab TLS-CCA (Client Certificate Authentication) puhul PIN koodi. See on samamoodi nagu TARA-s.

## Kui seanssi pikendatakse taustal skriptiga, siis kas on ka mingi jäik periood, mille järel nõutakse kasutaja taastuvastamist? (s.t, et kasutaja lahtine brauseriaken lõputult sisse ei jääks kui kasutajat ennast seal taga ei ole)

Klientrakendus saab pikendamise päringutega seanssi pikendada kuni teatud maksimaalse ajani. Hetkel on selleks 12 tundi, aga see võib tulevikus muutuda.
Pigem on mõistlik klientrakenduse enda poolel teostada kasutaja aktiivsuse tuvastamine, ehk et kui kui kasutaja mõnda aega ühtegi tegevust sooritanud pole, siis klientrakenduse enda seanss aegub/lõpetatakse. Klientrakenduse enda poolne kasutaja aktiivsuse perioodi pikkus sõltuks klientrakenduse spetsiifikast, näiteks pankade puhul on see tüüpiliselt väga lühike (nt 5 minutit), mõne teise infosüsteemi puhul võib see olla ka pikem, sõltuvalt infosüsteemi enda vajadusest (pool tundi, tund, rohkem).

## Kui kasutaja on mitmest browserist sisse logitud ning ühes browseris valib kõikidest välja logimise. Mis juhtub teise browseri sessiooniga?

Kui need on eraldi brauserid ehk käsitletavad eraldi seadmetena (s.t. igaühel on oma küpsisehoidla), siis nendes moodustatakse eraldi GovSSO seansid. Ühes seadmes / brauseris seansi lõpetamine ei mõjuta teist seadet / brauserit.

## Miks redirect_uri on *hard coded*?

Turvalisuse kaalutlustel, samamoodi nagu TARA-s. redirect_uri tuleb registreerida liitumistaotluses ja seda saab hiljem muuta kirjutades [help@ria.ee](mailto:help@ria.ee).
Kui Teie klientrakenduse arhitektuurist tulenevalt on vaja, siis on võimalik registreerida ühele klientrakendusele mitu redirect_uri väärtust ja klientrakendus saab autentimispäringut sooritades ise valida, millise eelregistreeritud redirect_uri väärtuse ta kaasa annab.

## Kas "given_name" on antud nimi või eesnimi? Samamoodi, kas "family_name" on perekonnanimi või viimane nimi? Mõnel juhul on need vahetuses (s.t perekonnanimi kirjutatakse esimese nimena)?

Nimede väärtused tulevad üks-ühele selliselt nagu need vastava autentimisvahendi juures on (ID-kaardi sertifikaat / Mobiil-ID sertifikaat / Smart-ID sertifikaat / eIDAS puhul välisriigi autentimisteenuse info).

## Miks refresh token kasutamist ei ole GovSSO-s kaalutud, kus saaks kasutaja sessiooni hallata?

OAuth standardist tulenevat refresh tokenit GovSSO teenus ei kasuta. Pikendamine on autentimispäringu kordamine prompt ja id_token_hint lisaparameetritega.

## Kas identsustõendi valideerimiseks, sessiooni uuendamiseks jms on plaanis teha integraatoritele abistavad teegid nagu seda on näiteks Web-eID puhul?

Kuna GovSSO ja TARA põhinevad OpenID Connect standarditel, siis peaks võimalik olema kasutada juba maailmas olemasolevaid levinud OpenID Connect protokolli toetavaid teeke. Täpsemalt oleneb konkreetsest teegist, milliseid OpenID Connect standardeid või alamosi see toetab.

## Kas on võimalik testida voogu, kus tokenisse ilmuks ka muu acr väärtus peale "high"? Kui jah, siis kuidas millised test autentimisvahendid lisavad ACR'i "low" või "substantial"?

Eesti autentimisvahendid (ID-kaart, Mobiil-ID, Smart-ID) on hetkel kõik "high" tasemega. Hetkel saavad "low" ja "substantial" tasemed tulla ainult "EU eID" ehk eIDAS autentimise kaudu. eIDAS riikide eID vahenditega testimise info on olemas TARA dokumentatsioonis https://e-gov.github.io/TARA-Doku/Testimine#eidas.

## Mida tähendab bufferdamine minu kui tavakasutaja jaoks? Millised tegevused tuleb teha selle jaoks?

Kui on mõeldud GovSSO Identity Token allkirjastamise võtme puhverdamist klientrakenduses, siis see ei puutu tavakasutajasse, sellega peab klientrakendus tegelema.

## Kas GovSSO tarbeks tehakse ka mock teenus (nagu TARA-mock), mida saab koormustestimisel kasutada (kui on vaja testida suurema hulga kasutajatega (nt 10 000 unikaalset kasutajat)?

GovSSO tarbeks on planeeritud luua ka mock teenus (nagu TARA-l), mille arendustega alustame tõenäoliselt 2022. aasta teises pooles.

## Kas iseteenindusportaali ei ole plaanis?

RIA on eID iseteeninduskeskkonna loomiseks jaoks taotlenud rahastust, mis võimaldaks alustada selle realisatsiooniga.

## `401 unauthorized` viga identsustõendi küsimisel /token otspunktist?

Veendu et:
- Authorization päis on korrektselt koostatud vastavalt juhendile [Technical Specification](/TechnicalSpecification#62-id-token-request).
- client_id ja client_secret väärtused on õiged ning väljastatud antud keskkonna jaoks.

Uue saladuse (client_secret) väljastamiseks kontakteeru [help@ria.ee](mailto:help@ria.ee). Kindlasti lisa vastava teenuse client_id millele saladust soovitakse.

## Kas brauseris peavad küpsised olema lubatud?

Jah, GovSSO vajab küpsiste lubamist.  


Ei leidnud oma küsimusele vastust? Kontakteeru Riigi Infosüsteemi Ametiga: [help@ria.ee](mailto:help@ria.ee). Kui sa oled juba GovSSO teenusega liitunud siis kindlasti lisa ka oma infopäringusse client_id väärtus.
{: .adv}
