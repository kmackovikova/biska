#BIS

##Implementace www stránky s přístupovým heslem, její bezpečnost a rychlost, ukázka útoku hrubou silou

V rámci práce vytvoříme testovací webovou stránku, zahrnující uživatelský účet. Budeme se zabývat také hlediskem 
zabezpečení této stránky a rychlostí jejího fungování. Zabezpečení webové stránky otestujeme demonstrací útoku
hrubou silou, kdy se pokusíme náhodně volit hesla a také použitím rainbow tables (tabulek obsahujících hashe,
pomocí nichž je možné nalézt shodu s hashem např. z odcizené databáze a získat tak heslo k účtu).

##Instalace

Abychom mohli rozjet projekt, potřebujeme nainstalovat dva hlavní programy, kterými jsou sqliteman a virtualenv.
Pomocí sqlitemanu můžeme prohlížet obsah odcizené databáze (ze které budeme porovnávat hashe s cílem získat heslo
k uživatelskému účtu). Virtualenv nám vytváří izolované Python prostředí, které nám slouží k nainstalování knihovny
pro běh tohoto projektu (rojekt tedy není nainstalován do systému, ale do tohoto virtuálního prostředí). Příkazem
activate spustíme složku, aby bylo možné do ní instalovat knihovny.
    
    sudo aptitude install sqliteman virtualenv
    cd PycharmProjects/biska
    virtualenv bis
    source bis/bin/activate
    pip install -r requirements.txt
       
## Popis webové aplikace

Webová stránka je implementována v programovacím jazyce Python s využitím webového frameworku Django.
Django jsme zvolila z důvodu urychlení vývoje webové stránky.

### Spuštění webové stránky

Pro spuštění na adrese [http://localhost:8000/](http://localhost:8000/) 
Pomocí prvního příkazu vytvoříme webovou databázi (tato činnost se provádí jen jednou) a pomocí druhého web spustíme.

    python app.py syncdb
    python app.py runserver
    
Vytvoříme uživatele.

    python app.py createsuperuser --username=admin --email=admin@test.com
    
### Funkce webové stránky

Webová stránka umožňuje přihlášení uživatele pomocí přihlašovacího jména a hesla, zadaného do formuláře. Po přihlášení
získá uživatel přístup k chráněným informacím.

### Bezpečnost webové stránky

V databázi používám pro ukládání hesel velmi slabé zabezpečení - neosolený algoritmus MD5. Zvolila jsem ho, abychom
viděli, jak je snadné ho prolomit a proč by se neměl používat. Jediným bezpečnějším použitím tohoto algoritmu je přidání
soli, díky které se změní hash a nedají se pak využít k prolomení hesla rainbow tabulky. 
Měla jsem v úmyslu pro zvýšení bezpečnosti webové stránky využít možnosti blokace účtu po několika neúspěšných
přihlášeních, ale bohužel to již nebylo v časových možnostech.
Formulář využívá zabudované bezpečnostní funkce frameworku Django, ochrany proti [CSRF útoku](http://cs.wikipedia.org/wiki/Cross-site_request_forgery)

## Útok

### Útok na běžící webovou stránku

U webové stránky jsem si řekla, že uživatelem bude "admin" a napsala jsem si skript, který se pokouší přihlásit na
stránku pod uživatelem "admin" a postupně zkouší všechna možná hesla složená z malých písmen anglické abecedy o
rozsahu jeden až čtyři znaky (více jsem nezkoušela z hlediska časové náročnosti - chtěla jsem jen demonstrovat použití
tohoto způsobu útoku).

    python hackit.py
    zkouseni 26 hesel o delce 1 trvalo 0.396 s
    zkouseni 676 hesel o delce 2 trvalo 9.017 s
    Heslo je abc
    Jeden pokus trval prumerne 0.0133911237325 s

Jak vidíme, v případě krátkého hesla je tento typ útoku snadno proveditelný a proto je potřeba ještě další zabezpečnení
(např. dříve zmíněné zablokování IP adresy, ze které je útok veden po několika neúspěšných pokusech o přihlášení nebo
jinou ochranou, např. limitováním počtu pokusů např. na jeden za vteřinu).

Implementovaná stránka odpovídala velmi rychle, průměrně za 0.014 s, což je v tomto případě na škodu, protože to 
umožňuje tento typ útoku.

### Útok s využitím rainbow tabulek (duhových tabulek)

Na webových stránkách [https://www.freerainbowtables.com](https://www.freerainbowtables.com) jsem stáhla program na použití rainbow tabulek, nicméně v rámci
datové a časové úspory jsem stáhla jen tabulku, která obsahuje hashe o kombinacích malých písmen a čísel (a-z, 0-9) délky
do 8 znaků.

#### Ukázka použitého hashe z pokusné DB

Tato varianta je zajímavá, když získáme přístup k nějaké databázi, kde vidíme hashe. Pomocí rainbow tabulek je pak
můžeme porovnat a prolomit heslo. Jakékoliv heslo, jehož hash je obsahem rainbow tabulky a k němuž máme hash z databáze,
odhalí tyto tabulky během okamžiku.

./rcracki_mt -h bae60998ffe4923b131e3d6e4c19993e "cesta k adresari s rainbow tabulkami"
   
    statistics
    -------------------------------------------------------
    plaintext found:                          1 of 1(100.00%)
    total disk access time:                   107.14s
    total cryptanalysis time:                 5.20s
    total pre-calculation time:               12.62s
    total chain walk step:                    49985001
    total false alarm:                        5544
    total chain walk step due to false alarm: 20415090

    result
    -------------------------------------------------------
    bae60998ffe4923b131e3d6e4c19993e	bad	hex:626164

#### Ukázka dlouhé slovo - jen  písmena

./rcracki_mt -h e8dc4081b13434b45189a720b77b6818 "cesta k adresari s rainbow tabulkama"

    statistics
    -------------------------------------------------------
    plaintext found:                          1 of 1(100.00%)
    total disk access time:                   57.22s
    total cryptanalysis time:                 2.13s
    total pre-calculation time:               12.39s
    total chain walk step:                    49985001
    total false alarm:                        2240
    total chain walk step due to false alarm: 8190237
    
    result
    -------------------------------------------------------
    e8dc4081b13434b45189a720b77b6818	abcdefgh	hex:6162636465666768


#### Ukázka kombinovaná písmena a čísla

./rcracki_mt -h 11cfd4fab26bbf25df59f59fb8ccd9b1 "cesta k adresari s rainbow tabulkama"

    statistics
    -------------------------------------------------------
    plaintext found:                          1 of 1(100.00%)
    total disk access time:                   190.39s
    total cryptanalysis time:                 7.94s
    total pre-calculation time:               25.68s
    total chain walk step:                    99970002
    total false alarm:                        8075
    total chain walk step due to false alarm: 30700068
    
    result
    -------------------------------------------------------
    11cfd4fab26bbf25df59f59fb8ccd9b1	068877op	hex:3036383837376f70


Nebo to samé můžeme docílit s pomocí webu, bez nutnosti něco stahovat.

[http://www.md5crack.com/](http://www.md5crack.com/)


### Bruteforce pomoci Hascat

Také útok hrubou silou s využitím programu který zkouší všechny možnosti, všechny délky určitých znaků. Můžeme zde
využít možnosti, kdy víme, jak má heslo vypadat - program poté odhalí heslo snadněji. V případě dlouhých hesel je
tato varianta časově velmi náročná (už pro heslo o 8 znacích při využití PC se 4jádrovým procesorem jsem čekání vzdala
po 10 minutách).

Zadání typu hesla:

    ?l = abcdefghijklmnopqrstuvwxyz
    ?u = ABCDEFGHIJKLMNOPQRSTUVWXYZ
    ?d = 0123456789
    ?s = «space»!"#$%&'()*+,-./:;<=>?@[\]^_`{|}~
    ?a = ?l?u?d?s
    ?b = 0x00 - 0xff

    
    ./hashcat-cli64.bin -m0 -a3 hashes.txt ?l?l?l # abecední znaky o délce 3
    ./hashcat-cli64.bin -m0 -a3 hashes.txt ?a?a?a # všechny znaky o délce 3
    ./hashcat-cli64.bin -m0 -a3 hashes.txt ?l?d?l?d?l?d?l?d?l?d?l?d?l?d?l?d # znaky o délce 8 ve formátu 8x[0-9a-z]
    ./hashcat-cli64.bin -m0 -a3 hashes.txt ?d?d?d?d?d?d?l?l (123456az) 6x[0-9]2x[a-z]