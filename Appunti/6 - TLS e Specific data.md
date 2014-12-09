Lezione 13 - Programmazione di sistema

Codice rientrante e Thread Safety
=
Piccolo chiarimento su funzioni rientranti e thread safe. Una funzione rientrante non utilizza _variabili statiche_, e non ritorna puntatori a dati statici.
La ricetta più semplice è che la funzione per essere rientrante, riceve tutti i dati di cui ha bisogno dalal funzione chiamante.
Può ovviamente utilizzare le variabili _automatiche_ ma tutti gli altri dati necessari per inviare valori di ritorno tipo buffer dovrebbero essere passati dalla funzione chiamante.

Posso ottenere una funzione __thread safe__, anche se non ho una funzione rientrante, introducendo ad esempio dei lock per controllare le _risorse condivise_.

Posso utilizzare i lock anche per accedere non solo a sezioni chritiche, ma anche ai dati. Quindi come risultato posso ottenere una funzione thread safe se in quella funzione utilizzo meccanismi di lock.
Se non utilizzo dati statici ho ovviamente una funzione threadsafe.

	int diff(int x, int y) {
		int delta;
		delta = y-x;
		if(delta<0)
			delta = -delta;
		return delta;
	}
Viene ritornato proprio il valore, e non l' indirizzo del buffer.

Generalmente, se una funzione è thread safe non è per forza rientrante, e una funzione rientrante è una thread safe.

#Thread Local Storage TLS
Ho dei thread che condividono gran parte dello spazio di indirizzamento. Ognuno ha uno stack separato ma condividono lo spazio di indirizzamento. In questo ho i dati statici e ho anche la parte dell' heap (ci focalizziamo nel caso di un solo heap).

Un problema che posso avere, è quello in cui ho più thread che devono fare delle allocazioni dinamiche e voglio che ognuno abbia un'area allocata con malloc propria, e non condivisa - teoricamente condivisa, ma separata per ognuno.

Per allocare una riga nello TLS, usiamo `DWORD TlsAlloc(VOID)`.
Possono essere allocate almeno fino a 64 indici. Se non sono disponibili indici viene ritornato -1.
Per accedere per inizializzare uso TlsSetValue, mentre per prendere il valore uso TlsGetValue(DWORd dwTlsIndex);

Per liberare un indice, utilizzo `BOOL TlsFree ( DWORD dwTlsIndex);`.
Nessun thread può accedere al TLS di un altro thread, ma un thread può invocare TLSFree distruggendo l' indice di tutti i thread.

Questo meccanismo viene molto utilizzato da applicazioni grosse multithreading (es:dll).

##Specific Data - pthread
L' idea è la stessa che c'è per il TLS di Windows, purtroppo però la sintassi è completamente differente.
In questo caso non usiamo matrici, bensì coppie di chiave-valore.

La primitiva `pthread_key_create` permette di riservare un thread-specific data item.
Esiste una funzone pthread_once, viene chiamata ogni volta che viene usato un thread-specific-data ma viene eseguita una volta per processo.

Tutti i thread hanno una chiave che viene utilizzata per accedere al proprio puntatore. Negli specific data, è possibile avere una callback da richiamare quando il dato viene rilasciato.

Per usarle:

1. Si chiama `pthread_once` per fare l' inizializzazione. Apparentemente tutti la chiamano, ma viene eseguita solo la prima volta. All' interno si invoca la funzione `pthread_key_create` (TLSalloc). In questo caso ritorna una chiave. Quest'ultima funzione prende anche un puntatore a funzione, tipicamente usato per fare cleanup (fare la free).
2. Una volta che ho la chiave, le associo un valore con `pthread_setspecific` (potrebbe essere qualsiasi cosa - ma di solito è un puntatore).
3. Lo vado a leggere con `pthread_getspecific` (TLSgetValue)

Nella maggior parte dei casi l'associazione si fa una _volta sola_. Se devo usare più puntatori, vado o creare un'altra riga nella matrice o mi creo un'altra chiave. Non si riciclano entry di solito.

>Esempio: `thread/readline1.c` e `lib/readline.c`
>Se il puntatore associato alla chiave è effettivamente associato alla chiave, mi salvo il puntatore. 
>Altrimenti `pthread_getspecific` torna null. In quel caso alloco con Calloc e associo il puntatore con `pthread_setspecific(rl_key,tsd);`

----
#Priorità e Scheduling
###Gestione dei thread da parte del sistema operativo.
Sotto windows, vengono schedulati i singoli thread. Sotto unix/linux i processi. Il supporto thread sotto unix all' inizio non era un supporto separato (esisteva proprio una primitiva clone per i processi).
Nel caso unix le priorità sono ordinate in una maniera non molto intuitiva: i processi con priorità più bassa sono quelli che hanno la più alta probabilità di essere eseguiti.
Sotto windows, il thread con priorità più alta è quello che viene effettivamente schedulato. In windows poi, esiste anche la **classe di priorità**.
In unix lo scheduling è fatto sui singoli processi senza usare classi, in windows esistono 4 classi di priorità:

 1 `IDLE_PRIORITY_CLASS`, meno usato, priorità base 4
 2 `NORMAL_PRIORITY_CLASS`: priorità base 9 o 7, viene gestita a livello di applicazioni. Se la finestra ha il focus la priorità di base è 9 sennò 7.
 3 `HIGHT_PRIORITY_CLASS`: non viene quasi mai usata nelle applicazioni di windows.
 4 `REALTIME_PRIORITY_CLASS` : poco usata, priorità base 24. Windows e Linux sono di base non realtime, ma esistono apposite versioni.

Ogni classe ha 4 possibili valori di priorità. Esiste una priorità di base, e ci si sposta sempre nella stessa classe.

Si manipolano utilizzando delle primitive:

	Bool SetPriorityClass 
	(
		HANDLE hProcess,
		DWORD dwPriorityClass
	);
	DWORD GetPriorityClass
	(
		HANDLE hProcess
	);

Per poter modificare la priority class ho bisogno di diritti particolari. Normalmente queste funzioni non si utilizzano nella pratica perchè è una **pessima idea** modificare la priorità. In alcuni limitati casi possono essere applicate, ma normalmente non è una buona idea utilizzarle.
Il singolo processo ha tutti i diritti su se stesso.
Si utilizzano delle MACRO per gestire la variazione di priorità (da -2 a +2).
Esistono primitive per settare e ricevere la priorità di un thread.
Le proprità di un thread cambiano nel corso dell 'esecuzione in base alle regole di sistema operativo.
Le priorità possono essere cambiate ma è comunque una cattiva idea. E' possibile usare la funzione sleep per lasciare il controllo della cpu, e si specifica un tempo di sleep in millisecondi. Impostando sleep(0) dice allo scheduler che il tempo di esecuzione del processo è terminato.

#Possibili problemi con thread
E' importante ricordarsi della natura asincrona della loro esecuzione. Ad esempio, creo un certo numero di thread, quindi ho creato un certo numero di flussi di esecuzione.
Intuitivamente, questi flussi procedono veramente in maniera concorrente, ovvero avanzano allo stesso modo. Ovviamente questa pratica non è vera: ad esempio se creo un numero di thrad maggiore dei core disponibili, cerctamente non possono avanzare parallelamente.
Anche se avessi un core per ogni thread, avrei un flusso asincrono.
Non è scontato che tutti i thread si trovino allo stesso punto. Se i thread eseguono azioni diverse,è  intuitivo che non avanzino allo stesso modo. Se invece eseguono le stesse operazioni e avessi la giusta quantità di core, potremmo pesnare che avanzano allo stesso modo : **nonè così**.
E' possibile sincronizzare i thread in modo che accedano i dati o avanzino allo stesso modo.
Non si fanno quindi mai assunzioni sull' avanzamento della esecuzione.

##Ciclo di vita del thread
![Ciclo di vita di un thread](images/ciclo-di-vita-thread)

Un thread rimane in *esecuzione* finchè gli rimane il tempo preassegnatogli, oppure quando non può procedere perchè ha bisogno di risorse specifiche.
Tipico di windows sugli object, per unix potrei andare in attesa per la lettura di un file.
Entro nello stato di attesa, e quì rimango finchè l'evento con la quale sono in attesa per un certo oggetto si verifica.

