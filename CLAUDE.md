# Automatyzacja Bitrix-GUS-Firmao (n8n)

## Status: W TRAKCIE TESTOWANIA - GUS dziala, Firmao do dokonczenia

## Aktualny stan (2026-01-27)
- Workflow zaimportowany do n8n
- Webhook dziala (testowy: `https://n8n.public.asterisk-dev.pl/webhook-test/bitrix-company-webhook`)
- Bitrix GET dziala
- Walidacja NIP dziala
- **GUS API dziala** - uzywamy JSON endpoints (ajaxEndpoint)
- **Firmao** - do dokonczenia (nazwy pol poprawione, adres do ustalenia)

## Pliki
- `Bitrix_GUS_Firmao.json` - workflow n8n (20 node'ow)
- `CLAUDE.md` - ten plik kontekstowy

## Skonfigurowane wartosci (HARDCODED w workflow)

### Bitrix24
- **Webhook URL:** `https://b24-x44o93.bitrix24.pl/rest/154/f07c2dmvuwsi52qb/`
- **Pole NIP:** `UF_CRM_661903D52335E`
- **Pole Status GUS:** `UF_CRM_1769435766` (lista: PENDING=1384, OK=1386, ERROR=1388)
- **Pole Komunikat GUS:** `UF_CRM_1769435867`
- **Pole REGON:** `UF_CRM_1769435935`
- **Pole KRS:** `UF_CRM_1769435983`

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
- `label` - krotka nazwa (wymagane)
- `name` - pelna nazwa (wymagane)
- `nipNumber` - NIP (nie "nip"!)
- `identificationNumber` - REGON (nie "regon"!)
- `krs` - KRS
- `officeAddress` - obiekt adresu siedziby (przy GET zwraca, przy POST moze nie dzialac):
  - `street` - ulica z numerem
  - `city` - miasto
  - `postCode` - kod pocztowy
  - `country` - kraj
  - `county` - powiat
- `correspondenceAddress` - obiekt adresu korespondencyjnego (taka sama struktura)
- `description` - opis
- `email`, `email2`, `email3` - emaile
- `phone`, `phoneOther`, `phoneOther2` - telefony
- `website` - strona www

## Testowa firma w Bitrix
- **ID:** 4492
- **NIP:** 1133175087
- **Firma:** MaCode Maciej Polak (pobrane z GUS!)

## Przeplyw (20 node'ow)
```
Webhook -> BitrixGet -> IF(sync?) -> WalidujNIP -> IF(valid?)
  -> GUSLogin -> GUSSearch -> IF(found?) -> GUSReport
  -> BitrixUpdate(OK) -> FirmaoSearch -> IF(exists?)
    -> FirmaoCreate / FirmaoUpdate -> END

Bledy: -> BitrixUpdate(ERROR) -> Notify -> END
```

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
1. ~~**Dokonczyc Firmao** - ustalic jak zapisywac adres~~ DONE (officeAddress dziala)
2. ~~**Dodac _TEST do nazwy** - przy tworzeniu klienta w Firmao~~ DONE
3. ~~**Poprawic Status GUS** - zmienione na ID listy (OK=1386, ERROR=1388)~~ DONE
4. **Przetestowac caly flow** - zaimportowac nowy JSON do n8n
5. Skonfigurowac webhook w Bitrix (produkcyjny URL)
6. Usunac _TEST i aktywowac workflow na produkcji

## Testowanie webhook (curl)
```bash
curl -X POST "https://n8n.public.asterisk-dev.pl/webhook-test/bitrix-company-webhook" \
  -H "Content-Type: application/json" \
  -d '{"COMPANY_ID": "4492"}'
```

## Przydatne linki
- Portal API GUS: https://api.stat.gov.pl/Home/RegonApi
- Dokumentacja BIR: https://regonapi.readthedocs.io/en/latest/bir_versions.html
