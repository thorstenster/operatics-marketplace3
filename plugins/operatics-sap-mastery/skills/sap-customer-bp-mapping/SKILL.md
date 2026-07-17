# Debitor/Kreditor ↔ Geschäftspartner (CVI-Verknüpfung)

## Der Fallstrick

`KUNNR` (Debitor, z. B. aus VBAK) und `BU_PARTNER` (Geschäftspartner, z. B. aus BUT000)
sind **verschiedene Nummernkreise**. Eine identische Ziffernfolge (z. B. beide `0000000001`)
ist **reiner Zufall** und darf niemals als Gleichheit angenommen werden. Die korrekte
Zuordnung läuft immer über eine eigene Verknüpfungstabelle, nicht über einen Vergleich
der Nummern.

## Vorgehen

Immer `query_table` verwenden (kein `run_abap_code` nötig) — zwei Schritte:

**1. Debitor/Kreditor → Partner-GUID**

| Richtung | Tabelle | Schlüsselfeld | Ergebnisfeld |
|---|---|---|---|
| Debitor → Partner | `CVI_CUST_LINK` | `CUSTOMER` = KUNNR | `PARTNER_GUID` |
| Kreditor → Partner | `CVI_VEND_LINK` | `VENDOR` = LIFNR | `PARTNER_GUID` |

**2. Partner-GUID → Geschäftspartnernummer + Name**

Tabelle `BUT000`, Feld `PARTNER_GUID` gegen das Ergebnis aus Schritt 1 abgleichen.
Liefert `PARTNER` (= BU_PARTNER), sowie je nach Partnertyp `NAME_LAST`/`NAME_FIRST`
(Person) oder `NAME_ORG1` (Organisation).

⚠️ Feldliste bei `query_table` auf `BUT000` **schmal halten** (z. B. nur `PARTNER`) —
eine zu breite Feldauswahl kann zu einem Parser-Fehler führen ("Arbeitsbereich hat
weniger Felder als selektiert werden").

## Beispiel (verifiziert auf S4S)

```
Debitor 0000000001
  → CVI_CUST_LINK: CUSTOMER = 0000000001 → PARTNER_GUID = Y8se9InHH9Gggn/2ULGFpQ==
  → BUT000: PARTNER_GUID = Y8se9InHH9Gggn/2ULGFpQ== → PARTNER = 0000000012,
    NAME_ORG1 = "Kundenfutter GmbH"
```

Der naheliegende, aber falsche Kurzschluss wäre gewesen: Debitor `0000000001` =
Geschäftspartner `0000000001` ("Willi Winzig"). Tatsächlich gehört Debitor `0000000001`
zum Geschäftspartner `0000000012` ("Kundenfutter GmbH").

## Merksatz

**KUNNR ≠ BU_PARTNER.** Immer über `CVI_CUST_LINK`/`CVI_VEND_LINK` + `BUT000` auflösen,
nie über Nummerngleichheit raten.

## Bevorzugter Weg, wenn ein CDS-View mit Assoziation verfügbar ist

Wenn der Ausgangsbeleg über einen modernen CDS-View gelesen wird (z. B. `I_SalesOrder`
für Verkaufsaufträge), NICHT den manuellen CVI-Umweg gehen. Stattdessen die Assoziation
direkt nutzen:

1. `I_SalesOrder` liefert `SoldToParty` (= KUNNR) zum Beleg.
2. `I_Customer`, gefiltert auf `Customer` = `SoldToParty`, liefert bereits aufgelöste
   Klartextfelder — u. a. `CustomerName`, `CustomerFullName`, `BPCustomerName`,
   `OrganizationBPName1`, Adresse (Stadt, PLZ, Straße).

Das ist der direkte, assoziative Weg (so wie es ein Fiori-App-Entwickler auch tun würde)
und spart den Zwischenschritt über `CVI_CUST_LINK`/`BUT000`. Der CVI-Weg (oben) bleibt
nötig, wenn nur klassische Tabellen wie `VBAK`/`EKKO` ohne zugehörigen CDS-View zur
Verfügung stehen.

**Beispiel (verifiziert auf S4S):**
```
I_SalesOrder: SalesOrder = 0000000003 → SoldToParty = 0000000001
I_Customer:   Customer = 0000000001 → CustomerName = "Kundenfutter GmbH",
              CustomerFullName = "Firma Kundenfutter GmbH/53179 Bonn"
```
