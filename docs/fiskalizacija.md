# Fiskalizacija 1.0 i 2.0 – Kompletni tehnički vodič

## Sadržaj

1. [Pregled sustava](#pregled-sustava)
2. [Fiskalizacija 1.0 - Gotovinski računi](#fiskalizacija-10---gotovinski-računi)
   - [Obvezni elementi računa](#obvezni-elementi-računa)
   - [ZKI - Zaštitni Kod Izdavatelja](#zki---zaštitni-kod-izdavatelja)
   - [JIR - Jedinstveni Identifikator Računa](#jir---jedinstveni-identifikator-računa)
   - [QR kod specifikacija](#qr-kod-specifikacija)
   - [SOAP komunikacija i XML strukture](#soap-komunikacija-i-xml-strukture)
   - [Digitalni potpis (XML Signature)](#digitalni-potpis-xml-signature)
   - [Certifikati](#certifikati)
3. [Fiskalizacija 2.0 - eRačun B2B](#fiskalizacija-20---eračun-b2b)
   - [EN 16931 i UBL 2.1](#en-16931-i-ubl-21)
   - [Hrvatska CIUS specifikacija](#hrvatska-cius-specifikacija)
   - [Informacijski posrednici](#informacijski-posrednici)
4. [Razlike F1 vs F2](#razlike-f1-vs-f2)
5. [Implementacija](#implementacija)
6. [Testiranje](#testiranje)
7. [Izvori](#izvori)

---

## Pregled sustava

| Aspekt | Fiskalizacija 1.0 | Fiskalizacija 2.0 |
|--------|-------------------|-------------------|
| Početak | 2013. | **1. siječnja 2026.** |
| Primjena | B2C (krajnja potrošnja) | B2B i B2G (poslovni subjekti) |
| Način plaćanja | Gotovina, kartice | Bezgotovinsko (virman, eRačun) |
| Format | SOAP/XML | UBL 2.1 XML + EN 16931 |
| Komunikacija | Direktno s CIS-om | Preko informacijskih posrednika |
| Identifikatori | ZKI, JIR, QR | OIB, IKOF, JIR |

### Ključni datumi

| Datum | Događaj |
|-------|---------|
| 01.09.2025. | Početak testne faze, novi certifikati od kvalificiranih pružatelja |
| 01.01.2026. | Obvezna fiskalizacija eRačuna B2B za PDV obveznike |
| 01.01.2027. | Obveza za ne-PDV obveznike |

### Produkcijski endpoint

```
https://cis.porezna-uprava.hr:8509/FiskalizacijaService
```

---

## Fiskalizacija 1.0 - Gotovinski računi

### Obvezni elementi računa

```
┌─────────────────────────────────────────────────────────────────┐
│                     STRUKTURA RAČUNA                            │
├─────────────────────────────────────────────────────────────────┤
│  IDENTIFIKACIJA OBVEZNIKA                                       │
│  ├── OIB obveznika fiskalizacije (11 znamenki)                  │
│  ├── Naziv tvrtke                                               │
│  └── Adresa sjedišta                                            │
│                                                                 │
│  BROJ RAČUNA (format: BROJ/PP/NU)                               │
│  ├── BrOznRac: Brojčana oznaka računa (redni broj)              │
│  ├── OznPosPr: Oznaka poslovnog prostora                        │
│  └── OznNapUr: Oznaka naplatnog uređaja                         │
│                                                                 │
│  VREMENSKA OZNAKA                                               │
│  └── Datum i vrijeme izdavanja (DD.MM.GGGGTHH:MM:SS)            │
│                                                                 │
│  FINANCIJSKI PODACI                                             │
│  ├── Stavke (naziv, količina, cijena)                           │
│  ├── PDV po stopama (25%, 13%, 5%, 0%)                          │
│  │   ├── Stopa                                                  │
│  │   ├── Osnovica                                               │
│  │   └── Iznos                                                  │
│  ├── Porez na potrošnju (PNP) ako postoji                       │
│  └── Ukupni iznos                                               │
│                                                                 │
│  NAČIN PLAĆANJA                                                 │
│  └── G=Gotovina, K=Kartica, C=Ček, T=Transakcijski, O=Ostalo    │
│                                                                 │
│  FISKALIZACIJSKI PODACI                                         │
│  ├── ZKI (Zaštitni Kod Izdavatelja) - 32 hex znaka              │
│  ├── JIR (Jedinstveni Identifikator Računa) - UUID              │
│  └── QR kod (obavezno od 01.01.2021.)                           │
│                                                                 │
│  OPERATER                                                       │
│  └── OIB operatera koji izdaje račun                            │
└─────────────────────────────────────────────────────────────────┘
```

### ZKI - Zaštitni Kod Izdavatelja

ZKI je 32-znamenkasti heksadecimalni broj koji potvrđuje vezu između obveznika i računa. Generira se **lokalno** na naplatnom uređaju.

#### Ulazni podaci za ZKI

```
┌──────────────────────────────────────────────────────────────┐
│  1. OIB obveznika fiskalizacije    │  11 znamenki            │
│  2. Datum i vrijeme izdavanja      │  DD.MM.GGGG HH:MM:SS    │
│  3. Brojčana oznaka računa         │  Redni broj             │
│  4. Oznaka poslovnog prostora      │  String                 │
│  5. Oznaka naplatnog uređaja       │  String                 │
│  6. Ukupni iznos računa            │  Decimalni broj         │
└──────────────────────────────────────────────────────────────┘
```

#### Algoritam generiranja ZKI

```
┌─────────────────────────────────────────────────────────────────┐
│  KORAK 1: Konkatenacija podataka                                │
│  ─────────────────────────────────────────────────────────────  │
│  Format: OIB + DatumVrijeme + BrojRačuna + OznPP + OznNU + Iznos│
│                                                                 │
│  Primjer:                                                       │
│  "12345678901" + "01.01.2025 12:00:00" + "1" + "PP1" + "1" +    │
│  "100.00"                                                       │
│                                                                 │
│  Rezultat: "1234567890101.01.2025 12:00:001PP11100.00"          │
├─────────────────────────────────────────────────────────────────┤
│  KORAK 2: RSA potpis privatnim ključem                          │
│  ─────────────────────────────────────────────────────────────  │
│  - Koristi privatni ključ iz FINA certifikata                   │
│  - Algoritam: SHA1withRSA ili SHA256withRSA                     │
│  - Rezultat: binarni potpis                                     │
├─────────────────────────────────────────────────────────────────┤
│  KORAK 3: MD5 hash potpisa                                      │
│  ─────────────────────────────────────────────────────────────  │
│  - Uzmi RSA potpis iz koraka 2                                  │
│  - Izračunaj MD5 hash                                           │
│  - Pretvori u lowercase hex string                              │
│                                                                 │
│  Rezultat: 32 heksadecimalna znaka (0-9, a-f)                   │
│  Primjer: "e4d909c290d0fb1ca068ffaddf22cbd0"                    │
└─────────────────────────────────────────────────────────────────┘
```

#### Pseudokod za ZKI

```
function generateZKI(oib, dateTime, billNumber, ppCode, deviceCode, totalAmount, privateKey):
    // Korak 1: Konkatenacija
    concatenated = oib + dateTime + billNumber + ppCode + deviceCode + formatAmount(totalAmount)

    // Korak 2: RSA potpis
    signature = RSA_Sign(concatenated, privateKey, "SHA1withRSA")

    // Korak 3: MD5 hash
    zki = MD5_Hash(signature).toLowerCase()

    return zki  // 32 hex znaka
```

#### Primjer u Go

```go
package fiscalization

import (
    "crypto"
    "crypto/md5"
    "crypto/rsa"
    "crypto/x509"
    "encoding/hex"
    "fmt"
    "strings"
)

func GenerateZKI(
    oib string,
    dateTime string,      // format: "DD.MM.YYYY HH:MM:SS"
    billNumber string,
    ppCode string,
    deviceCode string,
    totalAmount string,   // format: "100.00"
    privateKey *rsa.PrivateKey,
) (string, error) {
    // Korak 1: Konkatenacija
    data := oib + dateTime + billNumber + ppCode + deviceCode + totalAmount

    // Korak 2: SHA1 hash podataka
    hash := crypto.SHA1.New()
    hash.Write([]byte(data))
    hashed := hash.Sum(nil)

    // Korak 3: RSA potpis
    signature, err := rsa.SignPKCS1v15(nil, privateKey, crypto.SHA1, hashed)
    if err != nil {
        return "", fmt.Errorf("RSA sign failed: %w", err)
    }

    // Korak 4: MD5 hash potpisa
    md5Hash := md5.Sum(signature)
    zki := hex.EncodeToString(md5Hash[:])

    return strings.ToLower(zki), nil
}
```

### JIR - Jedinstveni Identifikator Računa

JIR generira **Porezna uprava** nakon uspješne fiskalizacije.

```
┌─────────────────────────────────────────────────────────────────┐
│  FORMAT: UUID (Universally Unique Identifier)                   │
│  ─────────────────────────────────────────────────────────────  │
│  Struktura: 8-4-4-4-12 (36 znakova s crticama)                  │
│                                                                 │
│  Primjer: f47ac10b-58cc-4372-a567-0e02b2c3d479                  │
│                                                                 │
│  Znakovi: 0-9, a-f (lowercase heksadecimalni)                   │
│                                                                 │
│  Vrijeme odgovora: CIS mora odgovoriti unutar 2 sekunde         │
└─────────────────────────────────────────────────────────────────┘
```

#### Obrada JIR odgovora

```
┌─────────────────────────────────────────────────────────────────┐
│  SCENARIJ 1: Uspješna fiskalizacija                             │
│  ─────────────────────────────────────────────────────────────  │
│  - CIS vraća JIR u odgovoru                                     │
│  - Spremi JIR uz račun                                          │
│  - Ispiši JIR na račun                                          │
│  - Generiraj QR kod s JIR-om                                    │
├─────────────────────────────────────────────────────────────────┤
│  SCENARIJ 2: Greška / Timeout                                   │
│  ─────────────────────────────────────────────────────────────  │
│  - Ispiši račun samo sa ZKI-om (bez JIR-a)                      │
│  - Označi račun za naknadnu dostavu                             │
│  - Implementiraj retry mehanizam                                │
│  - Generiraj QR kod sa ZKI-om                                   │
├─────────────────────────────────────────────────────────────────┤
│  SCENARIJ 3: Naknadna dostava                                   │
│  ─────────────────────────────────────────────────────────────  │
│  - Postavi NakijeDostave flag na true                           │
│  - Pošalji isti račun ponovno                                   │
│  - CIS će vratiti JIR za već evidentirani račun                 │
└─────────────────────────────────────────────────────────────────┘
```

### QR kod specifikacija

Obavezan od 01.01.2021. na svim fiskaliziranim računima.

#### Sadržaj QR koda

```
┌─────────────────────────────────────────────────────────────────┐
│  URL FORMAT                                                     │
│  ─────────────────────────────────────────────────────────────  │
│  https://porezna.gov.hr/rn?jir={JIR}&datv={DATUM}&izn={IZNOS}   │
│                                                                 │
│  ili (ako nema JIR-a):                                          │
│                                                                 │
│  https://porezna.gov.hr/rn?zki={ZKI}&datv={DATUM}&izn={IZNOS}   │
├─────────────────────────────────────────────────────────────────┤
│  PARAMETRI                                                      │
│  ─────────────────────────────────────────────────────────────  │
│  jir    : JIR (36 znakova s crticama)                           │
│  zki    : ZKI (32 heksadecimalna znaka)                         │
│  datv   : Datum i vrijeme (format: GGGGMMDD_HHMM)               │
│  izn    : Ukupni iznos u centima (cijeli broj)                  │
├─────────────────────────────────────────────────────────────────┤
│  PRIMJER                                                        │
│  ─────────────────────────────────────────────────────────────  │
│  https://porezna.gov.hr/rn?                                     │
│    jir=f47ac10b-58cc-4372-a567-0e02b2c3d479&                    │
│    datv=20250101_1200&                                          │
│    izn=10000                                                    │
│                                                                 │
│  (iznos 100.00 EUR = 10000 centi)                               │
└─────────────────────────────────────────────────────────────────┘
```

#### Tehničke specifikacije QR koda

```
┌─────────────────────────────────────────────────────────────────┐
│  DIMENZIJE                                                      │
│  ─────────────────────────────────────────────────────────────  │
│  - Minimalna veličina: 2 x 2 cm                                 │
│  - Prazan prostor (quiet zone): min 2 mm sa svih strana         │
│  - QR kod verzija 4 (33x33 modula)                              │
├─────────────────────────────────────────────────────────────────┤
│  KOREKCIJA GREŠKE                                               │
│  ─────────────────────────────────────────────────────────────  │
│  - Razina: L (Low) - 7% oporavak                                │
│  - Standard: ISO/IEC 15415                                      │
├─────────────────────────────────────────────────────────────────┤
│  OGRANIČENJA                                                    │
│  ─────────────────────────────────────────────────────────────  │
│  - NE smije sadržavati logo ili sliku                           │
│  - NE smije biti ispisan preko drugog grafičkog elementa        │
│  - Kapacitet: 50-114 alfanumeričkih znakova                     │
└─────────────────────────────────────────────────────────────────┘
```

#### Primjer generiranja QR koda (Go)

```go
package fiscalization

import (
    "fmt"
    "time"

    "github.com/skip2/go-qrcode"
)

type QRCodeData struct {
    JIR       string    // opcijski, ako postoji
    ZKI       string    // obavezno ako nema JIR
    DateTime  time.Time
    Amount    int64     // u centima
}

func GenerateQRCode(data QRCodeData) ([]byte, error) {
    var url string
    dateStr := data.DateTime.Format("20060102_1504") // GGGGMMDD_HHMM

    if data.JIR != "" {
        url = fmt.Sprintf(
            "https://porezna.gov.hr/rn?jir=%s&datv=%s&izn=%d",
            data.JIR, dateStr, data.Amount,
        )
    } else {
        url = fmt.Sprintf(
            "https://porezna.gov.hr/rn?zki=%s&datv=%s&izn=%d",
            data.ZKI, dateStr, data.Amount,
        )
    }

    // Generiraj QR kod s L razinom korekcije
    return qrcode.Encode(url, qrcode.Low, 256)
}
```

### SOAP komunikacija i XML strukture

#### Namespace i sheme

```xml
<!-- Namespace -->
xmlns:tns="http://www.apis-it.hr/fin/2012/types/f73"

<!-- Shema -->
FiskalizacijaSchema.xsd
```

#### RacunZahtjev struktura

```xml
<?xml version="1.0" encoding="UTF-8"?>
<soapenv:Envelope
    xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/"
    xmlns:tns="http://www.apis-it.hr/fin/2012/types/f73">
    <soapenv:Body>
        <tns:RacunZahtjev Id="RacunZahtjev">
            <tns:Zaglavlje>
                <tns:IdPoruke>f81d4fae-7dec-11d0-a765-00a0c91e6bf6</tns:IdPoruke>
                <tns:DatumVrijeme>01.01.2025T12:00:00</tns:DatumVrijeme>
            </tns:Zaglavlje>
            <tns:Racun>
                <tns:Oib>12345678901</tns:Oib>
                <tns:USustPdv>true</tns:USustPdv>
                <tns:DatVrijeme>01.01.2025T12:00:00</tns:DatVrijeme>
                <tns:OznSlijed>P</tns:OznSlijed>
                <tns:BrRac>
                    <tns:BrOznRac>1</tns:BrOznRac>
                    <tns:OznPosPr>PP1</tns:OznPosPr>
                    <tns:OznNapUr>1</tns:OznNapUr>
                </tns:BrRac>
                <tns:Pdv>
                    <tns:Porez>
                        <tns:Stopa>25.00</tns:Stopa>
                        <tns:Osnovica>80.00</tns:Osnovica>
                        <tns:Iznos>20.00</tns:Iznos>
                    </tns:Porez>
                </tns:Pdv>
                <tns:IznosUkupno>100.00</tns:IznosUkupno>
                <tns:NacinPlac>G</tns:NacinPlac>
                <tns:OibOper>98765432109</tns:OibOper>
                <tns:ZastKod>e4d909c290d0fb1ca068ffaddf22cbd0</tns:ZastKod>
                <tns:NakijeDost>false</tns:NakijeDost>
            </tns:Racun>
            <!-- Signature element će biti dodan -->
        </tns:RacunZahtjev>
    </soapenv:Body>
</soapenv:Envelope>
```

#### RacunOdgovor struktura

```xml
<?xml version="1.0" encoding="UTF-8"?>
<soapenv:Envelope
    xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/"
    xmlns:tns="http://www.apis-it.hr/fin/2012/types/f73">
    <soapenv:Body>
        <tns:RacunOdgovor>
            <tns:Zaglavlje>
                <tns:IdPoruke>f81d4fae-7dec-11d0-a765-00a0c91e6bf6</tns:IdPoruke>
                <tns:DatumVrijeme>01.01.2025T12:00:01</tns:DatumVrijeme>
            </tns:Zaglavlje>
            <tns:Jir>f47ac10b-58cc-4372-a567-0e02b2c3d479</tns:Jir>
        </tns:RacunOdgovor>
    </soapenv:Body>
</soapenv:Envelope>
```

#### Elementi RacunType

| Element | Tip | Obavezan | Opis |
|---------|-----|----------|------|
| `Oib` | String(11) | Da | OIB obveznika fiskalizacije |
| `USustPdv` | Boolean | Da | Je li obveznik u sustavu PDV-a |
| `DatVrijeme` | DateTime | Da | Datum i vrijeme izdavanja |
| `OznSlijed` | String(1) | Da | P=prostor, N=naplatni uređaj |
| `BrRac` | Complex | Da | Broj računa (BrOznRac, OznPosPr, OznNapUr) |
| `Pdv` | Complex | Ne | PDV stavke (Stopa, Osnovica, Iznos) |
| `Pnp` | Complex | Ne | Porez na potrošnju |
| `IznosOslobPdv` | Decimal | Ne | Iznos oslobođen PDV-a |
| `IznosMarza` | Decimal | Ne | Marža za posebno oporezivanje |
| `IznosNePodl662` | Decimal | Ne | Iznos koji ne podliježe oporezivanju |
| `Naknade` | Complex | Ne | Lista naknada |
| `IznosUkupno` | Decimal | Da | Ukupni iznos računa |
| `NacinPlac` | String(1) | Da | G/K/C/T/O |
| `OibOper` | String(11) | Da | OIB operatera |
| `ZastKod` | String(32) | Da | ZKI |
| `NakijeDost` | Boolean | Da | Naknadna dostava (true/false) |
| `ParagonBrRac` | String | Ne | Broj paragon bloka |
| `SpecNamj` | String | Ne | Specifična namjena |

### Digitalni potpis (XML Signature)

#### Struktura XML Enveloped Signature

```xml
<Signature xmlns="http://www.w3.org/2000/09/xmldsig#">
    <SignedInfo>
        <CanonicalizationMethod
            Algorithm="http://www.w3.org/2001/10/xml-exc-c14n#"/>
        <SignatureMethod
            Algorithm="http://www.w3.org/2000/09/xmldsig#rsa-sha1"/>
        <Reference URI="#RacunZahtjev">
            <Transforms>
                <Transform
                    Algorithm="http://www.w3.org/2000/09/xmldsig#enveloped-signature"/>
                <Transform
                    Algorithm="http://www.w3.org/2001/10/xml-exc-c14n#"/>
            </Transforms>
            <DigestMethod
                Algorithm="http://www.w3.org/2000/09/xmldsig#sha1"/>
            <DigestValue>base64_encoded_digest</DigestValue>
        </Reference>
    </SignedInfo>
    <SignatureValue>base64_encoded_signature</SignatureValue>
    <KeyInfo>
        <X509Data>
            <X509Certificate>base64_encoded_certificate</X509Certificate>
            <X509IssuerSerial>
                <X509IssuerName>CN=FINA Demo CA,O=FINA,C=HR</X509IssuerName>
                <X509SerialNumber>123456789</X509SerialNumber>
            </X509IssuerSerial>
        </X509Data>
    </KeyInfo>
</Signature>
```

#### Algoritmi

```
┌─────────────────────────────────────────────────────────────────┐
│  KANONIZACIJA                                                   │
│  ─────────────────────────────────────────────────────────────  │
│  Algorithm: http://www.w3.org/2001/10/xml-exc-c14n#             │
│  (Exclusive XML Canonicalization)                               │
├─────────────────────────────────────────────────────────────────┤
│  POTPIS                                                         │
│  ─────────────────────────────────────────────────────────────  │
│  RSA-SHA1:  http://www.w3.org/2000/09/xmldsig#rsa-sha1          │
│  RSA-SHA256: http://www.w3.org/2001/04/xmldsig-more#rsa-sha256  │
├─────────────────────────────────────────────────────────────────┤
│  DIGEST                                                         │
│  ─────────────────────────────────────────────────────────────  │
│  SHA1:   http://www.w3.org/2000/09/xmldsig#sha1                 │
│  SHA256: http://www.w3.org/2001/04/xmlenc#sha256                │
├─────────────────────────────────────────────────────────────────┤
│  TRANSFORM                                                      │
│  ─────────────────────────────────────────────────────────────  │
│  Enveloped: http://www.w3.org/2000/09/xmldsig#enveloped-signature│
└─────────────────────────────────────────────────────────────────┘
```

#### Proces potpisivanja

```
┌─────────────────────────────────────────────────────────────────┐
│  KORAK 1: Pripremi XML dokument                                 │
│  ─────────────────────────────────────────────────────────────  │
│  - Kreiraj RacunZahtjev s Id atributom                          │
│  - Dodaj sve obavezne elemente                                  │
├─────────────────────────────────────────────────────────────────┤
│  KORAK 2: Kanonizacija                                          │
│  ─────────────────────────────────────────────────────────────  │
│  - Primijeni Exclusive C14N na element koji se potpisuje        │
│  - Ovo osigurava konzistentnu reprezentaciju                    │
├─────────────────────────────────────────────────────────────────┤
│  KORAK 3: Digest                                                │
│  ─────────────────────────────────────────────────────────────  │
│  - Izračunaj SHA1/SHA256 hash kanoniziranog sadržaja            │
│  - Enkodira u Base64 za DigestValue                             │
├─────────────────────────────────────────────────────────────────┤
│  KORAK 4: Potpis                                                │
│  ─────────────────────────────────────────────────────────────  │
│  - Kanonikaliziraj SignedInfo element                           │
│  - Potpiši RSA privatnim ključem                                │
│  - Enkodira u Base64 za SignatureValue                          │
├─────────────────────────────────────────────────────────────────┤
│  KORAK 5: Umetni Signature                                      │
│  ─────────────────────────────────────────────────────────────  │
│  - Dodaj Signature element unutar RacunZahtjev                  │
│  - Uključi X509 certifikat u KeyInfo                            │
└─────────────────────────────────────────────────────────────────┘
```

### Certifikati

#### FINA aplikativni certifikat

```
┌─────────────────────────────────────────────────────────────────┐
│  IZDAVATELJ                                                     │
│  ─────────────────────────────────────────────────────────────  │
│  - FINA (Financijska agencija)                                  │
│  - Od 01.09.2025.: kvalificirani pružatelji usluga povjerenja   │
├─────────────────────────────────────────────────────────────────┤
│  FORMAT                                                         │
│  ─────────────────────────────────────────────────────────────  │
│  - PKCS#12 (.p12 / .pfx)                                        │
│  - Sadrži privatni ključ i lanac certifikata                    │
│  - Zaštićen lozinkom                                            │
├─────────────────────────────────────────────────────────────────┤
│  ZAHTJEVI                                                       │
│  ─────────────────────────────────────────────────────────────  │
│  - OIB mora biti u certifikatu (Subject ili Extension)          │
│  - Key Usage: Digital Signature                                 │
│  - Extended Key Usage: Client Authentication                    │
├─────────────────────────────────────────────────────────────────┤
│  VALJANOST                                                      │
│  ─────────────────────────────────────────────────────────────  │
│  - Pratiti rok isteka                                           │
│  - Obnoviti prije isteka (30+ dana unaprijed)                   │
│  - Implementirati monitoring isteka                             │
└─────────────────────────────────────────────────────────────────┘
```

#### Root certifikati

```
FINA Root CA: https://www.fina.hr/ca-fina-root-certifikati
- Fina RDC CA 2015 (stariji sustavi)
- Fina RDC CA 2020 (novi certifikati)
```

#### TLS konfiguracija

```
┌─────────────────────────────────────────────────────────────────┐
│  PROTOKOLI                                                      │
│  ─────────────────────────────────────────────────────────────  │
│  Podržani: TLS 1.1, TLS 1.2, TLS 1.3                            │
│  Preporučeni: TLS 1.2 ili TLS 1.3                               │
├─────────────────────────────────────────────────────────────────┤
│  AUTENTIKACIJA                                                  │
│  ─────────────────────────────────────────────────────────────  │
│  - 1-way TLS (server authentication)                            │
│  - Klijent verificira CIS certifikat                            │
│  - CIS certifikat izdan od Fina RDC CA                          │
├─────────────────────────────────────────────────────────────────┤
│  TRUST STORE                                                    │
│  ─────────────────────────────────────────────────────────────  │
│  - Instalirati FINA Root CA certifikate                         │
│  - Pratiti obavijesti o zamjeni CIS certifikata                 │
└─────────────────────────────────────────────────────────────────┘
```

#### Primjer učitavanja certifikata (Go)

```go
package fiscalization

import (
    "crypto/rsa"
    "crypto/tls"
    "crypto/x509"
    "encoding/pem"
    "os"

    "golang.org/x/crypto/pkcs12"
)

type CertificateStore struct {
    Certificate *x509.Certificate
    PrivateKey  *rsa.PrivateKey
    TLSConfig   *tls.Config
}

func LoadP12Certificate(path, password string) (*CertificateStore, error) {
    p12Data, err := os.ReadFile(path)
    if err != nil {
        return nil, err
    }

    privateKey, cert, err := pkcs12.Decode(p12Data, password)
    if err != nil {
        return nil, err
    }

    rsaKey, ok := privateKey.(*rsa.PrivateKey)
    if !ok {
        return nil, fmt.Errorf("private key is not RSA")
    }

    tlsCert := tls.Certificate{
        Certificate: [][]byte{cert.Raw},
        PrivateKey:  rsaKey,
    }

    tlsConfig := &tls.Config{
        Certificates: []tls.Certificate{tlsCert},
        MinVersion:   tls.VersionTLS12,
    }

    return &CertificateStore{
        Certificate: cert,
        PrivateKey:  rsaKey,
        TLSConfig:   tlsConfig,
    }, nil
}
```

---

## Fiskalizacija 2.0 - eRačun B2B

### EN 16931 i UBL 2.1

#### Osnovni koncepti

```
┌─────────────────────────────────────────────────────────────────┐
│  EN 16931 - Europski standard za e-fakturiranje                 │
│  ─────────────────────────────────────────────────────────────  │
│  - Semantički model podataka eRačuna                            │
│  - Definira BT (Business Term) i BG (Business Group)            │
│  - Obvezujući za B2G, preporučen za B2B                         │
├─────────────────────────────────────────────────────────────────┤
│  UBL 2.1 (Universal Business Language)                          │
│  ─────────────────────────────────────────────────────────────  │
│  - XML sintaksa za EN 16931                                     │
│  - OASIS standard                                               │
│  - Peppol BIS 3.0 baziran na UBL 2.1                            │
├─────────────────────────────────────────────────────────────────┤
│  CII (Cross-Industry Invoice)                                   │
│  ─────────────────────────────────────────────────────────────  │
│  - Alternativna XML sintaksa                                    │
│  - UN/CEFACT standard                                           │
│  - Također podržan u Hrvatskoj                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### Ključni BT (Business Terms)

| BT | Naziv | Opis | Obavezan |
|----|-------|------|----------|
| BT-1 | Invoice number | Jedinstveni broj računa | Da |
| BT-2 | Invoice issue date | Datum izdavanja | Da |
| BT-3 | Invoice type code | Tip dokumenta (380, 381, 389...) | Da |
| BT-5 | Invoice currency code | Valuta (EUR) | Da |
| BT-9 | Payment due date | Rok plaćanja | Ne |
| BT-23 | Business process type | Poslovni proces | Ne |
| BT-24 | Specification identifier | CIUS identifikator | Da |
| BT-27 | Seller name | Naziv prodavatelja | Da |
| BT-29 | Seller country | Država prodavatelja | Da |
| BT-31 | Seller VAT identifier | OIB prodavatelja | Da* |
| BT-44 | Buyer name | Naziv kupca | Da |
| BT-48 | Buyer VAT identifier | OIB kupca | Da* |
| BT-106 | Sum of Invoice line net amount | Ukupno neto | Da |
| BT-109 | Tax exclusive amount | Iznos bez PDV-a | Da |
| BT-110 | Tax amount | Iznos PDV-a | Da |
| BT-112 | Amount due for payment | Iznos za plaćanje | Da |

### Hrvatska CIUS specifikacija

#### Identifikator specifikacije

```
urn:cen.eu:en16931:2017#compliant#urn:mfin.gov.hr:cius-2025:1.0#conformant#urn:mfin.gov.hr:ext-2025:1.0
```

#### Hrvatska proširenja

```
┌─────────────────────────────────────────────────────────────────┐
│  OBAVEZNA PROŠIRENJA                                            │
│  ─────────────────────────────────────────────────────────────  │
│  - KPD šifra (Klasifikacija Proizvoda i Usluga) za svaku stavku │
│  - OIB u formatu HR:VAT (npr. HR12345678901)                    │
│  - Podaci za fiskalizaciju (ZKI/JIR ako primjenjivo)            │
├─────────────────────────────────────────────────────────────────┤
│  FISKALNA PORUKA                                                │
│  ─────────────────────────────────────────────────────────────  │
│  - OIB izdavatelja i primatelja                                 │
│  - Datum i vrijeme                                              │
│  - Broj i iznos računa                                          │
│  - Oznaka načina plaćanja                                       │
│  - KPD šifra za svaku stavku                                    │
├─────────────────────────────────────────────────────────────────┤
│  DIGITALNI POTPIS                                               │
│  ─────────────────────────────────────────────────────────────  │
│  - Kvalificirani elektronički potpis                            │
│  - Kvalificirani vremenski žig                                  │
└─────────────────────────────────────────────────────────────────┘
```

#### Primjer UBL 2.1 eRačuna

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Invoice xmlns="urn:oasis:names:specification:ubl:schema:xsd:Invoice-2"
         xmlns:cac="urn:oasis:names:specification:ubl:schema:xsd:CommonAggregateComponents-2"
         xmlns:cbc="urn:oasis:names:specification:ubl:schema:xsd:CommonBasicComponents-2">

    <!-- BT-24: Specification identifier -->
    <cbc:CustomizationID>
        urn:cen.eu:en16931:2017#compliant#urn:mfin.gov.hr:cius-2025:1.0#conformant#urn:mfin.gov.hr:ext-2025:1.0
    </cbc:CustomizationID>

    <!-- BT-23: Business process type -->
    <cbc:ProfileID>urn:fdc:peppol.eu:2017:poacc:billing:01:1.0</cbc:ProfileID>

    <!-- BT-1: Invoice number -->
    <cbc:ID>INV-2025-001</cbc:ID>

    <!-- BT-2: Invoice issue date -->
    <cbc:IssueDate>2025-01-01</cbc:IssueDate>

    <!-- BT-9: Payment due date -->
    <cbc:DueDate>2025-01-31</cbc:DueDate>

    <!-- BT-3: Invoice type code -->
    <cbc:InvoiceTypeCode>380</cbc:InvoiceTypeCode>

    <!-- BT-5: Invoice currency code -->
    <cbc:DocumentCurrencyCode>EUR</cbc:DocumentCurrencyCode>

    <!-- BG-4: Seller -->
    <cac:AccountingSupplierParty>
        <cac:Party>
            <!-- BT-27: Seller name -->
            <cac:PartyName>
                <cbc:Name>Tvrtka d.o.o.</cbc:Name>
            </cac:PartyName>
            <!-- BT-31: Seller VAT identifier -->
            <cac:PartyTaxScheme>
                <cbc:CompanyID>HR12345678901</cbc:CompanyID>
                <cac:TaxScheme>
                    <cbc:ID>VAT</cbc:ID>
                </cac:TaxScheme>
            </cac:PartyTaxScheme>
            <!-- BT-29: Seller country -->
            <cac:PostalAddress>
                <cac:Country>
                    <cbc:IdentificationCode>HR</cbc:IdentificationCode>
                </cac:Country>
            </cac:PostalAddress>
        </cac:Party>
    </cac:AccountingSupplierParty>

    <!-- BG-7: Buyer -->
    <cac:AccountingCustomerParty>
        <cac:Party>
            <!-- BT-44: Buyer name -->
            <cac:PartyName>
                <cbc:Name>Kupac d.o.o.</cbc:Name>
            </cac:PartyName>
            <!-- BT-48: Buyer VAT identifier -->
            <cac:PartyTaxScheme>
                <cbc:CompanyID>HR98765432109</cbc:CompanyID>
                <cac:TaxScheme>
                    <cbc:ID>VAT</cbc:ID>
                </cac:TaxScheme>
            </cac:PartyTaxScheme>
        </cac:Party>
    </cac:AccountingCustomerParty>

    <!-- BG-22: Document totals -->
    <cac:LegalMonetaryTotal>
        <!-- BT-106: Sum of Invoice line net amount -->
        <cbc:LineExtensionAmount currencyID="EUR">100.00</cbc:LineExtensionAmount>
        <!-- BT-109: Invoice total amount without VAT -->
        <cbc:TaxExclusiveAmount currencyID="EUR">100.00</cbc:TaxExclusiveAmount>
        <!-- BT-110 + BT-109: Invoice total amount with VAT -->
        <cbc:TaxInclusiveAmount currencyID="EUR">125.00</cbc:TaxInclusiveAmount>
        <!-- BT-112: Amount due for payment -->
        <cbc:PayableAmount currencyID="EUR">125.00</cbc:PayableAmount>
    </cac:LegalMonetaryTotal>

    <!-- BG-25: Invoice line -->
    <cac:InvoiceLine>
        <cbc:ID>1</cbc:ID>
        <cbc:InvoicedQuantity unitCode="C62">1</cbc:InvoicedQuantity>
        <cbc:LineExtensionAmount currencyID="EUR">100.00</cbc:LineExtensionAmount>
        <cac:Item>
            <cbc:Name>Usluga programiranja</cbc:Name>
            <!-- KPD šifra (hrvatsko proširenje) -->
            <cac:CommodityClassification>
                <cbc:ItemClassificationCode listID="KPD">62.01</cbc:ItemClassificationCode>
            </cac:CommodityClassification>
            <cac:ClassifiedTaxCategory>
                <cbc:ID>S</cbc:ID>
                <cbc:Percent>25</cbc:Percent>
                <cac:TaxScheme>
                    <cbc:ID>VAT</cbc:ID>
                </cac:TaxScheme>
            </cac:ClassifiedTaxCategory>
        </cac:Item>
        <cac:Price>
            <cbc:PriceAmount currencyID="EUR">100.00</cbc:PriceAmount>
        </cac:Price>
    </cac:InvoiceLine>
</Invoice>
```

### Informacijski posrednici

```
┌─────────────────────────────────────────────────────────────────┐
│  ULOGA INFORMACIJSKOG POSREDNIKA                                │
│  ─────────────────────────────────────────────────────────────  │
│  - Posreduje u razmjeni eRačuna između obveznika                │
│  - Komunicira s CIS-om Porezne uprave                           │
│  - Osigurava dostavu i potvrdu primitka                         │
├─────────────────────────────────────────────────────────────────┤
│  REGISTRIRANI POSREDNICI                                        │
│  ─────────────────────────────────────────────────────────────  │
│  - FINA (Financijska agencija)                                  │
│  - moj-eRačun                                                   │
│  - Drugi registrirani kod Porezne uprave                        │
├─────────────────────────────────────────────────────────────────┤
│  PROTOKOL RAZMJENE                                              │
│  ─────────────────────────────────────────────────────────────  │
│  - AS4 (Applicability Statement 4)                              │
│  - Peppol infrastruktura za prekograničnu razmjenu              │
│  - HTTPS za komunikaciju s CIS-om                               │
├─────────────────────────────────────────────────────────────────┤
│  METAPODATKOVNI SERVIS (MPS)                                    │
│  ─────────────────────────────────────────────────────────────  │
│  - Dohvat elektroničke adrese primatelja po OIB-u               │
│  - Lista identifikatora poreznih obveznika                      │
│  - Javno dostupan na portalu Porezne uprave                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Razlike F1 vs F2

```
┌─────────────────────────────────────────────────────────────────┐
│                    FISKALIZACIJA 1.0 vs 2.0                     │
├────────────────────┬────────────────────┬───────────────────────┤
│     ASPEKT         │    FISKALIZACIJA 1 │    FISKALIZACIJA 2    │
├────────────────────┼────────────────────┼───────────────────────┤
│ Primjena           │ B2C (krajnja       │ B2B, B2G              │
│                    │ potrošnja)         │ (poslovni subjekti)   │
├────────────────────┼────────────────────┼───────────────────────┤
│ Plaćanje           │ Gotovina, kartice  │ Bezgotovinsko         │
│                    │                    │ (virman, eRačun)      │
├────────────────────┼────────────────────┼───────────────────────┤
│ Format računa      │ Slobodan (sa       │ Strukturirani XML     │
│                    │ obveznim poljima)  │ (UBL 2.1 / CII)       │
├────────────────────┼────────────────────┼───────────────────────┤
│ Standard           │ Nacionalni         │ EN 16931 + HR CIUS    │
├────────────────────┼────────────────────┼───────────────────────┤
│ Komunikacija       │ SOAP direktno s    │ Preko informacijskih  │
│                    │ CIS-om             │ posrednika / MPS      │
├────────────────────┼────────────────────┼───────────────────────┤
│ Identifikatori     │ ZKI, JIR, QR       │ OIB, IKOF, JIR        │
├────────────────────┼────────────────────┼───────────────────────┤
│ Podaci o naplati   │ Ne                 │ Da (eIzvještavanje)   │
├────────────────────┼────────────────────┼───────────────────────┤
│ Predujmovi         │ Standardno         │ Posebna pravila       │
│                    │                    │ (izdavanje, storno,   │
│                    │                    │ konačni račun)        │
├────────────────────┼────────────────────┼───────────────────────┤
│ KPD šifra          │ Ne                 │ Da (za svaku stavku)  │
├────────────────────┼────────────────────┼───────────────────────┤
│ Potpis             │ RSA-SHA1/SHA256    │ Kvalificirani         │
│                    │                    │ elektronički potpis   │
│                    │                    │ + vremenski žig       │
└────────────────────┴────────────────────┴───────────────────────┘
```

---

## Implementacija

### Arhitektura sustava

```
┌─────────────────────────────────────────────────────────────────┐
│                         POS / ERP                               │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    Fiskalizacijski modul                 │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────────────┐  │   │
│  │  │   ZKI      │  │   XML      │  │   Certifikati      │  │   │
│  │  │ Generator  │  │ Signer     │  │   Manager          │  │   │
│  │  └────────────┘  └────────────┘  └────────────────────┘  │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────────────┐  │   │
│  │  │   SOAP     │  │   QR       │  │   Retry            │  │   │
│  │  │  Client    │  │ Generator  │  │   Queue            │  │   │
│  │  └────────────┘  └────────────┘  └────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    eRačun modul (F2)                     │   │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────────────┐  │   │
│  │  │   UBL      │  │   KPD      │  │   Informacijski    │  │   │
│  │  │ Generator  │  │ Mapper     │  │   posrednik        │  │   │
│  │  └────────────┘  └────────────┘  └────────────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   CIS (Porezna uprava)                          │
│                                                                 │
│  https://cis.porezna-uprava.hr:8509/FiskalizacijaService        │
└─────────────────────────────────────────────────────────────────┘
```

### Slijed operacija (F1)

```
┌─────────────────────────────────────────────────────────────────┐
│  1. PRIPREMA RAČUNA                                             │
│  ─────────────────────────────────────────────────────────────  │
│  - Prikupi podatke o transakciji                                │
│  - Validiraj OIB, PDV stope, iznose                             │
│  - Generiraj jedinstveni broj računa                            │
├─────────────────────────────────────────────────────────────────┤
│  2. GENERIRANJE ZKI                                             │
│  ─────────────────────────────────────────────────────────────  │
│  - Konkatenacija: OIB + vrijeme + broj + PP + NU + iznos        │
│  - RSA potpis privatnim ključem                                 │
│  - MD5 hash → 32 hex znaka                                      │
├─────────────────────────────────────────────────────────────────┤
│  3. KREIRANJE SOAP PORUKE                                       │
│  ─────────────────────────────────────────────────────────────  │
│  - Sastavi RacunZahtjev XML                                     │
│  - Dodaj Zaglavlje (IdPoruke UUID, DatumVrijeme)                │
│  - Uključi sve elemente Racun                                   │
├─────────────────────────────────────────────────────────────────┤
│  4. DIGITALNI POTPIS                                            │
│  ─────────────────────────────────────────────────────────────  │
│  - Kanonikalizacija (Exclusive C14N)                            │
│  - SHA1/SHA256 digest                                           │
│  - RSA potpis SignedInfo                                        │
│  - Umetni Signature element                                     │
├─────────────────────────────────────────────────────────────────┤
│  5. SLANJE NA CIS                                               │
│  ─────────────────────────────────────────────────────────────  │
│  - HTTPS POST na endpoint                                       │
│  - TLS s FINA certifikatom                                      │
│  - Timeout: 5 sekundi                                           │
├─────────────────────────────────────────────────────────────────┤
│  6. OBRADA ODGOVORA                                             │
│  ─────────────────────────────────────────────────────────────  │
│  - Uspjeh: Izvuci JIR                                           │
│  - Greška: Log i retry queue                                    │
│  - Spremi zahtjev/odgovor za audit                              │
├─────────────────────────────────────────────────────────────────┤
│  7. GENERIRANJE QR KODA                                         │
│  ─────────────────────────────────────────────────────────────  │
│  - URL s JIR/ZKI, datum, iznos                                  │
│  - QR verzija 4, L korekcija                                    │
│  - Min 2x2 cm                                                   │
├─────────────────────────────────────────────────────────────────┤
│  8. ISPIS RAČUNA                                                │
│  ─────────────────────────────────────────────────────────────  │
│  - Ispiši sve obvezne elemente                                  │
│  - Uključi ZKI, JIR (ako postoji), QR                           │
│  - Arhiviraj kopiju                                             │
└─────────────────────────────────────────────────────────────────┘
```

### Greške i retry logika

```go
package fiscalization

import (
    "context"
    "time"
)

type RetryConfig struct {
    MaxRetries     int
    InitialBackoff time.Duration
    MaxBackoff     time.Duration
    BackoffFactor  float64
}

var DefaultRetryConfig = RetryConfig{
    MaxRetries:     5,
    InitialBackoff: 1 * time.Second,
    MaxBackoff:     60 * time.Second,
    BackoffFactor:  2.0,
}

type FiscalizationError struct {
    Code      string
    Message   string
    Retryable bool
}

var (
    ErrTimeout        = FiscalizationError{"TIMEOUT", "CIS timeout", true}
    ErrNetworkFailure = FiscalizationError{"NETWORK", "Network failure", true}
    ErrServerError    = FiscalizationError{"SERVER", "CIS server error", true}
    ErrInvalidRequest = FiscalizationError{"INVALID", "Invalid request", false}
    ErrCertificate    = FiscalizationError{"CERT", "Certificate error", false}
)

func (c *Client) SendWithRetry(ctx context.Context, req *RacunZahtjev) (*RacunOdgovor, error) {
    var lastErr error
    backoff := c.retryConfig.InitialBackoff

    for attempt := 0; attempt <= c.retryConfig.MaxRetries; attempt++ {
        resp, err := c.Send(ctx, req)
        if err == nil {
            return resp, nil
        }

        fiscErr, ok := err.(FiscalizationError)
        if !ok || !fiscErr.Retryable {
            return nil, err
        }

        lastErr = err

        select {
        case <-ctx.Done():
            return nil, ctx.Err()
        case <-time.After(backoff):
            backoff = time.Duration(float64(backoff) * c.retryConfig.BackoffFactor)
            if backoff > c.retryConfig.MaxBackoff {
                backoff = c.retryConfig.MaxBackoff
            }
        }
    }

    // Označi za naknadnu dostavu
    req.NakijeDost = true
    c.retryQueue.Push(req)

    return nil, lastErr
}
```

---

## Testiranje

### Testno okruženje

```
┌─────────────────────────────────────────────────────────────────┐
│  EDUKACIJSKI SERVIS                                             │
│  ─────────────────────────────────────────────────────────────  │
│  URL: Dostupan u tehničkoj dokumentaciji PU                     │
│  WSDL: WSDL-EDUC (v1.10)                                        │
│  Certifikat: Demo certifikati od FINA                           │
├─────────────────────────────────────────────────────────────────┤
│  PRODUKCIJSKI SERVIS                                            │
│  ─────────────────────────────────────────────────────────────  │
│  URL: https://cis.porezna-uprava.hr:8509/FiskalizacijaService   │
│  WSDL: WSDL-PROD (v1.10)                                        │
│  Certifikat: Produkcijski FINA certifikat                       │
└─────────────────────────────────────────────────────────────────┘
```

### Test scenariji

```
┌─────────────────────────────────────────────────────────────────┐
│  OSNOVNI SCENARIJI                                              │
│  ─────────────────────────────────────────────────────────────  │
│  ✓ Uspješna fiskalizacija s PDV-om (25%, 13%, 5%)               │
│  ✓ Uspješna fiskalizacija bez PDV-a                             │
│  ✓ Različiti načini plaćanja (G, K, T, O)                       │
│  ✓ Naknadna dostava (NakijeDost=true)                           │
│  ✓ Storniranje računa                                           │
├─────────────────────────────────────────────────────────────────┤
│  EDGE CASES                                                     │
│  ─────────────────────────────────────────────────────────────  │
│  ✓ Minimalni iznos (0.01 EUR)                                   │
│  ✓ Maksimalni iznos                                             │
│  ✓ Posebni znakovi u nazivima                                   │
│  ✓ Dugački nazivi stavki                                        │
│  ✓ Više PDV stopa na istom računu                               │
├─────────────────────────────────────────────────────────────────┤
│  ERROR HANDLING                                                 │
│  ─────────────────────────────────────────────────────────────  │
│  ✓ Neispravan OIB                                               │
│  ✓ Istekli certifikat                                           │
│  ✓ Neispravan ZKI                                               │
│  ✓ Timeout (simulacija)                                         │
│  ✓ Duplikat broja računa                                        │
├─────────────────────────────────────────────────────────────────┤
│  F2 SCENARIJI                                                   │
│  ─────────────────────────────────────────────────────────────  │
│  ✓ UBL 2.1 validacija                                           │
│  ✓ EN 16931 usklađenost                                         │
│  ✓ KPD šifre                                                    │
│  ✓ Predujam workflow (izdavanje → storno → konačni)             │
│  ✓ eIzvještavanje o naplati                                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Izvori

### Službena dokumentacija

- [Porezna uprava - Bezgotovinski računi (F2)](https://porezna.gov.hr/fiskalizacija/bezgotovinski-racuni)
- [Porezna uprava - Tehnički podaci](https://porezna.gov.hr/fiskalizacija/gotovinski-racuni/tehnicki-podaci/o/tehnicki-podaci)
- [Tehnička specifikacija v2.6 (PDF)](https://porezna-uprava.gov.hr/UserDocsImages/Fiskalizacija/Tehničke%20specifikacije/Fiskalizacija%20-%20Tehnicka%20specifikacija%20za%20korisnike_v2.6.pdf)

### Fiskalizacija 2.0

- [Fiskalizacija 2.0 - Zakon pojednostavljeno](https://fiskalizacija2.hr/zakon-o-fiskalizaciji-pojednostavljeno/)
- [Fiskalizacija 2.0 - Identifikatori](https://fiskalizacija2.hr/rjecnik-fiskalizacije-2-0/identifikatori/)
- [Fiskalizacija 2.0 - Tehnička specifikacija eRačuna](https://fiskalizacija2.hr/rjecnik-fiskalizacije-2-0/tehnicka-specifikacija-eracuna/)

### Certifikati

- [FINA - Poslovni certifikati za fiskalizaciju](https://www.fina.hr/poslovni-digitalni-certifikati/poslovni-certifikati-za-fiskalizaciju)
- [FINA Root certifikati](https://www.fina.hr/ca-fina-root-certifikati)

### EN 16931 i UBL

- [EU eInvoicing - Croatia](https://ec.europa.eu/digital-building-blocks/sites/display/DIGITAL/eInvoicing+in+Croatia)
- [HR CIUS Specifikacija](https://porezna.gov.hr/fiskalizacija/api/dokumenti/99)

### Implementacije

- [Cis.Fiscalization (.NET)](https://github.com/tgrospic/Cis.Fiscalization)
- [Fisver Java API](https://fisver.eu/croatia/hr/)
- [Evolva API](https://www.evolva.hr/hr/fiskalizacijski_api.html)

---

*Dokument ažuriran: prosinac 2025*
*Verzija: 1.0*
