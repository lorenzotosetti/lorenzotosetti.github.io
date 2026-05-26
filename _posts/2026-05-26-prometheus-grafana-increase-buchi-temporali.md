# Prometheus & Grafana — attenzione ai “buchi” temporali con `increase()`

Breve post su Prometheus/Grafana riguardo ad un problema con cui mi sono scontrato ultimamente.

---

## Il problema

Per estrarre i numeri normalmente tramite PromQL nelle dashboard di Grafana utilizziamo la funzione `increase()`:

```promql
sum by(client_name) (
  increase(
    files_input_counter{
      client_name=~"$ClientName",
      ...tags vari...
    }[$__range]
  )
)
```

Il parametro `__$range` ci permette di ottenere il delta tra il valore assunto dal contatore ad inizio e fine intervallo temporale.

---

## Il limite di `increase()`

La funzione `increase()`, pur gestendo i **counter resets**, ha un grosso limite (come molte altre funzioni PromQL):

- ha bisogno di **almeno 2 campioni**
- non funziona correttamente in presenza di **buchi temporali**

Quando parlo di *buchi* su Prometheus, intendo situazioni in cui Prometheus ha eseguito almeno uno scrape mentre il microservizio non esponeva alcuna metrica.

Questo può capitare, ad esempio:

- dopo un riavvio del microservizio
- durante un deploy Kubernetes
- durante una fase di startup in cui le metriche non sono ancora disponibili

---

## Cosa succede nel concreto?

Nel nostro caso ci siamo accorti che i conteggi su Grafana erano sempre corretti…

**tranne dopo un redeploy del microservizio.**

Il motivo era proprio il “buco” temporale privo di dati.

Questo comportamento è spesso accettabile in sistemi *always-on* con traffico continuo, ma diventa molto problematico in sistemi batch/scheduler-based.

Nel nostro caso:

- il sistema può rimanere fermo per minuti
- viene attivato tramite scheduler Spring
- può processare centinaia di file in pochi secondi

---

## Esempio reale

Supponiamo che:

- prima del deploy non esista alcun campione
- dopo il restart il microservizio processi subito **200 file**
- Prometheus veda soltanto:

```text
files_input_counter = 200
```

In questo scenario `increase()` può restituire:

```text
0
```

Perché?

Perché manca il campione precedente necessario per calcolare il delta.

---

# Possibili soluzioni

Esistono principalmente due approcci.

---

# Soluzione 1 — inizializzare le serie temporali a zero

La soluzione che ho adottato consiste nel:

1. inizializzare a zero tutte le possibili serie temporali all’avvio di Spring Boot
2. attendere abbastanza tempo affinché Prometheus effettui almeno uno scrape
3. soltanto dopo iniziare il vero processing

In questo modo esisteranno già due campioni:

- il primo a zero
- il secondo con il valore reale

e `increase()` potrà funzionare correttamente fin dal primo incremento utile.

---

## Nel mio caso

Ho introdotto un `initialDelay` nello scheduler Spring:

```java
@Service
public class FileSchedulerService {

    @Scheduled(
        initialDelayString =
            "${mio-microservizio.sftp.scheduler.initial-delay}"
    )
    public void schedulerInputDirectory() {

    }
}
```

Nel mio caso un delay di circa **60 secondi** si è dimostrato sufficiente.

---

# Quando questa soluzione funziona bene?

Funziona bene quando conosciamo a priori tutti i possibili valori delle labels.

Ad esempio:

- tag `client_name`
  - mario
  - luca

- tag `file_extension`
  - `.txt`
  - `.csv`

Possiamo inizializzare le metriche così:

```text
files_input_counter{client_name="mario",file_extension=".txt"} 0
files_input_counter{client_name="mario",file_extension=".csv"} 0
files_input_counter{client_name="luca",file_extension=".txt"} 0
files_input_counter{client_name="luca",file_extension=".csv"} 0
```

---

## Limite di questo approccio

Questa soluzione NON è applicabile quando:

- non conosciamo in anticipo i valori delle labels
- le combinazioni possibili sono dinamiche
- le cardinalità sono elevate

In questi casi consiglio il secondo approccio.

---

# Soluzione 2 — usare Prometheus Pushgateway

L’alternativa consiste nell’utilizzare un:

# Prometheus Pushgateway

Invece di esporre le metriche tramite l’endpoint management del microservizio:

- il microservizio esegue una **push**
- i contatori vengono salvati nel Pushgateway
- Prometheus scraperà sempre il Pushgateway

---

## Vantaggi

Con questo approccio:

- il restart del microservizio non azzera nulla
- non esistono più buchi temporali
- i contatori restano persistenti
- `increase()` continua a funzionare correttamente

Anche se il microservizio è momentaneamente fermo:

- il valore rimane stabile
- ma non scompare mai

---

# Quando Pushgateway è particolarmente adatto?

Il pattern Pushgateway è molto usato quando il deploy avviene tramite:

- Kubernetes Jobs
- batch temporanei
- processi short-lived

ovvero servizi che:

1. si avviano
2. eseguono il job
3. terminano

In questo scenario il concetto di “push finale” verso Pushgateway è perfettamente naturale.

---

# Considerazione finale

Probabilmente il nostro microservizio batch avrebbe potuto essere progettato fin dall’inizio come Kubernetes Job.

Tuttavia l’architettura era già consolidata come servizio always-on, quindi ho preferito applicare la soluzione 1:

- inizializzazione delle serie temporali
- delay iniziale dello scheduler
- mantenimento del microservizio sempre attivo

e nel nostro caso il risultato si è dimostrato affidabile e stabile.
