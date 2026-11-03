# server-TCP-multithread-in-Java
Guida passo passo: server TCP multithread in Java
Metodo riutilizzabile per esercizi client/server testuali, con esempi dall'esercizio Vendita Biglietti
1. Che tipo di esercizio è
Gli esercizi di questa famiglia chiedono quasi sempre le stesse quattro cose:
• un protocollo testuale a righe, con comandi e risposte decisi dalla traccia;
• un server che gestisce più client insieme, con un thread per ogni connessione;
• uno stato condiviso in memoria (qui i biglietti rimasti) che tutti i thread leggono e modificano;
• una gestione degli errori precisa, con i messaggi esatti della traccia.
Di solito la valutazione guarda, in quest'ordine: rispetto del protocollo, assenza di race condition, qualità del codice, estensioni. Conviene quindi procedere per gradi: prima un server che funziona con un client, poi con molti client, poi la rifinitura degli errori.
2. Prima di scrivere codice: schematizza la traccia
Dedica dieci minuti alla traccia, con carta e penna. Ricava:
1. Porta e ruoli: chi apre la connessione e chi parla per primo (qui il server manda WELCOME).
2. Stato condiviso: cosa deve ricordare il server e con quali valori iniziali (Gold 10, Pit 30, Parterre 60, totale 100).
3. Comandi: per ognuno scrivi cosa invia il client e cosa risponde il server. Attenzione ai comandi su più righe: BUY è seguito da una seconda riga con tipo e quantità.
4. Errori: per ogni situazione sbagliata, il messaggio esatto (ERR SYNTAX, ERR UNKNOWNCMD, ERR LOGINREQUIRED, ERR UNKNOWNTYPE, ERR USERINUSE).
5. Contraddizioni: se due punti della traccia dicono cose diverse (qui KO oppure ERR UNKNOWNTYPE per un tipo inesistente), scegli una versione, annotala in un commento e, se puoi, chiedi al docente.
Copia l'esempio di scambio della traccia: sarà il tuo primo test.
3. Architettura: tre pezzi
• Main: crea lo stato condiviso, apre il ServerSocket e, in un ciclo infinito, chiama accept() e avvia un thread per ogni client.
• Thread del client (classe Runnable): conosce il proprio socket e i riferimenti allo stato condiviso; contiene login, ciclo dei comandi e gestione errori.
• Stato condiviso: una Map per le disponibilità (usa LinkedHashMap per mantenere l'ordine Gold, Pit, Parterre) e un Set per gli username. Si crea una sola volta in Main e si passa ai thread nel costruttore.
Regola: lo stato non va mai creato dentro il thread, altrimenti ogni client avrebbe il proprio stock privato.
4. Passo 1: un server che risponde a un client
Obiettivo: verificare che socket e stream funzionino, prima di aggiungere il resto.
try (ServerSocket ss = new ServerSocket(3000)) {
    Socket s = ss.accept();
    BufferedReader in = new BufferedReader(new InputStreamReader(s.getInputStream()));
    PrintWriter out = new PrintWriter(s.getOutputStream(), true);
    out.println("WELCOME");
    String riga = in.readLine();
    out.println("ECHO " + riga);
}
Da ricordare:
• il true nel PrintWriter attiva l'autoflush: senza, println non spedisce subito e il client resta in attesa;
• readLine() restituisce null quando il client chiude la connessione: controllalo sempre;
• usa println e non print, perché il protocollo è a righe.
Prova con nc localhost 3000.
5. Passo 2: più client, un thread ciascuno
while (true) {
    Socket socket = ss.accept();
    new Thread(new MioThread(socket, disponibilita, usernames)).start();
}
accept() è bloccante: il ciclo si ferma lì finché arriva un client. Dopo aver avviato il thread, il main torna subito ad aspettare il successivo. Con una sola accept() il server serve un client e poi termina: è l'errore più tipico.
6. Passo 3: chiudere bene socket e stream
Dentro run() i metodi di I/O lanciano IOException, che run() non può dichiarare. Si risolve con try-with-resources, catch e finally:
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
        // qui si fanno le pulizie, ad esempio liberare lo username
    }
}
Il finally viene eseguito sia dopo QUIT sia con una disconnessione brusca, quindi è il posto giusto per rimuovere lo username.
7. Passo 4: il login
Schema: ripeti finché il client non manda un LOGIN valido.
• prima parola diversa da LOGIN: ERR LOGINREQUIRED;
• LOGIN senza username, o con troppi campi: ERR SYNTAX;
• username già in uso: ERR USERINUSE;
• altrimenti: inserisci lo username e rispondi OK.
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
    return false; // il client ha chiuso la connessione
}
Punti chiave: add() di un Set restituisce false se l'elemento esiste già, quindi controllo e inserimento sono un'unica operazione. Va comunque messa in un blocco synchronized, perché HashSet non è thread-safe. L'OK si invia solo nel ramo di successo.
8. Passo 5: il ciclo dei comandi
private void comandi(BufferedReader in, PrintWriter out) throws IOException {
    String comando;
    while ((comando = in.readLine()) != null) {
        switch (comando.trim()) {
            case "N":
                out.println(riepilogo());
                break;
            case "BUY": {
                String richiesta = in.readLine();   // seconda riga: <Tipo> <Quantita>
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
• split(" ", 2) divide al primo spazio: parti[0] è il comando, parti[1] il resto, se esiste. Controlla sempre parti.length prima di usare parti[1].
• Per i comandi su più righe (BUY) la seconda riga si legge dentro il case con un altro readLine().
• Metti la logica in metodi separati (riepilogo(), acquista(String)) che restituiscono la stringa di risposta: il codice resta leggibile, e questo conta per il criterio sulla qualità.
• Per il comando N costruisci la riga a mano (AVAIL Gold:10 Pit:30 Parterre:60): stampare direttamente la mappa darebbe {Gold=10, ...}, che è un formato diverso da quello richiesto.
9. Passo 6: stato condiviso e synchronized
È il punto in cui si perdono più punti. Il problema nasce quando due thread eseguono insieme la sequenza leggo, decido, scrivo.
Versione sbagliata:
int disp = disponibilita.get(tipo);
if (quantita <= disp) {
    // un altro thread può modificare disponibilita proprio qui
    disponibilita.put(tipo, disp - quantita);
}
Con 5 biglietti Gold rimasti, due client che ne chiedono 5 possono leggere entrambi 5 e vendere 10 biglietti.
Versione corretta: tutta la sequenza in un unico blocco, sullo stesso oggetto.
synchronized (disponibilita) {
    Integer disp = disponibilita.get(tipo);
    if (disp == null) return "ERR UNKNOWNTYPE";
    if (quantita > disp) return "KO";
    disponibilita.put(tipo, disp - quantita);
    return "OK";
}
Regole pratiche:
1. Ogni accesso allo stato condiviso, anche la sola lettura per il comando N, va in synchronized sullo stesso oggetto.
2. Il blocco deve contenere il controllo e la modifica, non solo la modifica.
3. Tieni il blocco corto: niente readLine() o altro I/O lento al suo interno. Costruisci la risposta dentro e inviala fuori.
4. Alternativa più ordinata: metti stato e operazioni in una classe a parte (per esempio Biglietteria con un metodo synchronized acquista(...)), così il lock è automatico e non puoi dimenticarlo.
10. Passo 7: gestione degli errori
Ordine consigliato dei controlli per BUY: sintassi, tipo, quantità, disponibilità.
Situazioni che fanno cadere il thread se non gestite:
• map.get(tipo) restituisce null per un tipo inesistente: appena lo confronti con un numero ottieni una NullPointerException;
• Integer.parseInt su testo non numerico lancia NumberFormatException: catturala e rispondi ERR SYNTAX;
• una quantità zero o negativa: una quantità negativa aumenterebbe lo stock, quindi va rifiutata;
• readLine() che restituisce null: il client è sparito, esci dal ciclo;
• parti[1] quando la riga ha una sola parola: ArrayIndexOutOfBoundsException, evitala controllando la lunghezza.
Principio: il server non deve mai cadere per colpa dell'input di un client.
11. Passo 8: testare
Test manuale con nc localhost 3000 (o telnet). Scenari minimi:
1. l'esempio completo della traccia, riga per riga;
2. un comando prima del login: ERR LOGINREQUIRED;
3. LOGIN senza nome: ERR SYNTAX;
4. lo stesso username da due terminali: ERR USERINUSE; dopo il QUIT del primo, il nome torna libero;
5. BUY con tipo sbagliato, quantità non numerica, quantità troppo alta, riga mancante;
6. un comando inesistente: ERR UNKNOWNCMD;
7. chiusura brusca di nc con Ctrl+C: il server deve continuare a funzionare.
Test di concorrenza: apri due terminali, porta un tipo a pochi biglietti e lancia due BUY quasi insieme. Per un test serio scrivi un piccolo client Java che avvia 50 thread, ognuno dei quali compra 1 biglietto: alla fine i biglietti venduti non devono mai superare quelli iniziali.
12. Passo 9: estensioni (bonus)
• Notifica sold-out: tieni un elenco sincronizzato dei PrintWriter dei client connessi. Quando una quantità arriva a 0, invia a tutti NOTIFY SoldOut Gold (con il tipo giusto) e rimuovi il writer dall'elenco nel finally. Invia la notifica fuori dal blocco che modifica lo stock.
• Client CLI: servono due thread, uno che legge dal server e stampa, l'altro che legge la tastiera e invia. Con un solo thread perdi le NOTIFY che arrivano mentre il programma aspetta l'input.
• Persistenza: salva lo stock su file dopo ogni acquisto riuscito, sempre dentro il lock.
13. Errori più comuni
Usa questa lista come revisione finale del tuo codice:
• una sola accept() e nessun thread;
• stato condiviso creato dentro il thread;
• condizioni invertite nel login (length != 2 al posto di == 2);
• OK inviato nel punto sbagliato, o inviato insieme a un messaggio di errore;
• mappa stampata direttamente invece del formato della traccia;
• nessun controllo su null (tipo inesistente, readLine());
• controllo e modifica non racchiusi in un unico synchronized;
• username mai rimosso a fine connessione;
• classi in package diversi, o IOException non gestita in run();
• PrintWriter senza autoflush, o print al posto di println;
• messaggi di errore diversi da quelli della traccia, anche di una sola lettera.
14. Prima di consegnare
• Il server rispetta riga per riga l'esempio della traccia.
• Ogni errore elencato nella traccia produce il messaggio esatto.
• Più client contemporanei funzionano e gli acquisti non superano lo stock.
• Nessun input strano (righe vuote, parole mancanti, numeri sbagliati) fa cadere il server.
• Il codice è diviso in metodi con nomi chiari e ha pochi commenti, ma utili.
• Hai provato davvero con nc e con almeno due terminali aperti.
15. Scheletro riutilizzabile
public class Main {
    public static void main(String[] args) throws IOException {
        Map<String, Integer> stato = new LinkedHashMap<>();
        // TODO: valori iniziali
        Set<String> usernames = new HashSet<>();

        try (ServerSocket ss = new ServerSocket(3000)) {
            while (true) {
                Socket s = ss.accept();
                new Thread(new Handler(s, stato, usernames)).start();
            }
        }
    }
}

public class Handler implements Runnable {
    private final Socket socket;
    private final Map<String, Integer> stato;
    private final Set<String> usernames;
    private String username;

    // costruttore: assegna i tre campi

    @Override
    public void run() {
        try (Socket s = socket;
             BufferedReader in = new BufferedReader(new InputStreamReader(s.getInputStream()));
             PrintWriter out = new PrintWriter(s.getOutputStream(), true)) {
            out.println("WELCOME");
            if (login(in, out)) comandi(in, out);
        } catch (IOException e) {
            System.out.println("Errore: " + e.getMessage());
        } finally {
            // TODO: libera lo username (synchronized)
        }
    }

    private boolean login(BufferedReader in, PrintWriter out) throws IOException { /* TODO */ return false; }
    private void comandi(BufferedReader in, PrintWriter out) throws IOException { /* TODO */ }
}
