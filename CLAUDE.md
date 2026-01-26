# Automatyzacja Bitrix-GUS-Firmao (n8n)

## Status: WORKFLOW GOTOWY - wymaga konfiguracji

## Cel
Workflow n8n: Bitrix webhook → walidacja NIP → GUS SOAP → update Bitrix → upsert Firmao → notyfikacje błędów.

## Pliki
- `Bitrix_GUS_Firmao.json` - workflow do importu w n8n (20 node'ów)
- `CLAUDE.md` - ten plik kontekstowy

## Stack
- **n8n** - workflow (JSON export)
- **Bitrix24** - CRM, REST webhook
- **GUS BIR1** - SOAP API (klucz produkcyjny)
- **Firmao** - REST API (Basic Auth)

## Przepływ (20 node'ów)
```
Webhook → BitrixGet → IF(sync?) → WalidujNIP → IF(valid?)
  → GUSLogin → GUSSearch → IF(found?) → GUSReport
  → BitrixUpdate(OK) → FirmaoSearch → IF(exists?)
    → FirmaoCreate / FirmaoUpdate → END

Błędy: → BitrixUpdate(ERROR) → Notify → END
```

## KONFIGURACJA WYMAGANA

### 1. Sprawdź nazwę pola NIP w Bitrix
URL: `https://TWOJA_DOMENA.bitrix24.pl/rest/WEBHOOK/crm.company.fields`
Szukaj pola z "NIP" - zanotuj nazwę techniczną (np. `UF_CRM_1234567890`)

### 2. Utwórz pola w Bitrix (CRM → Ustawienia → Pola → Firma)
| Pole | Typ | Wartości |
|------|-----|----------|
| UF_CRM_GUS_SYNC_STATUS | Lista | PENDING, OK, ERROR |
| UF_CRM_GUS_SYNC_MESSAGE | Tekst | - |
| UF_CRM_REGON | Tekst | - |
| UF_CRM_KRS | Tekst | - |

### 3. Zmienne ENV w n8n (Settings → Environment Variables)
```
BITRIX_WEBHOOK_URL=https://xxx.bitrix24.pl/rest/1/xxx/
GUS_API_KEY=twoj_klucz_produkcyjny
FIRMAO_API_URL=https://system.firmao.pl/api
FIRMAO_LOGIN=twoj_login
FIRMAO_PASSWORD=twoje_haslo
```

### 4. Credentials w n8n
Utwórz credential typu "HTTP Basic Auth" o nazwie "Firmao API"

### 5. Po imporcie - dostosuj nazwy pól
W node "IF: Czy synchronizować?" i "Waliduj NIP" zmień `UF_CRM_NIP` na prawdziwą nazwę pola NIP

### 6. Webhook Bitrix
Skonfiguruj webhook w Bitrix żeby uderzał do:
`https://twoj-n8n.com/webhook/bitrix-company-webhook`
Event: ONCRMUPDATECOMPANY, payload: `{"data": {"FIELDS": {"ID": "$ID"}}}`

## GUS SOAP
- Produkcja: `https://wyszukiwarkaregon.stat.gov.pl/wsBIR/UslugaBIRzworny.svc`
- Test: `https://wyszukiwarkaregontest.stat.gov.pl/wsBIR/UslugaBIRzworny.svc`

## Walidacja NIP
Wagi checksum: [6,5,7,2,3,4,5,6,7], suma mod 11 == ostatnia cyfra

## Testy
1. Test webhook - czy n8n odbiera z Bitrix
2. Test błędnego NIP - czy przychodzi notyfikacja
3. Test poprawnego NIP - czy GUS zwraca dane
4. Test Firmao - czy klient został utworzony
5. Test anty-pętli - druga edycja NIE uruchamia workflow (STATUS=OK)
