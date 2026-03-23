# Biedkracht

**TenderNed aanbestedingen monitor + AI tender response generator.**

Plak een TenderNed URL, ontvang een gestructureerde tender-inschrijving. Biedkracht automatiseert het analyseren en beantwoorden van Nederlandse overheidsaanbestedingen.

## Wat het doet

```
TenderNed URL plakken
    → Metadata ophalen via TenderNed API
        → Eisen en criteria extraheren uit Bestek
            → Concept-inschrijving genereren per sectie
                → Klaar voor review en indienen
```

### Kernfunctionaliteit

| Functie | Beschrijving |
|---------|-------------|
| **Tender Monitor** | Dagelijkse scan op nieuwe aanbestedingen per CPV-code en sector |
| **Analyse** | Automatische extractie van eisen, EMVI-criteria, deadlines en vereiste documenten |
| **Response Generator** | AI-gegenereerde concept-inschrijving in het juiste register — per sectie |
| **Compliance Check** | Verificatie of alle vereiste onderdelen aanwezig zijn voor indienen |

## Architectuur

```
┌─────────────────────────────────────────────────┐
│  Browser (Vue 3 SPA)                            │
│  ┌────────────┐  ┌───────────────────────────┐  │
│  │ Home       │  │ Response Viewer           │  │
│  │ URL input  │  │ Documenten + compliance   │  │
│  └─────┬──────┘  └────────────┬──────────────┘  │
└────────┼───────────────────────┼─────────────────┘
         │ REST                  │ REST + polling
┌────────┼───────────────────────┼─────────────────┐
│  FastAPI Backend                                  │
│                                                   │
│  ┌──────────┐  ┌───────────┐  ┌───────────────┐  │
│  │/analyse  │  │/genereer  │  │/auth          │  │
│  │URL→meta  │  │job queue  │  │magic links    │  │
│  └────┬─────┘  └─────┬─────┘  └───────────────┘  │
│       │               │                           │
│  ┌────▼───────────────▼───────────────────────┐   │
│  │  Pipeline                                  │   │
│  │                                            │   │
│  │  1. TenderNed API → metadata + documenten  │   │
│  │  2. Extractor → eisen, EMVI, deadlines     │   │
│  │  3. Generator → concept per sectie         │   │
│  │  4. Compliance → completeness check        │   │
│  └────────────────────┬───────────────────────┘   │
└───────────────────────┼───────────────────────────┘
                        │
┌───────────────────────┼───────────────────────────┐
│  PostgreSQL                                        │
│  ┌────────┐  ┌────────┐  ┌──────────┐             │
│  │ jobs   │  │ users  │  │responses │             │
│  │(queue) │  │        │  │          │             │
│  └────────┘  └────────┘  └──────────┘             │
└────────────────────────────────────────────────────┘
```

### Data Flow

```
POST /api/analyse
  → Parse TenderNed URL
  → Fetch metadata via TenderNed REST API
  → Return: titel, aanbestedende dienst, CPV-codes, deadline, type procedure

POST /api/genereer
  → Fetch tender documenten (Bestek, Nota van Inlichtingen)
  → Extraheer eisen en EMVI-criteria
  → Genereer concept-inschrijving per sectie:
      - Management samenvatting
      - Geschiktheidseisen
      - Plan van Aanpak
      - EMVI-onderbouwing
      - Prijscomponent
  → Sla resultaat op, return job_id

GET /api/response/:id
  → Poll voor status
  → Return gegenereerde documenten + compliance score
```

## Code Sample: TenderNed API Client

De integratie met de officiële TenderNed API voor het ophalen van aanbestedingsdata:

```python
class TenderNedClient:
    """Client voor de TenderNed REST API (publicatieplatform.aanbestedingen.nl)"""

    BASE_URL = "https://publicatieplatform.aanbestedingen.nl/api/v1"

    async def fetch_tender(self, tender_url: str) -> TenderMetadata:
        """Parse een TenderNed URL en haal metadata op."""
        tender_id = self._extract_id(tender_url)

        async with self.session.get(
            f"{self.BASE_URL}/tenders/{tender_id}",
            headers=self._auth_headers()
        ) as response:
            data = await response.json()

        return TenderMetadata(
            id=tender_id,
            titel=data["title"],
            aanbestedende_dienst=data["contracting_authority"]["name"],
            cpv_codes=self._parse_cpv(data["cpv_codes"]),
            deadline=datetime.fromisoformat(data["submission_deadline"]),
            type_procedure=data["procedure_type"],
            documenten=await self._fetch_documents(tender_id)
        )

    async def monitor(
        self, cpv_codes: list[str], callback: Callable
    ) -> None:
        """Dagelijkse scan op nieuwe aanbestedingen per CPV-code."""
        for cpv in cpv_codes:
            tenders = await self._search(cpv_code=cpv, since=self.last_check)
            for tender in tenders:
                await callback(tender)
        self.last_check = datetime.now()
```

## Stack

| Component | Technologie |
|-----------|-------------|
| Frontend | Vue 3 + Vite + TypeScript |
| UI | shadcn-vue + Tailwind v4 |
| Backend | FastAPI + Python |
| Database | PostgreSQL 16 |
| AI | Anthropic Claude API |
| Auth | Magic links via Resend |
| Betaling | Stripe (iDEAL + card) |
| Infra | Docker Compose |

## Status

Biedkracht is in actieve ontwikkeling. De analyse-pipeline en response generator zijn operationeel.

## Contact

Gebouwd door **Youssef Hajar** — AI consultancy voor Nederlandse MKB.

Interesse? [yhajar@biedkracht.nl](mailto:yhajar@biedkracht.nl)
