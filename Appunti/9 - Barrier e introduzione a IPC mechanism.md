##Condition Variables in Win32
E' possibile creare un meccanismo equivalente di Condition Variables in windows può essere raggiunto usando una combinazione di mutex ed eventi.
Cominciamo dal produttore: lo usa per ontrollare l'accesso. Sotto al mutex cambia lo stato (aggiorna il conto degli item) e segnala l'evento. Questa segnalazione la esegue usando la PulseEvent.
Dopo aver segnalato l'evento, rilascia il mutex.
Il consumatore, per poter controllare lo stato (*predicato*) deve prima di tutto acquisire il controllo del mutex.
Dopo aver acquisito il mutex, controlla il predicato, se non è soddisfatto a questo punto il consumatore rilascia il mutex, e va in attesa che cambi lo stato (che il produttore produca un nuovo item). Quì si crea un problema: abbiamo visto sotto Linux che il rilascio del mutex era implicito quando si fa la wait.
Il problema è: cosa avviene se il consumatore rilascia il mutex e prima di andare in attesa il produttore prende il controllo del mutex e segnala l'evento? L'evento è perso.
*Potrebbe esserci un deadlock*.
La soluzione è facile (Anche se poco elegante): la wait permette di associare il timeout. Quindi questo ci permette dopo tot tempo dalla wait per controllare se il predicato è soddisfatto.
Il consumatore deve quindi mantenere il controllo del mutex e rilasciarlo solo quando ece dal loop, altrimenti può succedere che se non ha il controllo del mutex potrebbe venire preso dal produttore che cambierebbe lo stato senza che il consumatore se ne accorga. Per questo motivo abbiamo bisogno di questo wait con timeout.
##Barriera
Vediamo un esempio di barriera che usa una combinazione di mutex ed evento, e che risolve il problema di avere un certo numero di entità che devono arrivare ad un punto di barriera.
Il meccanismo si basa sul mutex e l' evento. Quindi nella struttura dati ho gli handle per manipolare questi due oggetti.
Poi ho un valore di soglia (`b_threshold`) che è il numero che deve essere raggiunto. Ho un contatore per sapere quanti thread hanno raggiunto la barriera (`b_count`), e poi ho una ulteriore variabile che serve a definire se la barriera è ancora valida o meno (`b_destroyed`) se destroyed è zero la barriera è ancora valida.

    /* THRESHOLD BARRIER - TYPE DEFINITION AND FUNCTIONS */
    typedef struct THRESHOLD_BARRIER_TAG { 	/* Threshold barrier */
    	HANDLE b_guard;	/* mutex for the object */
    	HANDLE b_broadcast;	/* auto reset event: b_count >= b_threshold */
    	volatile DWORD b_destroyed;	/* Se la barriera è valida o no */
    	volatile DWORD b_count;		/* Numero di thread che hanno raggiunto la barriera */
    	volatile DWORD b_threshold;	/* Numero di thread che devono raggiungere la barriera (soglia) */
    } THRESHOLD_BARRIER, *THB_HANDLE;

    /**********************/
    /*  THRESHOLD BARRIER OBJECTS */
    /**********************/
    
    DWORD CreateThresholdBarrier (THB_HANDLE *pthb, DWORD b_value)
    {
    	THB_HANDLE hthb;
    	/* Initialize a barrier object */
    	hthb = malloc (sizeof(THRESHOLD_BARRIER));
    	if (hthb == NULL) return 1;
    
    	hthb->b_guard = CreateMutex (NULL, FALSE, NULL);
    	if (hthb->b_guard == NULL) return 2;
    	
    	hthb->b_broadcast = CreateEvent (NULL, FALSE, FALSE, NULL);
    	if (hthb->b_broadcast == NULL) return 3;
    	
    	hthb->b_threshold = b_value;
    	hthb->b_count = 0;
    	hthb->b_destroyed = 0;  /* the object is now valid. */
    
    	*pthb = hthb;
    
    	return 0;
    }
    
    
    DWORD WaitThresholdBarrier (THB_HANDLE thb)
    {
    	/* Wait for the specified number of thread to reach */
    	/* the barrier, then broadcast on the CV */
    
    	/* Assure that the object is still valid */
    	if (thb->b_destroyed == 1) return 1;
    	
    	WaitForSingleObject (thb->b_guard, INFINITE);
    	thb->b_count++;  /* A new thread has arrived */
    	while (thb->b_count < thb->b_threshold) {
    		ReleaseMutex (thb->b_guard);
    		WaitForSingleObject (thb->b_broadcast, CV_TIMEOUT);
    		WaitForSingleObject (thb->b_guard, INFINITE);
    	}
    	
    	SetEvent (thb->b_broadcast); /* Broadcast to all waiting threads */
    	ReleaseMutex (thb->b_guard);	
    	return 0;
    }
    
    
    DWORD CloseThresholdBarrier (THB_HANDLE thb)
    {
    	/* Destroy the component mutexes and event once it is safe to do so */
    	/* This is not entirely satisfactory, but is consistent with what would */
    	/* happen if a mutex handle were destroyed while another thread waited on it */
    	if (thb->b_destroyed == 1) return 1;
    	WaitForSingleObject (thb->b_guard, INFINITE);
    	thb->b_destroyed = 1; /* No thread will try to use it  during this time inteval. */
    
    	/* Now release the mutex and close the handle */
    	ReleaseMutex (thb->b_guard);
    	CloseHandle (thb->b_broadcast);
    	CloseHandle (thb->b_guard);
    	free (thb);
    	return 0;
    
    }

> Esercizio: creare barriera per i pthread

----
##DeadLock

    static struct { HANDLE guard;
    struct List;
    } ListaA, ListaB;
    
    DWORD AddSharedElement (void *arg) {
    	WaitForSingleObject(ListA.guard, INFINITE);
    	WaitForSingleObject(ListB.guard,INFINITE);
    	
    	/** AddShared element **/

		Release(ListA.guard); Release(ListB.guard);
		return 0;
		}


##Processi leggeri
In linux nascono processi che vengono suddivisi in thread, mentre in Windows abbiamo threads e processi come contenitori di q uesti.
In Windows, il sistema operativo si limita a mettere in esecuzione i Kernel Thread. Questi vengono gesitti direttamente a livello di sistema operativo (dal kernel), vengono solitamente creati per svolgere compiti particolari (non della gestione ordinaria all' interno del kernel) sono estremamenti veloci nel content switch e sono poco onerosi da creare.
Un **Lightweight Process** possiamo vederlo come un processo utente supportato dal kernel thread.
Se si fanno chiamate di sistema che vanno ad accedere frequentemente strutture dati comuni, il meccanismo di sincronizzazione potrebbe diventare molto pesante (alto overhead) tale da eliminare i vantaggi dell' utilizzo di questi (es: applicazioni che usano spesso I/O).
La possibilità di supportare i thread a livello utente, tutto lo scheduling viene fatto a livello dell' applicazione.

## Inter Process Comunication (IPC)
Meccanismi di comunicazione tra processi: un esempio è il file mapping (visto in passato). Il concetto di IPC è un concetto però più generale, e nasce molto presto e nei sistemi Linux si concretizza con la Pipe.
Con la pipe c'è un canale di comunicazione è tale per cui c'è un processo che lo utilizza per inviare dei dati, ed un altro processo per ricevere dei dati.
La pipe nella forma più elementare implementa il concetto di *half-mutex* nel quale c'è sempre qualcuno che scrive da un alto ed un'altro che riceve dall' altro.
Per quanto potente ed utilizzato, il meccanismo ha il problema di essere unidirezionale.
Per questo motivo si usano le named-pipe. Mentre la pipe tradizionale, è una pipe senza nome, nella named pipe abbiamo un nome.
Questo nome è importante perchè permette di creare manipolazioni che una pipe anonima non permette. Oltre a questo nome, la named pipe permette una comunicazione full duplex: posso scrivere e ricevere da entrambi i lati della pipe.
Come ulteriorie possibilità, sotto windows possiamo usare le named pipe per comunicare in rete. La pipe funziona solo in locale.
Sotto Linux invece si utilizzano i sockets.

##Anonymous pipe in Win32
Vediamo come funziona il concetto di pipe anonima sotto windows:

	BOOL CreatePipe( 
		PHANDLE hREadPipe,
		PHANDLE hWritePipe,
		LPSECURITY_ATTRIBUTES lpPipeAttributes,
		DWORD nSize);

Sotto windows abbiamo due `PHANDLE` (=>`HANDLE`) uno per scrivere e uno per leggere, sotto linux abbiamo due **file descriptor**.
Se specifico 0 come **nSize**, utilizza il valore di default come dimensione. Sennò posso specificare un valore diverso per indicare la dimensione della pipe.
Quando il processo scrive nella pipe, e nessuno legge, quello che scrive si accumula nella pipe **fino ad un certo punto**. Quando la pipe è piena,  il processo che prova a scrivere **va in attesa**.
La dimensione standard della pipe è relativamente piccola (4kb): è importante perchè ce la ritroveremo tale e quale quando lavoreremo con i sockets.
Mentre nel caso di una applicazione che utilizza una pipe certi meccanismi ci si può accorgere, nelle socket potrebbe diventare più difficile.

Tornando a Windows, questo ci permette di definire una dimensione diversa da quella di default usando l'argomento di `nSize`.
La logica di utilizzo è abbastanza simile a quella già vista con linux: 

 - Si crea la pipe, 
 - Si ottengono due handle, 
 - Si processo crea due processi e associa un handle per ogni processo.
 - Una pipe si scrive con `WriteFile(hWrite)` e si legge usando `ReadFile(hRead)`

Il numero di byte che gli viene richiesto di leggere, non è detto che sia lo stesso del numero di byte richiesto dalla read.

Devo sempre controllare che il numero di byte letto sia almeno quello che mi aspetto o meno.
Le pipe anonime sono pensate per essere utilizzate fra due processi, e con un solo verso di lettura/scrittura.

##Named pipe in win32
Sono molto più sofisticate, visto che permettono anche la bidirezionalità (leggere e scrivere da entrambi i lati).
La pipe anonima utilizza un tipo di invio a flusso, non esiste il concetto di datagramma. Sotto windows con le namedpipe possiamo usare anche un meccanismo di pacchetti.
La differenza principale fra le due è molto profonda. Ad esempio, la prima differenza è che quello che abbiamo detto prima del numero di byte inferiore rispetto a quello che volevo leggere, non si applica più.
Quando ho una named pipe posso avere più istanze che utilizzano la stessa pipe. Più client che usano la stessa pipe (n client ad 1). e in più il server può rispondere ad ognuno di essi.
Le named pipe hanno un nome che può essere usato anche in locale. Sotto Windows possiamo usarlo anche per trasferire dati da due sistemi basta che abbiano lo stesso nome.

La NamedPipe usa la primitiva `CreateNamedPipe` :
lpName deve essere nella forma  `\\.\pipe\pipename`
mMaxInstances quanti client posso avere
nDefaultTimeOut: timeout per le operazioni. E' particolare perchè il timeout che si specifica non ha validità per questa funzione, ma per la funzione WaitNamedPipe. è l' unica funzione in Win del tipo createxx che specifica il timeout per un'altra funzione.
Dimensioni dei buffer input e output 
dwPipeMode identifica se:
 -  è `message-oriented` o `byte-oriented`.
 - La lettura è per messaggi o per blocchi;
 - Le operazioni di lettura sono bloccanti.

Normalmente c'è un solo processo che crea la named pipe: di solito il server. A quel punto gli utilizzatori della pipe accedono alla pipe usando la `CreateFile` e specificando come nome del file il nome della namedpipe.

Questo pipe può essere utilizzato anche in rete, l' unica differenza che bisogna specificare il nome della macchina come prima parte della sintassi di definizione del nome.
Si sostituisce al punto (.) col nome del server (servername).
