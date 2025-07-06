# Analiza błędów w skryptach rsync-incremental-backup

## 🔴 Krytyczne błędy

### 1. Niepoprawna transformacja ścieżek
**Problem:** W liniach 18/21/19 we wszystkich skryptach:
```bash
tempLogPath="${ownFolderPath}/local_${dst//[\/]/\\}"
```
**Błąd:** Zastąpienie `/` na `\` w ścieżce jest niepoprawne w systemach Unix/Linux. Powoduje to tworzenie nieprawidłowych nazw katalogów.

**Poprawka:** Powinno być:
```bash
tempLogPath="${ownFolderPath}/local_${dst//[\/]/_}"
```

### 2. Błędna logika rotacji w skrypcie remote
**Problem:** Linia 67 w `rsync-incremental-backup-remote`:
```bash
if (ssh ${remote} "[ ! -d ${partialFolderPath} ] && [ ! -e ${rotationLockFilePath} ]")
```
**Błąd:** Używa `&&` zamiast `||`, co oznacza że rotacja NIE wystąpi jeśli:
- Folder partial nie istnieje AND plik lock nie istnieje
- To jest odwrotność zamierzonej logiki

**Poprawka:** Powinno być:
```bash
if (ssh ${remote} "[ ! -d ${partialFolderPath} ] || [ ! -e ${rotationLockFilePath} ]")
```

### 3. Brak sprawdzania błędów SSH
**Problem:** Wszystkie połączenia SSH w skrypcie remote nie sprawdzają statusu wyjścia.

**Przykład problemu:**
```bash
ssh ${remote} "mkdir -p ${dst} ${logPath}"
# Brak sprawdzenia czy się udało
```

## 🟡 Poważne problemy

### 4. Niewłaściwe użycie `true` i `export`
**Problem:** W liniach 44/45, 74, 83:
```bash
export bak$i="${dst}/${pathBakN}/${nameBakN}.$i"
true $((i = i + 1))
```
**Problemy:**
- `export` dynamicznych zmiennych jest niepotrzebne i potencjalnie problematyczne
- `true $((i = i + 1))` jest dziwną konstrukcją; `true` zawsze zwraca 0, więc `$((i = i + 1))` nie jest używane

**Poprawka:**
```bash
declare bak$i="${dst}/${pathBakN}/${nameBakN}.$i"
((i = i + 1))
```

### 5. Brak walidacji konfiguracji
**Problem:** Skrypty nie sprawdzają czy:
- Katalog źródłowy istnieje
- Katalog docelowy jest dostępny
- Plik exclude.txt istnieje (jeśli jest używany)
- Zmienne konfiguracyjne są poprawnie ustawione

### 6. Problemy z quotowaniem
**Problem:** Zmienne nie są prawidłowo quotowane, co może powodować problemy ze ścieżkami zawierającymi spacje.

**Przykład:**
```bash
rsync ${rsyncFlags} ${src}/ ${bak0}/
```
**Powinno być:**
```bash
rsync ${rsyncFlags} "${src}/" "${bak0}/"
```

### 7. Niekonsekwentna obsługa błędów rsync
**Problem:** W skrypcie remote, gdy rsync się nie udaje, plik log jest i tak kopiowany przez `scp`, ale nie sprawdza się czy się udało.

**Problematyczny kod:**
```bash
scp ${logFile} ${remoteLogPath}
if [ "$?" -eq "0" ]
then
    rm ${logFile}
fi
```

## 🟠 Problemy średniej wagi

### 8. Brak sprawdzenia uprawnień
**Problem:** Skrypty nie sprawdzają czy mają uprawnienia do:
- Odczytu z katalogu źródłowego
- Zapisu w katalogu docelowym
- Tworzenia plików lock

### 9. Problemy z concurrent execution
**Problem:** Mechanizm lock file nie chroni przed jednoczesnym uruchomieniem skryptu (race condition).

### 10. Hardcoded exclusions w skrypcie system
**Problem:** Linia 109-110:
```bash
rsync ${rsyncFlags} --exclude={"/dev/*","/proc/*","/sys/*","/tmp/*","/run/*","/mnt/*","/media/*","/lost+found"} \
${src}/ ${bak0}/
```
**Problem:** Hardcoded wykluczenia mogą nie być odpowiednie dla wszystkich systemów.

## 🟢 Drobne problemy

### 11. Zbędne nawiasy
**Problem:** Używanie `([ condition ])` zamiast `[[ condition ]]` lub `[ condition ]`.

### 12. Nieoptymalne wywołania date
**Problem:** Wielokrotne wywołania `date` w tej samej linii:
```bash
logName="rsync-incremental-backup_$(date -Id)_$(date +%H-%M-%S).log"
```

### 13. Brak sprawdzenia dostępności rsync
**Problem:** Skrypty nie sprawdzają czy `rsync` jest zainstalowany i dostępny.

## 🔧 Rekomendacje poprawek

1. **Natychmiastowe:** Poprawić transformację ścieżek i logikę rotacji
2. **Krytyczne:** Dodać walidację konfiguracji i sprawdzanie błędów SSH
3. **Ważne:** Poprawić quotowanie zmiennych i obsługę błędów
4. **Zalecane:** Dodać sprawdzenie uprawnień i concurrent execution protection

## 📋 Podsumowanie

Skrypty zawierają **13 kategorii problemów**, z czego **3 są krytyczne** i mogą powodować nieprawidłowe działanie lub utratę danych. Większość problemów może być łatwo naprawiona, ale wymagają one systematycznego podejścia do testowania po wprowadzeniu zmian.