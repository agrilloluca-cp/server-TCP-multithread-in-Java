# Guida Passo-Passo: Server TCP Multithread in Java
*Metodo riutilizzabile per esercizi client/server testuali (ispirato all'esercizio "Vendita Biglietti")*

---

## Indice
1. [Che tipo di esercizio è](#1-che-tipo-di-esercizio-è)
2. [Prima di scrivere codice: schematizza la traccia](#2-prima-di-scrivere-codice-schematizza-la-traccia)
3. [Architettura dell'applicazione](#3-architettura-dellapplicazione)
4. [Implementazione Passo-Passo](#4-implementazione-passo-passo)
   - [Passo 1: Un server che risponde a un client](#passo-1-un-server-che-risponde-a-un-client)
   - [Passo 2: Più client, un thread ciascuno](#passo-2-più-client-un-thread-ciascuno)
   - [Passo 3: Chiusura corretta di socket e stream](#passo-3-chiusura-corretta-di-socket-e-stream)
   - [Passo 4: Gestione del login](#passo-4-gestione-del-login)
   - [Passo 5: Il ciclo dei comandi](#passo-5-il-ciclo-dei-comandi)
5. [Stato condiviso e concorrenza (`synchronized`)](#5-stato-condiviso-e-concorrenza-synchronized)
6. [Gestione degli errori e robustezza](#6-gestione-degli-errori-e-robustezza)
7. [Testing e scenari di verifica](#7-testing-e-scenari-di-verifica)
8. [Estensioni e funzionalità avanzate (Bonus)](#8-estensioni-e-funzionalità-avanzate-bonus)
9. [Errori più comuni da evitare](#9-errori-più-comuni-da-evitare)
10. [Checklist prima di consegnare](#10-checklist-prima-di-consegnare)
11. [Scheletro riutilizzabile (Boilerplate)](#11-scheletro-riutilizzabile-boilerplate)

---

## 1. Che tipo di esercizio è
Gli esercizi di questa famiglia richiedono quasi sempre le stesse quattro caratteristiche fondamentali:
* **Un protocollo testuale a righe**, con comandi e risposte definiti rigorosamente dalla traccia.
* **Un server multithread**, capace di gestire più client contemporaneamente (un thread dedicato per ogni connessione).
* **Uno stato condiviso in memoria** (es. i biglietti rimasti), letto e modificato concorrentemente da tutti i thread.
* **Una gestione degli errori precisa**, con i messaggi testuali esatti indicati nella specifica.

> **Criteri di valutazione tipici (in ordine di importanza):**
> 1. Rispetto rigoroso del protocollo
> 2. Assenza di *race condition* (sincronizzazione corretta)
> 3. Qualità e pulizia del codice
> 4. Eventuali estensioni

**Consiglio:** Procedi sempre per gradi. Prima fai funzionare il server con un singolo client, poi estendilo a molti client, e infine rifinisci la gestione degli errori.

---

## 2. Prima di scrivere codice: schematizza la traccia
Dedica almeno 10 minuti iniziali con carta e penna per estrarre le informazioni chiave:
1. **Porta e ruoli:** Chi apre la connessione e chi parla per primo (es. il server invia subito `WELCOME`).
2. **Stato condiviso:** Cosa deve ricordare il server e quali sono i valori iniziali (es. *Gold 10, Pit 30, Parterre 60, totale 100*).
3. **Comandi:** Per ognuno, annota cosa invia il client e cosa risponde il server. *Attenzione ai comandi su più righe* (es. `BUY` seguito da una seconda riga con tipo e quantità).
4. **Errori:** Per ogni situazione anomala, annota il messaggio esatto (es. `ERR SYNTAX`, `ERR UNKNOWNCMD`, `ERR LOGINREQUIRED`, `ERR UNKNOWNTYPE`, `ERR USERINUSE`).
5. **Contraddizioni:** Se la traccia presenta incongruenze (es. `KO` vs `ERR UNKNOWNTYPE` per un tipo inesistente), scegli una versione, documentala in un commento e, se possibile, chiedi chiarimenti al docente.
6. **Esempio di scambio:** Copia l'esempio di interazione fornito dalla traccia: diventerà il tuo primo test d'integrazione.

---

## 3. Architettura dell'applicazione
L'applicazione si compone tipicamente di tre macro-componenti:
* **`Main`**: Inizializza lo stato condiviso, apre il `ServerSocket` e avvia un ciclo infinito che chiama `accept()` e lancia un nuovo thread per ogni client connesso.
* **Thread del client (`Runnable`)**: Gestisce il ciclo di vita della singola connessione (socket dedicato, riferimenti allo stato condiviso, autenticazione/login, ciclo dei comandi e gestione errori).
* **Stato condiviso**: Una struttura dati centralizzata (es. `LinkedHashMap` per preservare l'ordine delle categorie e un `HashSet` per gli username attivi). Si crea una **sola volta** nel `Main` e viene passata ai thread tramite costruttore.

> ⚠️ **Regola d'oro:** Lo stato condiviso **non** va mai istanziato dentro il thread, altrimenti ogni client finirebbe per avere il proprio stock privato e isolato.

---

## 4. Implementazione Passo-Passo

### Passo 1: Un server che risponde a un client
Verifica che socket e flussi di I/O funzionino correttamente prima di aggiungere logica complessa:

```java
try (ServerSocket ss = new ServerSocket(3000)) {
    Socket s = ss.accept();
    BufferedReader in = new BufferedReader(new InputStreamReader(s.getInputStream()));
    PrintWriter out = new PrintWriter(s.getOutputStream(), true);
    
    out.println("WELCOME");
    String riga = in.readLine();
    out.println("ECHO " + riga);
}
