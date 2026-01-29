# Automatyzacja Bitrix-GUS-Firmao (n8n)

## Status: GOTOWE DO PRODUKCJI - wymaga konfiguracji webhookow w Bitrix

## Aktualny stan (2026-01-28)
- Workflow zaimportowany do n8n i przetestowany
- **Wszystkie komponenty dzialaja:**
  - Webhook n8n (produkcyjny: `https://n8n.public.asterisk-dev.pl/webhook/bitrix-company-webhook`)
  - Bitrix GET/UPDATE
  - Walidacja NIP (checksum)
  - GUS API (JSON endpoints)
  - Firmao (tworzenie klienta z NIP)
  - Sprawdzanie duplikatow w Firmao
- **Logika uruchamiania:** workflow wykonuje sie tylko gdy Status GUS = EXECUTE (1396)
- **Pozostalo:** skonfigurowac webhooki wychodzace w Bitrix

## Pliki
- `Bitrix_GUS_Firmao.json` - workflow n8n (20 node'ow)
- `CLAUDE.md` - ten plik kontekstowy

## Skonfigurowane wartosci (HARDCODED w workflow)

### Bitrix24
- **Webhook URL:** `https://b24-x44o93.bitrix24.pl/rest/154/e7ipt5izcl5pwz2n/`
- **Pole NIP:** `UF_CRM_661903D52335E`
- **Pole Status GUS:** `UF_CRM_1769435766` (lista: PENDING=1384, OK=1386, ERROR=1388, **EXECUTE=1396**)
- **Pole Komunikat GUS:** `UF_CRM_1769435867`
- **Pole REGON:** `UF_CRM_1769435935`
- **Pole KRS:** `UF_CRM_1769435983`
- **Pole Nazwa Gabinetu:** `UF_CRM_1757666334249`
- **Pole Email:** `EMAIL` (standardowe, format multi-field: `[{"VALUE": "...", "VALUE_TYPE": "WORK"}]`)

### GUS BIR API (JSON endpoints - dzialaja!)
- **Klucz API:** `b901f957c1f847c79d06`
- **Endpoint Zaloguj:** `https://wyszukiwarkaregon.stat.gov.pl/wsBIR/UslugaBIRzewnPubl.svc/ajaxEndpoint/Zaloguj`
- **Endpoint Szukaj:** `https://wyszukiwarkaregon.stat.gov.pl/wsBIR/UslugaBIRzewnPubl.svc/ajaxEndpoint/daneSzukajPodmioty`
- **Endpoint Raport:** `https://wyszukiwarkaregon.stat.gov.pl/wsBIR/UslugaBIRzewnPubl.svc/ajaxEndpoint/DanePobierzPelnyRaport`

### Nazwy raportow GUS (wedlug SilosID)
- Osoba prawna: `BIR11OsPrawna`
- Osoba fizyczna CEIDG (SilosID=1): `BIR11OsFizycznaDzialalnoscCeidg`
- Osoba fizyczna rolnicza (SilosID=2): `BIR11OsFizycznaDzialalnoscRolnicza`
- Osoba fizyczna pozostala (SilosID=3): `BIR11OsFizycznaDzialalnoscPozostala`
- Osoba fizyczna skreslonakreslona (SilosID=4): `BIR11OsFizycznaDzialalnoscSkreslonaDo20141108`

### Firmao
- **URL API:** `https://system.firmao.pl/scanlab/svc/v1/customers`
- **Login:** `scanlab.api@firmao.pl`
- **Haslo:** `d36e725ba59b4a7e`
- **Credential w n8n:** Header Auth z `Authorization: Basic c2NhbmxhYi5hcGlAZmlybWFvLnBsOmQzNmU3MjViYTU5YjRhN2U=`

### Pola Firmao dla customers (wazne!)

#### POST (tworzenie)
- `label` - krotka nazwa (wymagane)
- `name` - pelna nazwa (wymagane)
- `nipNumber` - NIP (nie "nip"!)
- `identificationNumber` - REGON (nie "regon"!)
- `krs` - KRS
- `description` - opis
- Odpowiedz POST: `{"changelog": [{"objectId": 123, ...}]}` - ID nowego klienta w `changelog[0].objectId`

#### PUT (aktualizacja) - NOTACJA KROPKOWA!
Firmao PUT NIE akceptuje zagniezdzonych obiektow! Trzeba uzywac notacji z kropka:
- `"customFields.custom5"` - Opiekun Klienta (nie `{customFields: {custom5: ...}}`)
- `"customFields.custom6"` - Typ skanera
- `"officeAddress.street"` - ulica z numerem
- `"officeAddress.city"` - miasto
- `"officeAddress.postCode"` - kod pocztowy
- `"officeAddress.country"` - kraj
- `"phones"` - tablica telefonow (nie `phone`!)
- `"emails"` - tablica emaili (nie `email`!)
- `description` - opis (flat field, dziala normalnie)

#### GET (odczyt)
- Zwraca zagniedzone obiekty: `officeAddress: {street, city, ...}`, `customFields: {custom5, ...}`
- Telefony: `phone`, `phoneOther`, `phoneOther2` (flat fields)
- Emaile: `email`, `email2`, `email3` (flat fields)
- `website` - strona www

## Testowa firma w Bitrix
- **ID:** 4492
- **NIP:** 1133175087
- **Firma:** MaCode Maciej Polak (pobrane z GUS!)

## Przeplyw (23 node'y)
```
Webhook -> RespondOK -> BitrixGet -> IF(Status=EXECUTE?)
  -> WalidujNIP -> IF(valid?) -> GUSLogin -> GUSSearch
  -> IF(found?) -> GUSReport -> ParseGUS -> BitrixUpdate(OK)
  -> BitrixGetUser -> PrzygotujDaneFirmao -> FirmaoSearch -> CheckNIP -> IF(exists?)
    -> FirmaoCreate (jesli nie istnieje) -> FirmaoUpdate -> END
    -> SKIP (jesli istnieje) -> END

Bledy: -> BitrixUpdate(ERROR) -> Notify -> END
Brak EXECUTE: -> SKIP -> END
```

### Warunek uruchomienia
- NIP musi byc wypelniony
- Status GUS musi byc rowny EXECUTE (1396)
- Jesli warunek nie spelniony - workflow konczy sie bez akcji

## Walidacja NIP
Wagi checksum: [6,5,7,2,3,4,5,6,7], suma mod 11 == ostatnia cyfra

## Rozwiazane problemy

### GUS SOAP/MTOM
- **Problem:** n8n HTTP Request nie obsluguje MTOM (multipart response)
- **Rozwiazanie:** Uzycie JSON endpoints (`/ajaxEndpoint/`) zamiast SOAP

### Nazwy raportow GUS
- **Problem:** Bledna nazwa raportu powodowala ErrorCode 5
- **Rozwiazanie:** Uzywanie SilosID z odpowiedzi wyszukiwania do okreslenia typu raportu

### Firmao API
- **Problem:** Bledne URL i nazwy pol
- **Rozwiazanie:** URL: `/svc/v1/` (nie `/api/`), pola: `nipNumber`, `identificationNumber`

## Kolejne kroki do wykonania
1. ~~**Dokonczyc Firmao** - ustalic jak zapisywac adres~~ DONE
2. ~~**Dodac _TEST do nazwy** - przy tworzeniu klienta w Firmao~~ DONE
3. ~~**Poprawic Status GUS** - zmienione na ID listy~~ DONE
4. ~~**Przetestowac caly flow**~~ DONE
5. ~~**Dodac Nazwa Gabinetu i Email**~~ DONE
6. ~~**Zmienic logike na EXECUTE trigger**~~ DONE - workflow odpala sie tylko gdy Status GUS = EXECUTE
7. **Skonfigurowac webhooki wychodzace w Bitrix** - INSTRUKCJA PONIZEJ
8. Usunac _TEST z nazw klientow Firmao i aktywowac workflow na produkcji

---

## INSTRUKCJA: Konfiguracja webhookow w Bitrix24

### Krok 1: Wejdz w panel webhookow
URL: `https://b24-x44o93.bitrix24.pl/devops/section/standard/`
Lub: Aplikacje -> Developer resources -> Inne -> Outbound webhooks

### Krok 2: Dodaj webhook dla TWORZENIA firmy
- **Event type:** `ONCRMCOMPANYADD`
- **Handler URL:** `https://n8n.public.asterisk-dev.pl/webhook/bitrix-company-webhook`
- Zapisz

### Krok 3: Dodaj webhook dla EDYCJI firmy
- **Event type:** `ONCRMCOMPANYUPDATE`
- **Handler URL:** `https://n8n.public.asterisk-dev.pl/webhook/bitrix-company-webhook`
- Zapisz

### Krok 4: Aktywuj workflow w n8n
- Wejdz w n8n -> Workflows -> Bitrix-GUS-Firmao
- Ustaw przelacznik **Active = ON**

### Jak to dziala
1. Uzytkownik tworzy/edytuje firme w Bitrix
2. Bitrix wysyla webhook do n8n (kazda zmiana)
3. n8n sprawdza czy Status GUS = EXECUTE (1396)
   - Jesli TAK -> wykonuje synchronizacje z GUS i Firmao
   - Jesli NIE -> ignoruje request (nic nie robi)
4. Po synchronizacji Status GUS zmienia sie na OK (1386) lub ERROR (1388)

### Test po konfiguracji
1. Utworz nowa firme w Bitrix
2. Wpisz NIP
3. Ustaw Status GUS = EXECUTE
4. Zapisz firme
5. Sprawdz czy dane zostaly pobrane z GUS i firma utworzona w Firmao

---

## Testowanie webhook (curl)

### URL testowy (wymaga Listen for test event w n8n)
```bash
curl -X POST "https://n8n.public.asterisk-dev.pl/webhook-test/bitrix-company-webhook" \
  -H "Content-Type: application/json" \
  -d '{"COMPANY_ID": "4492"}'
```

### URL produkcyjny (wymaga Active workflow w n8n)
```bash
curl -X POST "https://n8n.public.asterisk-dev.pl/webhook/bitrix-company-webhook" \
  -H "Content-Type: application/json" \
  -d '{"COMPANY_ID": "4492"}'
```

**UWAGA:** Produkcyjny URL (`/webhook/` bez `-test`) dziala TYLKO gdy workflow jest aktywny!

## Przydatne linki
- Portal API GUS: https://api.stat.gov.pl/Home/RegonApi
- Dokumentacja BIR: https://regonapi.readthedocs.io/en/latest/bir_versions.html
