# Odpowiedzi do projektu

## 1. Rozróżnianie uprawnień

Model `User` jest powiązany z modelem `Role`. Metody `isAdmin()` i `isRegularUser()` sprawdzają rolę użytkownika. Przykład: zwykły użytkownik nie może wejść do `/admin`, ponieważ trasa jest chroniona przez `auth` oraz `can:access-admin-panel`, a Gate sprawdza, czy użytkownik jest administratorem.

## 2. Akcja – zakup karnetu

1. **Widok `klient.blade.php`**: przycisk „Kup teraz” wysyła formularz `POST` na `/klient/karnety/{ticket}/kup`.

   ![Widok klienta – formularz zakupu](01_widok_klienta.png)

2. **`web.php`**: routing kieruje żądanie do `ClientDashboardController::buyTicket`.

   ![Routing zakupu karnetu](odpowiedzi_assets/02_routing_web.png)

3. **Kontroler** sprawdza użytkownika i wywołuje `TicketPurchaseService::buy`.

   ![Kontroler buyTicket](odpowiedzi_assets/03_kontroler_buyTicket.png)

4. **Serwis** wykonuje zapis zakupu do bazy przez Eloquent.

   ![Serwis zakupu karnetu](odpowiedzi_assets/04_serwis_zakupu.png)

5. Relacja `ticketPurchases()` w `User.php` łączy użytkownika z jego zakupami i umożliwia zapis oraz pobieranie powiązanych rekordów przez Eloquent.

   ![Relacja ticketPurchases w User](odpowiedzi_assets/05_relacja_user.png)

## 3. Poprawność danych

`TicketCatalogFilterRequest` sprawdza dane formularza filtrowania. `price_min` i `price_max` muszą być liczbami, a `type` może mieć tylko dozwolone wartości, np. `CZASOWY` lub `ILOSCIOWY`. Przy błędnych danych Laravel odrzuca walidację i wraca do formularza z informacją o błędach.

## 4. Dobrze zorganizowany widok

Widok `klient.blade.php` jest podzielony na osobne sekcje dotyczące konta, posiadanych karnetów i oferty. Wykorzystuje układ Grid w Tailwind CSS oraz wspólny `layout.blade.php`, dzięki czemu nie trzeba powtarzać wspólnych elementów strony i łatwiej ją rozbudować.

## 5. „Niewidoczna” funkcjonalność

Przykładem jest logowanie zdarzeń w tle. W `TicketPurchaseService::buy` wykonywane jest `Log::info('ticket.purchase.created', [...])`. Użytkownik tego nie widzi, ale zapis pozwala śledzić wykonane operacje i ułatwia diagnozowanie problemów.

## 6. Rozpoznawanie intencji użytkownika

Aplikacja rozpoznaje akcję na podstawie adresu URL i metody HTTP. Przykładowo usunięcie zakupu wykorzystuje trasę `/klient/karnety/{purchase}` i metodę `DELETE`, przekazywaną z formularza przez `@method('DELETE')`. Routing kieruje następnie żądanie do odpowiedniej metody kontrolera.

## 7. Powiązania danych

Użytkownik i jego zakupy są powiązani przez tabelę `user_tickets`. Zawiera ona identyfikator użytkownika i zakupionego karnetu. W modelu `User` relacja `ticketPurchases()` typu `HasMany` pozwala pobrać wszystkie zakupy danego użytkownika i wyświetlić je w jego panelu.

## 8. Nietypowa sytuacja

Przykładem jest próba anulowania zakupu należącego do innego użytkownika. `TicketPurchaseService::cancel` porównuje `user_id` zakupu z ID zalogowanego użytkownika. Jeżeli się nie zgadzają, zgłaszany jest błąd autoryzacji i użytkownik otrzymuje odpowiedź `403`.

## 9. Do poprawy technicznej

W formularzu filtrów w `klient.blade.php` dane są walidowane, ale błędy nie są wyświetlane bezpośrednio pod polami. Można to poprawić, dodając np. `@error('price_min') {{ $message }} @enderror`, dzięki czemu użytkownik zobaczy, co wpisał niepoprawnie.

## 10. Uzasadnienie oceny 3.0 i najważniejsza poprawka

Projekt spełnia wymagania na ocenę 3.0, ponieważ udostępnia publiczne zasoby aplikacji. Dla karnetów został wykonany pełny CRUD: Create, Read, Update i Delete. System posiada administratora oraz zwykłego użytkownika z różnymi uprawnieniami. Projekt korzysta z migracji, seederów, Eloquent i relacji między modelami. Formularze wykorzystują walidację, a zakup karnetu obsługuje osobna warstwa serwisowa. W pierwszej kolejności poprawiłbym wyświetlanie błędów walidacji, ponieważ zwiększyłoby to czytelność aplikacji dla użytkownika.
