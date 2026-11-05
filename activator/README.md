# Activator – Notifica per eventi USGS sopra soglia

## Sorgente
Eventstream esUSGSEarthquakes, tramite la destinazione
AlertsEarthquakes collegata all'uscita di ValidMagnitude.

## Regola
- Valutazione: On each event when.
- Campo: magnitude.
- Condizione: Is greater than.
- Soglia: 5.
- Azione: invio messaggio Teams.

## Test end-to-end
È stato eseguito un invio dal producer Python utilizzando dati reali
del feed USGS all_day.geojson.

Il messaggio ha attraversato il percorso:
Python → Custom endpoint → Eventstream → Activator → messaggio Teams.

- Data e ora del test: 29/10/2026 - 17.50
- Esito: notifica messaggio ricevuto.
- Evidenza della configurazione: rule.png.
- Evidenza della notifica: alert.png.

## Limiti
La notifica segnala la ricezione di un messaggio sopra soglia,
non necessariamente un terremoto appena avvenuto.

Il reinvio dello stesso evento può generare un'altra notifica.
La regola non garantisce un solo messaggio per event_id tra diversi invii.

