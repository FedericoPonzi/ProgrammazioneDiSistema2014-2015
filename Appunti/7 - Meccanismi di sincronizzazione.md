Programmazione di sistema - Lezione  15
#"Fibre" in win32
Una piccola curiosità intellettuale. Sono infatti poco utilizzate, e permettono un approccio alternativo alla esecuzione concorrente.
La caratteristica principale è che lo scheduling dell'applicazione viene eseguita non dal kernel ma gestita direttamente dall' applicazione.
La fibra (filo) può essere creata da un singolo thread e un thread può creare uno o più fibre, e le fibre stesse determinano quale sarà eseguita successivamente.
Ho fibre con stack indipendenti, però la parte di spazio di indirizzamento privato ma globale (local storage) è condiviso.
Sono disponibili sututte le versioni di windows.
Questo meccanismo è poco utilizzato, un esempio è per esempio per supportare il polling in modo da evitare di bloccare l' intero thread.
Le fibre poi permettono di imlementare il concetto di "co-routine"
> Esistono flussi di esecuzione concorrenti, ma il flusso da eseguire viene scelto in maniera esplicita dall' applicazione. Una funzione di un flusso di esecuzione, chiama non una funzione del suo flusso, ma una funzione di un'altro flusso. Questa è una co-routine.

##Come usare le fibre
Usa un set abbastanza ridotto di 6 primitive, per creare l'ambiente si chiama una rotuine `ConvertThreadToFiber` che permette di rendere il thread un contenitore di fibre.
Ritorna l' indirizzo della fibra, è l' equivalente dell' handle nella gestione dei thread. Una differenza fra i due è che le fibre non sono oggetti kernel (handle), quindi si utilizzano gli indirizzi per la manipolazione delle fibre.
Una volta convertito il thread, posso creare nuove fibre usando la primitiva `CreateFiber` che fa partire una funzione passandogli un puntatore generico. Si può specificare per ogni fibra la dimensione dello stack. Anche quì viene ritornato un puntatore. Il valore di ritorno non è il puntatore allla funzione associata  alla fibra, ma ad un blocco di controllo usato per la manipolazione della fibra (per mandarla in esecuzione) ma che **non** coincide con l' indirizzo della funzione.

La funzione `SwitchToFiber` prende il puntatore al blocco di controllo che serve alla manipolazione della fibra.
Prende o il valore di ritorno di CreateFiber, o comunque di una qualunque fibra già creata.
Quando chiamo switchtofiber, la fibra invocante non torna da questa invocazione. Passando il controllo ad un'altra fibra, il controllo viene rilasciato. Potenzialmente potrebbe non tornare mai.

Esistono altre funzioni di utilità, una fibra può essere anche cancellata da un'altra fibra usando la funzione `DeleteFiber(LPVOID lpFiber)` che permette di richiamare un suicidio collettivo, dando come parametro l' indirizzo del chiamante in modo a cancellare l' intero thread.
sisopii/FiberEample.c

#Sincronizzazione dei thread
##Win32
I meccanismi di sincronizzazione in Win32 possono essere utilizzati non solo tra thread all' interno dello stesso processo ma, in molti casi, anche tra thread appartenenti a processi separati.
Alcune semplici regole generali sono le seguenti:

 1. I thread dovrebbero cambiare il process environment solo nel caso in cui tutti i thread devono essere impattati. Ovvero i meccanismi di sincronizzazione devono essere selettivi. Ci sono casi in cui anche se è selettivo, si applica a più thread, ma in questi casi succede che riapplico lo stesso meccanismo a più casi. Questo discorso fa sempre riferimento al principio per cui quello che andiamo a controllare dovrebbe avere un diretto impatto solo a livello locale (nella singola applicazione) evitando di interferire ai compiti del sysop. 
 2. Se un thread deve mantenere dati accessibili ma privati nel suo flusso di esecuzione deve allocarle sul TLS.
 3. Le variabili condivise si devono specificare come `volatile`. Questo perchè se ho 2 o più thread che devono utilizzare la stessa variabile, vorrei che fosse coerente. Quando il secondo thread la legge dovrebbe leggere l'ultimo valore scritto. 

> **Volatile** richiede che tutti gli accessi vengano fatti direttamente in memoria (e non salvate per esempio sui registri). In alcuni casi può essere anche sconsigliata come utilizzo in quanto può avere un impatto pesante nell' applicazione.

La più semplice funzioni di sincronizzazione che risolvono comunque un problema molto comune sono due primitive base

    LONG InterlockedIncrement ( LONG volatile* Addend);
    LONG InterlockedDecrement (LONG volatile* Adend);

##Critical_SECTION
La critical section è un meccanismo messo a disposizione da windows, per supportare il concetto generico secondo cui ho una parte (sezione -> flusso di esecuzione) che voglio controllare, e fare in modo che ci sia solo un thread per volta la esegua.
Può venire fuori come diverse circostanze, e windows mette a disposizione un meccanismo specifico che non può essere usato da processi.
Questo meccanismo non offre supporto per thread di processi distinti (questo perchè hanno codici e spazi differenti perciò non possono darsi fastidio).
Si usa una variabile di tipo CRITICAL_SECTION, e si inizializza usando `InitializeCriticalSection`
e ci si può entrare usando  `EnterCriticalSection(LPCRITICAL_SECTION lpCriticalSection);`
Un thread che vuole entrare nella sezione critica, chiama la primitiva ed aspetta (viene bloccato) finchè un thread già presente non ne esce.
Per uscire dalla sezione critica, deve richiamare la funzione LeaveCriticalSection ( LPCRITICAL_SECTION)
Nonostante la semplicità, ci sono diverse considerazioni da fare.

 1. Che succede se il thread che è già entrato nella sezione critica, e richiede nuvoamente di entrarvi? Se un thread possiede il controllo di una CS, può entrare più volte senza essere bloccato. Quando esco però, devo uscire per lo stesso numero di volte per cui sono entrato. Se un thread entra tre volte in una CS, deve uscire per 3 volte.
 2. Invocare LeaveCriticalSection senza avere il controllo può portare a risultati impredicibili.
 3. Non esiste un meccanismo di timeout, però si può utilizzare la funzione `boolean TryEnterCriticalSection` che è una variante non bloccante. Se riesco entro altrimenti torno. Il booleano di ritorno value `true` sono dentro la critical section, se è `false`, ci devo provare di nuovo. E' fondamentale quindi controllare il valore di ritorno.

		CRITICAL_SECTION CriticalSection;
		void main() {
		//Initialize the critical section one time only
			if(!InitializeCriticalSectionAndSpinCount())
			return;
		}
		
	
>Esempio: simplePC.c in chaptr09		

    /* Chapter 9. simplePC.c										*/
    /* Maintain two threads, a producer and a consumer				*/
    /* The producer periodically creates checksummed data buffers, 	*/
    /* or "message block" which the consumer displays when prompted	*/
    /* by the user. The conusmer reads the most recent complete 	*/
    /* set of data and validates it before display					*/
    
    #include "EvryThng.h"
    #include <time.h>
    #define DATA_SIZE 256
    
    typedef struct msg_block_tag { /* Message block */
    	volatile DWORD		f_ready, f_stop; 
    		/* ready state flag, stop flag	*/
    	volatile DWORD sequence; /* Message block sequence number	*/
    	volatile DWORD nCons, nLost;
    	time_t timestamp;
    	CRITICAL_SECTION	mguard;	/* Guard the message block structure	*/
    	DWORD	checksum; /* Message contents checksum		*/
    	DWORD	data[DATA_SIZE]; /* Message Contents		*/
    } MSG_BLOCK;
    /*	One of the following conditions holds for the message block 	*/
    /*	  1)	!f_ready || f_stop										*/
    /*			 nothing is assured about the data		OR				*/
    /*	  2)	f_ready && data is valid								*/
    /*			 && checksum and timestamp are valid					*/ 
    /*  Also, at all times, 0 <= nLost + nCons <= sequence				*/
    
    /* Single message block, ready to fill with a new message 	*/
    MSG_BLOCK mblock = { 0, 0, 0, 0, 0 }; 
    
    DWORD WINAPI produce (void *);
    DWORD WINAPI consume (void *);
    void MessageFill (MSG_BLOCK *);
    void MessageDisplay (MSG_BLOCK *);
    	
    DWORD _tmain (DWORD argc, LPTSTR argv[])
    {
    	DWORD Status, ThId;
    	HANDLE produce_h, consume_h;
    	
    	//Inizializza la sezione critica
    	/* Initialize the message block CRITICAL SECTION */
    	InitializeCriticalSection (&mblock.mguard);
    
    	/* Create the two threads */
    	produce_h = (HANDLE)_beginthreadex (NULL, 0, produce, NULL, 0, &ThId);
    	if (produce_h == NULL) 
    		ReportError (_T("Cannot create producer thread"), 1, TRUE);
    	consume_h = (HANDLE)_beginthreadex (NULL, 0, consume, NULL, 0, &ThId);
    	if (consume_h == NULL) 
    		ReportError (_T("Cannot create consumer thread"), 2, TRUE);
    	
    	/* Wait for the producer and consumer to complete */
    	
    	Status = WaitForSingleObject (consume_h, INFINITE);
    	if (Status != WAIT_OBJECT_0) 
    		ReportError (_T("Failed waiting for consumer thread"), 3, TRUE);
    	Status = WaitForSingleObject (produce_h, INFINITE);
    	if (Status != WAIT_OBJECT_0) 
    		ReportError (__T("Failed waiting for producer thread"), 4, TRUE);
    
    	DeleteCriticalSection (&mblock.mguard);
    
    	_tprintf (_T("Producer and consumer threads have terminated\n"));
    	_tprintf (_T("Messages produced: %d, Consumed: %d, Known Lost: %d\n"),
    		mblock.sequence, mblock.nCons, mblock.nLost);
    	return 0;
    }
    // Produttore: userà il meccanismo Critical Section
    DWORD WINAPI produce (void *arg)
    /* Producer thread - Create new messages at random intervals */
    {
    	
    	srand ((DWORD)time(NULL)); /* Seed the random # generator 	*/
    	
    	while (!mblock.f_stop) {
    		/* Random Delay */
    		Sleep(rand()/100);
    		
    		/* Get the buffer, fill it */
    		
    		EnterCriticalSection (&mblock.mguard);
    		__try {
    			if (!mblock.f_stop) {
    				mblock.f_ready = 0;
    				MessageFill (&mblock);
    				mblock.f_ready = 1;
    				mblock.sequence++;
    			}
    		} 
    		__finally { LeaveCriticalSection (&mblock.mguard); }
    	}
    	return 0;
    }
    
    DWORD WINAPI consume (void *arg)
    {
    	DWORD ShutDown = 0;
    	CHAR command, extra;
    	/* Consume the NEXT message when prompted by the user */
    	while (!ShutDown) { /* This is the only thread accessing stdin, stdout */
    		_tprintf (_T("\n**Enter 'c' for consume; 's' to stop: "));
    		_tscanf ("%c%c", &command, &extra);
    		if (command == 's') {
    			EnterCriticalSection (&mblock.mguard);
    			ShutDown = mblock.f_stop = 1;
    			LeaveCriticalSection (&mblock.mguard);
    		} else if (command == 'c') { /* Get a new buffer to consume */
    			EnterCriticalSection (&mblock.mguard);
    			__try {
    				if (mblock.f_ready == 0) 
    					_tprintf (_T("No new messages. Try again later\n"));
    				else {
    					MessageDisplay (&mblock);
    					mblock.nCons++;
    					mblock.nLost = mblock.sequence - mblock.nCons;
    					mblock.f_ready = 0; /* No new messages are ready */
    				}
    			} 
    			__finally { LeaveCriticalSection (&mblock.mguard); }
    		} else {
    			_tprintf (_T("Illegal command. Try again.\n"));
    		}
    	}
    	return 0;		
    }
    
    void MessageFill (MSG_BLOCK *mblock)
    {
    	/* Fill the message buffer, and include checksum and timestamp	*/
    	/* This function is called from the producer thread while it 	*/
    	/* owns the message block mutex					*/
    	
    	DWORD i;
    	
    	mblock->checksum = 0;	
    	for (i = 0; i < DATA_SIZE; i++) {
    		mblock->data[i] = rand();
    		mblock->checksum ^= mblock->data[i];
    	}
    	mblock->timestamp = time(NULL);
    	return;
    }
    
    void MessageDisplay (MSG_BLOCK *mblock)
    {
    	/* Display message buffer, timestamp, and validate checksum	*/
    	/* This function is called from the consumer thread while it 	*/
    	/* owns the message block mutex					*/
    	DWORD i, tcheck = 0;
    	
    	for (i = 0; i < DATA_SIZE; i++) 
    		tcheck ^= mblock->data[i];
    	_tprintf (_T("\nMessage number %d generated at: %s"), 
    		mblock->sequence, _tctime (&(mblock->timestamp)));
    	_tprintf (_T("First and last entries: %x %x\n"),
    		mblock->data[0], mblock->data[DATA_SIZE-1]);
    	if (tcheck == mblock->checksum)
    		_tprintf (_T("GOOD ->Checksum was validated.\n"));
    	else
    		_tprintf (_T("BAD  ->Checksum failed. message was corrupted\n"));
    		
    	return;
    
    }

Le critical section, non hanno un equivalente nei pthread. Sotto *nix non abbiamo un meccanismo equivalente. Possiamo immaginarlo, ma non abbiamo primitive simile. Naturalmente useremo meccanismi alternativi di sincronizzazione.

##Mutex
Sono supportati sia su windows che su *nix. Ragionevolmente simili, sintassi completamente diversa.
Il Mutex è più sofisticato: posso ottenere lo stesso riusltato dellla CS, ma anche molto di più.
Fanno parte dei meccanismi di sincronizzazione usati anche per sincronizzare thread di processi distinti.
Posso associare un time-out, ed usare la primitiva `WaitForSingleObject.` In questo caso posso automaticamente definire un timeout.
E' presente anche nei mutex il problema delle CS: se un thread ha un mutex e vuole riprendere i lcontrolo di nuovo? La situazione in windows e *nix è leggermente diversa, in Windows la situazione è identica a quella della CS.
Per creare un mutex, si usa CreateMutex che è una primitiva semplice ma con caratteristiche da evidenziare:

 - Devo associare al mutex un nome, in modo che lo possa aprire da un'altro processo.
 - Può avere associato un blocco di attributi di sicurezza.
 - Quando viene specificato null, si assumono gli attributi di sicurezza di default,
 - La mutex mi ritorna un handle. O fallisce se il mutex esiste già un mutex. Posso usare lo stesso nome di un file, e i nomi sono casesensitive
 - Tutti i thread di un processo possono accedervi direttamente avendo accesso all' handle.
 - Se voglio sincronizzare due thread di processi distinti, uso la primitiva OpenMutex (equivalente di openprocess, openthread ecc). Fallisce se il mutex non esiste ancora.

Per ottenere il controllo del mutex, uso la primtiva generica WaitForSingleObject. Come al solito posso fare polling impostando dwMilliseconds=0;
Posso usare anche WaitForMultipleObject per più mutex (ma è poco usato).
Se voglio rilasciare il mutex, chiamo `ReleaseMutex(HANDLE hMutex)`. Se richiamo la releasemutex senza avere il controllo, l' effetto è inaspettato.
Va invocato per ogni volta che il mutex è stato acquisito.

##Mutex e pthread
Un Mutex è una variabile di tipo `pthread_mutex_t`.
Per lockkare un mutex uso:`pthread_mutex_lock(pthread_mutex_t * mptr);`  e' una chiamata bloccante.
Per rilasciare il mutex `pthread_mutex_unlock(pthread_mutex_t *mptr);`.
L' ordine con cui dei thread bloccati su un lock aspettano di entrare nella sezione critica vengono sbloccati, è indefinito e **non è su base fifo**.

Nel caso dei pthread, è più complesso capire che succede quando un thread ha già il mutex e rivuole il controllo.  Dipende dal tipo di mutex. Esistono 3 tipi di mutex:

 - Fast:è un mutex che non permette di riottenere il controllo del mutex da parte dello stesso thread. (se lo riaquisice, va in errore ed entra in deadlock).
 - Ricorsivo: Può riacquisire il controllo di quel mutex, ma deve rilasciarlo lo stesso numero di volte altrimenti abbiamo un problema.
 - Error Check: Ultimo tipo di mutex, una sorta di via di mezzo. Non posso riaquisire il controllo, e mi genera un *errore esplicito*. Fast invece và in deadlock.

