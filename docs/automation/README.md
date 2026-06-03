# Automation — Power Platform · RPA · Chatbots

## Overzicht

Automatisering elimineert repetitief handwerk, vermindert fouten en verhoogt de snelheid van processen. Het Microsoft Power Platform combineert low-code automatisering (Power Automate), app-bouw (Power Apps), RPA (desktop flows) en AI-gedreven chatbots (Copilot Studio).

---

## Power Automate

### Cloud Flows

Cloud flows reageren op triggers en voeren acties uit — zonder server of code:

```
Trigger: Wanneer een nieuw item in SharePoint lijst wordt aangemaakt
    ↓
Voorwaarde: Kolom "Goedkeuring vereist" = Ja?
    ↓ Ja
Actie: Stuur goedkeuringsverzoek naar manager
    ↓
Wacht op reactie
    ↓ Goedgekeurd
Actie: Update item status → "Goedgekeurd"
Actie: Stuur e-mail naar aanvrager
    ↓ Afgewezen
Actie: Update item status → "Afgewezen"
Actie: Stuur e-mail met reden
```

### Integratie met REST API

```json
// HTTP action in Power Automate
{
  "method": "POST",
  "uri": "https://myapp.azurewebsites.net/api/orders",
  "headers": {
    "Authorization": "Bearer @{body('Get_token')?['access_token']}",
    "Content-Type": "application/json"
  },
  "body": {
    "orderNumber": "@triggerBody()?['OrderNumber']",
    "customerId": "@triggerBody()?['CustomerId']",
    "deliveryDate": "@triggerBody()?['DeliveryDate']"
  }
}
```

### Veelgebruikte triggers

| Trigger | Gebruik |
|---------|---------|
| HTTP Request | Aanroepen vanuit externe systemen |
| SharePoint item aangemaakt/gewijzigd | Documentworkflows |
| E-mail ontvangen (Outlook) | E-mail verwerking |
| Recurrence (timer) | Geplande taken |
| Business Central webhook | ERP-driven automation |
| Service Bus message | Enterprise messaging |

---

## RPA — Desktop Flows (Power Automate Desktop)

RPA (Robotic Process Automation) automatiseert applicaties die geen API hebben — via de UI.

### Wanneer RPA gebruiken?

- Legacy systemen zonder API (mainframes, oudere Windows apps)
- Handmatig kopiëren tussen systemen
- Formulieren invullen in webapplicaties
- Data extractie uit PDF of Excel

### Power Automate Desktop — Voorbeeld flow

```
// Pseudo-code voor een desktop flow
Launch Application: "C:\TAS\TAS.exe"
Wait for application to be ready

// Log in
Click on: [Gebruikersnaam veld]
Type text: %Config_Username%
Click on: [Wachtwoord veld]
Type text: %Config_Password%
Click on: [Inloggen knop]

// Navigeer naar transportorders
Click on: [Menu → Transport → Orders]

// Loop door orders
Loop through items in: %OrderList%
    Set current order = %Loop_Item%

    Click on: [Nieuwe order knop]
    Fill text field: [Referentie] with: %current_order['Reference']%
    Fill text field: [Klant] with: %current_order['CustomerCode']%

    Select from dropdown: [Prioriteit] value: %current_order['Priority']%

    Click on: [Opslaan]
    Wait for success message
    Extract text from: [Order ID veld] → save to: %NewOrderId%

    // Sla resultaat op
    Add row to data table: %Results%
        columns: [Reference, NewOrderId, Status]
        values: [%current_order['Reference']%, %NewOrderId%, 'Aangemaakt']

Close application
Return: %Results%
```

---

## RPA Best Practices

- Gebruik **RPA alleen als API niet mogelijk is** — API's zijn stabieler en sneller
- Bouw **foutafhandeling** in: wat als een popup verschijnt? Wat als een element niet gevonden wordt?
- Gebruik **Image anchoring** zo min mogelijk — werkt slecht bij schermresolutie/schaal wijzigingen
- **Parameteriseer** alles: gebruikersnaam, URL, bestandspaden via configuratie — niet hardcoden
- Test flows in een **testomgeving** die de productie-UI spiegelt
- Log alles naar een centrale locatie (SharePoint lijst of database)

---

## Chatbots — Microsoft Copilot Studio

### Architectuur

```
Gebruiker (Teams / SharePoint / Website)
    ↓
Copilot Studio Bot
    ├── Topics (gespreksflows)
    │   ├── FAQ Topics (veelgestelde vragen)
    │   ├── Task Topics (acties uitvoeren)
    │   └── Escalation Topic (doorverbinden naar mens)
    ├── Entities (data extraheren uit berichten)
    ├── Variables (informatie bewaren binnen gesprek)
    └── Actions
        ├── Power Automate Flow aanroepen
        ├── HTTP API aanroepen
        └── Dataverse query
```

### Topic voorbeeld: Orderstatus opvragen

```
Trigger phrases:
- "Wat is de status van mijn order?"
- "Order status"
- "Waar is mijn bestelling?"

Bot vraagt: "Wat is je ordernummer?"
Gebruiker: "ORD-2026-042"

[Action: Roep Power Automate flow aan]
    Input: OrderNumber = "ORD-2026-042"
    Returnt: OrderStatus, DeliveryDate, TrackingCode

Bot antwoordt:
"Je order ORD-2026-042 heeft status: [OrderStatus].
Verwachte levering: [DeliveryDate].
Track je zending met code: [TrackingCode]."

[Variant: Order niet gevonden]
Bot: "Ik vond geen order met dat nummer. Wil je doorbellen naar onze klantenservice?"
    → Ja: Escaleer naar live agent
    → Nee: Einde gesprek
```

---

## Power Apps

### Wanneer Power Apps vs. custom Angular?

| Criterium | Power Apps | Custom Angular |
|-----------|-----------|----------------|
| Complexiteit | Eenvoudig tot matig | Matig tot complex |
| Tijdlijn | Weken | Maanden |
| Integratie | Microsoft/Dataverse | Elke API |
| Maatwerk UI | Beperkt | Volledig |
| Offline | Ja (canvas) | Maatwerk nodig |
| Doelgroep | Citizen developers | Pro developers |

### Model-driven vs. Canvas

- **Canvas App**: volledig zelfontworpen scherm, beste voor custom workflows
- **Model-driven App**: gebouwd op Dataverse-tabellen, automatische formulieren en views

---

## Automation Center of Excellence (CoE)

Bij schaalbaar gebruik van Power Platform:

1. **Governance**: Wie mag wat bouwen? Welke connectoren zijn toegestaan?
2. **CoE Starter Kit**: Microsoft's gratis toolkit voor monitoring en beheer van alle flows/apps
3. **Environment strategie**: Dev → Test → Productie
4. **Naming conventions**: `flow_[team]_[proces]_v1`, `app_[team]_[doel]`
5. **ALM**: Solutions gebruiken voor versiebeheer en deployments

---

*[← Terug naar overzicht](../../README.md)*
