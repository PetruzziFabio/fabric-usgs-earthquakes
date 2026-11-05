# Visual della dashboard Earthquakes_Dashboard

## Elenco dei visual

| Tile | Tipo visual | Query KQL / sorgente |
|---|---|---|
| Terremoti di magnitudo ≥ 4 | Table | Q1, con filtro dashboard _area |
| Terremoti per ora | Time chart | Q2, con filtro dashboard _area |
| Eventi per area e ora: confronto con la media | Table | Q3, con filtro dashboard _area |
| Distribuzione dei terremoti per magnitudo | Column chart | Q4, con filtro dashboard _area |
| Mappa degli epicentri | Fabric Map incorporata | Item Map collegato alla funzione EarthquakesMap() |

## Organizzazione

- Pagina 1: tabella Q1, grafico Q2, tabella Q3 e grafico Q4.
- Pagina 2: Fabric Map a dimensione ampliata.

## Filtri

Il parametro _area permette di selezionare tutte le aree oppure una
delle fasce definite nel producer: americas, europe_africa, asia_pacific.

Il parametro si applica alle quattro tile KQL della prima pagina.
Non si applica alla Fabric Map incorporata.

Le aree sono fasce definite dalla longitudine, non confini esatti
dei continenti.

## Periodi analizzati

- Q1, Q2 e Q4: ultimi 7 giorni, secondo le rispettive query.
- Q3: ultime 24 ore complete.
- Mappa: ultimi 7 giorni, eventi di magnitudo almeno 4.
- Mappa test: eventi ultime 24 ore, magnitudo di almeno 4.

## Query effettive delle tile

### Q1 — Tabella terremoti

Earthquakes
| summarize arg_max(updated_time, *) by event_id
| where event_time between (ago(7d) .. now())
| where _area == "tutte" or area == _area
| where magnitude >= 4
| extend Profondita = case(
    depth_km < 70, "Superficiale (<70 km)",
    depth_km < 300, "Intermedia (70-300 km)",
    "Profonda (>=300 km)"
)
| project
    event_time,
    area,
    magnitude,
    place,
    depth_km,
    Profondita,
    event_id
| order by magnitude desc, event_time desc

### Q2 - Time chart numero terremoti per ora

let Fine = bin(now(), 1h);
let Inizio = Fine - 7d;
Earthquakes
| summarize arg_max(updated_time, *) by event_id
| where event_time >= Inizio and event_time < Fine
| where _area == "tutte" or area == _area
| make-series NumeroEventi = count() default=0
    on event_time from Inizio to Fine step 1h
| render timechart

### Q3 - Tabella Eventi per area e ora: confronto con la media

let Fine = bin(now(), 1h);
let Inizio = Fine - 24h;
let Ore = range Ora from Inizio to (Fine - 1h) step 1h;
let Aree = datatable(area:string)
[
    "americas",
    "europe_africa",
    "asia_pacific"
];
let Conteggi =
    Earthquakes
    | summarize arg_max(updated_time, *) by event_id
    | where event_time >= Inizio and event_time < Fine
    | summarize NumeroEventi = count() by area, Ora = bin(event_time, 1h);
let ConteggiCompleti =
    Aree
    | extend Chiave = 1
    | join kind=inner (Ore | extend Chiave = 1) on Chiave
    | project area, Ora
    | join kind=leftouter Conteggi on area, Ora
    | project area, Ora, NumeroEventi = coalesce(NumeroEventi, tolong(0));
let Medie =
    ConteggiCompleti
    | summarize MediaOraria = avg(todouble(NumeroEventi)) by area;
ConteggiCompleti
| join kind=inner Medie on area
| where _area == "tutte" or area == _area
| extend
    SopraMedia = NumeroEventi > MediaOraria,
    Scostamento = round(NumeroEventi - MediaOraria, 2)
| project
    Ora,
    area,
    NumeroEventi,
    MediaOraria = round(MediaOraria, 2),
    SopraMedia,
    Scostamento
| order by Ora desc, area asc

### Q4 - Distribuzione dei terremoti per magnitudo

Earthquakes
| summarize arg_max(updated_time, *) by event_id
| where event_time between (ago(7d) .. now())
| where _area == "tutte" or area == _area
| where isnotnull(magnitude)
| extend FasciaMagnitudo = case(
    magnitude < 2, "< 2",
    magnitude < 3, "2 - <3",
    magnitude < 4, "3 - <4",
    magnitude < 5, "4 - <5",
    magnitude < 6, "5 - <6",
    ">= 6"
)
| extend Ordine = case(
    magnitude < 2, 1,
    magnitude < 3, 2,
    magnitude < 4, 3,
    magnitude < 5, 4,
    magnitude < 6, 5,
    6
)
| summarize NumeroEventi = count() by Ordine, FasciaMagnitudo
| order by Ordine asc
| project FasciaMagnitudo, NumeroEventi