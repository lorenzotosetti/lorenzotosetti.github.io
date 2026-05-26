# Problemi con `increase()` in Prometheus/Grafana e buchi temporali dopo i redeploy

Breve post su un problema con cui mi sono scontrato recentemente utilizzando **Prometheus**, **PromQL** e **Grafana**.

---

# Il problema

Per estrarre i numeri nelle dashboard Grafana, normalmente utilizziamo la funzione `increase()` di PromQL, che rappresenta il delta del contatore all'interno della finestra temporale selezionata.

## Esempio

```promql
sum by(client_name) (
  increase(
    files_input_counter(client_name=~"$ClientName", ....tags vari..}[$__range]
  )
)
```

La variabile `$__range` permette di ottenere il delta tra il valore assunto dal contatore a inizio e fine intervallo temporale.

---

# Attenzione ai "buchi temporali"

La funzione `increase()`, pur gestendo correttamente i **counter reset**, ha un grosso limite, come molte altre funzioni PromQL:

- ha bisogno di almeno **2 campioni**
- non funziona correttamente in presenza di **buchi temporali** all'interno della finestra di analisi

Quando parlo di "buchi", intendo situazioni in cui Prometheus ha effettuato uno scrape mentre il microservizio non esponeva alcuna metrica.

Questo può capitare, ad esempio:

- dopo un restart
- durante un redeploy Kubernetes
- in fase di startup del microservizio

---

# Il caso reale

Nel nostro caso i conteggi Grafana risultavano sempre corretti...

tranne durante i redeploy.

Il motivo era proprio il buco temporale privo di dati su Prometheus.

Questo problema è spesso trascurabile nei sistemi **always-on** con traffico continuo, ma nel nostro scenario era molto penalizzante.

## Perché?

Perché il nostro microservizio:

- resta inattivo anche per diversi minuti
- viene attivato tramite Spring Scheduler
- può processare moltissimi file in pochissimo tempo

Quindi può succedere questo scenario:

1. il microservizio viene riavviato
2. Prometheus esegue uno scrape durante il periodo "vuoto"
3. il servizio processa improvvisamente 200 file
4. Prometheus vede solo il valore finale `200`

Risultato:

```promql
increase(...)
```

restituisce `0`, perché manca il campione precedente necessario per il calcolo del delta.

---

# Soluzione 1 - Inizializzare le serie temporali all'avvio

La prima soluzione consiste nell'inizializzare a zero tutte le possibili serie temporali durante lo startup del microservizio.

## Concetto

Occorre:

1. creare i contatori a zero
2. attendere che Prometheus effettui almeno uno scrape
3. solo dopo iniziare il processing reale

In questo modo esisterà già un campione iniziale pari a `0`, permettendo a `increase()` di funzionare correttamente già dal primo incremento reale.

---

# Esempio Spring Scheduler

Nel mio caso ho introdotto semplicemente un delay iniziale di 60 secondi:

```java
@Service
public class FileSchedulerService {

    @Scheduled(
        initialDelayString = "${mio-microservizio.sftp.scheduler.initial-delay}"
    )
    public void schedulerInputDirectory() {

    }
}
```

Questa è la soluzione che ho adottato e si è dimostrata funzionare perfettamente.

---

# Limite della soluzione

Questa soluzione funziona bene solo se conosciamo a priori tutti i possibili valori delle labels.

## Esempio

Supponiamo di avere:

### Labels disponibili

- `client_name`
  - mario
  - luca

- `file_extension`
  - .txt
  - .csv

Possiamo quindi inizializzare le metriche così:

```text
files_input_counter{client_name="mario",file_extension=".txt"} 0.0
files_input_counter{client_name="mario",file_extension=".csv"} 0.0
files_input_counter{client_name="luca",file_extension=".txt"} 0.0
files_input_counter{client_name="luca",file_extension=".csv"} 0.0
```

---

# Quando questa soluzione NON è applicabile

Se non conosciamo in anticipo i valori delle labels, questa tecnica diventa poco praticabile.

Ad esempio:

- nomi clienti dinamici
- identificativi variabili
- labels generate runtime

In questi casi consiglio la seconda soluzione.

---

# Soluzione 2 - Utilizzare Pushgateway

L'alternativa consiste nell'utilizzare un **Prometheus Pushgateway**.

In questo scenario:

- il microservizio non espone direttamente le metriche
- i contatori vengono pushati verso Pushgateway
- Pushgateway mantiene i valori anche durante i restart del servizio

## Vantaggio principale

Quando il microservizio si riavvia:

- Pushgateway continua a mantenere le metriche precedenti
- Prometheus non vede buchi temporali
- il contatore resta fermo all'ultimo valore noto

Quindi `increase()` continua a funzionare correttamente.

---

# Quando Pushgateway è particolarmente indicato

La soluzione con Pushgateway è particolarmente adatta quando il microservizio è in realtà un:

- Kubernetes Job
- batch temporaneo
- processo schedulato che si avvia e termina

In questi casi il pattern corretto è:

1. il job parte
2. esegue il processing
3. invia le metriche tramite PUSH
4. termina

---

# Considerazione finale

Probabilmente il nostro microservizio avrebbe dovuto essere concepito come un vero job batch fin dall'inizio.

Tuttavia l'architettura era già stata definita, quindi ho preferito:

- mantenere il servizio always-on
- applicare la soluzione 1
- introdurre un delay iniziale per permettere a Prometheus di creare le serie temporali

Nel nostro caso questa soluzione si è dimostrata semplice, efficace e sufficiente.
