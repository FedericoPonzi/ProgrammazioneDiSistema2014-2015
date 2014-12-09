#Ancora su IPC
Chi rende a disposizione un server -> servizio, rende disponibile una modalità d'accesso.
Quello che avviene è che il server crea la namedpipe e poi in qualche modo rimane in attesa che qualcuno si connetta alla PIPE. PEr farlo, si usa:

	Bool ConnectNamedPipe(

Ha una sorta di equivalente per l'altro lato:

	BOOL WaitNamedPipe(
	LPCTSTR lpNAmedPipeName,
	DWORD nTimeOut)

Viene usata dal client per sincronizzarsi col server. Il server crea la pipe, si mette in attesa con la connectnamedpipe, e il client si connette usando la wait named pipe.
La `WaitNamedPipe` ha una finestra temporale e funziona se la pipe è stata creata, anche se non è stata ancora invocato la connect named pipe. Se non è stata neanche creata la wait named pipe fallisce.

##Funzioni avanzate per le named pipe
Windows permette di usare le named pipe in un certo modo: anche quì ci sono funzioni di utilità per usare una pipe senza effettuare tutte le chiamate viste.
Ad esempio, si può utilizzare la `TransactNamedPipe` che è una sorta di transazione e unione fra writefile e readfile. La uso in caso in cui mando un messaggio e mi aspetto una risposta.
lpBytesRead -> quanti byte effettivamente letti.
Io mando una richiesta di una certa dimesione, mi aspetto una risposta di dimensione diversa, ma l' effettivo numero di byte ricevuti si trova in lpBytesRead

>Quando si legge da Pipe o Socket, è **fondamentale dimenticarsi** che è scontato leggere il numero di byte che effettivamente si ha richiesto.

Mentre nel caso dei file c'è una specie di precauzione, quando leggiamo da file l' unico problema potremmo trovarlo nell' ultima parte del file che leggiamo meno di quanto ci aspettiamo. Per le pipe possiamo ricevere in qualsiasi momento un numero di byte minore (mai maggiore) rispetto al numero di byte che mi aspetto.

Per ottenere una forma ancora più conmpatta può essere utilizzata la CallNamedppe. Questa funzione è sincrona mentre la precedente offre la possibilità di effettuare I/O asincrono (usando la struttura lpOverlapped).

La funzione `PeekNamedPipe` permette di leggere quello che c'è nella pipe (viene effettivamente letto) ma lo lascio nella pipe. E' come se io facessi una sorta di copia.
Ci sono un certo numero di byte nella pipe, li leggo (e quindi li ho nel buffer che passo come lpBuffer) però quei byte rimangono anche nella pipe.
Questo significa che se rifaccio una operazione di lettura, rileggo gli stessi byte.
Questa funzione può essere utile se voglio controllare se c'è il numero di byte che voglio oppure a leggere comunque una parte(quella già disponibile) perchè contiene informazioni utili su come gestire il payload.
Il numero totale di byte che si può leggere dalla pipe ci viene ritornato dal valore lpTotalBytesAvail passato come input alla funzione.
Passando un buffer di dimensione 0, non viene letto niente ma mi dice quanti byte son pronti nella lettura della pipe.

#Pipe in Unix/Linux

Si creano usando int pipe(int filedes[2]);
L' 1 è aperto in lettura il 2 in scrittura.
Normalmente se dobbiamo inviare i dati da A verso B, A chiude il filedes[0 ] e B chiude filedes[1].
La pipe è riconosciuta come un tipo di file speciale (FIFO).
L' associazione tra un estremo dalla pipe e lo standard input o output di un processo deve essere fatta dal processo stesso. La read da una pipe il cui estremo di scrittura è stato chiuso ritorna 0.
Se faccio una operazione di lettura e la pipe è vuota, l' operazione è bloccante e non torno. Quindi la pipe di per se non torna mai zero come numero di byte letti (Essendo bloccante si blocca finchè non legge).
Però se faccio una read su una pipe il cui estremo è stato chiuso, torna proprio 0.

Se l' estremo di scrittura di una pipe è chiuso (tramite la funzione close()), e provo a leggere, mi torna zero questo perchè non c' è modo che qualcuno scriva nella pipe.

Se si prova a scrivere in una pipe il cui estremo di lettura viene chiuso, il processo riceve un segnale SIGPIPE. Se non viene gestita la SIGPIPE, il processo che lo riceve viene chiuso.
Perchè c'è questa asimmetria? Se il processo prova a scrivere sulla pipe, e dall'altra parte non può essere letto nulla perchè l' estremo è stato chiuso, il processo che vuole scrivere deve ricevere un messaggio forte ad indicare che i byte che stà scrivendo non verranno mai letti.

>La stessa cosa avviene anche nei socket in cui l'estremo di un socket è stato chiuso


Esiste una variante chiamate `popen` e `pclose` che permettono di usare il concetto di pipe tramite il filestream. Non c'è nulla di diverso nel concetto di pipe, ma è solo una funzione di utilità del mondo unix-linux. Quello che ci permette di utilizzare è usare una popen per mandare in esecuzione un certo comando che deve essere un estremo della type che dipende dal tipo (che ci indica se si è connesso allo stdin o allo stdout). popen esegue una fork ed una exec al volo.

>`Popen` e `Pclose` non fanno parte della libreria standard

##Coprocessi e named pipe in Unix/Linux
Vediamo come l'utilizzo delle named pipe ci aiuta a risolvere un problema.
Sotto Unix/linux la named pipe è concettualmente uguale a quella di windows.
In unix si usa la primitiva `include <sys/stat.h> mkfifo` per creare il file (simile all' apertura di un file).
Una volta creata la pipe con mkfifo, la fifo può essere acceduta con l funzioni di I/O open, close, read, write, unlink ecc.
Da un punto di vista pratico è usata da due soli processi, ma si può utilizzare comunque da più processi.

>Supponiamo di avere un processo che processa dei dati, e che poi deve mandare a due procesi diversi i risultati del processamento.
>Utilizzo una namedpipe:
>
>     mkfifo fifo1
>     prog3 < fifo1 &
>     prog1 < input_file | tee fifo1 | prog2
> Tee riceve quello che viene prodotto dalla prma parte, lo scrive sul suo stdout perciò prog2, e lo scrive anche su un file (fifo1) che è una pipe. Visto che prog3 era in attesa, leggerà il contenuto della fifo1.
> Combinando le  named pipe e la namedpipe  ottengo questo effetto senza toccare nessuno dei tre programmi.

#Altre tecniche di IPC
Tecniche più sofisticate ma meno diffuse. 
In Win32 ricordiamo mailslot: sono un meccanismo un pò diverso da quelli visti fin'ora perchè sono *point to point.* Questo invece permette di inviare messaggi via broadcast.
La comunicazione è inviata da 1 a n.
Sono meccanismi di comunicazione unidirezionali e possono essere distribuite in rete, cioè si può utilizzare per inviare una informazione a dei client su una rete distribuita.
Tutti quanti ricevono lo stesso messaggio.

In Unix/Linux ci sono le message queue. Invece di avere un buffer, posso inviare delle vere e proprie liste: posso inserire in una coda dei messaggi collegati fra di loro tramite una lista linkata.
E' possibile accedere i messaggi in qualsiasi ordine (non necessariamente FIFO).

#Shared Memory IPC in Unix/Linux
La memoria condivisa è un IPC diverso (non c'è più uno scambio di messaggi ma uno scambio di zone di memoria).
Vedendo il file mapping abbiamo visto la condivisione dello spazio di indirizzamento usando mmap.
Esiste però un altro metodo di condivisione della memoria abbastanza utilizzato.
L' idea è specificare un segmnto che voglio condividere tramite una chiave (Che vinee utiizzata per identificare il segmento da processi distinti) questo segmento può avere una certa dimensione massima di 256mb, e ci sono una serie di flag che ci permettono di specificare le modalità di accesso al segmento.
Normalmente la dimensione del segmento è arrotondata ad un multiplo della dimensione di una singola pagina.
Esistono limiti sulla dimensione del blocco allocato, e sul numero di segmenti (il processo può invocare la primitva più volte per segmenti che si distinguono grazie alla chiave) ma esiste poi un limite massimo sul numero di segmenti che posso condividere.
Una volta definito il segmento, lo devo collegare allo spazio di indirizzamento del processo.

Lo creo con shmget, che lo crea e lo rende accessibile ma non lo collega. Quando è disponibile uso la primitiva `void *shmat` che prende l' indirizzo da shmget e lo collega all' indirizzo preso in input.
Usando null, si lascia al sistema attaccare il segmento dove meglio crede. L' ndirizzo di dove si è attaccato viene ritornato come puntatore a void altrimenti ritorna-1.
Per rimuovere lo spazio di indirizzamento di un processo un segmento si utilizza la `int shmdt (const void *shmaddr)`.
Posso usare la `smctl` per effettuare operazioni sul segmento. Ad esempio posso sapere quanti processi sono attaccati allo stesso spazio di indirizzamento.

#Linux
Esistono due tipi di utenti: utente normale, e superutenti. (pure in windows). In linux gli utenti sono raggruppati in gruppi. Quando si definisce un file, c'è il proprietario, il gruppo del proprietario, e tutti gli altri.
Questa gerarchia è ottima, ma è rigida: esiste solo questa possibilità. Esiste l' owner, il gruppo dell' owner, e tutti gli altri. Però io non posso andare a fare operazioni specifiche per un singolo utente.
Non posso dire che voglio permettere l'accesso in lettura ad un singolo utente.

#Attributi di sicurezza in Win32
Da windows 2000 in poi, ha un modello di sicurezza più sofisticato: usa un modello discretionary. Permette di regolare l'accesso ai singoli file. 
Esiste una classificazione del livello di sicurezza del sistema (come rappresentazione).  A livello di modello si usa la classificazione dell' Orange Book (windows è classificato a livello C2).
Questo modello discretionary, si basa sul concetto di attributo di sicurezza che viene gestito tramite liste d'accesso.
Il modello che vedremo quindi utilizza delle liste per associare attributi di sicurezza a tutta una serie di entità (tutte quelle viste fin'ora). Rispetto ai file, processi, mutex, file mapping, ecc.

Il tipo fondamentale è la struttura SECURITY_ATTRIBUTES

Il security descriptor (lpSecurityDescriptor) descrive chi è proprietario dell' oggetto (mutex file ecc) e a quali singoli utenti sono permessi o negati i singoli permessi.
Se il descrittore è NULL, significa utilizzare il descrittore di default del processo chiamante. C'è un descrittore di default che viene usato in caso non vogliamo definire permessi particolari. 
Solo l' utnete rappresentato dall' accesso token può accedere all' oggetto.
Quindi il modello di base è molto restrittivo.
Ogni modello di processo è identificato dal suo accesso token, ed  è il modo in sostanza con cui a livello kernel viene identificiato il proprietario.

##Security Descriptor in Win32
Si manipola usando una serie di primitive:

Inizializzato usando la funzione InitizalizeSecurityDescriptor che contiene l' identificatore dell' owner (SetOwner......), e del gruppo (Esiste il concetto di gruppo simile di linux anche in windows anche se è pocousato).

La parte improtante è la DACL Access Control List, specifica chi può fare cosa.

BOOL InitializeAcl (
	PACL pAcl,
	DWORD mAclLenght,
	DWORD dwAclRevision
	);

La dimensione nAclLenght del buffer pAcl può essere determinata con precisione ma 1kb è sicuramente adeguato.
Esistono due tipi di ACL: la DACL, e una seconda lista che non vedremo in dettaglio.

PRima inizializzo la lista con InitializeAcl, poi aggiungo gli ACE (Access Control Entry).
Ogni ACE contiene un SIDed una access mask che specifica i diritti da concedere o negare.
Applicare un' ACL vuota ad un oggetto nega tutti gli accessi all' oggetto! 

I valori possibili dell' AccessMask dipendono dal tipo di oggetto a cui si deve applicare l' ACE.
Una volta creata la ACL e riempita di ACE, aggiungo la lista al descrittorecon SetSecurityDescriptorDacl(

##Costruzione di un security descriptor
I passaggi per creare un security descriptor diverso da quello di default impostabile da null.
 1. InitializeSecurityDescriptor
 2. SetSecurityDescriptorOwner
 3. SetSecurityDescriptorGroup
 4. InitializeAcl
 5. AddAccessDeniedAce
 6. AddAccessAllowedAce
 7. SetSecurityDescriptorDacl

