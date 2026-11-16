# Analizator siły hasła

Prosta aplikacja webowa do oceny siły hasła, napisana w czystym HTML/CSS/JavaScript (jeden plik, bez bibliotek). Projekt na zajęcia z cyberbezpieczeństwa.

## Opis

Narzędzie szacuje odporność hasła na atak siłowy (brute-force). Po wpisaniu hasła pokazuje jego **entropię**, szacowany **czas złamania** oraz listę spełnionych i niespełnionych **kryteriów bezpieczeństwa**. Dodatkowo umożliwia sprawdzenie, czy hasło pojawiło się w znanych **wyciekach danych**, korzystając z API Have I Been Pwned w modelu k-anonimowości.

Kluczowa cecha: analiza entropii odbywa się **lokalnie w przeglądarce**. Hasło nie jest nigdzie wysyłane ani zapisywane — to świadoma decyzja projektowa, bo narzędzie, które przesyłałoby sprawdzane hasło na serwer, byłoby samo w sobie zagrożeniem. Sprawdzenie wycieku również nie wysyła hasła (szczegóły niżej).

## Jak uruchomić

Otwórz plik `analizator-hasla.html` dwuklikiem w dowolnej przeglądarce. Nie wymaga instalacji, serwera ani połączenia z internetem.

## Jak to działa

### Entropia

Entropia hasła (w bitach) to miara jego nieprzewidywalności. Liczona jest wzorem:

```
E = L × log₂(N)
```

gdzie:
- `L` — długość hasła (liczba znaków),
- `N` — rozmiar puli znaków, z której hasło mogło pochodzić.

Pula znaków `N` zależy od użytych typów znaków:

| Typ znaku           | Dodaje do puli |
|---------------------|----------------|
| małe litery (a–z)   | 26             |
| wielkie litery (A–Z)| 26             |
| cyfry (0–9)         | 10             |
| znaki specjalne     | 33             |

Przykład: hasło `Kot@2026!` ma długość 9 i używa małych liter, wielkich liter, cyfr oraz znaków specjalnych, więc `N = 26 + 26 + 10 + 33 = 95`, a entropia to `9 × log₂(95) ≈ 59,1 bita`.

### Czas złamania

Liczba wszystkich możliwych kombinacji to `N^L`. Dzieląc ją przez zakładaną prędkość atakującego, otrzymujemy czas potrzebny na przejście całej przestrzeni:

```
czas = N^L / (próby na sekundę)
```

Aplikacja przyjmuje **10 miliardów prób na sekundę** — wartość realistyczna dla ataku offline na słabo zabezpieczony skrót (np. MD5 lub SHA-1 na GPU). To założenie pesymistyczne, czyli „gorszy przypadek" dla obrońcy.

### Progi oceny

| Entropia       | Ocena          |
|----------------|----------------|
| < 28 bitów     | bardzo słabe   |
| 28–40 bitów    | słabe          |
| 40–60 bitów    | umiarkowane    |
| 60–80 bitów    | silne          |
| > 80 bitów     | bardzo silne   |

### Sprawdzanie wycieku (model k-anonimowości)

Samo wymyślenie silnego hasła nie wystarcza — jeśli hasło wyciekło już w jakimś naruszeniu danych, trafia do słowników używanych przez atakujących, niezależnie od jego entropii. Aplikacja sprawdza to przez publiczne API [Have I Been Pwned](https://haveibeenpwned.com/Passwords).

Problem: jak zapytać serwer „czy znasz moje hasło?", nie ujawniając mu hasła? Rozwiązaniem jest **k-anonimowość**:

1. Hasło jest haszowane **lokalnie** w przeglądarce algorytmem SHA-1 (`crypto.subtle.digest`).
2. Do serwera wysyłane jest tylko **pierwszych 5 znaków** skrótu (tzw. prefiks).
3. Serwer odsyła **wszystkie** końcówki skrótów zaczynających się tym prefiksem (zwykle kilkaset) wraz z liczbą wystąpień w wyciekach.
4. Porównanie reszty skrótu z tą listą odbywa się **u klienta**.

Dzięki temu serwer nigdy nie wie, które konkretnie hasło sprawdzasz — widzi tylko prefiks pasujący do setek różnych haseł. Pełne hasło ani pełny skrót nie opuszczają przeglądarki.

> **Uwaga techniczna:** ta funkcja wymaga **bezpiecznego kontekstu** (HTTPS lub localhost), ponieważ Web Crypto API nie działa przy otwarciu pliku przez `file://`. Dlatego do pełnego działania projekt należy uruchomić na GitHub Pages lub Vercel. Analiza entropii działa niezależnie, również offline.

## Ograniczenia i uproszczenia

- Model entropii zakłada hasło **losowe**. Hasło zbudowane ze słów ze słownika (np. `CzerwonyRower123`) ma w praktyce entropię znacznie niższą niż wynika ze wzoru, bo atak słownikowy nie przechodzi po pojedynczych znakach. Wzór `L × log₂(N)` to więc *górne* oszacowanie.
- Założona prędkość ataku dotyczy słabych algorytmów skrótu. Nowoczesne funkcje haszujące hasła (**bcrypt**, **scrypt**, **Argon2**) są celowo wolne i wydłużają czas złamania o wiele rzędów wielkości.
- Analiza entropii ocenia wyłącznie strukturę hasła. Sprawdzenie wycieku (HIBP) uzupełnia ją o drugie, niezależne kryterium, ale dotyczy tylko haseł, które faktycznie trafiły do znanych baz naruszeń — brak trafienia nie jest gwarancją bezpieczeństwa.
- Wykorzystany tu SHA-1 służy wyłącznie do odpytania API HIBP. Nie jest to zalecany algorytm do *przechowywania* haseł — do tego używa się bcrypt, scrypt lub Argon2.

## Dobre praktyki (wnioski)

- Długość ma większe znaczenie niż złożoność — każdy dodatkowy znak zwiększa entropię liniowo.
- Najlepsze są długie, losowe hasła lub passphrase'y generowane przez menedżer haseł.
- Unikać słów ze słownika, dat i typowych wzorców (`qwerty`, `123456`, `admin`).

## Struktura projektu

```
.
├── analizator-hasla.html   # cała aplikacja (HTML + CSS + JS)
└── README.md               # ten plik
```
