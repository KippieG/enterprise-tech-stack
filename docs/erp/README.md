# ERP — Business Central

## Overzicht

Microsoft Dynamics 365 Business Central (BC) is het ERP-systeem voor kleine tot middelgrote bedrijven. Het beheert financiën, verkoop, inkoop, voorraad, productie en projecten. Business Central is uitbreidbaar via **AL-code** (Application Language) en integreert met de rest van de Microsoft stack.

---

## Architectuur

```
Business Central (SaaS / On-Prem / Docker)
├── Base Application (Microsoft standard)
├── Extensions (jouw aanpassingen — AL code)
│   ├── Table Extensions       (extra velden op bestaande tabellen)
│   ├── Page Extensions        (extra UI op bestaande pagina's)
│   ├── Report Extensions      (aanpassingen op rapporten)
│   └── Codeunit Extensions    (aanpassingen op business logic)
├── Integraties
│   ├── API Pages → REST API voor externe systemen
│   ├── Web Services (OData / SOAP legacy)
│   └── Azure Service Bus / Logic Apps
└── Microsoft 365
    ├── Teams integratie
    ├── Outlook Add-in
    └── Power BI rapportage
```

---

## AL — Application Language

### Table Extension

```al
tableextension 50100 "Ext. Customer" extends Customer
{
    fields
    {
        field(50100; "Customer Category"; Enum "Customer Category")
        {
            Caption = 'Customer Category';
            DataClassification = CustomerContent;
        }
        field(50101; "External Reference"; Code[30])
        {
            Caption = 'External Reference';
            DataClassification = CustomerContent;
        }
        field(50102; "Last Sync Date"; DateTime)
        {
            Caption = 'Last Sync Date';
            DataClassification = SystemMetadata;
        }
    }
}

enum 50100 "Customer Category"
{
    Extensible = true;

    value(0; " ") { Caption = ' '; }
    value(1; "A") { Caption = 'A - Strategisch'; }
    value(2; "B") { Caption = 'B - Standaard'; }
    value(3; "C") { Caption = 'C - Occasioneel'; }
}
```

### Page Extension

```al
pageextension 50100 "Ext. Customer Card" extends "Customer Card"
{
    layout
    {
        addafter(Name)
        {
            field("Customer Category"; Rec."Customer Category")
            {
                ApplicationArea = All;
                ToolTip = 'Geeft de categorie van de klant aan.';
            }
            field("External Reference"; Rec."External Reference")
            {
                ApplicationArea = All;
                ToolTip = 'Externe referentie voor integraties.';
            }
        }
    }

    actions
    {
        addlast(processing)
        {
            action("Sync to WMS")
            {
                ApplicationArea = All;
                Caption = 'Synchroniseer naar WMS';
                Image = TransmitElectronic;
                ToolTip = 'Stuur klantgegevens naar het Warehouse Management System.';

                trigger OnAction()
                var
                    WMSSyncMgt: Codeunit "WMS Sync Management";
                begin
                    WMSSyncMgt.SyncCustomer(Rec);
                    Message('Klant succesvol gesynchroniseerd naar WMS.');
                end;
            }
        }
    }
}
```

### Codeunit — Business Logic

```al
codeunit 50100 "WMS Sync Management"
{
    procedure SyncCustomer(Customer: Record Customer)
    var
        HttpClient: HttpClient;
        HttpContent: HttpContent;
        HttpResponse: HttpResponseMessage;
        JsonPayload: Text;
        WMSApiUrl: Text;
    begin
        WMSApiUrl := GetSetup().WMSEndpoint + '/api/customers';
        JsonPayload := BuildCustomerJson(Customer);

        HttpContent.WriteFrom(JsonPayload);
        HttpContent.GetHeaders().Add('Content-Type', 'application/json');
        HttpContent.GetHeaders().Add('Authorization', 'Bearer ' + GetSetup().WMSApiKey);

        if not HttpClient.Post(WMSApiUrl, HttpContent, HttpResponse) then
            Error('Verbinding met WMS mislukt. Controleer de netwerkinstellingen.');

        if not HttpResponse.IsSuccessStatusCode then
            Error('WMS synchronisatie gefaald: %1', HttpResponse.ReasonPhrase);

        Customer."Last Sync Date" := CurrentDateTime;
        Customer.Modify();
    end;

    local procedure BuildCustomerJson(Customer: Record Customer): Text
    var
        Json: JsonObject;
        JsonText: Text;
    begin
        Json.Add('externalId', Customer."No.");
        Json.Add('name', Customer.Name);
        Json.Add('address', Customer.Address);
        Json.Add('city', Customer.City);
        Json.Add('country', Customer."Country/Region Code");
        Json.Add('category', Format(Customer."Customer Category"));
        Json.WriteToText(JsonText);
        exit(JsonText);
    end;

    local procedure GetSetup(): Record "WMS Integration Setup"
    var
        Setup: Record "WMS Integration Setup";
    begin
        Setup.Get();
        exit(Setup);
    end;
}
```

---

## API Pages — REST API blootstellen

```al
page 50110 "Customer API"
{
    PageType = API;
    APIPublisher = 'mycompany';
    APIGroup = 'integration';
    APIVersion = 'v1.0';
    EntitySetName = 'customers';
    EntityName = 'customer';
    SourceTable = Customer;
    ODataKeyFields = SystemId;
    DelayedInsert = true;

    layout
    {
        area(Content)
        {
            repeater(Group)
            {
                field(id; Rec.SystemId) { }
                field(number; Rec."No.") { }
                field(name; Rec.Name) { }
                field(city; Rec.City) { }
                field(country; Rec."Country/Region Code") { }
                field(category; Rec."Customer Category") { }
                field(lastSyncDate; Rec."Last Sync Date") { }
            }
        }
    }
}

// Aanroepen:
// GET /api/mycompany/integration/v1.0/companies({id})/customers
// POST /api/mycompany/integration/v1.0/companies({id})/customers
```

---

## Events & Subscribers

Business Central werkt met een **Publisher/Subscriber** patroon voor clean extensies:

```al
// Publisher (in base app of jouw extension)
codeunit 50200 "Order Events"
{
    [IntegrationEvent(false, false)]
    procedure OnAfterOrderPosted(var SalesInvoiceHeader: Record "Sales Invoice Header")
    begin
    end;
}

// Subscriber (in een andere extension — losse koppeling)
codeunit 50201 "Order Posted Subscriber"
{
    [EventSubscriber(ObjectType::Codeunit, Codeunit::"Order Events",
        'OnAfterOrderPosted', '', false, false)]
    local procedure HandleOrderPosted(var SalesInvoiceHeader: Record "Sales Invoice Header")
    var
        WMSSyncMgt: Codeunit "WMS Sync Management";
    begin
        WMSSyncMgt.NotifyWMSOrderPosted(SalesInvoiceHeader);
    end;
}
```

---

## Integratiepaden

| Richting | Technologie | Gebruik |
|----------|-------------|---------|
| BC → Extern | API Page (OData) | Externe app leest BC data |
| Extern → BC | API Page (POST/PATCH) | Externe app schrijft naar BC |
| BC → Extern (event) | Azure Service Bus via AL | BC stuurt events bij wijzigingen |
| Extern → BC (batch) | Azure Logic Apps | Periodieke synchronisaties |
| BC → Power BI | OData feed | Rapportage en dashboards |

---

## Extension Lifecycle

```bash
# Ontwikkelen
code .           # VS Code met AL Language extension
al: download symbols    # Symbolen ophalen van BC server
F5               # Publiceer en debug in sandbox

# Builden
al: build        # Genereert .app bestand

# Deployen
# Via Business Central Admin Center of:
az storage blob upload --file MyExtension_1.0.0.0.app ...
```

---

## Best Practices

- Gebruik **objectrange** boven 50000 voor eigen objecten (per licentieafspraak)
- Nooit `Codeunit.Run` met globale variabelen — geef parameters door
- Gebruik **SetRange / SetFilter** voor queries — nooit loop + if
- Schrijf events op kritische punten zodat andere extensies kunnen subscriben
- Test altijd in een **sandbox** of **Docker container** voor acceptatie
- Gebruik **AppSourceCop** en **PerTenantExtensionCop** linting

---

*[← Terug naar overzicht](../../README.md)*
