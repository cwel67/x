[odpowiedzi_na_obrone.md](https://github.com/user-attachments/files/31703712/odpowiedzi_na_obrone.md)
# ODPOWIEDZI NA OBRONĘ — GameHUB

---

## PYTANIE 1
**Opisz, jak aplikacja rozróżnia użytkowników o różnych uprawnieniach.**

### ODPOWIEDŹ:

W moim projekcie są dwa rodzaje użytkowników — admin i zwykły gracz. Rozróżniam ich po adresie e-mail. Jeśli e-mail zawiera słowo „admin", to użytkownik jest adminem.

Przykład: admin może wejść na listę użytkowników (`/dashboard/users`) i np. usunąć czyjeś konto. Zwykły gracz tego nie może — jeśli spróbuje, dostanie błąd 403.

W kontrolerze `RegisteredUserController.php` na początku metod admina mam takie sprawdzenie:

```php
if (!str_contains(Auth::user()->email, 'admin')) {
    abort(403, 'Nie masz uprawnień do przeglądania tej strony.');
}
```

Poza tym w widokach np. w `dashboard.blade.php` wyświetlam inną zawartość w zależności od roli:

```php
@if(str_contains(Auth::user()->email, 'admin'))
    // panel admina — katalog gier, dodawanie, użytkownicy
@else
    // polecane gry dla gracza
@endif
```

A trasy wymagające zalogowania są chronione middleware `auth` w `web.php`.

### CO MOGĘ POKAZAĆ:
- `RegisteredUserController.php` — `abort(403)`
- `dashboard.blade.php` — `@if` z podziałem na admina i gracza
- `web.php` — middleware `auth`

---

## PYTANIE 2
**Wybierz jedną akcję i opisz krok po kroku drogę danych.**

### ODPOWIEDŹ:

Wybieram **kupowanie gry**.

**Krok 1 — Widok (formularz)**

W `games/index.blade.php` przy każdej grze jest przycisk „KUP":

```html
<form action="{{ route('games.purchase', $game) }}" method="POST">
    @csrf
    <button type="submit">KUP ({{ $game->price ?? 100 }} 🪙)</button>
</form>
```

**Krok 2 — Routing**

W `web.php` jest trasa, która łapie to żądanie POST:

```php
Route::post('/games/{game}/purchase', [GameController::class, 'purchase'])->name('games.purchase');
```

Jest w grupie z middleware `auth`, więc niezalogowany użytkownik zostanie przekierowany na logowanie.

**Krok 3 — Kontroler**

Żądanie trafia do metody `purchase()` w `GameController.php`:

```php
public function purchase(Request $request, Game $game)
{
    $user = Auth::user();
    $price = (int) round($game->price ?? 100);

    if ($user->balance < $price) {
        return redirect()->back()->with('error', 'Masz za mało monet!');
    }
    if ($user->games()->where('game_id', $game->id)->exists()) {
        return redirect()->back()->with('error', 'Posiadasz już tę grę!');
    }

    $user->balance -= $price;
    $user->save();
    $user->games()->attach($game->id, ['purchase_price' => $price]);

    return redirect()->back()->with('success', 'Zakupiono grę: ' . $game->title);
}
```

Sprawdza, czy wystarczy monet i czy gracz już nie ma tej gry. Potem odejmuje cenę od salda i dodaje grę do tabeli pośredniej.

**Krok 4 — Model i baza danych**

W `User.php` jest relacja wiele-do-wielu:

```php
public function games()
{
    return $this->belongsToMany(Game::class)->withPivot('purchase_price');
}
```

`attach()` dodaje wiersz do tabeli `game_user` (user_id, game_id, purchase_price). Jednocześnie `$user->save()` aktualizuje `balance` w tabeli `users`.

**Krok 5 — Odpowiedź**

Po zakupie użytkownik wraca do katalogu. Widzi zmienione saldo i zamiast „KUP" pojawia się „ZWRÓĆ GRĘ":

```html
@if(Auth::user()->games()->where('game_id', $game->id)->exists())
    <button>ZWRÓĆ GRĘ</button>
@else
    <button>KUP</button>
@endif
```

### CO MOGĘ POKAZAĆ (5 fragmentów):
1. `games/index.blade.php` — formularz z przyciskiem KUP
2. `web.php` — trasa `games.purchase`
3. `GameController.php` — metoda `purchase()`
4. `User.php` — relacja `games()` z `belongsToMany`
5. `games/index.blade.php` — przycisk ZWRÓĆ po zakupie + saldo portfela

---

## PYTANIE 3
**Wskaż miejsce, w którym aplikacja sprawdza poprawność danych.**

### ODPOWIEDŹ:

Walidacja jest w kontrolerach, metodą `$request->validate()`. Najlepszy przykład to dodawanie gry.

W `GameController.php`, metoda `store()`:

```php
$validated = $request->validate([
    'title' => 'required|max:255',
    'description' => 'required',
    'developer' => 'required',
    'genre' => 'required',
    'release_date' => 'required|date',
    'image_url' => 'nullable|string',
    'price' => 'required|numeric|min:0',
]);
```

Sprawdzane jest np.: czy tytuł nie jest pusty, czy nie jest za długi (max 255 znaków), czy cena jest liczbą i nie jest ujemna, czy data premiery ma prawidłowy format.

Jeśli coś jest źle, Laravel przekierowuje z powrotem na formularz i pod każdym polem z błędem wyświetla się czerwony komunikat. W widoku `games/create.blade.php`:

```html
@error('title') <span class="text-red-400 text-xs">{{ $message }}</span> @enderror
```

A `old('title')` zachowuje dane wpisane wcześniej, żeby użytkownik nie musiał wpisywać wszystkiego od nowa.

### CO MOGĘ POKAZAĆ:
- `GameController.php` — reguły walidacji w `store()`
- `games/create.blade.php` — `@error` i `old()` przy polach formularza

---

## PYTANIE 4
**Wskaż przykład widoku, który jest dobrze zorganizowany.**

### ODPOWIEDŹ:

Wybieram `games/create.blade.php` — formularz dodawania gry.

Jest dobrze zorganizowany, bo:
- Używa layoutu `<x-app-layout>`, więc nawigacja, nagłówek i cała struktura strony są w jednym miejscu (`layouts/app.blade.php`) i nie trzeba ich kopiować do każdego widoku.
- Formularz jest podzielony logicznie — każde pole ma swoją etykietę, input i obsługę błędów. Niektóre pola są w gridzie obok siebie (gatunek i data premiery).
- Gdybym chciał dodać nowe pole, np. „platforma", wystarczy dopisać jeden blok z `<label>`, `<input>` i `@error` — i dodać regułę walidacji w kontrolerze. Nie trzeba ruszać innych plików.
- Zmiana wyglądu nawigacji w `navigation.blade.php` zmienia ją na wszystkich stronach naraz.

### CO MOGĘ POKAZAĆ:
- `games/create.blade.php` — formularz
- `layouts/app.blade.php` — layout z `$slot`
- `layouts/navigation.blade.php` — nawigacja używana wszędzie

---

## PYTANIE 5
**Wskaż funkcjonalność, której nie widać w interfejsie, ale jest ważna.**

### ODPOWIEDŹ:

Napisałem własny middleware `NoCacheHeaders`.

W pliku `app/Http/Middleware/NoCacheHeaders.php`:

```php
public function handle(Request $request, Closure $next): Response
{
    $response = $next($request);

    if ($request->user()) {
        $response->headers->set('Cache-Control', 'no-cache, no-store, max-age=0, must-revalidate');
        $response->headers->set('Pragma', 'no-cache');
        $response->headers->set('Expires', 'Sat, 01 Jan 2000 00:00:00 GMT');
    }

    return $response;
}
```

Jest zarejestrowany globalnie w `bootstrap/app.php`:

```php
$middleware->web(append: [
    \App\Http\Middleware\NoCacheHeaders::class,
]);
```

Do czego to służy: kiedy zalogowany użytkownik się wyloguje i kliknie „Wstecz" w przeglądarce, bez tego middleware przeglądarka mogłaby pokazać poprzednią stronę z cache'u — razem z danymi użytkownika (saldo, panel admina). Ten middleware mówi przeglądarce: „nie zapisuj tej strony, zawsze pobierz od nowa". Dzięki temu po wylogowaniu nie da się wrócić do chronionych stron przyciskiem Wstecz.

### CO MOGĘ POKAZAĆ:
- `NoCacheHeaders.php` — kod middleware
- `bootstrap/app.php` — rejestracja

---

## PYTANIE 6
**Opisz, jak aplikacja rozpoznaje, co użytkownik chce zrobić.**

### ODPOWIEDŹ:

Aplikacja rozpoznaje akcję po adresie URL i metodzie HTTP. Wszystko jest zdefiniowane w `routes/web.php`.

Na przykładzie gier:

| Akcja | Adres | Metoda | Co robi |
|---|---|---|---|
| Lista gier | `/` | GET | `GameController@index` |
| Dodaj grę (formularz) | `/admin/games/create` | GET | `GameController@create` |
| Zapisz grę | `/admin/games` | POST | `GameController@store` |
| Edytuj grę | `/admin/games/{game}/edit` | GET | `GameController@edit` |
| Aktualizuj grę | `/admin/games/{game}` | PUT | `GameController@update` |
| Usuń grę | `/admin/games/{game}` | DELETE | `GameController@destroy` |

Przeglądarki nie obsługują PUT i DELETE bezpośrednio, więc w formularzach używam ukrytego pola:

```html
<form method="POST" action="{{ route('games.update', $game) }}">
    @csrf
    @method('PUT')
</form>
```

`@method('PUT')` generuje ukryte pole, dzięki któremu Laravel wie, że to aktualizacja, a nie tworzenie.

### CO MOGĘ POKAZAĆ:
- `web.php` — trasy
- `games/edit.blade.php` — `@method('PUT')`
- `games/index.blade.php` — formularz usuwania z `@method('DELETE')`

---

## PYTANIE 7
**Opisz przykład powiązania danych w projekcie.**

### ODPOWIEDŹ:

Użytkownik i gra są w relacji wiele-do-wielu — jeden użytkownik może mieć wiele gier, a jedną grę może mieć wielu użytkowników.

**W bazie danych** łączy je tabela pośrednia `game_user` z kolumnami: `user_id`, `game_id` i `purchase_price`.

**W modelu** `User.php`:

```php
public function games()
{
    return $this->belongsToMany(Game::class)->withPivot('purchase_price');
}
```

**W kontrolerze** — przy kupowaniu dodaję wpis do tabeli pośredniej:

```php
$user->games()->attach($game->id, ['purchase_price' => $price]);
```

Przy zwrocie usuwam:

```php
$user->games()->detach($game->id);
```

**W widoku** `games/my_library.blade.php` wyświetlam gry użytkownika:

```php
@forelse($games as $game)
    <td>{{ $game->title }}</td>
    <td>{{ $game->developer }}</td>
    <td>{{ $game->genre }}</td>
@empty
    Nie posiadasz jeszcze żadnych gier...
@endforelse
```

Dane pobierane są w `web.php`:

```php
$games = auth()->user()->games;
```

### CO MOGĘ POKAZAĆ:
- `User.php` — `belongsToMany`
- `GameController.php` — `attach()` / `detach()`
- `games/my_library.blade.php` — widok biblioteki

---

## PYTANIE 8
**Opisz, jak aplikacja reaguje na nietypową sytuację.**

### ODPOWIEDŹ:

Pokażę kilka przykładów z `GameController.php`:

**Za mało monet na kupno gry:**

```php
if ($user->balance < $price) {
    return redirect()->back()->with('error', 'Masz za mało monet!');
}
```

Użytkownik wraca do sklepu z komunikatem „Masz za mało monet!".

**Gra już kupiona:**

```php
if ($user->games()->where('game_id', $game->id)->exists()) {
    return redirect()->back()->with('error', 'Posiadasz już tę grę!');
}
```

**Zwykły użytkownik próbuje wejść na panel admina** (`RegisteredUserController.php`):

```php
if (!str_contains(Auth::user()->email, 'admin')) {
    abort(403, 'Nie masz uprawnień do przeglądania tej strony.');
}
```

Dostaje stronę z błędem 403.

**Admin próbuje usunąć sam siebie:**

```php
if (Auth::id() === $user->id) {
    return redirect()->route('users.index')
        ->with('error', 'Nie możesz usunąć aktualnie zalogowanego konta administratora!');
}
```

**Pusta lista gier** (`games/index.blade.php`):

```html
@forelse($games as $game)
    ...
@empty
    <p>Brak gier w bazie.</p>
@endforelse
```

### CO MOGĘ POKAZAĆ:
- `GameController.php` — sprawdzenie salda i duplikatu
- `RegisteredUserController.php` — `abort(403)`
- `games/index.blade.php` — `@empty`

---

## PYTANIE 9
**Wskaż jedno miejsce do poprawy.**

### ODPOWIEDŹ:

Sprawdzanie, czy użytkownik jest adminem, opiera się na zawartości e-maila:

```php
str_contains(Auth::user()->email, 'admin')
```

Problem jest taki, że ktoś może się zarejestrować z mailem np. `admin_fan@gmail.com` i zostanie potraktowany jako admin. Poza tym ten warunek jest powtórzony w kilku plikach — w kontrolerze, w dashboardzie, w nawigacji, w katalogu gier.

Lepszym rozwiązaniem byłoby dodanie kolumny `role` w tabeli `users` i sprawdzanie po niej:

```php
if (Auth::user()->role !== 'admin') {
    abort(403);
}
```

Najlepiej jeszcze stworzyć do tego osobny middleware, żeby nie powtarzać tego w każdej metodzie kontrolera.

### CO MOGĘ POKAZAĆ:
- `RegisteredUserController.php` — powtórzony `str_contains`
- `dashboard.blade.php` — ten sam warunek w widoku

---

## PYTANIE 10
**Napisz 6–10 zdań, dlaczego projekt zasługuje na wskazaną ocenę.**

### ODPOWIEDŹ:

Mój projekt to działający sklep z grami „GameHUB" zbudowany w Laravelu. Aplikacja obsługuje pełny CRUD na grach — dodawanie, edycja, usuwanie i wyświetlanie z walidacją danych. Mam system logowania i rejestracji oparty na Laravel Breeze, plus zmianę hasła i usuwanie konta. Są dwie role — admin zarządza grami i użytkownikami, a gracz kupuje i zwraca gry za wirtualne monety, ma swój portfel i bibliotekę gier. Relacja między użytkownikiem a grami to wiele-do-wielu z tabelą pośrednią, w której zapisuję cenę zakupu. Widoki korzystają z layoutów Blade i Tailwind CSS z ciemnym motywem, więc kod się nie powtarza. Katalog gier ma filtrowanie po gatunku, wyszukiwanie po tytule i sortowanie, które jest zabezpieczone whitelistą. Napisałem też własny middleware `NoCacheHeaders`, który chroni przed podglądaniem stron po wylogowaniu. Kontrolery obsługują sytuacje jak brak monet, duplikat zakupu czy próba usunięcia konta admina.

---

## PYTANIE 11
**Co poprawić, żeby projekt zasługiwał na wyższą ocenę?**

### ODPOWIEDŹ:

Najważniejsze to zmienić sposób sprawdzania roli admina. Teraz sprawdzam po e-mailu (`str_contains(email, 'admin')`), co jest słabe — bo każdy może się zarejestrować z „admin" w mailu. Poza tym ten warunek jest powtórzony w kilku plikach.

Lepiej byłoby dodać kolumnę `role` w tabeli `users` i stworzyć middleware `AdminMiddleware`, który sprawdza rolę w jednym miejscu. Wtedy w `web.php` wystarczyłoby podpiąć middleware do tras admina i kontrolery byłyby czyste. To poprawia naraz bezpieczeństwo, jakość kodu i łatwość rozbudowy.

---

## PYTANIE 12
**Wskaż projekt mocniejszy i słabszy od Twojego z grupy.**

### ODPOWIEDŹ:

Nie mam dostępu do projektów innych osób z grupy, więc nie jestem w stanie porównać. Żeby odpowiedzieć, musiałbym zobaczyć przynajmniej dwa inne projekty albo ich opisy.

---

## PYTANIE 13
**Uzasadnij wybór tych dwóch projektów.**

### ODPOWIEDŹ:

Tak samo jak wyżej — potrzebuję informacji o projektach kolegów, żeby się do nich odnieść. Nie chcę wymyślać porównania bez konkretnych podstaw.
