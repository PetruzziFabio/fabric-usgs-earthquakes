# Eventstream USGS Earthquakes

## Scopo

Ricevere gli eventi sismici dal producer Python tramite un Custom endpoint,
trasformarli e salvarli nella tabella Earthquakes dell'Eventhouse.

## Item Fabric

- Eventstream: esUSGSEarthquakes
- Sorgente: usgs-source
- Eventhouse: EH_Earthquakes
- Tabella di destinazione: Earthquakes

## Topologia

usgs-source → esUSGSEarthquakes → NormalizeFields
→ ValidMagnitude → EarthquakesDestination

## Configurazione dei nodi

| Nodo | Tipo | Configurazione |

| usgs-source | Custom endpoint | Riceve eventi JSON piatti dal producer Python tramite il protocollo compatibile Event Hubs. |
| NormalizeFields | Manage fields | Converte event_time, updated_time e ingest_time in DateTime; magnitude, latitude, longitude e depth_km in Double; significance e tsunami in Long/Int64. Conserva gli altri campi. |
| ValidMagnitude | Filter | Mantiene gli eventi con magnitude non nulla. |
| EarthquakesDestination | Eventhouse | Scrive nella tabella Earthquakes del database selezionato in EH_Earthquakes, con elaborazione prima dell'ingestione. |

## Credenziali

La connection string del Custom endpoint è conservata nel file .env locale.

## Evidenze

- topologia.png: screenshot della topologia completa in Live view.
- preview.png: screenshot del Data preview della destinazione con dati ricevuti.

## Problemi e soluzioni

La sottoscrizione Azure non era più attiva: abbiamo sostituito
Azure IoT Hub con il Custom endpoint di Fabric Eventstream.

## Estensione: Fabric Map

Abbiamo creato una Fabric Map autonoma collegata all'Eventhouse
tramite la funzione KQL EarthquakesMap().

La mappa mostra gli epicentri degli eventi di magnitudo almeno 4
degli ultimi 7 giorni, mantenendo l'ultima versione per event_id.

La mappa è stata incorporata nella Real-Time Dashboard.

