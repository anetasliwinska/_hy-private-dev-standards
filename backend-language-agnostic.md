# Standardy backendowe niezależne od języka i frameworka

> Ten dokument nie zakłada żadnego konkretnego języka programowania, frameworka ani biblioteki — nadaje się tak samo do serwisu w Pythonie, PHP, Javie, Go, jak i w TypeScript/Node. Cel: zasady architektoniczne i projektowe, które przenoszą się między technologiami, bo dotyczą tego, *jak myśleć o systemie*, nie *jakiej składni użyć*.
>
> Format: pogrubiona reguła + krótkie uzasadnienie/przykład. Bez fragmentów kodu w konkretnej składni — jeśli reguła wymaga ilustracji, ilustracja jest opisowa (np. "kod 403", "format ISO 8601"), nie zapisem w konkretnym języku.
>
> Pochodzenie: zdestylowane z dojrzałych standardów wewnętrznych dwóch niezależnych projektów backendowych (NestJS/TypeScript), po odrzuceniu wszystkiego, co jest specyficzne dla tej konkretnej technologii — zostały wyłącznie zasady, które broniły się same, niezależnie od implementacji.
>
> Dokument towarzyszący: `frontend-react-typescript-nextjs.md` (ten sam folder) — tamten jest celowo przywiązany do jednego stosu frontendowego, ten jest celowo od żadnego stosu niezależny.

## 1. Zasady architektoniczne

- **Warstwa logiki biznesowej zależy od abstrakcyjnego kontraktu, nigdy od konkretnej implementacji.** Kod domenowy najwyższego poziomu wywołuje interfejs/kontrakt danej odpowiedzialności (np. "magazyn danych", "dostawca płatności", "system tożsamości"); to, która konkretna implementacja/dostawca stoi pod spodem, jest szczegółem podłączonym z zewnątrz. Jeśli kontraktowi brakuje metody potrzebnej warstwie wyższej — dodaj metodę do kontraktu i zaimplementuj we wszystkich implementacjach, nigdy nie omijaj kontraktu bezpośrednim wywołaniem konkretnej implementacji.
- **Kontrakt implementuje się w całości, bez opcjonalnych metod.** Częściowa implementacja łamie podstawialność — konsument kontraktu musi móc zaufać, że każda zgodna implementacja obsługuje każdą metodę kontraktu, bez wyjątków typu "ta konkretna integracja tego nie wspiera".
- **Wybór, która implementacja obsługuje dany kontrakt, żyje w jednym, scentralizowanym miejscu konfiguracji uruchomieniowej — nie jest rozproszony po kodzie.** Podmiana dostawcy (np. z atrapy testowej na prawdziwy system, albo z jednego dostawcy na innego) powinna być zmianą w jednym miejscu, nie polowaniem po całym kodzie za każdym miejscem użycia.
- **Każda zależność, z której faktycznie korzysta dany moduł, musi być jawnie zadeklarowana w jego własnej konfiguracji — nie polegaj na tym, że "gdzieś wyżej" w aplikacji akurat jest już zarejestrowana.** Domyślne poleganie na przypadkowej dostępności zależności z zewnętrznego kontekstu jest kruche: działa, dopóki moduł nie zostanie użyty ponownie w innym kontekście (inna aplikacja, inny zestaw zarejestrowanych zależności, izolowany test) — wtedy zawodzi po cichu albo dostaje złą instancję.
- **Każda transformacja danych na granicy między warstwami przechodzi przez dedykowaną funkcję mapującą — nawet jeśli dziś wygląda trywialnie.** Trywialny mapper jutro urośnie; inline'owe mapowanie ukrywa miejsce, w którym trzeba będzie wprowadzić zmianę przy ewolucji kontraktu. Nigdy nie zwracaj surowej odpowiedzi zewnętrznego systemu bezpośrednio jako odpowiedzi warstwy wyższej — nawet jeśli kształt danych wygląda dziś identycznie, bo każda przyszła zmiana formatu po stronie zewnętrznego systemu natychmiast złamie wszystkich konsumentów bez mappera pomiędzy.
- **Warstwa integracji z zewnętrznym systemem nie przechowuje danych trwale.** Pobiera dane w czasie rzeczywistym (albo z efemerycznego cache z krótkim TTL), transformuje je do znormalizowanego modelu domenowego i zwraca — jedno źródło prawdy per domena, bez ryzyka desynchronizacji dwóch kopii tych samych danych.
- **Atrapa/implementacja zamockowana (statyczne dane, bez wywoływania prawdziwego systemu) to legalny, osobny wariant implementacji tego samego kontraktu** — przydatny do developmentu, testów, demo, prototypowania. Musi implementować dokładnie ten sam kontrakt co prawdziwa integracja, żeby była podmienialna bez zmiany reszty systemu.
- **Rozszerzanie istniejącej integracji: dziedziczenie/kompozycja nad wspólną abstrakcją, nie równoległy, zduplikowany moduł.** Duplikacja prowadzi do rozjazdu implementacji w czasie. Jeśli abstrakcja bazowa nie pozwala się rozszerzyć w potrzebny sposób — to sygnał do dyskusji o zmianie samej abstrakcji, nie do tworzenia obejścia.
- **Moduł eksportuje na zewnątrz tylko to, co faktycznie jest potrzebne innym modułom** — minimalizuj powierzchnię eksponowaną, żeby zredukować sprzężenie. Jeden moduł, jedna dobrze zdefiniowana odpowiedzialność domenowa.
- Jeśli istnieje generator/szkielet do powtarzalnej struktury (nowy moduł, nowa integracja) — użyj go zamiast ręcznego kopiowania innego modułu "jako wzorca". Ręczne kopiowanie pomija ukryte kroki konfiguracyjne i jest częstym źródłem cichych błędów produkcyjnych (komponent zarejestrowany w czterech miejscach z pięciu wymaganych — nic nie krzyczy, po prostu nie działa w runtime).

## 2. Modelowanie danych i mappery

- **Warstwa niezależna od konkretnego dostawcy danych definiuje znormalizowane modele — jeden model per koncepcja domenowa.** Każda integracja transformuje swój surowy kształt danych do tego wspólnego modelu, więc konsument (logika wyższego poziomu, warstwa prezentacji) zawsze widzi spójną strukturę niezależnie od tego, który system źródłowy akurat odpowiada.
- Zanim zadeklarujesz nowy typ dla powszechnej koncepcji (pole formularza, lista stronicowana, odnośnik, plik) — sprawdź, czy współdzielony typ już istnieje. Jeśli istniejący jest "prawie dobry, ale nie do końca" — rozszerz go, nie forkuj od zera.
- Nie deklaruj kształtu żądania/odpowiedzi inline w sygnaturze funkcji — wydziel do nazwanej, reużywalnej struktury w dedykowanym miejscu. Ukryty inline'owy kontrakt utrudnia ponowne użycie i śledzenie zmian.
- **Mapper jest czystą funkcją** — bez efektów ubocznych, bez mutacji danych wejściowych (logowanie wewnątrz mappera to zapach kodu).
- **Rozróżnij pola legitymnie opcjonalne od encji strukturalnie bezużytecznych.** Pole opcjonalne dostaje bezpieczną nawigację i fallback. Ale gdy relacja *wymagana* do sensownego wyświetlenia encji jest pusta — całą encję trzeba odfiltrować na granicy mappera, nie "załatać" łańcuchem fallbacków przepychającym puste stringi/zepsute odnośniki dalej w dół.
- **Translacja formatu (konwencje sortowania, nazewnictwo pól, słowniki statusów, formaty dat) między "językiem domenowym" a "językiem konkretnego systemu zewnętrznego" żyje wyłącznie w warstwie integracji, nigdy w logice biznesowej wyższego poziomu.** Logika domenowa mówi jednym, spójnym językiem; integracja mówi oboma i tłumaczy między nimi.
- Filtrowanie danych na podstawie widoczności/ról (np. dane wewnętrzne vs publiczne) odbywa się w warstwie mappera integracji, na podstawie kontekstu przekazanego przez wywołującego (np. flaga uprawnień) — nie jest wpychane w zapytanie do zewnętrznego systemu. To odseparowuje model autoryzacji aplikacji od szczegółów zapytania do konkretnego dostawcy.
- Struktury głęboko zagnieżdżone z zewnętrznego systemu spłaszczaj do prostszej struktury domenowej tam, gdzie to ma sens — konsument nie powinien znać wewnętrznego zagnieżdżenia API zewnętrznego dostawcy.
- Rozróżniaj konsekwentnie "brak wartości" od "wartości pustej" w całym systemie (jedna konwencja) — niespójność jest źródłem błędów trudnych do wyśledzenia.
- Warstwa prezentacji/konsument importuje modele przez oficjalny, publiczny punkt wejścia warstwy backendowej — nigdy nie sięga bezpośrednio do wewnętrznych plików modelu integracji. Zapobiega to przenikaniu detali implementacyjnych konkretnego dostawcy do kodu klienckiego.
- Testuj mappery jako osobną, izolowaną jednostkę (happy path + brakujące opcjonalne pola) — niezależnie od testowania samej integracji.

## 3. Walidacja wejścia

- **Waliduj wszystkie dane wejściowe na samym początku funkcji/operacji, przed jakąkolwiek dalszą logiką ("fail early")** — nie marnuj pracy/wywołań sieciowych na dane, które i tak są nieprawidłowe.
- **Waliduj w czasie wykonania, nawet jeśli statyczny system typów (jeśli język go ma) teoretycznie to gwarantuje.** Dane pochodzące z zewnątrz (żądanie, zewnętrzne API) nie są zaufane przez sam fakt zadeklarowanego typu — typ statyczny nie chroni przed danymi przychodzącymi z zewnątrz granicy systemu.
- Waliduj obecność i nie-pustość pól wymaganych osobno od pól opcjonalnych (którym wolno przypisać wartość domyślną).
- **Waliduj względem listy dozwolonych wartości (allowlist), nie listy zabronionych (blocklist).** Blocklisty są z natury niekompletne; allowlisty definiują dokładnie to, co akceptowalne — bezpieczniejsze i łatwiejsze w utrzymaniu. To ogólna zasada "fail closed by default" na granicy systemu.
- Waliduj zakresy liczbowe (np. rozmiar strony w paginacji) — zbyt duży limit jest wektorem przeciążenia/odmowy usługi.
- Waliduj format ciągów znaków o określonym znaczeniu semantycznym (e-mail, URL, identyfikator) jawnymi regułami/wyrażeniami/parserami — nie akceptuj dowolnego ciągu bez weryfikacji formatu.
- **Waliduj konfigurację/zmienne środowiskowe przy starcie aplikacji, nie w trakcie obsługi żądania.** Fail fast: lepiej, żeby aplikacja w ogóle nie wystartowała z błędną konfiguracją krytyczną, niż żeby zawodziła nieprzewidywalnie w runtime.
- **Nigdy nie ufaj bezkrytycznie danym z zewnętrznego systemu** — waliduj istnienie krytycznych pól i poprawność struktury przed dalszym przetwarzaniem, niezależnie od tego, czy dostawca deklaruje własny kontrakt danych.
- **Waliduj reguły biznesowe, nie tylko format danych.** Reguły przejść stanu (encja zamknięta nie przechodzi bezpośrednio do stanu "w trakcie"), reguły własności (można modyfikować tylko swój zasób, chyba że ma się rolę administracyjną), spójność pól powiązanych (data "do" nie może być wcześniejsza niż data "od").
- Komunikaty błędów walidacji są konkretne i wykonalne (co dokładnie jest nieprawidłowe, jaki jest oczekiwany format/zakres) — nie ogólnikowe.
- **Sanityzacja przeciw atakom iniekcyjnym:** koduj parametry przekazywane do zapytań/adresów; waliduj ścieżki plików przeciw przechodzeniu katalogów (odrzucaj `..` i separator ścieżki w nazwie pliku); waliduj rozszerzenia/typy plików względem allowlisty; waliduj adresy URL wywoływane przez system (protokół, domena z listy zaufanych) przed wywołaniem — zapobiega to fałszowaniu żądań po stronie serwera (SSRF).
- Testuj logikę walidacji systematycznie: przypadki poprawne i niepoprawne, naruszenia reguł biznesowych, przypadki brzegowe (puste, brak wartości, wartości ekstremalne).

## 4. Obsługa błędów i odporność

- **Używaj specyficznych, semantycznych typów błędów dopasowanych do sytuacji** (walidacja, brak zasobu, brak autoryzacji, konflikt, błąd zewnętrznego systemu) zamiast jednego generycznego wyjątku — typ błędu powinien nieść informację o tym, co się stało i jak to zmapować na odpowiedź.
- **Warstwa transportowa nie przechwytuje wyjątków biznesowych "na wszelki wypadek".** Pozwól im propagować się do centralnego mechanizmu obsługi błędów, który konwertuje je na odpowiedź w spójnym formacie. Przechwytywanie ma sens tylko, gdy trzeba dodać kontekst specyficzny dla żądania — i wtedy błąd trzeba rzucić ponownie, nie połknąć.
- **Nigdy nie przekazuj surowego błędu z integracji zewnętrznej wprost do użytkownika końcowego.** Warstwa integracyjna tłumaczy błędy specyficzne dla zewnętrznego systemu (komunikat walidacji dostawcy, stack trace bazy danych, błąd tokenu) na stabilny, przewidywalny kod błędu rozumiany przez warstwę wyższą. Konsument nie powinien zależeć od treści komunikatu wewnętrznego systemu — to kruche i wycieka szczegóły implementacji.
- Komunikaty widoczne dla użytkownika są pomocne, konkretne i actionable ("zasób X o identyfikatorze Y nie znaleziony", nie "not found"), ale nigdy nie ujawniają szczegółów technicznych/infrastrukturalnych (adresy, nazwy kolumn, stack trace, treść zapytań do bazy).
- **Rozróżnij funkcje krytyczne od opcjonalnych.** Błąd w ścieżce krytycznej przerywa operację i zwraca błąd. Błąd w ścieżce opcjonalnej (dekoracja, rekomendacja, telemetria) loguje się jako ostrzeżenie, a operacja kontynuuje z bezpieczną wartością domyślną — "graceful degradation": jeśli rekomendacje produktowe padają, główna treść strony i tak powinna się załadować.
- **Retry z limitem prób dla błędów przejściowych** (timeout sieciowy, przeciążenie, throttling) — **nigdy nie ponawiaj** błędów, które nie znikną przy kolejnej próbie (błędne dane wejściowe, zasób nie istnieje, dane uwierzytelniające nieprawidłowe niezależnie od liczby prób).
- **Zawsze ustawiaj timeout na wywołania do zewnętrznych systemów** — nieograniczone oczekiwanie blokuje zasoby i degraduje cały system.
- **Circuit breaker dla krytycznych integracji zewnętrznych**: po przekroczeniu progu błędów w krótkim czasie, otwórz obwód i krótko zwracaj szybki błąd zamiast dalej czekać na timeout, z automatycznym resetem po czasie. Chroni system przed kaskadowym pogorszeniem, gdy zależność zewnętrzna jest w złym stanie.
- Retry z rosnącym odstępem czasu między próbami (exponential backoff) dla błędów throttlingu/przeciążenia, nie stały interwał.
- Każdy zalogowany błąd zawiera ustandaryzowany zestaw pól kontekstowych: co się działo (operacja, typ i identyfikator zasobu), kto wykonywał akcję, szczegóły błędu, kontekst żądania.
- **Nigdy nie loguj danych wrażliwych** (hasła nawet zahaszowane, tokeny/klucze, dane osobowe, numery kart płatniczych) — nawet w logach diagnostycznych.
- Testuj ścieżki błędów tak samo systematycznie jak ścieżki sukcesu: typ zwróconego błędu, zachowanie retry, zachowanie timeout.

## 5. Uwierzytelnianie i autoryzacja

- **Rozdziel role (kto ma dostęp do endpointu) od uprawnień (co dany użytkownik może zrobić z konkretnym zasobem)** — to dwie osobne warstwy autoryzacji, nie jedna.
- Każdy endpoint ma jawnie zadeklarowaną politykę dostępu — nawet "publiczny" powinien być świadomą deklaracją, nie brakiem deklaracji.
- Uprawnienia to pary zasób+akcja przechowywane w tokenie/sesji; logika sprawdzania uprawnień powinna zwracać mapę akcja→bool, żeby warstwa prezentacji mogła pokazywać/ukrywać funkcje bez duplikowania logiki autoryzacji po swojej stronie.
- **Operacje "na własnych danych" wyprowadzają tożsamość użytkownika z sesji/tokenu, nigdy nie przyjmują identyfikatora użytkownika jako parametr wejściowy od klienta.** Inaczej zwykły użytkownik może podać cudzy identyfikator i modyfikować cudze dane. Rozróżnij jawnie endpointy "self-service" (tożsamość z tokenu, brak parametru id) od "admin" (identyfikator jako parametr, z dodatkową weryfikacją zakresu).
- **Nie stosuj fallbacku do pustej/domyślnej wartości przy braku wymaganej tożsamości** (np. brakujący identyfikator zamieniony cicho na pusty string) — to zamienia błąd braku autoryzacji w cichy błąd "brak wyników". Przy braku wymaganej tożsamości/tokenu rzuć błąd natychmiast (fail closed).
- **Rola administracyjna nie daje automatycznie nieograniczonego dostępu do wszystkich zasobów w systemie.** Jeśli system ma pojęcie organizacji/najemcy/grupy, operacje administracyjne muszą dodatkowo weryfikować, że docelowy zasób należy do zakresu wykonującego akcję — sama rola "admin" to za mało.
- Nadawanie ról/uprawnień jest skopowane do właściwego zakresu (np. organizacji), nie globalne — globalne nadanie to cicha eskalacja uprawnień.
- **Nowe pojęcie domenowe dostaje własny, granularny zasób w modelu uprawnień**, nie jest podpinane pod istniejący, ogólniejszy zasób — inaczej system uprawnień traci precyzję rozróżniania "może widzieć X" od "może widzieć Y".
- Zmiana nazwy zasobu w warstwie autoryzacji musi być zsynchronizowana ze zmianą w mapowaniu ról/uprawnień — to rodzaj kontraktu niewykrywalnego statycznie, wymaga manualnej weryfikacji przy każdej takiej zmianie.
- Kod 401 dla braku/nieprawidłowego uwierzytelnienia, kod 403 dla uwierzytelnionego, ale nieautoryzowanego dostępu — to rozróżnienie jest uniwersalne w HTTP, niezależnie od frameworka.
- **Kontrola dostępu ma dwie osobne granulacje: cały zasób i poszczególne jego pola.** To, że ktoś widzi/edytuje rekord jako całość, nie znaczy, że wolno mu widzieć/edytować każde pole na nim (np. pole wynagrodzenia widoczne tylko dla siebie samego i administratora, reszta pól widoczna dla każdego, kto widzi rekord). Zaprojektuj dostęp na obu poziomach niezależnie, nie tylko na poziomie całego zasobu.
- **Sprawdzenie dostępu może zwrócić filtr zawężający wynik, nie tylko prawda/fałsz.** Zamiast "czy wołający widzi ten zasób" (allow/deny), potężniejszy i bezpieczniejszy domyślnie wzorzec to "jaki podzbiór tego zasobu wołający widzi" — zapytanie zwraca z góry zawężone do zakresu wołającego (np. tylko wiersze jego organizacji, tylko rekordy o statusie publicznym). To przenosi filtrowanie z logiki aplikacji do samego zapytania i eliminuje klasę błędów "zapomniałem przefiltrować po drodze". (To dokładnie ten mechanizm, którego brak zrozumienia bywa mylony z "cichym gubieniem zapisu" — zapis się powodzi, tylko odczyt zwraca zawężony/pusty wynik dla wołającego bez uprawnień; patrz sekcja 4 o pozorach sukcesu integracji.)
- **Zaufane/wewnętrzne ścieżki wywołania (operacje administracyjne, zadania w tle, bezpośredni dostęp do warstwy danych) domyślnie pomijają kontrolę dostępu — wymuszenie jej dla konkretnego wywołującego jest jawnym opt-in, łatwym do zapomnienia.** Jeśli mechanizm dostępu do danych ma tryb "zaufany" (pomija reguły) i tryb "w imieniu użytkownika" (reguły egzekwowane), przekazanie tożsamości użytkownika bez jawnego przełączenia na tryb egzekwowany jest cichą luką bezpieczeństwa — kod wygląda, jakby sprawdzał uprawnienia, a w rzeczywistości ich nie sprawdza.

## 6. Projektowanie API / warstwy transportowej

- Projektuj adresy wokół zasobów (rzeczowniki), nie akcji (`/articles`, nie `/getArticle`).
- Używaj standardowych metod HTTP semantycznie zgodnie z przeznaczeniem: odczyt bezstanowy i cache'owalny, tworzenie zasobu/złożone zapytanie, pełna aktualizacja, częściowa aktualizacja, usunięcie.
- **Warstwa transportowa jest cienka**: przyjmuje żądanie, deleguje logikę biznesową do warstwy serwisowej, nie zawiera transformacji danych ani logiki biznesowej.
- Endpoint akceptuje kontekstowe metadane żądania (locale, strefa czasowa, identyfikator organizacji) w ustandaryzowanej, typowanej formie i przekazuje je dalej.
- **Walidacja parametrów (paginacja, filtrowanie, sortowanie) odbywa się w warstwie logiki biznesowej, nie tylko przez typ danych żądania** — typ nie gwarantuje poprawności biznesowej (że wartość sortowania jest jedną z dozwolonych).
- Kolekcje wyników zwracaj w spójnym, przewidywalnym kształcie: dane + metadane paginacji — jeden ustandaryzowany format "koperty" dla wszystkich list w systemie.
- Stosuj semantyczne kody statusu HTTP: 2xx dla sukcesu (odpowiednio do operacji), 4xx dla błędów po stronie wołającego, 5xx dla błędów serwera.
- **Identyfikator zasobu, na którym operuje żądanie, żyje w ścieżce, nie w parametrach zapytania** — parametry zapytania są dla filtrów/sortowania/paginacji, nie głównego identyfikatora encji.
- **Nie duplikuj identyfikatora zasobu jednocześnie w ścieżce i w ciele żądania przy operacjach modyfikujących.** Ciało żądania aktualizującego zasób nigdy nie zawiera pola identyfikatora (nawet opcjonalnie) — identyfikator pochodzi wyłącznie ze ścieżki.
- **Odpowiedź na operację modyfikującą zwraca wynikowy stan zasobu, nie pusty komunikat typu "sukces".** Klient zwykle i tak potrzebuje nowego stanu — to oszczędza dodatkowe zapytanie odczytu. Wyjątek: usunięcie może zwracać brak treści; operacje masowe mogą zwracać podsumowanie liczbowe.
- Kształty żądań i odpowiedzi to nazwane, jawne struktury zdefiniowane w osobnym miejscu — nie inline w sygnaturze endpointu.

## 7. Kontrakty API i ich ewolucja

- **Jeden dokument specyfikacji API jest źródłem prawdy dla kontraktu**, z jawnym numerem wersji samej specyfikacji ORAZ osobnym numerem wersji API — to dwie różne wersje, obie muszą być udokumentowane.
- **Konwencje "wire format" (format danych na drucie) są jawnie spisane i wymuszone, nie domyślne** — konwencja nazewnictwa pól (może różnić się od wewnętrznej konwencji języka backendu), enumy jako czytelne, wyliczone stringi (nie magiczne liczby/kody).
- **Ujednolicony, maszynowo czytelny format błędów w całym API**, z osobnym rozszerzonym kształtem dla błędów walidacji pól — jeśli warstwa pośrednia (proxy/BFF) opakowuje ten kształt, konsument musi obsłużyć obie postaci.
- Niestandardowe nagłówki wymagane przez API są nazwane i udokumentowane tak samo jak endpointy — to część kontraktu, nie ukryty szczegół implementacyjny.
- **Typy/klienty konsumujące API generuj z kontraktu, nigdy nie pisz ręcznie** — zapobiega to rozjazdowi między specyfikacją a implementacją konsumenta.
- **Gdy specyfikacja kontraktu jest kopiowana ręcznie między repozytoriami (bez automatycznej synchronizacji), lokalna kopia jest cichym źródłem dryfu kontraktu.** Przy każdej zmianie API po stronie źródłowej: odśwież kopię, przegeneruj klienta, zweryfikuj że wersja w kopii zgadza się z wersją źródła — dopiero potem zaufaj wygenerowanym typom.
- **Topologia dostępu do API jest częścią specyfikacji kontraktu, nie tylko lista endpointów.** Dokument kontraktu jawnie mówi, kto ma prawo wywoływać API bezpośrednio (np. nigdy przeglądarka, tylko warstwa pośrednia) — to jest reguła bezpieczeństwa, nie szczegół implementacyjny do domyślenia się.

## 8. Integracja z API typu zapytaniowego (protokół, nie konkretny dostawca)

Dotyczy dowolnego API opartego o typowany język zapytań (nie tylko jednego konkretnego produktu) — reguły są na poziomie protokołu.

- Cała komunikacja z danym protokołem/API przechodzi przez jeden wspólny klient/wrapper — logika domenowa nigdy nie tworzy własnego surowego wywołania protokołu.
- Zapytania/operacje są zdefiniowane deklaratywnie w osobnych, nazwanych jednostkach, nie budowane inline jako stringi w kodzie.
- **Żądanie tworzące encję nie wysyła pól, za które odpowiedzialny jest serwer** (identyfikator, znaczniki czasu utworzenia/modyfikacji) — wysłanie ich ręcznie ryzykuje kolizję albo próbę nadpisania czegoś, co serwer i tak ustali sam.
- **Odniesienie do istniejącej powiązanej encji przekazuje wyłącznie jej identyfikator, nie pełny zagnieżdżony obiekt** — wysłanie pełnego obiektu w wielu API oznacza "utwórz nową powiązaną encję", co albo się nie powiedzie, albo utworzy duplikat.
- Formaty o ścisłej walidacji (znaczniki czasu, typy niestandardowe) muszą być wysyłane w dokładnie oczekiwanym formacie — API może przyjąć żądanie składniowo poprawnie, ale zwrócić puste/nieoczekiwane wyniki bez jawnego błędu, jeśli format nie pasuje.
- **Operator filtrowania działający na jednym typie pola może po cichu nie działać (zero wyników, brak błędu) na innym typie** — zweryfikuj wsparcie operatora dla konkretnego typu pola przed użyciem, zamiast zakładać, że działa wszędzie tak samo.
- Treść ustrukturyzowaną (dokumenty rich-text i podobne) konwertuj oficjalnym konwerterem dostawcy, nie ręcznie napisanym parserem drzewa dokumentu — ręczna implementacja niemal zawsze pomija subtelne przypadki.
- Po zmianie kontraktu zapytania (dodanie/zmiana pola) narzędzia generujące typy statyczne muszą zostać ponownie uruchomione — inaczej system typów może w ogóle nie widzieć nowego pola albo używać przestarzałego typu z cache.

## 9. Integracje z systemami zewnętrznymi

- Warstwa integracji obsługuje różne mechanizmy uwierzytelniania w sposób odizolowany od reszty aplikacji — logika biznesowa nie wie, jaki mechanizm autoryzacji stosuje dana zewnętrzna usługa.
- **Token dostępu z krótkim czasem życia jest cache'owany w pamięci procesu integracji i odświeżany przed wygaśnięciem (z buforem bezpieczeństwa)**, nie pobierany na nowo przy każdym wywołaniu.
- Klucze/sekrety uwierzytelniające żyją w konfiguracji środowiskowej/menedżerze sekretów, nigdy nie są commitowane do repozytorium.
- **Zasoby wymagające autoryzacji dostępowej (pliki na zewnętrznym storage) są rozwiązywane po stronie serwera, nie przez frontend** — przeglądarka nie potrafi dołączyć nagłówka autoryzacyjnego do prostego odnośnika/obrazka. Strategia rozwiązywania takiego adresu (podpisany URL z ograniczonym czasem życia, publiczny URL, proxy przez backend) jest szczegółem ukrytym za jedną funkcją — konsument dostaje już gotowy, działający adres.
- Wybór wzorca integracji (bezpośrednie wywołania, protokół zapytań typowanych, oficjalny klient dostawcy, klient generowany ze specyfikacji, dane zamockowane) zależy od charakterystyki API systemu docelowego — nie ma jednego uniwersalnego wzorca.
- Waliduj dane pochodzące z zewnętrznego systemu przed użyciem (patrz sekcja 3) — nigdy nie ufaj im bezkrytycznie.
- **Preferuj wywołania równoległe nad sekwencyjnymi, gdy operacje są od siebie niezależne** — istotnie poprawia to całkowity czas odpowiedzi. Wywołanie sekwencyjne tylko wtedy, gdy druga operacja faktycznie potrzebuje wyniku pierwszej.
- **Mechanizm "kombinowania wielu operacji równoległych" użyty z tylko jedną operacją jest zbędnym opakowaniem** — jeśli łączysz tylko jedno źródło danych, zwróć je bezpośrednio zamiast owijać w konstrukcję przeznaczoną do wielu źródeł.
- Preferuj oficjalne, wysokopoziomowe API biblioteki komunikacyjnej nad sięganiem do jej wewnętrznej, niższopoziomowej implementacji.
- **Nie zaszywaj na stałe identyfikatora konfiguracyjnego encji domenowej wewnątrz logiki serwisu** — identyfikator płynie jako parametr z żądania. Wyjątek: komponent z natury mający jeden, stały identyfikator (singleton) — wtedy zadeklaruj tę stałą jawnie na poziomie modułu, z komentarzem uzasadniającym, nie chowaj jej inline.
- Cache'uj kosztowne operacje z TTL dopasowanym do zmienności danych; klucz cache zawiera wszystkie parametry wpływające na wynik.
- Checklist dobrej integracji: uwierzytelnianie, walidacja wejścia/wyjścia, obsługa błędów z retry/timeout, logowanie z kontekstem, testy (w tym scenariuszy błędów), dokumentacja wymaganej konfiguracji.

## 10. Testowanie integracji

- **Testy jednostkowe integracji weryfikują logikę integracji (konfigurację, delegację, mapowanie, obsługę błędów), nie zachowanie samego zewnętrznego dostawcy.** Deterministyczne, szybkie, nigdy nie uderzają w prawdziwy system zewnętrzny.
- Co warto testować: konfigurację klienta (czy powstaje z wartości z poprawnego źródła, w tym scenariusz fallbacku); delegację (czy publiczna metoda usługi woła właściwą metodę zależności z tymi samymi parametrami, i właściwą zależność, gdy jest wybór między kilkoma); mapowanie odpowiedzi (dla reprezentatywnej odpowiedzi, wynik odpowiada oczekiwanemu kształtowi, w tym przypadki brzegowe — puste kolekcje, brakujące pola, błąd rzucony przez klienta); obserwowalny efekt obsługi błędu (zwrócona wartość/rzucony wyjątek), nie szczegóły implementacyjne.
- Mockuj zależności zewnętrzne na poziomie modułu, żeby nie zależeć od prawdziwej sieci ani szczegółów implementacyjnych wygenerowanego kodu.
- **Testy nie polegają na kolejności wykonania ani współdzielonym stanie między testami** — każdy test jest niezależny; jeśli test modyfikuje stan globalny, przywraca go po zakończeniu, nawet jeśli sam test zawiedzie.

## 11. Logowanie i obserwowalność

- **Każde żądanie do API jest automatycznie logowane w jednym, spójnym miejscu** (metoda/operacja, ścieżka/zasób, kod statusu, czas trwania, identyfikator korelacji) — nie ręcznie w każdym endpoincie osobno.
- **Structured logging (pola/obiekty, nie interpolowane stringi)** — ułatwia wyszukiwanie, filtrowanie, agregację w narzędziach do analizy logów.
- Rozróżniaj poziomy semantycznie: informacyjny (normalna operacja), debug (szczegóły diagnostyczne), warning (problem niekrytyczny — wolna odpowiedź zależności, brak trafienia w cache, użycie przestarzałej ścieżki), error (błąd ze stack trace).
- Standardowy zestaw pól kontekstowych: co się dzieje, kto wykonuje (jeśli dostępne), kiedy/jak długo, jaki wynik.
- **Nigdy nie loguj danych wrażliwych** (patrz sekcja 4) — nawet w logach diagnostycznych.
- Poziom logowania konfigurowalny per środowisko (produkcja: głównie błędy/ostrzeżenia; development: dodatkowo debug).
- **Rozróżnij "czy proces w ogóle żyje" (liveness) od "czy jest w stanie faktycznie obsługiwać żądania" (readiness)** — to drugie zależy od stanu krytycznych zależności zewnętrznych. Szeroko przyjęty wzorzec w systemach rozproszonych/orkiestrowanych, niezależny od języka. Endpoint zdrowia musi być tani i bez zależności od downstreamów, bo napędza automatyczne mechanizmy restartu/rutingu ruchu.

## 12. Konfiguracja i sekrety

- **Zbuduj artefakt raz, konfiguruj w czasie uruchomienia, wdrażaj wszędzie ten sam artefakt** — środowiska (dev/test/staging/prod) różni wyłącznie wstrzyknięta konfiguracja, nigdy zaszyta na etapie budowania wartość.
- **Twardy podział sekretów od jawnego configu, dostarczanych osobnymi mechanizmami** (magazyn sekretów szyfrowany vs zwykły magazyn konfiguracji). Żaden mechanizm oznaczający zmienną jako "bezpieczną do ujawnienia klientowi" (np. konwencja nazewnicza "publiczny") nie może być użyty dla sekretu.
- **Pełna inwentaryzacja zmiennych konfiguracyjnych jako żywy dokument, pogrupowana wg odpowiedzialności** (sekrety, uwierzytelnianie, routing, i18n, logowanie, flagi funkcji, integracje zewnętrzne, tożsamość środowiska) — z opisem przeznaczenia każdej. To coś więcej niż plik przykładowy z pustymi wartościami — to dokument kontraktowy z uzasadnieniem "po co" każdej zmiennej.
- **Dodanie nowej zmiennej konfiguracyjnej do kodu i dodanie jej do infrastruktury/pipeline'u budowania to dwa osobne, wymagane kroki** — pominięcie drugiego powoduje, że zmienna zostaje po cichu odcięta w zhermetyzowanym środowisku budowania/wdrożenia.
- Wartości biznesowe/flagi funkcji wstrzykiwane przez konfigurację środowiskową mają udokumentowane zachowanie fallback, gdy zmienna nie jest ustawiona.
- Ograniczenia uruchomieniowe jako część kontraktu wdrożenia: system plików tylko do odczytu poza jawnie wskazanymi zapisywalnymi katalogami, użytkownik bez uprawnień administracyjnych w kontenerze, ustandaryzowana zmienna portu nasłuchu.

## 13. Hooki cyklu życia encji / zdarzenia domenowe

Dotyczy dowolnego mechanizmu, w którym warstwa danych pozwala dopiąć własną logikę do momentów cyklu życia rekordu (walidacja, zapis, odczyt, usunięcie) — niezależnie czy nazywa się to "hook", "callback", "sygnał" czy "zdarzenie domenowe".

- **Dobieraj fazę hooka do rodzaju logiki, nie wrzucaj wszystkiego do jednej.** Formatowanie/normalizacja danych należy do fazy przed walidacją; reguły biznesowe zależne od operacji (np. auto-ustawienie znacznika czasu publikacji) — do fazy przed zapisem; efekty uboczne (powiadomienia, rewalidacja cache'u, audyt) — do fazy po udanym zapisie; pola wyliczane/pochodne — do fazy po odczycie.
- **Maskowanie/redakcja pola zależna od roli czytającego należy do fazy odczytu w warstwie danych, nie do zaufania, że klient sam ukryje to, czego nie powinien widzieć.** Jeśli pole ma być widoczne w innej postaci dla różnych ról (np. część adresu e-mail ukryta dla zwykłego użytkownika), transformacja dzieje się przy odczycie, po stronie serwera.
- **Zagnieżdżony zapis wywołany z wnętrza hooka musi być jawnie dopięty do tej samej transakcji/sesji co zapis wyzwalający.** Bez tego częściowa awaria (zapis główny się powiódł, zagnieżdżony nie, albo odwrotnie) zostawia dane w niespójnym stanie, którego nic nie wycofa — nawet jeśli mechanizm transakcyjny w warstwie danych istnieje, trzeba go świadomie przekazać dalej.
- **Operacja wywołana z wnętrza hooka, która wyzwala ten sam hook ponownie, tworzy nieskończoną pętlę.** Zapobiegaj temu jawną flagą przekazywaną przez kontekst zagnieżdżonego wywołania, wyłączającą ponowne odpalenie tego samego hooka dla tego konkretnego wywołania — nie polegaj na tym, że "to się nigdy nie zdarzy".

## 14. Organizacja pakietów/modułów

- Pakiet/moduł skupiony na jednej, dobrze zdefiniowanej odpowiedzialności (Single Responsibility na poziomie modułu, nie tylko funkcji).
- Minimalizuj zależności — dodawaj tylko faktycznie potrzebne; współdziel zależności wspólne przez mechanizm workspace/monorepo (jeśli dotyczy) zamiast duplikować w każdym pakiecie.
- Dokumentuj publiczne API pakietu (cel, instalacja/użycie, przykład) dla każdego pakietu przeznaczonego do reużycia.
- Wersjonuj konsekwentnie z jasną polityką (np. semantyczne wersjonowanie) i dokumentuj zmiany łamiące kompatybilność.
