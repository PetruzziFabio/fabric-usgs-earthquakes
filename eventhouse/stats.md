## Statistiche della tabella Earthquakes

- Righe totali nella tabella, inclusi reinvii e revisioni: 2257
- Eventi sismici distinti, mantenendo l'ultima versione: 1845

## Intervallo di ingestione

- Prima ingestione: 2026-10-29 11:05:58.9975035
- Ultima ingestione: 2026-10-30 17:21:17.2230523

## Caricamento iniziale

È stato eseguito un caricamento iniziale dal feed USGS all_week.geojson.
Il producer ha completato l'invio di 1845 eventi attraverso il percorso:

Python → Custom endpoint → Eventstream → Eventhouse.

Il caricamento settimanale si sovrappone a quello giornaliero iniziale.
Per le analisi dello stato corrente degli eventi utilizziamo
arg_max(updated_time, *) by event_id.

Il file sample_data.csv contiene 505 eventi selezionati dopo
la deduplicazione per event_id.