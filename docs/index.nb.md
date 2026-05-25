# AIS2203 E-bok

Velkommen til e-boken for emnet [AIS2203 — Datateknikk for kyberfysiske systemer](https://www.ntnu.no/studier/emner/AIS2203#tab=omEmnet).

Denne boken handler om å få programmer til å gjøre **mer enn én ting om gangen**, og om å få programmer på **ulike maskiner til å snakke sammen**. Disse to ideene — samtidighet og kommunikasjon — er det som gjør en enkeltstående øvingsoppgave om til en robot som leser sensorene sine, kjører en reguleringssløyfe, strømmer telemetri til en operatør og reagerer på kommandoer: alt samtidig, og alt i tide.

Bruk sidemenyen til å navigere, eller hopp rett til et emne med søkefeltet øverst. Ny her? Begynn med [Kom i gang](getting_started.md), og les deretter [Fra AIS1003 til AIS2203](from_ais1003.md) for å se hva emnet forutsetter at du allerede kan.

!!! note "Denne boken bygger på AIS1003"
    AIS2203 forutsetter at du allerede kan skrive, bygge og feilsøke et moderne C++-program med klasser, `std::vector`, RAII og CMake — innholdet i [AIS1003-e-boken](https://markaren.github.io/E-book_cpp/). Hvis noe av dette sitter løst, repeter det først; denne boken går raskere og bygger på alt sammen.

!!! note "Om oversettelsen"
    Denne boken er bare delvis oversatt til norsk (bokmål). Ikke alt er oversatt ennå — sider som ennå ikke er oversatt, vises på engelsk. Bytt språk med velgeren i menylinjen øverst.

## Hva du vil lære

Boken følger emnets oppbygning:

| Del | Hva den dekker |
|-----|----------------|
| **1. Moderne C++-verktøykasse** | Språkfunksjonene samtidighet og kommunikasjon hviler på: eierskap, flyttesemantikk, smartpekere, maler, lambda-uttrykk og `std::chrono`. |
| **2. Grunnleggende samtidighet** | Prosesser og tråder, kappløpssituasjoner, mutexer, tilstandsvariabler og atomære operasjoner — å dele data uten å ødelegge dem. |
| **3. Asynkront og sanntid** | Futures, trådbasseng, parallelle algoritmer, korutiner, og å overholde tidsfrister i et sanntidssystem. |
| **4. Datakommunikasjon** | Serialisering, sockets, TCP/UDP, serielinjer, Modbus, og mønstre på høyere nivå som MQTT og RPC. |
| **5. Embedded Linux** | Krysskompilering for en Raspberry Pi, styring av GPIO og busser fra Linux, og utrulling av programmet som en tjeneste. |
| **6. Bygge større prosjekter** | CMake med flere mål, tredjepartsavhengigheter med vcpkg, og å bygge bro mellom C++ og Python. |

Hvert kapittel avsluttes med **oppgaver**. Som i AIS1003 er løsningene sløret — prøv hver oppgave ærlig før du avslører svaret. Skriv koden inn i CLion og kjør den; å lese er ikke det samme som å kunne.

## Flere ressurser

Boken støtter læringen din, men er ikke en komplett referanse. Suppler den.

### Bøker

- **C++ Concurrency in Action, Second Edition** (Anthony Williams). Den definitive boken om C++-trådbiblioteket og minnemodellen — hovedfølgesvennen til del 2 og 3.
- **A Tour of C++** (Bjarne Stroustrup). En kortfattet oversikt over moderne C++ for de som allerede kan det grunnleggende.
- **Effective Modern C++** (Scott Meyers). Flyttesemantikk, smartpekere og samtidighet, forklart punkt for punkt.

### På nett

- [**cppreference**](https://en.cppreference.com/w/). Pålitelig og oppdatert dokumentasjon av språket og standardbiblioteket — særlig [`<thread>`](https://en.cppreference.com/w/cpp/thread)-delen.
- [**Boost.Asio**](https://www.boost.org/doc/libs/release/doc/html/boost_asio.html). De facto C++-biblioteket for nettverk og asynkron I/O.
- [**Stack Overflow**](https://stackoverflow.com/). De fleste programmeringsspørsmål er allerede stilt og besvart.

## Snarveier

- [Vanlige begreper](terms.md)
- [Ofte stilte spørsmål (FAQ)](faq.md)
