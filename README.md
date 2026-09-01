[odpowiedzi_na_obrone.md](https://github.com/user-attachments/files/31704888/odpowiedzi_na_obrone.md)
# ODPOWIEDZI NA OBRONĘ — GameHUB

---

## PYTANIE 1
**Opisz, jak aplikacja rozróżnia użytkowników o różnych uprawnieniach.**

### ODPOWIEDŹ:
Podzieliłem użytkowników na admina i zwykłych graczy. Zrobiłem to najprościej jak się dało — sprawdzam po prostu, czy w adresie e-mail użytkownika jest słowo "admin".

Na przykład: jak admin wejdzie na `/dashboard/users`, to widzi listę wszystkich kont i może je usuwać. Jak wejdzie tam zwykły gracz, to aplikacja wywali mu błąd 403 (brak dostępu).

Mam to zrobione w kontrolerze `RegisteredUserController.php`. Na samym początku metod dodawałem taki warunek:
```php
if (!str_contains(Auth::user()->email, 'admin')) {
    abort(403, 'Nie masz uprawnień do przeglądania tej strony.');
}
```
Dodatkowo w widokach, np. w `dashboard.blade.php`, używam zwykłego `@if`, żeby adminowi pokazywać przyciski zarządzania, a graczowi tylko polecane gry.

### CO POKAZAĆ:
- `RegisteredUserController.php` — warunek `abort(403)` na początku funkcji.
- `dashboard.blade.php` — `@if` z podziałem na role.

---

## PYTANIE 2
**Wybierz jedną akcję i opisz krok po kroku drogę danych.**

### ODPOWIEDŹ:
Opiszę to na przykładzie **kupowania gry**.

1. **Widok:** Użytkownik widzi grę i klika "KUP" w pliku `games/index.blade.php`. To jest formularz wysyłający żądanie POST.
2. **Trasa:** Żądanie leci do `web.php`. Tam łapie je trasa: `Route::post('/games/{game}/purchase', [GameController::class, 'purchase'])`.
3. **Kontroler (logika):** W `GameController.php` funkcja `purchase()` sprawdza, czy gracz ma w ogóle kasę (monety) i czy czasem już tej gry nie kupił. Jeśli jest okej, odejmuję cenę od portfela (`$user->balance -= $price;`) i dodaję grę do konta przez relację (`$user->games()->attach(...)`).
4. **Baza danych:** Laravel robi update tabeli `users` (zmienia balance) i dodaje wpis do tabeli `game_user`, która łączy gracza z grą (zapisuje też cenę zakupu).
5. **Koniec (widok):** Funkcja zwraca `redirect()->back()`. Strona się odświeża, gracz widzi swój nowy stan konta i przycisk "KUP" zmienia się na "ZWRÓĆ GRĘ".

### CO POKAZAĆ:
1. `games/index.blade.php` — formularz z przyciskiem KUP.
2. `web.php` — linijka z trasą `games.purchase`.
3. `GameController.php` — kod funkcji `purchase()`.
4. `User.php` — relacja `games()` z dopiskiem `belongsToMany`.
5. Baza danych (opcjonalnie) — jak wygląda tabela pośrednia `game_user`.

---

## PYTANIE 3
**Wskaż miejsce, w którym aplikacja sprawdza poprawność danych.**

### ODPOWIEDŹ:
Sprawdzam to w kontrolerach za pomocą funkcji `$request->validate()`. Fajna sprawa jest przy dodawaniu nowej gry (`GameController.php`, metoda `store`).

Sprawdzam tam np. czy tytuł w ogóle został wpisany, czy cena na pewno jest liczbą i czy nie jest mniejsza niż 0:
```php
$validated = $request->validate([
    'title' => 'required|max:255',
    'price' => 'required|numeric|min:0',
    // ...inne pola
]);
```
Jak ktoś wpisze bzdury (np. ujemną cenę), formularz nie przejdzie. Strona odświeży się sama, a pod złym polem pojawi się czerwony komunikat. W widoku `games/create.blade.php` odpowiada za to funkcja `@error`.
Co ważne, dzięki `old('title')` reszta poprawnych danych nie znika i nie trzeba wypełniać wszystkiego od nowa.

### CO POKAZAĆ:
- `GameController.php` — walidacja w metodzie `store()`.
- `games/create.blade.php` — jak używam `@error` i `old()`.

---

## PYTANIE 4
**Wskaż przykład widoku, który jest dobrze zorganizowany.**

### ODPOWIEDŹ:
Na pewno formularz dodawania gry — `games/create.blade.php`.

Uważam, że jest dobrze napisany, bo nie wklejałem tam całej struktury HTML, paska nawigacji ani stopki. Użyłem komponentu `<x-app-layout>`, który sam zaciąga ten cały "szkielet" z pliku `layouts/app.blade.php`. Dzięki temu kod się nie powtarza.
Sam formularz też jest poukładany: każde pole ma obok labelkę, inputa i obsługę błędów. Jakbym musiał teraz dodać pole "wymagania wiekowe", to po prostu kopiuję jeden div z inputem, zmieniam nazwę i gotowe. Zmiany nie zepsują reszty strony.

### CO POKAZAĆ:
- `games/create.blade.php` — ogólny wygląd pliku (użycie `<x-app-layout>`).
- `layouts/app.blade.php` — pokazanie, skąd bierze się główny szkielet strony.

---

## PYTANIE 5
**Wskaż funkcjonalność, której nie widać w interfejsie, ale jest ważna.**

### ODPOWIEDŹ:
Napisałem swój własny middleware o nazwie `NoCacheHeaders` (plik `app/Http/Middleware/NoCacheHeaders.php`).

Nie widać tego na stronie, ale robi bardzo ważną rzecz z bezpieczeństwem. Kiedy użytkownik jest zalogowany, ten kod dodaje do odpowiedzi serwera nagłówki, które blokują zapisywanie strony w pamięci przeglądarki (cache).
Dlaczego to takie ważne? Bo wyobraź sobie, że admin się wylogowuje, odchodzi od komputera, a ktoś podchodzi, klika strzałkę "Wstecz" w przeglądarce i widzi panel zarządzania, mimo że sesja już wygasła. Mój kod przed tym chroni — przeglądarka musi zawsze pytać serwera o nową stronę.

### CO POKAZAĆ:
- `NoCacheHeaders.php` — pokazanie kodu tego middleware'a.
- `bootstrap/app.php` — miejsce, gdzie go uruchomiłem globalnie.

---

## PYTANIE 6
**Opisz, jak aplikacja rozpoznaje, co użytkownik chce zrobić.**

### ODPOWIEDŹ:
Robi to na podstawie systemu tras (routingu) w pliku `routes/web.php`. Aplikacja patrzy na adres (URL) i typ zapytania (GET, POST itd.).

Dla przykładu z grami:
- Jak wejdziesz na `/admin/games/create` przez **GET**, to kontroler po prostu wyświetli Ci formularz.
- Ale jak wyślesz ten formularz na adres `/admin/games` przez **POST**, to kontroler wie, że ma zapisać nową grę w bazie.

Z aktualizowaniem i usuwaniem jest mały myk, bo same przeglądarki potrafią wysłać tylko GET i POST. Więc w formularzach usuwania dodaję np. `@method('DELETE')`. Wtedy Laravel pod spodem orientuje się, że chodzi o skasowanie rekordu.

### CO POKAZAĆ:
- `web.php` — pokazanie różnic między trasami GET i POST.
- `games/index.blade.php` — formularz usuwania gry z `@method('DELETE')`.

---

## PYTANIE 7
**Opisz przykład powiązania danych w projekcie.**

### ODPOWIEDŹ:
Zrobiłem powiązanie między użytkownikami a grami w systemie wiele-do-wielu (jeden gracz ma wiele gier, jedna gra należy do wielu graczy).

W bazie to są trzy tabele: `users`, `games` i tabela łącząca `game_user` (trzyma ID gracza, ID gry i za ile ją kupił).
W kodzie, w modelu `User.php` opisałem to funkcją:
```php
public function games()
{
    return $this->belongsToMany(Game::class)->withPivot('purchase_price');
}
```
Dzięki temu w kontrolerze łatwo dodaję komuś grę: `$user->games()->attach()`.
A w widoku mojej biblioteki (`my_library.blade.php`) po prostu lecę pętlą `@foreach($games as $game)` i wypisuję tytuły, które posiada zalogowany gość.

### CO POKAZAĆ:
- `User.php` — relacja `belongsToMany`.
- `GameController.php` — przypisywanie gry przez `attach()`.
- `games/my_library.blade.php` — pętla wyświetlająca gry gracza.

---

## PYTANIE 8
**Opisz, jak aplikacja reaguje na nietypową sytuację.**

### ODPOWIEDŹ:
Zabezpieczyłem projekt przed różnymi dziwnymi sytuacjami.

Na przykład przy kupowaniu gry: w `GameController.php` sprawdzam, czy gracz w ogóle ma wystarczająco monet na koncie. Jak nie ma, to anuluję akcję i rzucam go z powrotem do sklepu z alertem "Masz za mało monet!". Sprawdzam też, czy przypadkiem nie próbuje kupić gry, którą już ma.

Zabezpieczyłem też usuwanie kont. Normalnie admin może usunąć każdego, ale w kontrolerze `RegisteredUserController` zrobiłem warunek: jeśli admin próbuje usunąć samego siebie (`Auth::id() === $user->id`), to wywalam mu błąd. Dzięki temu przypadkiem nie zablokuje sobie dostępu do systemu.

### CO POKAZAĆ:
- `GameController.php` — kod sprawdzający, czy gracz ma monety na koncie.
- `RegisteredUserController.php` — if blokujący adminowi usunięcie własnego konta.

---

## PYTANIE 9
**Wskaż jedno miejsce do poprawy.**

### ODPOWIEDŹ:
Sposób, w jaki sprawdzam, czy użytkownik jest administratorem. 
Zamiast zrobić do tego osobną kolumnę w bazie danych, sprawdzam po prostu, czy adres e-mail zawiera słówko "admin" (`str_contains(Auth::user()->email, 'admin')`). Zrobiłem tak, żeby szybko rozdzielić widoki, ale to ogromna luka. Wystarczy, że ktoś założy konto o nazwie np. `jan.admin@gmail.com` i system z automatu da mu pełen dostęp do zarządzania sklepem i usuwania gier.

### CO POKAZAĆ:
- `RegisteredUserController.php` — pokazać linijkę z `str_contains(...)`.

---

## PYTANIE 10
**Napisz 6–10 zdań, dlaczego projekt zasługuje na wskazaną ocenę.**

### ODPOWIEDŹ:
Uważam, że projekt zasługuje na ocenę dostateczną (3), bo spełnia najważniejsze, podstawowe wymagania. Zrobiłem działającą aplikację w Laravelu, która normalnie łączy się z bazą danych PostgreSQL. Użytkownik może się zarejestrować, zalogować i przeglądać listę gier w sklepie. Działa też prosty system kupowania i zwracania gier za wirtualne monety. Strona jakoś wygląda, bo użyłem gotowych klas Tailwind CSS. Dodałem prostą walidację, więc formularz nie przepuści całkiem pustych danych ani ujemnej ceny. Zrobiłem też podział na admina i zwykłego gracza, chociaż w bardzo uproszczony sposób. Aplikacja nie wyrzuca błędów przy normalnym przeklikaniu zakładek. Jest to raczej prosty projekt, ale działa poprawnie i robi to, co do niego należy.

---

## PYTANIE 11
**Co poprawić w pierwszej kolejności, aby projekt zasługiwał na wyższą ocenę?**

### ODPOWIEDŹ:
Największym minusem mojego projektu jest to, jak działa konto administratora i to musiałbym poprawić w pierwszej kolejności. Teraz sprawdzam tylko, czy ktoś wpisał słowo "admin" w swoim adresie e-mail. Przez to każdy może sobie założyć takie konto i mieć dostęp do wszystkiego. Żeby aplikacja zasługiwała na wyższą ocenę, musiałbym po prostu dodać kolumnę "rola" w tabeli użytkowników w bazie danych i na tej podstawie blokować dostęp do panelu. To od razu rozwiązałoby główny problem z bezpieczeństwem.

---

## PYTANIA 12 i 13 (Inne projekty z grupy)

Żebym odpowiedział na to pytanie na obronie, prowadzący musiałby mi wcześniej pokazać (albo chociaż opisać) jakie projekty zrobili inni studenci z mojej grupy. Ponieważ ich nie znam, nie mam z czym porównać mojego sklepu GameHUB. 
(Jeśli znasz projekty kolegów, przypomnij sobie, kto zrobił coś bardziej rozbudowanego - np. z podpiętą prawdziwą płatnością, a kto zrobił coś prostszego, np. sam statyczny HTML bez panelu logowania).

---
