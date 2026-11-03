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
```
* **Autoflush:** Il parametro `true` nel `PrintWriter` attiva l'autoflush; senza di esso, `println` accumula i dati nel buffer e il client resta bloccato in attesa.
* **Controllo chiusura:** `readLine()` restituisce `null` se il client chiude bruscamente la connessione. Gestisci sempre questo caso.
* **Protocollo a righe:** Usa sempre `println` e `readLine` (mai `print` isolati).
* *Test rapido:* Puoi testare il server da terminale con il comando `nc localhost 3000`.

### Passo 2: Più client, un thread ciascuno
Sostituisci la gestione sequenziale con un ciclo `while(true)` associato all'avvio di thread concorrenti:

```java
while (true) {
    Socket socket = ss.accept();
    new Thread(new MioThread(socket, disponibilita, usernames)).start();
}
```
* **Attenzione:** `accept()` è bloccante. Il flusso del `Main` si ferma lì finché non si connette un nuovo client. Dopo aver avviato il thread, il main torna immediatamente in ascolto. 

### Passo 3: Chiusura corretta di socket e stream
All'interno del metodo `run()`, le operazioni di I/O lanciano `IOException` (che l'interfaccia `Runnable` non dichiara). La soluzione pulita prevede l'utilizzo di **try-with-resources**, `catch` e `finally`:

```java
@Override
public void run() {
    try (Socket s = mySocket;
         BufferedReader in = new BufferedReader(new InputStreamReader(s.getInputStream()));
         PrintWriter out = new PrintWriter(s.getOutputStream(), true)) {

        out.println("WELCOME");
        if (login(in, out)) {
            comandi(in, out);
        }
    } catch (IOException e) {
        System.out.println("Errore di connessione: " + e.getMessage());
    } finally {
        // Operazioni di pulizia (es. rimozione dello username dal Set condiviso)
    }
}
```
Il blocco `finally` viene eseguito sempre (sia dopo un comando `QUIT` pulito sia in caso di disconnessione anomala), risultando ideale per rilasciare risorse come lo username.

### Passo 4: Gestione del login
Richiedi l'autenticazione ripetendo la lettura finché non arriva un comando `LOGIN` valido:

```java
private boolean login(BufferedReader in, PrintWriter out) throws IOException {
    String linea;
    while ((linea = in.readLine()) != null) {
        String[] parti = linea.trim().split(" ", 2);
        if (!parti[0].equals("LOGIN")) {
            out.println("ERR LOGINREQUIRED");
        } else if (parti.length != 2 || parti[1].contains(" ")) {
            out.println("ERR SYNTAX");
        } else {
            boolean libero;
            synchronized (usernames) {
                libero = usernames.add(parti[1]);
            }
            if (libero) {
                username = parti[1];
                out.println("OK");
                return true;
            }
            out.println("ERR USERINUSE");
        }
    }
    return false; // Il client ha chiuso la connessione
}
```
* **Nota su `Set.add()`:** Restituisce `false` se l'elemento è già presente, rendendo il controllo e l'inserimento atomici. Va comunque protetto da `synchronized` poiché `HashSet` non è thread-safe.

### Passo 5: Il ciclo dei comandi
Gestisci lo smistamento dei comandi validi post-login:

```java
private void comandi(BufferedReader in, PrintWriter out) throws IOException {
    String comando;
    while ((comando = in.readLine()) != null) {
        switch (comando.trim()) {
            case "N":
                out.println(riepilogo());
                break;
            case "BUY": {
                String richiesta = in.readLine(); // Seconda riga: <Tipo> <Quantita>
                if (richiesta == null) return;
                out.println(acquista(richiesta));
                break;
            }
            case "QUIT":
                out.println("BYE");
                return;
            default:
                out.println("ERR UNKNOWNCMD");
        }
    }
}
```
* Usa `split(" ", 2)` per separare il comando dagli argomenti ed effettua sempre controlli sulla lunghezza dell'array prima di accedere a `parti[1]`.
* Per i comandi strutturati su più righe (`BUY`), leggi la riga aggiuntiva direttamente all'interno del rispettivo `case`.
* Isola la logica di business in metodi dedicati (`riepilogo()`, `acquista(String)`) per mantenere il codice pulito e leggibile.

---

## 5. Stato condiviso e concorrenza (`synchronized`)
Il rischio principale nei server multithread è la **race condition**, che si verifica quando due thread eseguono simultaneamente una sequenza di *lettura, decisione e scrittura*.

* **Versione ERRATA (non thread-safe):**
  ```java
  int disp = disponibilita.get(tipo);
  if (quantita <= disp) {
      // Un altro thread potrebbe modificare disponibilita esattamente in questo istante!
      disponibilita.put(tipo, disp - quantita);
  }
  ```

* **Versione CORRETTA:**
  Racchiudi l'intera transazione logica in un unico blocco sincronizzato sullo stesso oggetto di stato:
  ```java
  synchronized (disponibilita) {
      Integer disp = disponibilita.get(tipo);
      if (disp == null) return "ERR UNKNOWNTYPE";
      if (quantita > disp) return "KO";
      disponibilita.put(tipo, disp - quantita);
      return "OK";
  }
  ```

> **Regole per la sincronizzazione:**
> * Ogni accesso allo stato condiviso (inclusa la sola lettura per il comando `N`) deve essere protetto da `synchronized` sullo stesso oggetto monitor.
> * Il blocco deve racchiudere sia il controllo che la modifica.
> * Mantieni il blocco il più breve possibile: **mai** inserire operazioni di I/O lento (come `in.readLine()`) dentro un blocco `synchronized`.

---

## 6. Gestione degli errori e robustezza
Definisci un ordine di validazione coerente (es. per `BUY`: sintassi $\rightarrow$ tipo $\rightarrow$ quantità $\rightarrow$ disponibilità).

* **Prevenire eccezioni fatali:**
  * `map.get(tipo)` restituisce `null` se il tipo non esiste: controllarlo prima di qualsiasi operazione numerica evita `NullPointerException`.
  * `Integer.parseInt()` su input non numerici lancia `NumberFormatException`: catturala e rispondi con `ERR SYNTAX`.
  * Rifiuta quantità minori o uguali a zero (quantità negative incrementerebbero impropriamente lo stock).
  * Gestisci sempre i casi in cui `readLine()` restituisce `null` (disconnessione inaspettata).

> 💡 **Principio fondamentale:** Il server non deve mai andare in crash a causa di un input errato o malevolo inviato dal client.

---

## 7. Testing e scenari di verifica
Prima di considerare concluso l'esercizio, testa il server manualmente utilizzando `nc` (Netcat) o `telnet`:
1. Esegui lo scambio completo descritto nella traccia, riga per riga.
2. Invia un comando prima di effettuare il login (`ERR LOGINREQUIRED`).
3. Effettua un comando `LOGIN` senza specificare lo username (`ERR SYNTAX`).
4. Connettiti da due terminali diversi usando lo stesso username (`ERR USERINUSE`); verifica che, dopo il `QUIT` del primo, il nome torni disponibile.
5. Invia comandi `BUY` con categorie inesistenti, quantità non numeriche, quantità eccedenti o righe mancanti.
6. Invia un comando sconosciuto (`ERR UNKNOWNCMD`).
7. Simula una chiusura brusca del client (es. `Ctrl + C` su Netcat): il server deve continuare a funzionare regolarmente.

* **Test di concorrenza avanzato:** Scrivi un piccolo client di test in Java che avvia 50 thread concorrenti, ognuno dei quali tenta di acquistare biglietti. Verifica che la somma finale dei biglietti venduti non superi mai lo stock iniziale.

---

## 8. Estensioni e funzionalità avanzate (Bonus)
Se la traccia prevede requisiti extra o punti bonus:
* **Notifica Sold-Out:** Mantieni una lista sincronizzata dei `PrintWriter` di tutti i client connessi. Quando una categoria di biglietti si esaurisce, invia a tutti un messaggio di broadcast (es. `NOTIFY SoldOut Gold`) e rimuovi il writer corrispondente nel blocco `finally`. Effettua l'invio della notifica *fuori* dal blocco sincronizzato sullo stock.
* **Client CLI (interfaccia a riga di comando):** Richiede l'uso di due thread distinti sul client: uno dedicato alla lettura continua dei messaggi provenienti dal server, e un altro per la lettura dell'input da tastiera. Con un solo thread perderesti le notifiche asincrone in arrivo mentre il programma è in attesa di input.
* **Persistenza:** Salva lo stato aggiornato dello stock su file dopo ogni transazione andata a buon fine (sempre all'interno del blocco di sincronizzazione).

---

## 9. Errori più comuni da evitare
Controlla il tuo codice verificando di non essere incappato in questi errori classici:
* ❌ Dimenticare di avviare i thread e usare una sola chiamata a `accept()`.
* ❌ Creare lo stato condiviso all'interno del thread anziché nel `Main`.
* ❌ Invertire le condizioni di validazione nel login (es. `length != 2` al posto di `== 2`).
* ❌ Inviare `OK` nel punto sbagliato o accodarlo a messaggi d'errore.
* ❌ Stampare direttamente la mappa (`{Gold=10, ...}`) invece di formattare l'output come richiesto dalla traccia.
* ❌ Omettere i controlli sui valori `null`.
* ❌ Separare il controllo dallo stock e la sua modifica in blocchi non sincronizzati.
* ❌ Dimenticare di rimuovere lo username dal `Set` nel blocco `finally`.
* ❌ Dimenticare l'autoflush nel `PrintWriter` o usare `print` anziché `println`.
* ❌ Utilizzare messaggi d'errore differenti anche solo per un carattere rispetto a quelli imposti dalla traccia.

---

## 10. Checklist prima di consegnare
- [ ] Il server rispetta rigorosamente riga per riga l'esempio della traccia.
- [ ] Ogni codice di errore richiesto produce esattamente la stringa attesa.
- [ ] Più client concorrenti funzionano correttamente e gli acquisti non superano mai lo stock disponibile.
- [ ] Nessun input anomalo fa crashare il server.
- [ ] Il codice è suddiviso in metodi con nomi chiari e commenti essenziali ma significativi.
- [ ] È stato eseguito un test reale tramite terminale (`nc` o `telnet`) con almeno due client connessi contemporaneamente.

---

## 11. Scheletro riutilizzabile (Boilerplate)
Puoi usare questa struttura di base come punto di partenza per qualsiasi esercizio client/server testuale:

```java
import java.io.*;
import java.net.*;
import java.util.*;

public class Main {
    public static void main(String[] args) throws IOException {
        Map<String, Integer> stato = new LinkedHashMap<>();
        // TODO: Inizializzazione dei valori dello stato condiviso
        Set<String> usernames = new HashSet<>();

        try (ServerSocket ss = new ServerSocket(3000)) {
            System.out.println("Server avviato sulla porta 3000...");
            while (true) {
                Socket s = ss.accept();
                new Thread(new Handler(s, stato, usernames)).start();
            }
        }
    }
}

class Handler implements Runnable {
    private final Socket socket;
    private final Map<String, Integer> stato;
    private final Set<String> usernames;
    private String username;

    public Handler(Socket socket, Map<String, Integer> stato, Set<String> usernames) {
        this.socket = socket;
        this.stato = stato;
        this.usernames = usernames;
    }

    @Override
    public void run() {
        try (Socket s = socket;
             BufferedReader in = new BufferedReader(new InputStreamReader(s.getInputStream()));
             PrintWriter out = new PrintWriter(s.getOutputStream(), true)) {
            
            out.println("WELCOME");
            if (login(in, out)) {
                comandi(in, out);
            }
        } catch (IOException e) {
            System.out.println("Errore di connessione: " + e.getMessage());
        } finally {
            if (username != null) {
                synchronized (usernames) {
                    usernames.remove(username);
                }
            }
        }
    }

    private boolean login(BufferedReader in, PrintWriter out) throws IOException {
        // TODO: Implementare la logica di login
        return false;
    }

    private void comandi(BufferedReader in, PrintWriter out) throws IOException {
        // TODO: Implementare il ciclo di gestione dei comandi
    }
}
```
