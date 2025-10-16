# Analiza pliku `.htaccess`

Poniżej znajduje się opis i interpretacja poszczególnych dyrektyw z przekazanego pliku `.htaccess`, wraz z oceną ich wpływu na możliwość crawlowaia strony przez Ahrefs.

## 1. Blokada i przekierowania dla `/collections/`, `/shopdetail/` oraz `/pcmypage`

```apache
<IfModule mod_rewrite.c>
RewriteEngine On
RewriteRule ^collections/.*$ https://drewnobalcer.pl/ [R=301,L]
RewriteRule ^shopdetail/.*$ - [G,L]
RewriteCond %{QUERY_STRING} callback=product/detail/.*$
RewriteRule ^pcmypage$ - [G,L]
</IfModule>
```

- `RewriteEngine On` włącza mechanizm mod_rewrite.
- `RewriteRule ^collections/.*$ https://drewnobalcer.pl/ [R=301,L]` – każde żądanie zaczynające się od `/collections/` jest przekierowywane kodem 301 na stronę główną. To nie powinno blokować Ahrefs (zostanie przekierowany), ale powoduje, że te podstrony znikają z indeksu, bo w praktyce nie istnieją.
- `RewriteRule ^shopdetail/.*$ - [G,L]` – każde żądanie w katalogu `/shopdetail/` kończy się kodem 410 (Gone). Opcja `-` oznacza brak przekierowania, a `[G]` to skrót od `gone` (410). Zgłoszenie kodu 410 mówi robotom, że treść została trwale usunięta.
- `RewriteCond %{QUERY_STRING} callback=product/detail/.*$` + `RewriteRule ^pcmypage$ - [G,L]` – dla adresów `/pcmypage` z parametrem `callback=product/detail/...` zwracany jest kod 410. Bez tego parametru strona może działać normalnie.

## 2. Nagłówki `X-Robots-Tag`

```apache
<IfModule mod_headers.c>
    SetEnvIf Request_URI "^/collections/" noindex
    SetEnvIf Request_URI "^/shopdetail/" noindex
    SetEnvIf Request_URI "^/pcmypage$" noindex
    SetEnvIf Query_String "callback=product/detail/.*" noindex
    Header set X-Robots-Tag "noindex, nofollow" env=noindex
</IfModule>
```

- `SetEnvIf` ustawia zmienną środowiskową `noindex` dla wszystkich żądań, których ścieżka lub query string pasują do podanych wzorców. W praktyce te same adresy, które w sekcji wyżej otrzymują przekierowanie 301 lub kod 410, dostają też nagłówek `X-Robots-Tag: noindex, nofollow`.
- `Header set X-Robots-Tag "noindex, nofollow" env=noindex` dodaje wspomniany nagłówek, gdy zmienna `noindex` jest ustawiona. Roboty powinny więc unikać indeksowania tych stron. Nie wpływa to na resztę serwisu.

## 3. Reguły WordPressa

```apache
<IfModule mod_rewrite.c>
RewriteEngine On
RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]
RewriteBase /
RewriteRule ^index\.php$ - [L]
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule . /index.php [L]
</IfModule>
```

To standardowe reguły WordPressa:
- Przekazanie nagłówka `Authorization` do zmiennej środowiskowej (potrzebne dla REST API i logowania).
- Ustawienie katalogu podstawowego `RewriteBase /`.
- Umożliwienie bezpośredniego dostępu do `index.php`.
- W przypadku, gdy żądany plik ani katalog nie istnieją, WordPress przejmuje obsługę żądania (ładując `index.php`).

## 4. Sekcja `<Limit GET POST>`

```apache
<Limit GET POST>
order allow,deny
allow from all

# 🔒 Blokada podejrzanych IP (Singapur - ahrefs.net proxy)
deny from 202.8.43.232
deny from 202.8.43.233
deny from 202.8.43.234
deny from 202.8.43.235
deny from 202.8.43.236
deny from 92.204.40.219
</Limit>
```

- Sekcja `<Limit GET POST>` pozwala kontrolować dostęp dla żądań metodami GET i POST.
- `order allow,deny` oznacza, że Apache najpierw zastosuje reguły `allow`, a następnie `deny`. Ponieważ mamy `allow from all`, a potem konkretne `deny`, efekt jest taki, że każdy może wejść **oprócz** adresów IP wymienionych w `deny`.
- IP z komentarza to serwery Ahrefs (proxy z Singapuru). Te wpisy sprawiają, że wszystkie żądania crawlera Ahrefs są blokowane już na poziomie serwera WWW. To najprawdopodobniej główna przyczyna braku możliwości crawlowaia.

## Wnioski

1. Przekierowania i kody 410 dla `/collections/`, `/shopdetail/` i `/pcmypage` ograniczają indeksowanie tych sekcji, ale nie blokują Ahrefs na poziomie całej domeny.
2. Nagłówki `X-Robots-Tag` dodatkowo informują roboty, aby tych adresów nie indeksowały.
3. Reguły WordPressa są standardowe i nie stanowią problemu.
4. **Najważniejsze:** wpisy `deny from 202.8.43.232/233/234/235/236` i `deny from 92.204.40.219` w sekcji `<Limit>` blokują IP należące do Ahrefs. Aby umożliwić crawl, należy usunąć te dyrektywy lub w inny sposób dopuścić ruch z adresów Ahrefs.

