Real-Time Intelligence — USGS Earthquakes
1. Autore
Fabio Petruzzi
2. Obiettivo
Realizzare una pipeline di analisi degli eventi sismici con Microsoft Fabric: acquisizione da USGS, invio JSON tramite producer Python, ingestione e trasformazione in Eventstream, archiviazione in Eventhouse, analisi KQL, visualizzazione e notifiche.
3. Fonte dati e struttura dell'evento
Fonte: USGS Earthquakes, feed GeoJSON.
Endpoint utilizzati dal notebook:
•	Monitoraggio e test: https://earthquake.usgs.gov/earthquakes/feed/v1.0/summary/all_day.geojson
•	Caricamento iniziale: https://earthquake.usgs.gov/earthquakes/feed/v1.0/summary/all_week.geojson
Il feed giornaliero permette di acquisire gli eventi presenti nella finestra giornaliera; quello settimanale viene utilizzato per popolare inizialmente le analisi. L'invio di un evento storico non equivale al verificarsi di un nuovo terremoto.
Ogni feature GeoJSON viene trasformata in un evento JSON piatto:
Campo	Tipo nel payload	Significato
event_id	string	Identificativo USGS
event_time	string ISO 8601 UTC	Istante del sisma
updated_time	string ISO 8601 UTC	Ultima revisione della fonte
magnitude	number o null	Magnitudo
mag_type	string o null	Tipo di magnitudo
place	string o null	Descrizione geografica
latitude	number	Latitudine
longitude	number	Longitudine
depth_km	number	Profondità in km
area	string	Raggruppamento geografico del producer
event_type	string	Tipo di evento
status	string	Stato della revisione
network	string	Rete di rilevamento
significance	integer	Indicatore USGS di significatività
tsunami	integer	Indicatore USGS, non un allarme autonomo del progetto
ingest_time	string ISO 8601 UTC	Istante di costruzione del payload nel producer

area è calcolata mediante fasce di longitudine: americas per -170 ≤ lon < -30; europe_africa per -30 ≤ lon < 60; asia_pacific negli altri casi. È una classificazione semplificata, non un'identificazione geografica puntuale.
Esempio illustrativo di struttura, non prova di un sisma reale:
{
  "event_id": "example_event",
  "event_time": "2026-10-30T12:00:00+00:00",
  "updated_time": "2026-10-30T12:05:00+00:00",
  "magnitude": 4.5,
  "mag_type": "mb",
  "place": "Località di esempio",
  "latitude": 42.0,
  "longitude": 13.0,
  "depth_km": 10.0,
  "area": "europe_africa",
  "event_type": "earthquake",
  "status": "reviewed",
  "network": "example",
  "significance": 300,
  "tsunami": 0,
  "ingest_time": "2026-10-30T12:06:00+00:00"
}


4. Architettura
Workspace:  Esercitazione Real-Time
Schema logico; 
flowchart LR
    U[USGS GeoJSON] --> P[USGS-Earthquakes.ipynb]
    P --> C[Custom endpoint]
    C --> E[Eventstream: esUSGSEarthquakes]
    E --> T[Trasformazioni: dettaglio in eventstream/README.md]
    T --> H[Eventhouse / sorgente EH_Earthquakes]
    H --> K[Tabella Earthquakes]
    K --> Q[Query KQL]
    Q --> D[Real-Time Dashboard: Earthquakes_Dashboard]
    H --> M[Fabric Map: EarthquakeMap]
    T --> A[ActivatorEarthquakes: AlertsEarthquakes ]
    A --> N[Notifica messaggio su Teams]

Database KQL: EH_Earthquakes
La topologia effettiva, inclusi eventuali rami separati per gli alert, è documentata nello screenshot eventstream/topologia.png. 
5. Esecuzione del producer
Prerequisiti
•	Python; versione registrata nel notebook: 3.11.9.
•	Un ambiente capace di eseguire notebook Jupyter, ad esempio VS Code con supporto notebook o Jupyter già installato.
•	Accesso alla fonte USGS e al Custom endpoint configurato in Fabric.
•	Eventstream pubblicato e destinazioni configurate.
Il file consegnato è un notebook .ipynb 


Installazione e configurazione
Dalla cartella producer/:
python -m pip install -r requirements.txt

Dipendenze dirette e versioni registrate negli output del notebook:
azure-eventhub==5.15.1
requests==2.34.2
python-dotenv==1.2.4

Creare localmente un file .env nella cartella del notebook, copiando .env.example, e valorizzare:
FABRIC_CUSTOM_ENDPOINT_CONNECTION_STRING=

Il valore è la connection string del Custom endpoint. 
Parametri
Parametro	Valore nel notebook	Uso
FEED_URL	all_day.geojson	Feed iniziale; alcune celle lo modificano
DURATA_SEC	600	Finestra del ciclo di monitoraggio
PAUSA	60	Pausa tra i cicli, in secondi
MAX_EVENTI_SEC	10	Ritmo medio obiettivo dell'invio
MAX_RETRY	5	Numero massimo di tentativi per operazione
BATCH_SIZE	20	Numero di eventi per blocco applicativo

Il producer invia blocchi e poi attende len(blocco) / MAX_EVENTI_SEC: il valore 10 è un obiettivo medio, non un limite rigido su ogni singolo secondo. Chiamate di rete, retry e pause possono prolungare la durata effettiva.
Ordine delle celle e modalità operative
Non usare indiscriminatamente “Run All”: il notebook contiene tre modalità che possono reinviare gli stessi dati. 

Preparazione
1.	Installare le dipendenze tramite requirements oppure con la prima cella del notebook.
2.	Eseguire la cella degli import, della connessione e dei parametri.
3.	Eseguire la cella con iso_utc, get_terremoti, area_di e to_dict.
4.	Eseguire la cella che genera sample_events.json, dopo la definizione delle funzioni.
5.	Eseguire la cella che crea producer e definisce invia_batch.
A. Monitoraggio continuo:
•	Impostare FEED_URL al feed giornaliero, soprattutto se prima è stato usato quello settimanale.
•	Eseguire la cella che inizia con visti, totale, fine, ciclo = ....
•	Il primo giro acquisisce gli eventi del feed; i successivi selezionano versioni non ancora viste nella stessa esecuzione usando (event_id, updated_time).
•	Al termine, o dopo interruzione, il producer viene chiuso. Per ripetere il monitoraggio, ricrearlo prima di rieseguire il ciclo.
B. Caricamento iniziale settimanale:
•	Eseguire la cella “Caricamento iniziale: eventi dell'ultima settimana”.
•	Eseguire la successiva cella di invio di eventi_settimana, che ricrea e chiude il producer.
•	Vengono esclusi dalla lista gli eventi con magnitudo nulla.
C. Test giornaliero:
•	Eseguire la cella “Test completo con dati USGS reali dell'ultimo giorno”. Questa rilegge il feed, filtra le magnitudo nulle, deduplica la lista e mostra quanti eventi hanno magnitudo > 5.
•	Eseguire la cella finale di invio di eventi_test, che ricrea e chiude il producer.
•	Rieseguire soltanto la cella finale reinvia la lista già in memoria: non aggiorna i dati dalla fonte.
6. Trasformazioni, storage e analisi
Il dettaglio delle trasformazioni realmente pubblicate deve essere riportato in eventstream/README.md nella tabella Nodo | Tipo | Configurazione.. Il codice del notebook richiama un filtro Eventstream sulla magnitudo non nulla; i restanti operatori devono essere verificati sul canvas.
Lo schema e i tipi effettivi della tabella KQL sono documentati in eventhouse/schema.kql. I timestamp sono stringhe ISO 8601 nel payload e devono essere gestiti con tipi coerenti nello storage e nelle query.
Le domande di audit, le query e i risultati effettivamente realizzati si trovano in queries/queryset.kql e queries/results/. 
Distinzione temporale:
•	event_time: quando è avvenuto il sisma.
•	updated_time: quando la fonte ha revisionato l'evento.
•	ingest_time: quando il producer ha costruito il payload; non viene ricalcolato reinviando una lista già pronta.

7. Dashboard e Activator
Dashboard: Earthquakes_Dashboard
Configurazione dell'alert discussa durante il progetto, da confrontare con la definizione effettiva:
•	Item: ActivatorEarthquakes.
•	Regola: USGS_Evento_Maggiore5.
•	Condizione: magnitude > 5.
•	Azione: massaggio su Teams.
8. Estensione: Fabric Map
Mappa con sorgente EH_Earthquakes e funzione EarthquakesMapTest, con un layer di test per le ricezioni recenti e EarthquakesMap con le ricezioni dell’ultima settimana.
•	Nome reale della mappa: EarthquakeMap
•	Layer e finestra temporale effettivi: Epicentri magnitudo > 4 in last week / Test – Ricezioni ultimi 2 minuti in last day
Una finestra di ricezione breve può restituire zero righe quando il producer non sta inviando: questo non dimostra da solo un guasto della pipeline.

9. Prove registrate e limiti
Negli output salvati nel notebook risultano:
•	Monitoraggio: 205 eventi al primo ciclo, 1 al secondo e 0 al terzo.
•	Caricamento iniziale: 1.845 eventi inviati su 1.845.
•	Lista di test giornaliera: 199 eventi, di cui 12 con magnitudo > 5.
•	Invio della lista di test: 199 eventi inviati su 199; inizio stampato 2026-10-30T16:50:51.772923+00:00.
Limiti della versione consegnata:
•	La deduplicazione del monitoraggio è in memoria e si azzera a ogni riavvio della relativa cella.
•	Nel ciclo continuo le chiavi vengono marcate come viste prima della conferma di invio: se tutti i retry falliscono, gli eventi possono non essere riprovati nei cicli successivi della stessa esecuzione.
•	Un retry dopo un invio parziale può produrre duplicati; la gestione delle versioni nelle query resta necessaria.
•	Dopo l'esaurimento dei retry sul feed, get_terremoti() restituisce una lista vuota: controllare i log per distinguere l'assenza di novità dagli errori di lettura.
•	I caricamenti e i test sono replay di dati reali, non una simulazione di terremoti nuovi.
10. Stato dei requisiti e allegati

Requisito	Stato	Dove si trova / verifica restante
Producer JSON, timestamp UTC, parametri, batch e retry	Fatto	producer/USGS-Earthquakes.ipynb; limiti nella sezione 9
Dipendenze e istruzioni di avvio	Fatto	producer/requirements.txt ; 
Segreti esterni al codice	Fatto	Il notebook legge una variabile d'ambiente; 
Cinque eventi JSON di esempio	Fatto	Generare e allegare producer/sample_events.json
Eventstream pubblicato e almeno due trasformazioni	Fatto	eventstream/topologia.png, README.md

Schema, statistiche e almeno 500 righe esportate	Fatto	eventhouse/schema.kql, stats.md, sample_data.csv

Almeno due query di audit e operatori richiesti	Fatto	queries/queryset.kql e queries/results/
		
Dashboard, mappa, parametri e auto-refresh	Fatto	dashboard/dashboard.json, full.png, filtered.png, tiles.md

Regola Activator avviata e testata	Fatto	activator/rule.png, alert.png, README.md

Estensione Fabric Map	Fatto	Dashboard/mappa.png
		

11. Problemi riscontrati e soluzioni
Problema	Soluzione o stato
	
Invio limitato a 199 eventi	La cella scorre una sola volta eventi_test: la quantità dipende dalla lista preparata, non da un limite di Event Hubs.
Reinvio di una lista precedente	Rieseguire la preparazione del test prima dell'invio se si vogliono rileggere i dati della fonte.
Producer già chiuso	Ricreare EventHubProducerClient prima di un nuovo monitoraggio. Le celle di caricamento e test lo ricreano già.
Layer delle ricezioni recenti vuoto	Verificare nuove ingestioni e finestra temporale. Un replay può aggiornare le ricezioni senza cambiare i terremoti distinti.
Mancata ricezione di ulteriori alert	Problema segnalato durante i test. Causa: limiti di invio.

12. Strumenti AI
È stato utilizzato Perplexity come supporto alla preparazione e revisione del producer, alla lettura degli errori, al troubleshooting di mappa.



13. Organizzazione della consegna
  

