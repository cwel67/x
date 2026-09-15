[odpowiedzi_na_obrone_GameHUB_po_poprawkach.md](https://github.com/user-attachments/files/31727823/odpowiedzi_na_obrone_GameHUB_po_poprawkach.md)
---

## 1. Jak aplikacja rozróżnia użytkowników o różnych uprawnieniach?

Aplikacja rozróżnia role użytkowników po kolumnie `is_admin` w tabeli `users`. Zwykły użytkownik ma `is_admin = false`, a administrator ma `is_admin = true`. Dzięki temu uprawnienia nie zależą od adresu e-mail ani od nazwy użytkownika.

Administrator może dodawać, edytować i usuwać gry oraz zarządzać użytkownikami. Zwykły użytkownik może przeglądać gry, kupować je, zwracać, korzystać z portfela i własnej biblioteki. Nawet jeśli zwykły użytkownik ręcznie wpisze adres panelu administratora albo wyśle bezpośrednie żądanie HTTP, aplikacja zatrzyma go po stronie serwera i zwróci błąd `403`.

### Fragment: kolumna administratora w migracji

`database/migrations/2026_09_02_003000_add_is_admin_to_users_table.php`

```php
Schema::table('users', function (Blueprint $table) {
    $table->boolean('is_admin')->default(false);
});
```

### Fragment: model użytkownika

`app/Models/User.php`

```php
protected $fillable = [
    'name',
    'email',
    'password',
    'balance',
];

protected function casts(): array
{
    return [
        'password' => 'hashed',
        'is_admin' => 'boolean',
    ];
}
```

Ważne: `is_admin` nie znajduje się w `$fillable`, więc użytkownik nie może nadać sobie administratora przez zwykły formularz lub masowe przypisanie danych.

### Fragment: middleware administratora

`app/Http/Middleware/EnsureUserIsAdmin.php`

```php
public function handle(Request $request, Closure $next): Response
{
    if ($request->user()?->is_admin !== true) {
        abort(403);
    }

    return $next($request);
}
```

### Fragment: rejestracja aliasu middleware

`bootstrap/app.php`

```php
$middleware->alias([
    'admin' => \App\Http\Middleware\EnsureUserIsAdmin::class,
]);
```

### Fragment: trasy administratora

`routes/web.php`

```php
Route::middleware(['auth', 'admin'])->group(function () {
    Route::get('/admin/games/create', [GameController::class, 'create'])->name('games.create');
    Route::post('/admin/games', [GameController::class, 'store'])->name('games.store');
    Route::get('/admin/games/{game}/edit', [GameController::class, 'edit'])->name('games.edit');
    Route::put('/admin/games/{game}', [GameController::class, 'update'])->name('games.update');
    Route::delete('/admin/games/{game}', [GameController::class, 'destroy'])->name('games.destroy');

    Route::get('/dashboard/users', [RegisteredUserController::class, 'index'])->name('users.index');
    Route::get('/dashboard/users/{user}/edit', [RegisteredUserController::class, 'edit'])->name('users.edit');
    Route::put('/dashboard/users/{user}', [RegisteredUserController::class, 'update'])->name('users.update');
    Route::delete('/dashboard/users/{user}', [RegisteredUserController::class, 'destroy'])->name('users.destroy');
});
```

---

## 2. Jak działa proces zakupu gry?

Proces zakupu zaczyna się w widoku, gdzie użytkownik klika przycisk kupna. Formularz wysyła żądanie `POST` na trasę `games.purchase`. Trasa prowadzi do metody `purchase()` w `GameController`.

Kontroler sprawdza, czy użytkownik ma wystarczającą liczbę monet oraz czy nie posiada już tej gry. Jeśli wszystko jest poprawne, aplikacja odejmuje cenę gry z salda użytkownika i zapisuje zakup w tabeli pośredniej `game_user`. W tej tabeli zapisywana jest także cena zakupu jako `purchase_price`.

### Fragment: trasa zakupu

`routes/web.php`

```php
Route::middleware(['auth'])->group(function () {
    Route::post('/games/{game}/purchase', [GameController::class, 'purchase'])->name('games.purchase');
    Route::post('/games/{game}/refund', [GameController::class, 'refund'])->name('games.refund');
});
```

### Fragment: formularz zakupu w widoku

`resources/views/games/index.blade.php`

```blade
<form action="{{ route('games.purchase', $game) }}" method="POST" class="w-full">
    @csrf
    <button type="submit" class="w-full py-1.5 bg-indigo-600 hover:bg-indigo-500 text-white text-xs font-bold rounded-lg transition cursor-pointer">
        KUP ({{ $game->price ?? 100 }} 🪙)
    </button>
</form>
```

### Fragment: logika zakupu

`app/Http/Controllers/GameController.php`

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

---

## 3. Jak są połączone warstwy aplikacji?

Projekt korzysta z typowej struktury Laravela. Żądanie trafia najpierw do trasy w `routes/web.php`, następnie do kontrolera, kontroler używa modeli i bazy danych, a wynik jest zwracany do widoku Blade.

Przykład procesu zakupu:

1. Widok generuje formularz zakupu.
2. Formularz wysyła żądanie do trasy `games.purchase`.
3. Trasa kieruje żądanie do `GameController::purchase()`.
4. Kontroler pobiera zalogowanego użytkownika i grę.
5. Model `User` zapisuje relację z grą w tabeli `game_user`.
6. Użytkownik wraca na stronę z komunikatem sukcesu albo błędu.

### Fragment: relacja użytkownika z grami

`app/Models/User.php`

```php
public function games()
{
    return $this->belongsToMany(Game::class)->withPivot('purchase_price');
}
```

### Fragment: tabela pośrednia zakupów

`database/migrations/2026_06_05_113620_create_game_user_table.php`

```php
Schema::create('game_user', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained()->cascadeOnDelete();
    $table->foreignId('game_id')->constrained()->cascadeOnDelete();
    $table->integer('purchase_price')->default(0);
    $table->timestamp('created_at')->nullable();
    $table->timestamp('updated_at')->nullable();
    $table->unique(['user_id', 'game_id']);
});
```

---

## 4. Jak zabezpieczane są dane wprowadzane przez użytkownika?

Dane są walidowane w kontrolerach przed zapisem do bazy. Przykładowo przy dodawaniu gry wymagane są tytuł, opis, producent, gatunek, data premiery i cena. Cena musi być liczbą i nie może być ujemna. Dzięki temu aplikacja nie zapisuje niepełnych lub błędnych danych.

Laravel automatycznie chroni formularze przed atakami CSRF przez dyrektywę `@csrf`. W formularzach edycji i usuwania używane są też odpowiednie metody HTTP przez `@method('PUT')` albo `@method('DELETE')`.

### Fragment: walidacja gry

`app/Http/Controllers/GameController.php`

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

### Fragment: zabezpieczenie formularza tokenem CSRF

```blade
<form action="{{ route('games.store') }}" method="POST">
    @csrf
    <!-- pola formularza -->
</form>
```

### Fragment: usuwanie przez metodę DELETE

```blade
<form action="{{ route('games.destroy', $game) }}" method="POST">
    @csrf
    @method('DELETE')
    <button type="submit">Usuń</button>
</form>
```

---

## 5. Który fragment kodu jest dobrze zorganizowany?

Dobrym przykładem jest middleware `EnsureUserIsAdmin`. Jest krótki, ma jedną odpowiedzialność i znajduje się w osobnym pliku. Dzięki temu sprawdzanie uprawnień administratora nie jest powtarzane w wielu kontrolerach.

Jeśli w przyszłości trzeba zmienić sposób sprawdzania administratora, wystarczy zmienić jedno miejsce. Trasy pozostają czytelne, bo wystarczy dopisać middleware `admin`.

### Fragment: middleware jako jedno miejsce kontroli uprawnień

`app/Http/Middleware/EnsureUserIsAdmin.php`

```php
public function handle(Request $request, Closure $next): Response
{
    if ($request->user()?->is_admin !== true) {
        abort(403);
    }

    return $next($request);
}
```

### Fragment: użycie middleware na trasach

`routes/web.php`

```php
Route::middleware(['auth', 'admin'])->group(function () {
    Route::post('/admin/games', [GameController::class, 'store'])->name('games.store');
    Route::put('/admin/games/{game}', [GameController::class, 'update'])->name('games.update');
    Route::delete('/admin/games/{game}', [GameController::class, 'destroy'])->name('games.destroy');
});
```

---

## 6. Jaka funkcjonalność jest ważna, ale mało widoczna w interfejsie?

Ważna, ale mało widoczna funkcjonalność to zabezpieczenie panelu administratora po stronie serwera. Użytkownik może nie widzieć przycisków administracyjnych, ale prawdziwe zabezpieczenie działa dopiero wtedy, gdy trasy są chronione przez middleware.

Drugim przykładem jest blokowanie pamięci podręcznej dla stron po zalogowaniu. Dzięki temu po wylogowaniu użytkownik nie powinien wracać do chronionych ekranów przez przycisk „Wstecz” w przeglądarce.

### Fragment: nagłówki blokujące cache

`app/Http/Middleware/NoCacheHeaders.php`

```php
$response->headers->set('Cache-Control', 'no-cache, no-store, max-age=0, must-revalidate');
$response->headers->set('Pragma', 'no-cache');
$response->headers->set('Expires', 'Sat, 01 Jan 2000 00:00:00 GMT');
```

### Fragment: ochrona tras administratora

```php
Route::middleware(['auth', 'admin'])->group(function () {
    Route::get('/dashboard/users', [RegisteredUserController::class, 'index'])->name('users.index');
});
```

---

## 7. W jaki sposób aplikacja rozpoznaje, co użytkownik chce zrobić?

Aplikacja rozpoznaje akcję po adresie URL, metodzie HTTP i nazwie trasy. Przykładowo `POST /games/{game}/purchase` oznacza zakup gry, `POST /games/{game}/refund` oznacza zwrot gry, a `DELETE /admin/games/{game}` oznacza usunięcie gry przez administratora.

W formularzach Laravel używa tokenu CSRF oraz ukrytych pól metody, aby zwykły formularz HTML mógł wykonać akcję `PUT` albo `DELETE`.

### Fragment: trasy użytkownika

`routes/web.php`

```php
Route::middleware(['auth'])->group(function () {
    Route::post('/wallet/topup', [WalletController::class, 'topUp'])->name('wallet.topup');
    Route::post('/games/{game}/purchase', [GameController::class, 'purchase'])->name('games.purchase');
    Route::post('/games/{game}/refund', [GameController::class, 'refund'])->name('games.refund');
});
```

### Fragment: trasy administratora

```php
Route::middleware(['auth', 'admin'])->group(function () {
    Route::get('/admin/games/create', [GameController::class, 'create'])->name('games.create');
    Route::post('/admin/games', [GameController::class, 'store'])->name('games.store');
    Route::get('/admin/games/{game}/edit', [GameController::class, 'edit'])->name('games.edit');
    Route::put('/admin/games/{game}', [GameController::class, 'update'])->name('games.update');
    Route::delete('/admin/games/{game}', [GameController::class, 'destroy'])->name('games.destroy');
});
```

---

## 8. Opisz przykład powiązania danych w projekcie

Dobrym przykładem jest powiązanie użytkowników i gier. Jeden użytkownik może mieć wiele kupionych gier, a jedna gra może być kupiona przez wielu użytkowników. Jest to relacja wiele-do-wielu, zapisana w tabeli pośredniej `game_user`.

Tabela `game_user` przechowuje `user_id`, `game_id` i `purchase_price`. Dzięki temu aplikacja wie, kto kupił którą grę i za jaką cenę. Relacja jest widoczna w bazie danych, modelu `User` i w widoku biblioteki użytkownika.

### Fragment: migracja tabeli pośredniej

`database/migrations/2026_06_05_113620_create_game_user_table.php`

```php
Schema::create('game_user', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained()->cascadeOnDelete();
    $table->foreignId('game_id')->constrained()->cascadeOnDelete();
    $table->integer('purchase_price')->default(0);
    $table->timestamp('created_at')->nullable();
    $table->timestamp('updated_at')->nullable();
    $table->unique(['user_id', 'game_id']);
});
```

### Fragment: relacja w modelu użytkownika

`app/Models/User.php`

```php
public function games()
{
    return $this->belongsToMany(Game::class)->withPivot('purchase_price');
}
```

---

## 9. Jak aplikacja reaguje na nietypową sytuację?

Aplikacja obsługuje kilka sytuacji błędnych. Jeśli użytkownik nie ma wystarczającej liczby monet, zakup zostaje przerwany. Jeśli użytkownik już posiada grę, nie może kupić jej drugi raz. Jeśli próbuje zwrócić grę, której nie posiada, aplikacja zwraca komunikat błędu.

Dodatkowo administrator nie powinien usunąć samego siebie ani ostatniego konta administratora. To chroni system przed sytuacją, w której aplikacja zostaje bez administratora.

### Fragment: za mało monet

`app/Http/Controllers/GameController.php`

```php
if ($user->balance < $price) {
    return redirect()->back()->with('error', 'Masz za mało monet!');
}
```

### Fragment: blokada podwójnego zakupu

```php
if ($user->games()->where('game_id', $game->id)->exists()) {
    return redirect()->back()->with('error', 'Posiadasz już tę grę!');
}
```

### Fragment: zwrot tylko posiadanej gry

```php
$ownedGame = $user->games()->where('game_id', $game->id)->first();

if (! $ownedGame) {
    return redirect()->back()->with('error', 'Nie posiadasz tej gry, więc nie możesz jej zwrócić!');
}
```

### Fragment: ochrona ostatniego administratora

`app/Http/Controllers/Auth/RegisteredUserController.php`

```php
if ($user->is_admin && ! User::where('is_admin', true)->whereKeyNot($user->id)->exists()) {
    return redirect()->route('users.index')
        ->with('error', 'Bezpieczeństwo systemu: Nie możesz usunąć ostatniego administratora!');
}
```

---

## 10. Co warto poprawić w projekcie jako następny krok?

Najważniejszą rzeczą do poprawy jest przygotowanie kompletnego środowiska testowego, aby testy można było uruchamiać jedną komendą lokalnie lub na serwerze CI. W projekcie są testy autoryzacji administratora, ale warto dopilnować, żeby narzędzie testowe było zawsze dostępne po instalacji zależności.

Drugim usprawnieniem byłoby dodanie paginacji i wyszukiwarki do listy gier. Przy większej liczbie rekordów strona będzie wtedy wygodniejsza i szybsza w użyciu. Można też rozważyć upload okładek zamiast wpisywania samego adresu `image_url`.

---

## 11. Na jaką ocenę zasługuje projekt?

Projekt zasługuje na dobrą ocenę, ponieważ ma działające logowanie, role użytkowników, panel administratora, bazę PostgreSQL, relacje między użytkownikami i grami, portfel, zakup oraz zwrot gier. Po poprawkach system administratora jest bezpieczniejszy, ponieważ nie zależy od adresu e-mail, tylko od pola `is_admin` w bazie.

Projekt pokazuje znajomość Laravela: tras, kontrolerów, modeli, migracji, middleware, widoków Blade i relacji many-to-many. Dodatkowo została dodana komenda Artisan do bezpiecznego nadawania administratora istniejącemu użytkownikowi.

Nie jest to jeszcze projekt idealny, bo można rozbudować testy, paginację, upload grafik i panel raportów. Jednak jako projekt zaliczeniowy jest kompletny i pokazuje większość wymaganych elementów aplikacji internetowej.

---

## 12. Co poprawić, żeby dostać wyższą ocenę?

Aby projekt zasługiwał na wyższą ocenę, dodałbym pełną konfigurację testów automatycznych, paginację listy gier, wyszukiwarkę, upload okładek oraz bardziej rozbudowany panel administratora. Dobrym dodatkiem byłby raport sprzedaży: liczba zakupów, suma wydanych monet i najpopularniejsze gry.

Warto byłoby też przenieść część autoryzacji do polityk Laravela, jeśli projekt dalej by się rozrastał. Middleware jest dobry na tym etapie, ale przy większej liczbie zasobów polityki mogą być czytelniejsze.

---

