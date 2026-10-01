# Standardy React + TypeScript + Next.js (przenośne, poza projektami)

> Ten dokument nie jest przywiązany do żadnego konkretnego klienta ani frameworka firmowego. Cel: zwięzłe, sprawdzone reguły dla dowolnej aplikacji na stosie React + TypeScript + Next.js (z GraphQL/BFF jako warstwą integracji), do zabrania między projektami.
>
> Format: pogrubiona reguła + krótkie uzasadnienie/przykład. Bez rozbudowanych listingów kodu — to ma być szybkie do przejrzenia i tanie do wczytania, nie pełna dokumentacja referencyjna.
>
> Pochodzenie: zdestylowane z realnego code review (architekt, ~200 komentarzy w dwóch niezależnych projektach na tym samym stosie) oraz z dojrzałych standardów wewnętrznych frameworku firmowego, oczyszczone z nazw konkretnych firm/frameworków/pakietów.

## 1. Zasady ogólne

Sześć reguł, z których wynika większość punktów poniżej.

**I. W repozytorium nie ma prywatnej połowy.**
Wszystko, co zostaje w repozytorium — komentarze, fixture, dane mockowe, nazwy — czyta obcy człowiek bez twojego kontekstu, a w produkcie komercyjnym czyta to też klient. Cokolwiek ma sens wyłącznie z twoim dzisiejszym kontekstem (notatki robocze, żargon wewnętrzny innego projektu, nazwiska konkretnych ludzi, prawdziwe dane osobowe) jest wyciekiem, nawet jeśli merytorycznie jest trafne. Test: czy ktoś, kto ma tylko to repozytorium, jest w stanie to zrozumieć i sprawdzić?

**II. Promień zmiany to jej konsumenci, nie jej diff.**
Zmiana w czymś współdzielonym zmienia zachowanie albo odbiór rzeczy, których nie dotknąłeś: rozluźniona gwarancja obowiązuje wszystkich konsumentów, wzmocniona prezentacja ujawnia treść, którą wcześniej wybaczano, zawężone uprawnienie zmienia zachowanie w kodzie, którego nie napisałeś. Obowiązek jest dwuczęściowy: wylicz konsumentów, a jeśli nie da się wymusić nowego kontraktu, to go uwidocznij — egzekwowalnym przykładem tam, gdzie masz czym (np. story w Storybooku, gdy nie ma testów), albo zdaniem w opisie PR-a. Cicha zmiana zachowania jest gorsza od głośnej, nawet gdy jest słuszna.

**III. Nazwa mechanizmu bywa szersza niż jego zasięg.**
`resetForm` nie resetuje całego stanu komponentu, nazwa skryptu testowego w `package.json` nie znaczy, że CI go odpala, zielony build nie znaczy, że funkcja działa, a odpowiedź integracji wyglądająca na sukces nie znaczy, że cokolwiek się zapisało. Zanim potraktujesz sygnał jako dowód, sprawdź, co ten sygnał faktycznie obejmuje.

**IV. Najsłabiej obejrzane jest to, co właśnie napisałeś.**
Kończąc zmianę, jesteś myślami w kodzie, nie w jego wyniku, więc przypadek, pod który ta zmiana powstała, jest dokładnie tym, którego nie oglądasz. Odwróć to celowo: uruchom ten konkretny scenariusz, dla którego to robiłeś, i obejrzyj wynik oczami użytkownika, nie autora. To jest też granica automatycznego przeglądu kodu — nawet solidny checklist code review nie złapie błędu, który wymaga spojrzenia na rezultat, nie na diff. W praktyce sprawdzało się to wielokrotnie: przegląd, który zaczynał się od uruchomienia aplikacji i kliknięcia przez nią, znajdował awarie niewidoczne w samym kodzie.

**V. Jedna reguła egzekwowana w dwóch miejscach zawsze się rozjedzie, jeśli nie zmienia się jednym ruchem.**
Rola wymagana przez warstwę prezentacji i rola wymagana przez zewnętrzny system, z którym integruje się backend; konfiguracja środowiskowa mapująca role na jedną nazwę wewnętrzną i lista ról faktycznie akceptowana przez ten system; tabela przejść stanu w warstwie pośredniej i walidacja tego samego przejścia w systemie źródłowym danych — to ten sam mechanizm w skali dwóch systemów. Działa identycznie w skali jednego repozytorium: stała roli skopiowana w kilku plikach, ten sam typ zadeklarowany dwukrotnie, dwa bliźniacze moduły traktujące to samo pole inaczej. Każda z tych par miała innego właściciela albo powstała w innym momencie, i zmieniła się osobno. Rozjazd nie objawia się przy wdrożeniu, tylko przy pierwszym przypadku, który w niego trafi. Kiedy ta sama wiedza (reguła biznesowa, model uprawnień, typ, wzorzec pola) istnieje w dwóch miejscach — zmieniaj je razem, jednym commitem, albo zawężaj stronę bardziej pozwalającą, zamiast poszerzać obie niezależnie.

**VI. Coś zepsute ma wyglądać zepsute, nie wyglądać dobrze.**
Fallback string wpisany na sztywno w warstwie mapującej dane, generyczny komunikat po błędzie backendu, zapis z brakującym wymaganym polem zamiast rzuconego wyjątku, rola bez zestawu uprawnień, która po cichu zwraca `false` na każde pytanie — to różne mechanizmy, jeden wspólny błąd: coś maskuje defekt tak, że wygląda na poprawne. To inny mechanizm niż zasada III (III mówi o zbyt ufnym poleganiu na nazwie/zakresie sygnału; ta mówi o kodzie, który aktywnie ukrywa prawdziwy stan przed każdym obserwatorem, nie tylko przed tobą). Brakująca etykieta ma się objawić jako pusty string, nie tekst zastępczy; błąd integracji ma rzucić konkretny wyjątek, nie zapisać się z dziurą; rola bez uprawnień ma być widoczna jako problem konfiguracji, nie jako "przycisk się nie pokazuje" bez wyjaśnienia. Cicha poprawność jest gorsza niż widoczna awaria, bo nikt nie wie, że trzeba coś naprawić.

## 2. TypeScript

- Nigdy `any` — zawsze konkretny typ albo `unknown`.
- Union types zamiast gołego `string` dla ograniczonego zbioru wartości.
- Type narrowing przez discriminated unions i funkcje-strażniki (`data is Article`) zamiast rzutowań.
- `unknown` zamiast `any`, gdy typ faktycznie nieznany — z walidacją przed użyciem.
- **Unikaj `Record<string, unknown>` jako leniwego escape hatcha.** Używaj go tylko dla naprawdę nieprzezroczystych kontenerów klucz-wartość (np. surowe nagłówki HTTP), nie dla kształtów, które są znane (odpowiedzi integracji, query buildery, modele domenowe). Pytanie kontrolne: czy naprawdę nie znasz tego kształtu, czy unikasz pracy napisania go?
- Kolokacja typów blisko miejsca użycia; wydzielanie współdzielonych typów do osobnego pliku typów.
- `interface` dla kształtów obiektów (rozszerzalnych), `type` dla unii/przecięć/typów obliczanych.
- Opisowe nazwy generyków (`TData`, `TResponse`, `TError`) zamiast gołego `T`.
- Domyślne typy generyczne, gdy to sensowne; ograniczenia generyków (`extends`) dla bezpieczeństwa typów.
- Ścisła konfiguracja kompilatora (`strict`, `noImplicitAny`, `strictNullChecks`) jako punkt wyjścia dla każdego nowego projektu.
- Jawne sprawdzenia `null`/`undefined` zamiast `@ts-ignore`; optional chaining; operator `!` tylko gdy naprawdę pewne.
- Rzutowania typów oszczędnie i tylko uzasadnione (dostęp do DOM, walidacja danych zewnętrznych) — nigdy jako ucieczka od błędu typów.
- Wykorzystuj wbudowane utility types (`Partial`, `Required`, `Pick`, `Omit`, `Readonly`, `Record`) zamiast redefiniować te same wzorce ręcznie.
- **Odczyt generycznego `Record<string, T>` przez klucz typu `keyof T & string` nadal zawęża się do `T | undefined` pod flagą typu `noUncheckedIndexedAccess`, gdy `T` samo jest parametrem generycznym.** Kompilator nie zna konkretnego zbioru kluczy generyku, więc traktuje odczyt jako potencjalnie pusty, mimo że klucz jest już statycznie sprawdzony jako poprawny. Wzorce `&&`/`||` do wyboru samego klucza (`code && map[code]) || fallback`) dodatkowo przeciekają `string | undefined` z gałęzi fallbacku. Działające rozwiązanie: jawny ternary bez `&&`/`||` do wyboru klucza, plus jeden, wąski, udokumentowany `as` na samym końcu, tylko przy właściwym odczycie wartości — nie szerszy cast obejmujący całą funkcję.

## 3. Komponenty i hooki React

- `'use client'` na górze pliku dotyczy całego pliku i jego importów — świadomie decyduj, gdzie przebiega granica server/client, nie dodawaj dyrektywy "na wszelki wypadek".
- Zawsze `Readonly<Props>` dla typu propsów; destrukturyzuj propsy w sygnaturze funkcji zamiast dostępu przez `props.x`; `React.ReactNode` dla `children`.
- `useState` dla prostego stanu lokalnego, `useReducer` dla złożonej logiki stanu — z typami `State`/`Action` jako discriminated union.
- Preferuj komponenty kontrolowane (łatwiejsze do testowania) nad niekontrolowanymi przez refy.
- Dane przekazywane z komponentu serwerowego do klienckiego muszą być serializowalne (bez funkcji, bez instancji klas).
- **Preferuj `useTransition` zamiast ręcznego `useState<boolean>` na stan ładowania** — to natywny wzorzec React 19 dla akcji asynchronicznych, konwencja nazewnicza: `[isSubmitting, startSubmitTransition]`.
- Bądź konsekwentny w wyborze `undefined` vs `null` dla podobnych opcjonalnych wartości stanu w tym samym komponencie — nie mieszaj obu bez powodu.
- Wyodrębniaj złożone handlery inline (wieloetapowe `onSubmit`/`onClick`) do nazwanych funkcji.
- **`useMemo`/`useCallback` nie na wyrost.** Memoizacja ma sens tylko, gdy stabilizuje wartość dla realnego konsumenta (zależność `useEffect`, dziecko opakowane w `React.memo`, kosztowne obliczenie, wartość kontekstu) — w przeciwnym razie to szum zwiększający czytelniczy koszt bez korzyści.
- **Kolejność hooków wewnątrz komponentu**: `useContext` → `useState` → `useRef` → `useCallback`/`useMemo` → `useEffect`, z pustą linią między grupami — czytelna, powtarzalna konwencja.
- **`forwardRef` jest przestarzały od React 19 — `ref` to zwykły prop.** Nie pisz `forwardRef`, nie ustawiaj `displayName` "na wszelki wypadek".
- `dangerouslySetInnerHTML` tylko dla zaufanej, sanityzowanej treści (np. rich text z systemu treści po przejściu przez sanitizer) — nigdy dla treści wpisanej przez użytkownika.
- **Unikaj `useEffect`, gdy wystarczy handler albo `startTransition`.** `useEffect` służy do synchronizacji z czymś poza Reactem (DOM, subskrypcje, timery, sieć), nie jako ogólne "uruchom kod po zmianie stanu" — większość przypadków da się rozwiązać bezpośrednio w handlerze, który wywołał zmianę.
- Uważaj na `setTimeout` + `useRef`-guard wewnątrz `useEffect` w React 18+ Strict Mode — podwójne montowanie w trybie deweloperskim potrafi uruchomić cleanup przed właściwym efektem i uśpić flagę-strażnika na stałe.
- Nie owijaj handlera w zbędną strzałkę (`onClick={() => onContinue()}` zamiast `onClick={onContinue}`) — owijaj tylko, gdy trzeba przekazać dodatkowy argument albo odrzucić nie-`void` zwrot.
- Nie rozpakowuj (spread) zwracanych wartości z hooków zwracających obiekty z nazwanymi metodami — zachowuje to czytelność pochodzenia wywołania (`alerts.remove(id)` zamiast gołego `remove(id)`), z wyjątkiem konwencjonalnych wzorców jak `useState`.
- **Limit wielkości komponentu: ~300 linii to sygnał ostrzegawczy, 500 to twardy blocker.** Objawy przegrzania: 10+ `useState`, 5+ `useEffect`, 3+ dialogi w jednym komponencie, wiele niepowiązanych async handlerów. Dziel po naturalnej granicy: per krok formularza, per sekcja widoku, per dialog.
- Discriminated union pattern (pole `type`/`kind` + `switch` + exhaustiveness check przez `never`) do bezpiecznej typowo obsługi wariantów danych.
- Sprawdzaj długość listy przed renderowaniem kontenera — nie renderuj pustego `ul`, renderuj dopiero gdy `items.length > 0`, inaczej empty state.
- Jeśli ten sam wzorzec UI pojawia się w 3+ miejscach, wyodrębnij go do współdzielonego komponentu/hooka.
- Interpolacja treści z placeholderami w string renderowana jako elementy React — użyj do tego gotowej, przetestowanej biblioteki (np. odpowiednik `react-string-replace`), nie ręcznego `string.replace()` połączonego z `dangerouslySetInnerHTML`.
- Stałe modułowe (mapy statusów, maksymalne rozmiary, wzorce regex, opcje sortowania) definiuj na poziomie modułu, nie wewnątrz funkcji komponentu — inaczej są realokowane przy każdym renderze.
- **Własny renderer na kolumnie/polu, które bywa `null` lub `undefined`, może się nigdy nie wykonać, jeśli wspólny mechanizm renderowania robi wczesny zwrot dla pustej wartości przed dotarciem do Twojej gałęzi.** Jeśli jesteś pierwszym miejscem w kodzie, które nakłada niestandardowy renderer na pole mogące być puste, to jest decyzja do podjęcia jawnie (naprawić wspólny mechanizm dla wszystkich konsumentów vs obejść lokalnie), nie coś do cichego obejścia.

## 4. Next.js: server/client/Suspense

- Wzorzec server + client split: pobieranie danych w komponencie serwerowym (`async function`), przekazanie wyniku jako propsy do komponentu klienckiego odpowiedzialnego za interaktywność — mniejszy bundle, bezpieczniejszy dostęp do zasobów backendowych, lepsze SEO.
- Komponent serwerowy nie może używać hooków, API przeglądarki ani event handlerów — jeśli potrzebujesz którejś z tych rzeczy, ten fragment musi być klienckim komponentem-dzieckiem.
- Wzorzec "renderer": osobny, cienki komponent otaczający właściwy widok granicą `Suspense`, z `key` zależnym od identyfikatora danych (dla poprawnego re-montowania przy zmianie) i fallbackiem dopasowanym kształtem do realnej treści (skeleton, nie generyczny spinner).
- **`Suspense` żyje w komponencie-rendererze, nie w komponencie serwerowym, który sam czeka na dane.** Serwerowy `async` component sam już blokuje na `await` — otaczający go `Suspense` nigdy się nie wyzwoli, bo do czasu jego wyrenderowania dane już są gotowe. Granica Suspense ma sens dopiero jeden poziom wyżej, tam gdzie ktoś czeka na wynik tego komponentu.
- Dynamiczny import (`next/dynamic`) do redukcji rozmiaru bundla dla ciężkich komponentów, z jawną opcją `ssr: false` dla komponentów zależnych od `window`/`document`.
- Przekazuj kontekst żądania (locale, strefa czasowa, token uwierzytelniający z sesji) jawnie do warstwy pobierania danych w komponencie serwerowym — nie polegaj na globalnym stanie, który mógłby nie być dostępny na serwerze.
- Owijaj pobieranie danych w komponencie serwerowym w try-catch tak, żeby sam komponent nie rzucał nieobsłużonego wyjątku: zadeklaruj zmienną wyniku poza `try`, przypisz w środku, w `catch` zwróć `null`/fallback UI i zaloguj błąd po stronie serwera.
- Granica błędu (`error.tsx` w App Routerze) do przechwytywania nieobsłużonych wyjątków — świadomie zdecyduj o granularności (jedna root-level granica vs granice per segment), zwłaszcza przy dynamicznym/catch-all routingu.

## 5. Dostępność

- Semantyczny HTML zamiast divów na wszystko (`nav`, `main`, `article` zamiast `div className="nav"`).
- Nawigacja przez `a`, akcje przez `button` — nie mieszaj (nie rób linku z `onClick` usuwającym coś, ani buttona nawigującego).
- Każdy element interaktywny dostępny z klawiatury; jeśli używasz `div onClick`, dodaj `role="button"`, `tabIndex={0}` i obsługę `onKeyDown` (Enter/Spacja) — a lepiej użyj od razu `button`.
- Zawsze widoczny wskaźnik fokusu (`focus:ring`/`focus:outline`), naturalny porządek DOM zamiast ręcznego `tabIndex`.
- Skip-linki do głównej treści.
- Obrazy informacyjne mają opisowy `alt`; dekoracyjne mają `alt=""`; nigdy brak `alt` ani bezużyteczny alt typu "image".
- Ikona z tekstem obok → `aria-hidden="true"` na ikonie; ikona samodzielna → `aria-label` na przycisku.
- Każdy input ma powiązany `label` (`htmlFor`); nigdy sam placeholder zamiast labela.
- Błędy walidacji: `aria-invalid` + `aria-describedby` wskazujący na `role="alert"` z komunikatem; wymagane pola: widoczna gwiazdka + `aria-required="true"`.
- Minimalne kontrasty: tekst zwykły 4.5:1, duży tekst 3:1, komponenty UI 3:1. Nie polegaj wyłącznie na kolorze — dodaj ikonę/tekst obok kolorowego wskaźnika statusu.
- Landmarki (`role="banner"/"navigation"/"main"/"contentinfo"`), `aria-live="polite"` dla komunikatów niekrytycznych, `role="alert" aria-live="assertive"` dla krytycznych.
- Logiczna hierarchia nagłówków, bez przeskakiwania poziomów.
- Modale: fokus na przycisk zamknięcia przy otwarciu, focus trap, `role="dialog" aria-modal="true"`.
- Powtarzające się elementy renderuj jako `ul`/`ol` + `li`, nie jako sekwencję `div`.
- Obszar klikalny interaktywnego elementu (kafelek, karta) pokrywa cały wizualny obrys, nie tylko wewnętrzny tekst.
- Testowanie: automatyczne (axe-core) jako baseline, ale zawsze też manualnie klawiaturą i przy powiększeniu 200% — automatyczne narzędzia łapią ułamek realnych problemów.

## 6. Stylowanie (Tailwind/CVA) i responsywność

- Podejście utility-first zamiast custom CSS dla większości przypadków; helper do warunkowego łączenia klas (`cn`/`clsx`) zamiast ręcznej konkatenacji stringów.
- Warianty komponentu przez bibliotekę do tego przeznaczoną (np. `class-variance-authority`), nie przez ręczne ify w className.
- Design tokeny (semantyczne nazwy: `bg-primary`) zamiast hardkodowanych wartości (`bg-blue-500`) — ułatwia zmianę motywu i dark mode.
- Stany fokusu i disabled zawsze stylowane (`focus:ring`, `disabled:opacity-50`) — to też wymóg dostępności, nie tylko estetyki.
- Nie mieszaj paradygmatów w jednym miejscu (utility + custom CSS klasy na tym samym elemencie) i nie walcz z frameworkiem CSS stylami inline.
- Mobile-first: bazowe style dla najmniejszego ekranu, potem enhancement w górę przez breakpointy — nigdy odwrotnie.
- Touch targety min. 44×44px, z odpowiednim odstępem między elementami dotykowymi.
- Priorytet treści na mobile — najważniejsza treść pierwsza w DOM; można odwrócić kolejność wizualną (`order`, `flex-row-reverse`) na większych ekranach, ale nie chowaj krytycznej treści na żadnym rozmiarze ekranu.
- Tabele responsywne — układ kart na mobile, tabela na desktopie, zamiast poziomego przewijania jako jedynej odpowiedzi.
- Nie zakładaj, że duży ekran = brak dotyku (duże ekrany dotykowe istnieją) — `@media (hover: hover)` zamiast `md:hover:` tam, gdzie to ma znaczenie.
- Nie wyłączaj możliwości zoomu (`user-scalable=no`); minimalny rozmiar tekstu w inputach 16px (zapobiega auto-zoomowi na iOS).

## 7. Wydajność

- Komponent obrazu frameworka (np. `next/image`) zamiast gołego `img` — automatyczna optymalizacja formatu, lazy loading, zapobieganie przesunięciu layoutu. `priority` dla obrazów widocznych od razu, `sizes` dla responsywnych, zawsze podawaj `width`/`height` (albo `fill` w kontenerze o znanym rozmiarze).
- Suspense boundaries dla strumieniowania — szybka treść renderuje się natychmiast, wolna strumieniuje osobno; wiele niezależnych granic dla niezależnych sekcji.
- Dynamiczny import dla ciężkich komponentów (edytory, wykresy); `ssr: false` dla komponentów czysto klienckich; preload na hover przed interakcją.
- `React.memo` dla czystych, kosztownych komponentów; `useMemo` dla kosztownych obliczeń (filtrowanie/sortowanie/transformacje dużych tablic) — nie dla prostych operacji (patrz sekcja 3).
- Importy nazwane zamiast `import * as X` — ułatwia tree-shaking.
- Równoległe pobieranie danych (`Promise.all`) zamiast sekwencyjnych `await` jeden po drugim — to jedna z najczęściej pomijanych, a najbardziej opłacalnych optymalizacji w komponentach serwerowych.
- Skeletony ładowania dopasowane wymiarami/strukturą do rzeczywistej treści, nie generyczny spinner na środku ekranu.
- `next/font` (albo odpowiednik frameworka) z `display: 'swap'` zamiast ręcznego `@font-face`.
- Sprawdzaj rozmiar zależności przed dodaniem (bundle analyzer) — patrz sekcja 16.
- Monitoruj Core Web Vitals (LCP < 2.5s, FID/INP, CLS < 0.1) i React DevTools Profiler dla realnych re-renderów, nie zgadywania.

## 8. Formularze i walidacja

- Biblioteka zarządzania formularzem + biblioteka schematu walidacji (np. odpowiednik pary Formik+Yup albo React Hook Form+Zod) zamiast ręcznego zarządzania stanem pól i błędów.
- Walidacja warunkowa (pole wymagane zależnie od wartości innego pola) i porównanie pól (np. potwierdzenie hasła) przez mechanizmy schematu, nie ręczne ify w handlerze submitu.
- Błędy renderuj przez komponent typu Alert z ikoną i tekstem, nie goły `span`; `aria-invalid` gdy pole jest dotknięte i ma błąd (patrz sekcja 5).
- Walidacja po stronie serwera (server action / endpoint) to ostatnia linia obrony — walidacja we froncie jest wyłącznie UX-em, nigdy jedynym zabezpieczeniem.
- Sanityzacja przed `dangerouslySetInnerHTML` (np. DOMPurify) — React domyślnie escapuje treść, ale ta ścieżka omija to zabezpieczenie.
- **Waliduj URL przed przekierowaniem pochodzącym od użytkownika** (protokół + allowlista domen) zanim wywołasz nawigację programową — inaczej masz otwarte przekierowanie (open redirect).
- Typowanie jako walidacja kompilacyjna: union types zamiast `string` dla ograniczonego zbioru wartości, discriminated union dla stanu formularza (`idle`/`submitting`/`success`/`error`) z wyczerpującym `switch`.
- **Po udanym zapisie zresetuj cały stan, który formularz posiada — nie tylko ten zarządzany przez bibliotekę formularza.** `resetForm()` czyści pola zarządzane przez bibliotekę, ale osobny stan Reacta obok (np. przełącznik trybu, wybrana zakładka) zostaje z poprzedniej wartości i cicho dziedziczy się do następnego wypełnienia. Jeśli jakaś "lepkość" stanu jest celowa, napisz to w komentarzu — inaczej czyta się jako przeoczenie.
- **Przełącznik zmieniający znaczenie albo adresata wprowadzanych danych (np. widoczność publiczna/prywatna) musi być widoczny w miejscu akcji, nie tylko gdzieś obok.** Łatwo go przeoczyć przed wysłaniem — etykieta przycisku albo kolor formularza powinny iść za aktualnym trybem, nie być statyczne. Domyślny tryb powinien być tym bezpieczniejszym.
- **Placeholder i komunikat błędu to dwie różne role UI — jedna zmienna tekstowa nie powinna służyć obu naraz.** Prędzej czy później któraś rola odziedziczy treść nieadekwatną do siebie (np. placeholder pola użyty jako treść błędu walidacji, bo "przecież już tam jest jakiś tekst"). Rozdziel je od razu, przy pierwszym takim skopiowaniu.
- **Podniesienie rangi wizualnej prezentacji ujawnia każdy skrót w treści, którą komponent wyświetla.** Tekst niezauważalny jako mały, szary podpis zaczyna czytać się zupełnie inaczej (np. jak komunikat o awarii systemu) po zamianie na wyróżniony komponent typu Alert. Zmieniając sposób wyświetlania czegoś, przeczytaj jeszcze raz to, co się faktycznie wyświetla.

## 9. i18n

- Rozstrzygnij jawnie, co jest treścią domenową (tłumaczoną per encja/strona, zarządzaną przez osoby odpowiedzialne za treść) a co jest stringiem UI frameworka (tłumaczonym raz, żyjącym w repozytorium) — i nie mieszaj tych dwóch źródeł prawdy o tekście.
- Przekazuj locale jawnie i konsekwentnie przez cały łańcuch wywołań do warstwy danych, zarówno w komponentach serwerowych, jak i klienckich — nie polegaj wyłącznie na globalnym kontekście frameworka i18n.
- Nawigacja świadoma locale (linki, przekierowania, odczyt bieżącej ścieżki) przez wrapper biblioteki i18n, nie przez gołe API routingu frameworka — inaczej przekierowanie po akcji zgubi prefiks języka.
- Etykiety, które kod ustawia bezwarunkowo, muszą być typowane jako wymagane, nie opcjonalne (patrz sekcja 10) — inaczej `undefined` wycieka do DOM.
- Formatowanie dat/liczb przez mechanizm biblioteki i18n zgodny z aktywnym locale, nie ręczne stringi.
- Jeśli strona ma wersje językowe pod osobnymi URL-ami, zadbaj o meta-informację o alternatywnych wersjach językowych (SEO) dla każdej strony.
- Jeśli masz warstwę pośredniczącą (BFF) obsługującą wiele locale, przekazuj locale jawnie w nagłówku/parametrze każdego żądania do niej — nie zakładaj, że wie to skądinąd.

## 10. Obsługa błędów we frontendzie

- **Zawsze weryfikuj, że zapis faktycznie się powiódł — nie ufaj samemu kształtowi odpowiedzi integracji.** "Sukces" po stronie twojego kodu może nie znaczyć, że dane zapisały się po drugiej stronie.
- Sprawdzaj kod/status błędu i rozróżniaj typy (autoryzacja, walidacja, błąd serwera) zamiast traktować wszystko jako jedną kategorię — zbiorcze traktowanie błędów maskuje prawdziwą przyczynę i utrudnia diagnozę.
- **Nigdy nie renderuj surowego komunikatu błędu z backendu użytkownikowi.** Zmapuj stabilny kod błędu na zlokalizowany, przyjazny komunikat przez dedykowaną funkcję mapującą — niezależnie czy źródłem etykiet jest system treści, plik i18n czy stała w kodzie. Fallback do jednej generycznej etykiety, gdy zabraknie konkretnej, powoduje że sukces potrafi "ogłosić się" jako błąd i odwrotnie.
- Generyczny komunikat toastowy ukrywa realną regresję — jeśli backend zwraca kod, pokaż komunikat zależny od kodu.
- Opcjonalne dane (telemetria, dowód, dekoracja) jadące w tym samym żądaniu co właściwy payload nie mogą wywalać całej operacji, jeśli tylko ten dodatek się nie powiedzie.
- Komponenty serwerowe: try-catch wokół pobierania danych (patrz sekcja 4); komponenty klienckie: jawna obsługa `try/catch/finally` ze stanem `loading`/`error`; API routes i server actions: spójny format odpowiedzi błędu (`{ success, data | error }`), poprawne kody HTTP, walidacja wejścia przed przetwarzaniem.
- **Komunikaty błędów mają być przyjazne, nie techniczny żargon** — nie pokazuj kodów błędów sieciowych ani stack trace'ów; tłumacz na język zrozumiały dla użytkownika i, gdy to możliwe, dawaj kolejny krok (przycisk ponów, link do wsparcia), nie zostawiaj samego "wystąpił błąd".
- Graceful degradation dla brakujących opcjonalnych danych — optional chaining zamiast zakładania, że pole istnieje; fallback content zamiast pustego miejsca.
- **`catch` w kodzie klienta, który zbiera różne przyczyny w jeden definitywny komunikat, mimo że część z nich nigdy nie dotarła do serwera, jest błędem obserwowalności, nie tylko UX-u.** Sprawdź, czy poziom logowania jest właściwy: decyzja podjęta po stronie serwera powinna się logować po stronie serwera, nie ginąć w jednym uchwyceniu błędu na froncie.
- **Nieudana próba asynchronicznego załadowania danych nie powinna być cache'owana jako pusty, "udany" wynik (np. `setData([])` w `catch`).** Zostaw stan nietkniętym/`undefined`, żeby kolejna próba (kolejne otwarcie dialogu, kolejne zamontowanie) sama spróbowała jeszcze raz — zamiast utknąć trwale na jednej nieudanej próbie, bo stan wygląda na "już pobrany, po prostu pusty".
- **Sygnał "dane mogą być niepełne/ucięte" nie może być schowany za tym samym warunkiem pustości, który go wywołuje.** Jeśli komunikat ostrzegawczy renderuje się tylko wtedy, gdy lista ma już przynajmniej jeden element (`items.length > 0 && (... lista ... ostrzeżenie ...)`), to dokładnie w scenariuszu, dla którego ostrzeżenie powstało — realne dane istnieją, ale żadne nie trafiły do widocznego okna, więc lista jest pusta — ostrzeżenie znika razem z listą. Warunek renderowania kontenera musi obejmować "lista ma elementy LUB flaga niepełności jest ustawiona", z samą listą (nie całym kontenerem) osobno bramkowaną na długość.
- **Gdy dodajesz prawdziwe oznaczenie czegoś (flagę, pole), sprawdź czy gdzieś indziej w kodzie nie ma już heurystyki-przybliżenia tej samej rzeczy — i podmień ją, nie zostawiaj obok.** Przykład: banner cytujący "najnowszy publiczny komentarz" jako przybliżenie "treści ostatniej prośby o dane" działał tylko dopóki nic innego nie dopisało kolejnego zwykłego komentarza po prośbie. Dodanie prawdziwej flagi (np. `isDataRequest`) gdzie indziej w tej samej zmianie nie naprawia automatycznie heurystyki, która istniała tylko dlatego, że flagi wcześniej nie było — trzeba jawnie poszukać miejsc, które zgadywały to, co flaga teraz mówi wprost.

## 11. GraphQL na poziomie protokołu

Reguły niezależne od konkretnego dostawcy GraphQL — dotyczą każdej integracji z API typu GraphQL, niezależnie kto je hostuje.

- **Mutacje: nie wysyłaj z powrotem pól generowanych przez serwer** (`id`, `createdAt`, `updatedAt` i podobne) przy tworzeniu encji — serwer je ustawia; wysłanie ich z powrotem zaprasza do duplikatów albo cichego nadpisania.
- **Relacje: przekazuj samo `id`, nie cały zagnieżdżony obiekt** — jeśli relacja wskazuje na istniejący rekord, referencja wystarczy; wysyłanie pełnego obiektu miesza tworzenie z aktualizacją i zwiększa szansę na rozjazd danych.
- **Custom scalary są rygorystyczne co do formatu.** Np. scalar typu data-czas zwykle wymaga pełnego ISO 8601 z komponentem czasu, nie samej daty — API odrzuci albo (gorzej) po cichu źle zinterpretuje skrócony format.
- **Operatory filtrowania nie działają jednolicie na każdym typie kolumny.** Filtr tekstowy (`contains`) na polu typu rich text/JSON może po cichu zwrócić zero wyników zamiast błędu, bo silnik oczekuje innego kształtu wejścia dla tego typu pola. Przetestuj filtr na polu, na które go nakładasz, nie zakładaj że działa jak na zwykłym stringu.
- **Strukturalną treść (rich text) konwertuj oficjalnym konwerterem dostawcy, nie własnym walkerem po drzewie/AST.** Format bywa nietrywialny i zmienia się między wersjami; własny parser rich text to duplikacja pracy, która się psuje przy każdej aktualizacji.

## 12. BFF jako wzorzec architektoniczny

- **Jeśli istnieje dedykowana warstwa pośrednicząca (BFF) między frontendem a usługami zewnętrznymi, zawsze korzystaj z jej klienta — nigdy nie omijaj jej bezpośrednimi wywołaniami do zewnętrznych API z frontendu.** Cachowanie, mapowanie błędów i normalizacja danych powinny żyć w jednym miejscu, nie być powielane ad hoc przy każdym wywołaniu.
- **Adapter/port pattern dla integracji zewnętrznych: abstrakcyjny kontrakt (interfejs/token wstrzykiwany przez DI) oddzielony od konkretnej implementacji.** Pozwala podmienić dostawcę (np. jeden CMS na inny) bez zmian we frontendzie i w kodzie domenowym — konkretna implementacja zna szczegóły API dostawcy, kontrakt ich nie zna.
- Format translation (mapowanie surowego kształtu danych integracji na znormalizowany model domenowy) żyje w warstwie integracji, nie w komponencie ani w logice domenowej — komponent nigdy nie powinien znać kształtu danych konkretnego dostawcy.
- **Ścieżka zapisu i ścieżka odczytu muszą się zgadzać co do pola, którego dotyczą.** Jeśli operacja tworząca encję zapisuje dane w jednym polu, a zapytanie listujące/odczytujące czyta inne (bo np. ktoś dodał wygodniejsze do zapisu pole, nie aktualizując zapytań odczytu) — jedna ścieżka będzie zawsze pusta, mimo że dane technicznie istnieją. Po dodaniu/zmianie pola sprawdź drugą stronę przepływu.
- **Nigdy nie przekazuj surowego błędu integracji do użytkownika.** Backend/warstwa pośrednia mapuje błąd zewnętrzny na stabilny, własny kod błędu; frontend mapuje ten kod na przyjazny komunikat (patrz sekcja 10) — dwa niezależne mapowania, nie jeden skrót na oślep.
- Orkiestracja wielu niezależnych wywołań do usług zewnętrznych (np. dane oraz konfiguracja treści z osobnego źródła) powinna iść równolegle, nie sekwencyjnie jedno po drugim.
- **Rozdziel warstwę sesji/tożsamości od warstwy autoryzacji.** To, że ktoś jest zalogowany (sesja istnieje, token jest ważny) to inne pytanie niż to, co wolno mu zrobić (autoryzacja na konkretny zasób/akcję) — mechanizm uwierzytelniania (np. NextAuth po stronie frontendu) i logika autoryzacji w BFF to dwie oddzielne odpowiedzialności, nawet jeśli oba żyją "blisko siebie" architektonicznie.
- Akcje masowe (bulk) — jeden endpoint zbiorczy przyjmujący listę, nie pętla N pojedynczych zapytań wywołana z frontu.
- Nie zostawiaj gałęzi backendu, do których UI nie potrafi w praktyce dotrzeć (np. filtr przez konkretną wartość, gdy front zawsze wysyła tylko jedną stałą) — dorób wejście w UI albo uprość kod do tego, co realnie jest wywoływane.
- **Kontrakt oparty na łańcuchu tekstowym między dwiema niezależnie zmienianymi częściami systemu (kod błędu, nazwa zasobu uprawnień, klucz etykiety) zawodzi po cichu i permisywnie, nie głośno.** Jeśli obie strony po prostu wpisują ten sam string w dwóch miejscach, rozjazd nie wywali błędu kompilacji — po prostu przestanie działać dla jednej strony. Przypnij obie strony do tej samej stałej/typu, nie do kopii literału, i opisz kontrakt w miejscu neutralnym, nie tylko w głowie autora.

## 13. Testowanie i Storybook

- Storybook (albo odpowiednik) jako narzędzie do budowy i przeglądu komponentów w izolacji — nie tylko dokumentacja, realne środowisko do sprawdzenia stanów.
- Konwencje nazewnicze historii: PascalCase, opisowe nazwy (`Basic`, `WithActions`, `LoadingState`, `EmptyState`), krótki komentarz nad historią wyjaśniający jej cel, gdy nie jest oczywisty.
- **Checklist typów historii do pokrycia dla każdego komponentu**: wariant domyślny/podstawowy, warianty wizualne, stany interaktywne, stany danych (ładowanie/pusto/błąd/sukces), przypadki brzegowe (długi tekst, dużo elementów), responsywność, dostępność.
- Dane mockowe w historiach powinny pasować do domeny właściwej dla Twojej aplikacji, nie być losowym demo niezwiązanym z niczym — ułatwia to wykrycie realnych problemów (np. tekst nie mieszczący się w layoucie), które generyczne "Item 1, Item 2" by ukryły.
- **Rozluźnienie gwarancji we współdzielonym komponencie (np. nowy typ dopuszczalnej wartości propa) wymaga miejsca, w którym nowy kontrakt jest wykonywalny, nie tylko opisany w prozie.** Jeśli pakiet nie ma testów, ale ma Storybooka, dodaj historię demonstrującą właśnie ten nowy przypadek — to najtańsza egzekwowalna dokumentacja, jaka jest dostępna. Przy zmianie kontraktu przejrzyj też istniejące przykłady, bo mogły uczyć starego, już nieaktualnego zachowania.
- **Meta-zasada "discovery before invention": zanim napiszesz nowy komponent/wzorzec, sprawdź, czy już istnieje w bibliotece komponentów projektu** (prymitywy → komponenty złożone → istniejące podobne funkcje) — nie zgaduj ścieżek importu ani nie wymyślaj na nowo wzorca, który już jest w repozytorium.
- Gdy nie ma pasującego istniejącego wzorca, nie improwizuj w ciemno — poproś o referencję (zrzut ekranu, link do projektu, nazwę podobnego istniejącego komponentu) i przedstaw krótki plan (jakich prymitywów użyjesz, jaki layout) zanim napiszesz cały JSX.
- **Nazewnictwo atrybutów testowych (`data-testid` lub odpowiednik): konwencja `{zakres}-{rola}-{akcja}`, nie przypadkowe stringi.** Dla list identyfikuj element po jego prawdziwym ID, nie po indeksie w tablicy (indeks zmienia się przy sortowaniu/filtrowaniu i cicho podmienia test pod inny element). Złożone prymitywy (dialog, combobox) dostają prefiks + wyprowadzone z niego podczęści, nie osobne niepowiązane identyfikatory. Komponenty biblioteki UI powinny forwardować atrybut testowy przez rest-propsy, żeby dało się go nadać z zewnątrz.
- **Zdarzenie z przeglądarki wart przesłania na serwer, gdy inaczej nie zostawia śladu w logach/monitoringu** (np. błąd frontendowy, który nigdy nie trafia do żadnego requestu API). Zasady takiego logowania client→server: nigdy nie blokuj ani nie rzucaj z powodu samego logowania, żadnych danych osobowych w treści zdarzenia, statyczne/przewidywalne komunikaty (nie interpolowany wolny tekst), unikaj duplikowania tego, co i tak już loguje backend dla tego samego żądania.
- Test, który przypina niewłaściwą rzecz, jest gorszy niż brak testu — daje fałszywe poczucie pokrycia. Test, którego nazwa/asercja zakazuje zmiany, której nikt nie miał zamiaru zakazywać (przypadkowo utrwalony szczegół implementacji, nie kontrakt), będzie kiedyś blokował poprawną zmianę albo zostanie bez pytania poprawiony pod nowy kod, tracąc sens. Przy dodawaniu testu sprawdź, czy to, co przypina, jest kontraktem, nie przypadkiem.
- Zanim potraktujesz zielone testy jako sygnał jakości, sprawdź, czy CI faktycznie je odpala — a nie tylko czy komenda istnieje w konfiguracji projektu.

## 14. Organizacja kodu, nazewnictwo, komentarze

- Single Responsibility dla plików — rozdzielaj logikę/typy/style do osobnych plików zamiast mieszania odpowiedzialności w jednym.
- Kolejność importów: zależności zewnętrzne → pakiety wewnętrzne współdzielone → aliasy ścieżek → importy relatywne, najlepiej sortowana automatycznie przez narzędzie.
- Preferuj alias ścieżki nad długi import relatywny (`../../../`) — jeśli import przekracza jeden poziom katalogów w górę, użyj aliasu.
- Małe, fokusowane funkcje o jednej odpowiedzialności; docelowy rozmiar funkcji orientacyjnie do ~50 linii, powyżej — rozważ ekstrakcję.
- Wczesne zwroty i guard clauses zamiast głębokiego zagnieżdżenia `if-else`; `switch` zamiast długiego łańcucha `if-else` dla wielu wariantów.
- DRY: wyodrębniaj powtarzającą się logikę do funkcji/hooków/modułów, gdy używana w 2+ miejscach — nie wcześniej (patrz niżej o YAGNI).
- **Nowa abstrakcja z jednym wołającym to odwrotny błąd — YAGNI, nie DRY.** Wspólny helper/interfejs/generyczna funkcja używana tylko w jednym miejscu nie skraca niczego, tylko dodaje poziom wskazania, przez który trzeba przejść, żeby zrozumieć co się dzieje. Wydzielaj przy drugim realnym wystąpieniu, nie przy pierwszym "na wszelki wypadek".
- **Zduplikowana reguła biznesowa egzekwowana w dwóch miejscach zawsze się rozjedzie** (patrz Zasada V) — jeśli druga kopia jest potrzebna do czegoś innego (np. zasilenia dropdowna w UI), oznacz ją wprost jako projekcję pod ten cel, nie jako źródło prawdy.
- **Jeśli odduplikowujesz dwa niemal identyczne komponenty, zakwestionuj najpierw projekt (design), a nie tylko kod.** Duplikacja w kodzie bywa objawem duplikacji w designie (dwa osobne panele robiące to samo zamiast jednego z przełącznikiem) — wydzielenie wspólnego komponentu bez zakwestionowania układu utrwala zły projekt zamiast go naprawić.
- Nie commituj katalogów z kodem generowanym (np. przez codegen GraphQL/OpenAPI) — dodaj do `.gitignore`, generuj w skrypcie przygotowawczym/CI.
- Wyodrębniaj regexy do nazwanych stałych modułowych zamiast inline magic regex.
- Usuwaj martwy kod natychmiast — nieużywane importy/funkcje/komponenty, bez zostawiania zakomentowanego kodu "na wszelki wypadek" (historia gita go pamięta).
- Nazewnictwo: camelCase dla zmiennych/funkcji, PascalCase dla klas/typów/interfejsów/komponentów, kebab-case dla nazw plików, UPPER_SNAKE_CASE dla prawdziwych stałych kompilacyjnych. Prefiksy boolean (`is`/`has`/`should`/`can`). Interfejsy bez prefiksu `I` — opisowy sufiks zamiast tego. Unikaj generycznych nazw (`data`/`result`/`temp`/gołe `Params`).
- **Komentuj "dlaczego", nie "co"** — nieoczywistą logikę biznesową, obejścia ograniczeń biblioteki/frameworka, kod wrażliwy na bezpieczeństwo, złożone transformacje, punkty integracji z zewnętrznymi usługami. Nie komentuj tego, co widać z samego kodu.
- **Komentarze bez odniesień czasowych ("evergreen")** — nie "TODO: usunąć po migracji (dodane w styczniu)", tylko "TODO: zmigrować do nowego wzorca adaptera". Bez changelogów w komentarzach ("nowe", "zaktualizowane", daty napraw) — to zadanie historii gita.
- **Komentarz, który przeczy kodowi, jest błędem — i to groźniejszym niż literówka.** Aktualizuj komentarz razem z kodem podczas refaktoryzacji; komentarz oderwany od tego, co dokumentuje (bo kod się przesunął w tym samym refaktorze), czyta się jako wciąż prawdziwy, mimo że opisuje już nieistniejący stan.
- **Komentarz narracyjny ("wcześniej robiliśmy X, teraz Y") zamiast opisu aktualnego zachowania starzeje się z każdym kolejnym refaktorem** — za kilka zmian nikt już nie wie, czy "wcześniej" wciąż jest prawdziwe. Opisz, co kod robi teraz; historię zostaw commitowi.
- **Komentarz może wskazywać wyłącznie na rzeczy osiągalne z tego repozytorium.** Odwołania do prywatnych notatek, nazwisk konkretnych osób, materiałów planistycznych leżących gdzie indziej nie rozwiązują się dla nikogo poza autorem. Napisz wprost jedno zdanie o co chodziło, zamiast odsyłacza donikąd.
- **Dane testowe to kod wysyłany dalej (repozytorium, czasem klientowi) — fixture, mocki i historie Storybooka tak samo jak reszta.** Wyłącznie dane fikcyjne, wyłącznie domeny przykładowe (`example.com`), nigdy prawdziwe imię/nazwisko/e-mail.
- README pakietu/modułu: cel, instalacja/użycie, kluczowe eksporty, działający przykład — zwięźle, aktualizowane razem z kodem.
- **Nie umieszczaj w kodzie i komentarzach nazwy własnej Twojej firmy/organizacji, jeśli ten sam kod może kiedyś trafić do innego klienta albo zostać wydzielony do osobnego produktu.** Pisz neutralnie ("zespół wsparcia", "warstwa integracji") zamiast nazwy konkretnej firmy — dotyczy to też nazw pól i stałych, nie tylko prozy w komentarzu.
- **Jeśli dwa podobne moduły/komponenty traktują to samo pole albo zachowanie inaczej (jeden filtruje coś po stronie serwera, drugi wysyła zawsze) — to zwykle niezamierzona niespójność, nie świadomy wybór.** Sprawdź "bliźniaczy" plik/moduł przed uznaniem wzorca za gotowy.

## 15. Kontrola wersji i PR

- Nazewnictwo branchy wg prefiksu (`feature/`, `fix/`, `docs/`, `refactor/`, `chore/`) z opisową, konkretną nazwą.
- Conventional Commits: `type(scope): opis`, tryb rozkazujący, zwięźle, wyjaśniaj "dlaczego" nie "co", odnoś się do issue gdy dotyczy.
- PR skupiony na jednej funkcji/poprawce — nie mieszaj niepowiązanych zmian. Preferuj małe PR-y (<200 linii); duże (>500) dziel, jeśli to możliwe.
- Code review skupiony na poprawności (typy, obsługa błędów, testy, bezpieczeństwo), nie na stylu — formatowanie robi automatyczne narzędzie.
- **Zmiana, do której nikt się nie przyczepił, nadal musi być widoczna w opisie PR-a.** Cicha zmiana zachowania we współdzielonym kodzie (np. zawężenie zestawu uprawnień, poprawka kolejności w funkcji mapującej role), której recenzent nie ma jak zauważyć inaczej niż czytając diff linia po linii, potrzebuje własnego zdania w opisie — przed zgłoszeniem PR-a przejrzyj własny diff pod kątem zmian, które nie wynikają z żadnej konkretnej uwagi ani z treści zadania.
- Feature flags dla niekompletnych funkcji zamiast długo żyjących branchy — merge do głównej gałęzi często, z flagą wyłączającą niegotową funkcję; usuń flagę po ustabilizowaniu.

## 16. Zależności

- Minimalizuj i uzasadniaj zależności — nie dodawaj biblioteki tam, gdzie wystarczy natywny JS albo pakiet już obecny w projekcie.
- Preferuj pakiet już używany w projekcie zamiast dodawania drugiego o tej samej funkcji (np. drugiej biblioteki HTTP).
- Sprawdzaj rozmiar bundla przed dodaniem zależności frontendowej; porównaj alternatywy.
- Aktualizuj w małych, powiązanych partiach, nie wszystko naraz — łatwiej zidentyfikować przyczynę problemu. Czytaj changelog/migration guide przed aktualizacją major wersji.
- Regularne audyty bezpieczeństwa zależności; szybkie łatanie krytycznych podatności; commituj plik lock — zapewnia spójność wersji między środowiskami.
- W monorepo: wersje wewnętrznych pakietów jako referencja do najnowszej (nie sztywno przypięte), świadoma kolejność budowania między pakietami zależnymi.

## 17. Bezpieczeństwo frontendowe

- **Wartość od klienta użyta do autoryzacji albo ceny musi być zweryfikowana po stronie serwera.** Rola, status, flaga czy cena, którą przegląda przeglądarka i którą serwer honoruje bez ponownego sprawdzenia, to zaproszenie do spreparowanego żądania. Pytanie kontrolne: co dostaje ręcznie spreparowane żądanie, czego UI by nie zaproponował?
- **Rate limit kluczowany po wartości kontrolowanej przez atakującego ogranicza per-zasób, nie per-wołający.** Throttle liczony po polu z treści żądania albo po id zasobu omija się, po prostu zmieniając to pole. Właściwy klucz to zweryfikowany podmiot (np. id użytkownika z tokenu), jeśli taki istnieje.
- **`in` na niepewnym wejściu chodzi po łańcuchu prototypów** — użyj `Object.hasOwn`.
- **Na granicy systemu: allowlist, nie denylist.** Dla projekcji danych po wygenerowanym typie/DTO asercja wyczerpania sprawdzana w czasie kompilacji (exhaustiveness check) zamienia cichą utratę pola w błąd budowania zamiast w wyciek danych na produkcji.
- **Rozróżnienie statusów nie może wyciekać do przeglądarki bez potrzeby.** 400 kontra 404 mówi wołającemu "dobrze sformułowane, ale nieznane" od "źle sformułowane" — to często niepotrzebna informacja dla nieznajomego klienta API. Warstwa pośrednia może scalić oba na granicy z klientem i zalogować prawdziwy powód po stronie serwera.
- **Sekrety i tokeny trafiające do przeglądarki są znaleziskiem nawet wtedy, gdy komentarz dwie linie dalej twierdzi, że nie trafiają.** Sprawdź kod, nie komentarz.
- **Wartość zawierająca `/` w segmencie ścieżki URL wymaga uwagi.** Serwery pośredniczące, które routują po zdekodowanej ścieżce, potrafią ją ponownie podzielić. Jeśli naprawiono to na jednym przeskoku (np. w jednym endpoincie), sprawdź każdy kolejny, który wciąż przyjmuje tę wartość jako parametr ścieżki zamiast parametru zapytania.
- **Nowy identyfikator przyjmowany przez wyszukiwanie/lookup — sprawdź, jak łatwo go odgadnąć.** Akceptowanie losowego, długiego identyfikatora jako "wystarczającej ochrony" to inna decyzja niż akceptowanie sekwencyjnego licznika — sprawdź, co dokładnie zwraca trafienie, i czy zadeklarowany rate limiting faktycznie obejmuje tę ścieżkę.
- Waliduj URL przed przekierowaniem pochodzącym od użytkownika (patrz sekcja 8) — to też pozycja bezpieczeństwa, nie tylko UX.
- `dangerouslySetInnerHTML` tylko dla zaufanej, sanityzowanej treści (patrz sekcja 3) — nigdy dla treści wpisanej przez użytkownika.
- **Każda mutacja na cudzym zasobie (nie własnym rekordzie wołającego) musi jawnie sprawdzić przynależność/własność — i rzucić błąd przy braku dopasowania, nigdy zapisywać z brakującym identyfikatorem właściciela zamiast rzucić wyjątek.** Jeśli dodajesz takie sprawdzenie do jednej operacji w module, od razu przejrzyj siostrzane operacje w tym samym module — zwykle potrzebują tego samego, a łatwo się o jednej zapomina.
- **Komentarz twierdzący o ochronie, jakiej kod w rzeczywistości nie daje, jest samodzielnym znaleziskiem bezpieczeństwa, nie tylko nieścisłością dokumentacyjną.** Sprawdź, czy opisana w prozie granica zaufania faktycznie istnieje w kodzie pod spodem, nie tylko w komentarzu nad nim.

## 18. Weryfikacja przed otwarciem PR

- **Uruchom aplikację i przejdź przez zmienioną funkcję jako każda rola/persona, której dotyczy** — nie tylko jako siebie. Zielona automatyczna weryfikacja (build, lint, typy) nie jest dowodem, że funkcja działa; realne awarie UI/UX często wychodzą dopiero po kliknięciu.
- **Dodając walidację na ścieżce, która wcześniej jej nie miała, przetestuj każdą wartość, jaką UI potrafi faktycznie wyprodukować.** Allowlist jest z definicji zmianą łamiącą — jeśli lista dopuszczalnych wartości nie pokrywa się dokładnie z tym, co wysyła interfejs, wszystko zacznie się wywalać. Użyj do parsowania istniejącego mechanizmu/mapowania, jeśli już gdzieś jest, zamiast pisać drugi o innym formacie.
- **Porównaj zachowanie ze stanem sprzed zmiany (główna gałąź / poprzedni wdrożony release), żeby odróżnić regresję od zastanego, wcześniejszego błędu.** To rozstrzyga, czy problem trzeba naprawić w tym PR-ze, czy tylko odnotować osobno.
- **Dane testowe (fixture/seed/mock) muszą istnieć w konfiguracji faktycznie aktywnej w danym środowisku, nie w nieużywanym/martwym torze integracji.** Fixture w nieaktywnej integracji daje fałszywe poczucie pokrycia — jeśli kryterium akceptacji nie da się przeklikać bez ręcznego tworzenia danych, luka jest w danych testowych, nie w kodzie.
- **Sprawdź, czy model/kontrakt niesie to, czego persona faktycznie potrzebuje do wykonania zadania — nie tylko czy formalnie spełnia zapisane kryteria akceptacji.** Kryteria bywają niekompletne; potrzeba użytkownika jest ostatecznym testem.
- **Zanim potraktujesz zielone testy jako sygnał jakości, sprawdź, czy CI w ogóle je odpala** — nazwa skryptu w konfiguracji projektu nie gwarantuje, że jest wywoływana (patrz Zasada III).

## 19. Testy End-to-End (np. Playwright)

- **Generuj i uruchamiaj testy przez skompilowany skrypt (CLI), nie przez sesję eksploracyjną/sterowaną przez agenta AI, dla rutynowych przebiegów.** Tryb eksploracyjny (agent klika po żywej stronie i "widzi" jej stan) jest znacznie kosztowniejszy i mniej deterministyczny niż odpalenie zapisanego skryptu — zostaw go jako ograniczony fallback do znalezienia lokatora w naprawdę nowym, nieopisanym jeszcze fragmencie UI, nie jako sposób prowadzenia całego scenariusza.
- **Page Object Model z wstrzykiwanymi instancjami (fixture), nie ręcznym tworzeniem obiektu strony w teście.** Lokatory jako gettery (nie pola ustawiane w konstruktorze), akcje zwracają pustą obietnicę, asercje mogą żyć w Page Objekcie — ale logika scenariusza (kolejność kroków, warunki) zawsze zostaje w samym teście, nigdy w Page Objekcie.
- **Priorytet lokatorów: rola/etykieta semantyczna → atrybut testowy dedykowany → widoczny tekst → CSS jako ostateczność** (nigdy klasy stylistyczne/utility ani łańcuchy `nth-child`). Brak stabilnego zaczepienia — poproś o dedykowany atrybut testowy po stronie frontendu, nie pisz kruchego selektora na jego zastępstwo.
- Żadnego tekstu UI ani magicznych liczb wpisanych na sztywno w teście — centralne źródło etykiet i stałych (timeouty, progi).
- **Asercje "web-first" z wbudowanym oczekiwaniem** (czekaj na widoczność/URL/odpowiedź jako warunek, nie odczytaj-stan-i-porównaj) — nigdy sztywne uśpienie czasowe. Asertuj na efekt, nie na zegar.
- Nazwa/tag testu niesie identyfikator traceability wiążący go z wymaganiem/kryterium akceptacji, które weryfikuje — ten identyfikator jest kluczem łączącym z systemem zarządzania testami, nie osobnym plikiem mapującym.
- Żadnych zaszytych na sztywno adresów środowiska/sekretów w teście — przełączanie środowiska przez konfigurację, nigdy przez edycję samego testu.
- **Sprawdzenia dostępności na testach E2E są logowane, nie są twardą bramką na testach funkcjonalnych.** Regresja dostępności nie powinna zaczerwienić niepowiązanego testu funkcjonalnego — twarda bramka na dostępność to osobny, dedykowany test.
- **Triage awarii: napraw (heal) czy zgłoś jako błąd.** Czerwony test E2E to kandydat na zgłoszenie błędu, nie coś do zazielenienia przez przepisanie asercji. Dryf selektora/czasu (zachowanie się nie zmieniło) → napraw Page Object, uruchom ponownie. Zachowanie faktycznie odbiegające od kryterium akceptacji → zatrzymaj się, to prawdziwy błąd, nie test do poprawki.
- Test niestabilny (flaky) w sposób trwały trafia do kwarantanny z odnotowanym powodem — nigdy nie zostaje cicho ponawiany w nieskończoność.
- **Automatyzuj wszystko osiągalne przez interfejs; ręcznie zostaje tylko to, co fizycznie niemożliwe z poziomu uruchamiacza testów** (prawdziwe dostarczenie SMS/e-maila poza środowiskiem z atrapą, przekierowanie do zewnętrznego dostawcy bez zaczepu testowego, CAPTCHA, ocena wizualna wymagająca ludzkiego osądu) — i zapisz dlaczego, żeby wrócić do tego, gdy przeszkoda zniknie.
