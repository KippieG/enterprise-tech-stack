# ITSM — TopDesk

## Overzicht

TOPdesk is een IT Service Management (ITSM) tool die helpt bij het beheren van incidenten, wijzigingen, problemen en assets. Het is breed ingezet bij overheid, logistiek en industrie in de Benelux.

---

## Kernmodules

| Module | Doel |
|--------|------|
| **Incidentbeheer** | Meldingen van gebruikers registreren en oplossen |
| **Wijzigingsbeheer** | Gecontroleerd doorvoeren van IT-wijzigingen |
| **Probleembeheer** | Structurele oorzaken van incidenten aanpakken |
| **Assetbeheer** | Hardware, software en licenties bijhouden |
| **Selfservice Portal** | Eindgebruikers melden zelf en volgen op |
| **Rapportages** | SLA-bewaking, throughput, doorlooptijden |

---

## Incidentbeheer Workflow

```
Eindgebruiker meldt via:
    ├── Selfservice Portal
    ├── E-mail
    └── Telefonisch (1e lijn registreert)

1e Lijn Behandelaar:
    ├── Categoriseer (CI, categorie, subcategorie)
    ├── Prioriteit instellen (impact × urgentie matrix)
    │   ├── P1 — Kritisch: systeem volledig down
    │   ├── P2 — Hoog: ernstige verstoring
    │   ├── P3 — Middel: beperkte verstoring
    │   └── P4 — Laag: geen tijdsdruk
    ├── Probeer directe oplossing via kennisbank
    └── Kan niet oplossen → Escaleer naar 2e lijn

2e Lijn / Expert:
    ├── Analyse en oplossing
    ├── Werk bijhouden in actielogboek
    └── Oplossing gedocumenteerd

Afsluiting:
    ├── Oplossing beschreven
    ├── Eindgebruiker bevestigt (of auto-sluit na X dagen)
    └── Optioneel: kennisbankartikel aanmaken
```

---

## SLA Configuratie

```
Prioriteit P1:
    Response tijd:    15 minuten
    Oplossingstijd:   4 uur
    Escalatie na:     2 uur zonder update

Prioriteit P2:
    Response tijd:    1 uur
    Oplossingstijd:   8 uur werkuren
    Escalatie na:     4 uur zonder update

Prioriteit P3:
    Response tijd:    4 uur werkuren
    Oplossingstijd:   3 werkdagen

Prioriteit P4:
    Response tijd:    1 werkdag
    Oplossingstijd:   5 werkdagen
```

---

## TOPdesk REST API

### Authenticatie

```http
GET /api/incidents
Authorization: Basic base64(gebruikersnaam:appwachtwoord)
Content-Type: application/json
```

Of met API token:

```http
Authorization: TOKEN <jouw-api-token>
```

### Incident aanmaken via API

```http
POST /api/incidents
Content-Type: application/json

{
  "status": "firstLine",
  "briefDescription": "Printer op verdieping 2 geeft papierstoring",
  "request": "Gebruiker meldt dat de printer HP LaserJet een papierstoring geeft. Storing begon om 09:30.",
  "category": { "name": "Hardware" },
  "subcategory": { "name": "Printer" },
  "callType": { "name": "Incident" },
  "entryType": { "name": "Selfservice" },
  "impact": { "name": "Department" },
  "urgency": { "name": "Normal" },
  "priority": { "name": "P3" },
  "caller": {
    "email": "jan.janssen@bedrijf.be"
  }
}
```

### Incident opvragen

```http
GET /api/incidents?page_size=10&page_number=0&status=firstLine
```

### Incident bijwerken

```http
PATCH /api/incidents/id/{incident-id}
Content-Type: application/json

{
  "action": "Behandelaar heeft remote verbinding gemaakt. Papierstoring gevonden op lade 2.",
  "status": "secondLine",
  "operator": { "id": "behandelaar-uuid" }
}
```

---

## Integratie met .NET

```csharp
public class TopDeskService : ITopDeskService
{
    private readonly HttpClient _http;
    private readonly IOptions<TopDeskSettings> _settings;

    public TopDeskService(HttpClient http, IOptions<TopDeskSettings> settings)
    {
        _http = http;
        _settings = settings;

        var credentials = Convert.ToBase64String(
            Encoding.UTF8.GetBytes($"{settings.Value.Username}:{settings.Value.AppPassword}"));
        _http.DefaultRequestHeaders.Authorization =
            new AuthenticationHeaderValue("Basic", credentials);
        _http.BaseAddress = new Uri(settings.Value.BaseUrl);
    }

    public async Task<string> CreateIncidentAsync(CreateIncidentDto dto)
    {
        var payload = new
        {
            status = "firstLine",
            briefDescription = dto.Subject,
            request = dto.Description,
            category = new { name = dto.Category },
            priority = new { name = dto.Priority },
            caller = new { email = dto.CallerEmail }
        };

        var response = await _http.PostAsJsonAsync("/api/incidents", payload);
        response.EnsureSuccessStatusCode();

        var result = await response.Content.ReadFromJsonAsync<JsonElement>();
        return result.GetProperty("id").GetString()!;
    }

    public async Task<IncidentDto?> GetIncidentAsync(string id)
    {
        var response = await _http.GetAsync($"/api/incidents/id/{id}");
        if (response.StatusCode == HttpStatusCode.NotFound) return null;
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<IncidentDto>();
    }
}
```

---

## Wijzigingsbeheer (Change Management)

### RFC (Request for Change) workflow

```
1. Aanvraag (RFC aanmaken)
   → Beschrijving, impact, risico, roll-back plan

2. Review & Goedkeuring
   → Change Advisory Board (CAB) vergadering
   → Goedkeuring of afwijzing

3. Planning
   → Onderhoudsvenster inplannen
   → Betrokken systemen en teams notificeren

4. Implementatie
   → Stap-voor-stap uitvoeringsplan
   → Go/No-go checkpoint

5. Verificatie
   → Tests na implementatie
   → Bevestiging door eindgebruikers

6. Afsluiting
   → Succes of rollback gedocumenteerd
   → Kennisbank bijwerken
```

---

## Rapportages & KPI's

| KPI | Omschrijving | Streefwaarde |
|-----|--------------|--------------|
| First Call Resolution (FCR) | % incidenten opgelost door 1e lijn | > 70% |
| SLA naleving P1/P2 | % tijdig opgelost | > 95% |
| Gemiddelde doorlooptijd | Gemiddelde tijd tot sluiting per prioriteit | Varieert |
| Herhalende incidenten | % incidenten dat terugkomt | < 10% |
| Klanttevredenheid | Score na afsluiting | > 4/5 |

---

## Best Practices

- Gebruik **categorieën en subcategorieën** consistent — dit is de basis voor rapportage
- Documenteer **elke actie** in het actielogboek — niet in e-mails
- Schrijf **kennisbank artikelen** voor terugkerende problemen
- Koppel **assets aan incidenten** — zo zie je welke CI's de meeste incidenten veroorzaken
- Gebruik de **API** voor integratie met monitoring tools (Zabbix, Azure Monitor) zodat alerts automatisch incidenten aanmaken

---

*[← Terug naar overzicht](../../README.md)*
