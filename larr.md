# Odpowiedzi do projektu

## 1. Rozróżnianie uprawnień

Model `User` jest powiązany z modelem `Role`. Metody `isAdmin()` i `isRegularUser()` sprawdzają rolę użytkownika. Przykład: zwykły użytkownik nie może wejść do `/admin`, ponieważ trasa jest chroniona przez `auth` oraz `can:access-admin-panel`, a Gate sprawdza, czy użytkownik jest administratorem.

**Fragment kodu z projektu – `app/Models/User.php`:**

```php
public function isAdmin(): bool
{
    return $this->role?->name === Role::ADMIN;
}

public function isRegularUser(): bool
{
    return $this->role?->name === Role::USER;
}
```

**Fragment kodu z projektu – `app/Providers/AppServiceProvider.php` i `routes/web.php`:**

```php
Gate::define('access-admin-panel', fn (User $user) => $user->isAdmin());

Route::middleware(['auth', 'can:access-admin-panel'])
    ->prefix('admin')
    ->name('admin.')
    ->group(function () {
        // trasy panelu administratora
    });
```

## 2. Akcja – zakup karnetu

1. **Widok `klient.blade.php`**: przycisk „Kup teraz” wysyła formularz `POST` na `/klient/karnety/{ticket}/kup`.

   ![Widok klienta – formularz zakupu](odpowiedzi_z_kodem_assets/01_widok_klienta.png)

   **Kod odpowiadający temu etapowi:**

   ```blade
   <form action="{{ route('client.tickets.buy', $ticket) }}" method="POST">
       @csrf
       <button type="submit"
           class="bg-blue-600 text-white px-4 py-2 rounded-lg text-xs font-bold hover:bg-blue-700 transition shadow-sm uppercase tracking-wider cursor-pointer">
           Kup teraz
       </button>
   </form>
   ```

2. **`web.php`**: routing kieruje żądanie do `ClientDashboardController::buyTicket`.

   ![Routing zakupu karnetu](odpowiedzi_z_kodem_assets/02_routing_web.png)

   **Kod odpowiadający temu etapowi:**

   ```php
   Route::middleware('auth')->group(function () {
       Route::get('/klient', [ClientDashboardController::class, 'index'])->name('klient');
       Route::post('/klient/karnety/{ticket}/kup', [ClientDashboardController::class, 'buyTicket'])
           ->name('client.tickets.buy');
       Route::delete('/klient/karnety/{purchase}', [ClientDashboardController::class, 'cancelTicket'])
           ->name('client.tickets.destroy');
   });
   ```

3. **Kontroler** sprawdza użytkownika i wywołuje `TicketPurchaseService::buy`.

   ![Kontroler buyTicket](odpowiedzi_z_kodem_assets/03_kontroler_buyTicket.png)

   **Kod odpowiadający temu etapowi:**

   ```php
   public function buyTicket(Ticket $ticket): RedirectResponse
   {
       $user = request()->user()->loadMissing('role');
       $this->ensureRegularUser($user);

       $this->ticketPurchaseService->buy($user, $ticket);

       return redirect()->route('klient')
           ->with('success', 'Karnet kupiony pomyślnie.');
   }
   ```

4. **Serwis** wykonuje zapis zakupu do bazy przez Eloquent.

   ![Serwis zakupu karnetu](odpowiedzi_z_kodem_assets/04_serwis_zakupu.png)

   **Kod odpowiadający temu etapowi:**

   ```php
   public function buy(User $user, Ticket $ticket): TicketPurchase
   {
       return DB::transaction(function () use ($user, $ticket) {
           $purchase = $user->ticketPurchases()->create([
               'ticket_id' => $ticket->id,
               'zakupiono' => now(),
           ]);

           Log::info('ticket.purchase.created', [
               'user_id' => $user->id,
               'ticket_id' => $ticket->id,
               'purchase_id' => $purchase->id,
           ]);

           return $purchase;
       });
   }
   ```

5. Relacja `ticketPurchases()` w `User.php` łączy użytkownika z jego zakupami i umożliwia zapis oraz pobieranie powiązanych rekordów przez Eloquent.

   ![Relacja ticketPurchases w User](odpowiedzi_z_kodem_assets/05_relacja_user.png)

   **Kod odpowiadający temu etapowi:**

   ```php
   public function ticketPurchases(): HasMany
   {
       return $this->hasMany(TicketPurchase::class);
   }
   ```
## 3. Poprawność danych

`TicketCatalogFilterRequest` sprawdza dane formularza filtrowania. `price_min` i `price_max` muszą być liczbami, a `type` może mieć tylko dozwolone wartości, np. `CZASOWY` lub `ILOSCIOWY`. Jeżeli cena maksymalna jest niższa od minimalnej, również powstaje błąd walidacji. Przy błędnych danych Laravel odrzuca walidację i wraca do formularza z informacją o błędach.

**Fragment kodu z projektu – `app/Http/Requests/TicketCatalogFilterRequest.php`:**

```php
public function rules(): array
{
    return [
        'search' => ['nullable', 'string', 'max:255'],
        'type' => ['nullable', 'in:CZASOWY,ILOSCIOWY'],
        'price_min' => ['nullable', 'numeric', 'min:0'],
        'price_max' => ['nullable', 'numeric', 'min:0'],
    ];
}
```

```php
if (
    $this->filled('price_min')
    && $this->filled('price_max')
    && (float) $this->input('price_max') < (float) $this->input('price_min')
) {
    $validator->errors()->add(
        'price_max',
        'Cena maksymalna nie może być niższa od ceny minimalnej.'
    );
}
```

## 4. Dobrze zorganizowany widok

Widok `klient.blade.php` jest podzielony na osobne sekcje dotyczące konta, posiadanych karnetów i oferty. Wykorzystuje układ Grid w Tailwind CSS oraz wspólny `layout.blade.php`, dzięki czemu nie trzeba powtarzać wspólnych elementów strony i łatwiej ją rozbudować.

**Fragment kodu z projektu – `resources/views/klient.blade.php`:**

```blade
@extends('layout')

@section('content')

<div class="grid grid-cols-1 lg:grid-cols-3 gap-8 mb-8">
    <div class="flex flex-col gap-8">
        <!-- sekcja konta i posiadanych karnetów -->
    </div>

    <!-- sekcja cennika -->
</div>
@endsection
```

Wspólny layout wstawia zawartość widoku przez:

```blade
@yield('content')
```

## 5. „Niewidoczna” funkcjonalność

Przykładem jest logowanie zdarzeń w tle. W `TicketPurchaseService::buy` wykonywane jest `Log::info('ticket.purchase.created', [...])`. Użytkownik tego nie widzi, ale zapis pozwala śledzić wykonane operacje i ułatwia diagnozowanie problemów.

**Fragment kodu z projektu – `app/Services/TicketPurchaseService.php`:**

```php
Log::info('ticket.purchase.created', [
    'user_id' => $user->id,
    'ticket_id' => $ticket->id,
    'purchase_id' => $purchase->id,
]);
```

## 6. Rozpoznawanie intencji użytkownika

Aplikacja rozpoznaje akcję na podstawie adresu URL i metody HTTP. Przykładowo usunięcie zakupu wykorzystuje trasę `/klient/karnety/{purchase}` i metodę `DELETE`, przekazywaną z formularza przez `@method('DELETE')`. Routing kieruje następnie żądanie do odpowiedniej metody kontrolera.

**Fragment kodu z projektu – `routes/web.php`:**

```php
Route::delete(
    '/klient/karnety/{purchase}',
    [ClientDashboardController::class, 'cancelTicket']
)->name('client.tickets.destroy');
```

**Fragment kodu z projektu – `resources/views/klient.blade.php`:**

```blade
<form action="{{ route('client.tickets.destroy', $my) }}" method="POST">
    @csrf
    @method('DELETE')
    <button type="submit">Anuluj</button>
</form>
```

## 7. Powiązania danych

Użytkownik i jego zakupy są powiązani przez tabelę `user_tickets`. Zawiera ona identyfikator użytkownika i zakupionego karnetu. W modelu `User` relacja `ticketPurchases()` typu `HasMany` pozwala pobrać wszystkie zakupy danego użytkownika i wyświetlić je w jego panelu.

**Fragment kodu z projektu – `app/Models/User.php`:**

```php
public function ticketPurchases(): HasMany
{
    return $this->hasMany(TicketPurchase::class);
}
```

**Fragment kodu z projektu – `app/Models/TicketPurchase.php`:**

```php
protected $table = 'user_tickets';

public function user(): BelongsTo
{
    return $this->belongsTo(User::class);
}

public function ticket(): BelongsTo
{
    return $this->belongsTo(Ticket::class);
}
```

## 8. Nietypowa sytuacja

Przykładem jest próba anulowania zakupu należącego do innego użytkownika. `TicketPurchaseService::cancel` porównuje `user_id` zakupu z ID zalogowanego użytkownika. Jeżeli się nie zgadzają, zgłaszany jest błąd autoryzacji i użytkownik otrzymuje odpowiedź `403`.

**Fragment kodu z projektu – `app/Services/TicketPurchaseService.php`:**

```php
if ($purchase->user_id !== $user->id) {
    throw new AuthorizationException(
        'Nie możesz anulować cudzego zakupu.'
    );
}
```

Aplikacja posiada również własny widok błędu `403` w `resources/views/errors/403.blade.php`.

## 9. Do poprawy technicznej

W formularzu filtrów w `klient.blade.php` dane są walidowane, ale błędy nie są wyświetlane bezpośrednio pod polami. Walidacja działa w `TicketCatalogFilterRequest`, jednak przy polach formularza nie ma dyrektyw `@error`. Można to poprawić, wyświetlając komunikat bezpośrednio pod odpowiednim polem.

**Aktualny fragment kodu z projektu – `resources/views/klient.blade.php`:**

```blade
<input
    id="ticketPriceMin"
    type="number"
    step="0.01"
    min="0"
    name="price_min"
    value="{{ $filters['price_min'] ?? '' }}"
    class="w-full px-3 py-2 rounded-lg border border-gray-200"
>
```

**Proponowana poprawka techniczna:**

```blade
@error('price_min')
    <p class="text-red-600 text-xs mt-1">{{ $message }}</p>
@enderror
```

## 10. Uzasadnienie oceny 3.0 i najważniejsza poprawka

Projekt spełnia wymagania na ocenę 3.0, ponieważ udostępnia publiczne zasoby aplikacji. Dla karnetów został wykonany pełny CRUD: Create, Read, Update i Delete. System posiada administratora oraz zwykłego użytkownika z różnymi uprawnieniami. Projekt korzysta z migracji, seederów, Eloquent i relacji między modelami. Formularze wykorzystują walidację, a zakup karnetu obsługuje osobna warstwa serwisowa. W pierwszej kolejności poprawiłbym wyświetlanie błędów walidacji, ponieważ zwiększyłoby to czytelność aplikacji dla użytkownika.

**Przykładowe fragmenty kodu potwierdzające wymagania na 3.0 – `routes/web.php`:**

```php
// publicznie dostępny zasób
Route::get('/cennik', [PoolPageController::class, 'pricing'])
    ->name('pricing');

// CRUD karnetów
Route::resource('tickets', TicketController::class)
    ->except(['index']);

// panel administratora zabezpieczony uprawnieniami
Route::middleware(['auth', 'can:access-admin-panel'])
    ->prefix('admin')
    ->name('admin.')
    ->group(function () {
        // trasy administracyjne
    });
```
