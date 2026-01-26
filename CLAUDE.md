# Automatyzacja Bitrix-GUS-Firmao (n8n)

## Cel
Workflow n8n: Bitrix webhook → walidacja NIP → GUS SOAP → update Bitrix → upsert Firmao → notyfikacje błędów.

## Stack
- **n8n** - workflow (JSON export)
- **Bitrix24** - CRM, REST webhook
- **GUS BIR1** - SOAP API (klucz produkcyjny)
- **Firmao** - REST API (Basic Auth)

## Przepływ
```
Webhook(COMPANY_ID) → BitrixGet → NormalizujNIP → WalidujNIP
  → [błąd] → SetERROR → Notify → END
  → [ok] → GUSLogin(sid) → GUSSearch(REGON) → GUSReport
    → BitrixUpdate(+STATUS=OK) → FirmaoSearch(NIP)
      → [nowy] → FirmaoCreate
      → [istnieje] → FirmaoUpdate
```

## Pola Bitrix
| Pole | Typ | Opis |
|------|-----|------|
| `???` | string | NIP (sprawdzić nazwę!) |
| `UF_CRM_GUS_SYNC_STATUS` | lista | OK/ERROR/PENDING |
| `UF_CRM_GUS_SYNC_MESSAGE` | string | komunikat błędu |
| `UF_CRM_REGON` | string | do utworzenia |
| `UF_CRM_KRS` | string | do utworzenia |

## ENV n8n
```
BITRIX_WEBHOOK_URL=https://xxx.bitrix24.pl/rest/1/xxx/
GUS_API_KEY=xxx
FIRMAO_API_URL=https://system.firmao.pl/api
FIRMAO_LOGIN=xxx
FIRMAO_PASSWORD=xxx
```

## GUS SOAP endpoints
- Produkcja: `https://wyszukiwarkaregon.stat.gov.pl/wsBIR/UslugaBIRzworny.svc`
- Test: `https://wyszukiwarkaregontest.stat.gov.pl/wsBIR/UslugaBIRzworny.svc`

## Walidacja NIP (JS)
```js
// Normalizacja
nip = nip.replace(/[^0-9]/g, '');
// Checksum: wagi [6,5,7,2,3,4,5,6,7], suma mod 11 == ostatnia cyfra
```

## Mapowanie GUS → Bitrix/Firmao
Nazwa, Ulica+Nr, Miasto, KodPocztowy, REGON, KRS, PKD

## Anty-pętla
Workflow startuje tylko gdy: NIP istnieje AND STATUS != 'OK'

## TODO
- [ ] Sprawdzić nazwę pola NIP w Bitrix
- [ ] Utworzyć pola: GUS_SYNC_STATUS, GUS_SYNC_MESSAGE, REGON, KRS
- [ ] Ustawić ENV w n8n
- [ ] Wygenerować workflow JSON
- [ ] Test na firmie testowej
