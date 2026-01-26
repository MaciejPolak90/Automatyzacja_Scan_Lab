# Automatyzacja Bitrix-GUS-Firmao (n8n)

## Status: W TRAKCIE TESTOWANIA - problem z GUS API

## Aktualny stan (2026-01-26)
- Workflow zaimportowany do n8n
- Webhook działa (testowy: `https://n8n.public.asterisk-dev.pl/webhook-test/bitrix-company-webhook`)
- Bitrix GET działa
- Walidacja NIP działa
- **PROBLEM: GUS Zaloguj zwraca błąd** - do debugowania

### Ostatni błąd GUS
```
ActionNotSupported - The message with Action 'http://CIS/BIR/PUBL/2014/07/IUslugaBIRzewnPubl/Zaloguj'
cannot be processed at the receiver, due to a ContractFilter mismatch
```

### Możliwe przyczyny do sprawdzenia:
1. Klucz API GUS może być nieaktywny/wygasły
2. Format SOAP może wymagać dodatkowych elementów
3. API GUS może mieć inne wymagania (np. certyfikat, IP whitelist)

## Pliki
- `Bitrix_GUS_Firmao.json` - workflow n8n (20 node'ów)
- `CLAUDE.md` - ten plik kontekstowy

## Skonfigurowane wartości (HARDCODED w workflow)

### Bitrix24
- **Webhook URL:** `https://b24-x44o93.bitrix24.pl/rest/154/f07c2dmvuwsi52qb/`
- **Pole NIP:** `UF_CRM_661903D52335E`
- **Pole Status GUS:** `UF_CRM_1769435766`
- **Pole Komunikat GUS:** `UF_CRM_1769435867`
- **Pole REGON:** `UF_CRM_1769435935`
- **Pole KRS:** `UF_CRM_1769435983`

### GUS BIR API
- **Klucz API:** `b901f957c1f847c79d06`
- **Endpoint:** `https://wyszukiwarkaregon.stat.gov.pl/wsBIR/UslugaBIRzewnPubl.svc`
- **SOAP Action prefix:** `http://CIS/BIR/PUBL/2014/07/IUslugaBIRzewnPubl/`

### Firmao
- **URL API:** `https://system.firmao.pl/scanlab/api`
- **Login:** `scanlab.api@firmao.pl`
- **Haslo:** `d36e725ba59b4a7e`
- **Credential w n8n:** Header Auth z `Authorization: Basic c2NhbmxhYi5hcGlAZmlybWFvLnBsOmQzNmU3MjViYTU5YjRhN2U=`

## Testowa firma w Bitrix
- **ID:** 4488

## Przepływ (20 node'ów)
```
Webhook → BitrixGet → IF(sync?) → WalidujNIP → IF(valid?)
  → GUSLogin → GUSSearch → IF(found?) → GUSReport
  → BitrixUpdate(OK) → FirmaoSearch → IF(exists?)
    → FirmaoCreate / FirmaoUpdate → END

Bledy: → BitrixUpdate(ERROR) → Notify → END
```

## Walidacja NIP
Wagi checksum: [6,5,7,2,3,4,5,6,7], suma mod 11 == ostatnia cyfra

## Kolejne kroki do wykonania
1. **Zdebugować GUS API** - sprawdzić czy klucz jest aktywny, może przetestować w Postman
2. Po naprawie GUS - przetestować caly flow
3. Skonfigurowac webhook w Bitrix (produkcyjny URL)
4. Aktywowac workflow na produkcji

## Przydatne linki
- Portal API GUS: https://api.stat.gov.pl/Home/RegonApi
- Dokumentacja BIR: https://regonapi.readthedocs.io/en/latest/bir_versions.html
