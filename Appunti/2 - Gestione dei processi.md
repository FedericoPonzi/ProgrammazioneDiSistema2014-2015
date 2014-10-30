# Gestione dei processi in Win3w2
Windows è completamente diverso da Linux sotto questo aspetto.
Intanto, il processo non è l' unità di esecuzione fondamentale, ma è il Thread.
Il processo è una sorta di contenitore per uno o più thread.
La maggior parte delle apps windows in realtà utilizza i thread, e vedremo comevanno utilizzati.

Vediamo i processi prima.

Processi
---
C'è almeno un thread, e naturalmente ha un proprio spazio di indirizzamento virtuale e protetto. Ogni proceso può accedere al proprio spazio come se fosse l'unico ad accedere alla memoria anche se in realtà è il sysop che implementa un sistema di protezione.
Ovviamente lo spazio può essere immaginato come un insieme, anche se in realtà è un insieme di segmento. Un segmento è il modo con cui si riferisce la suddivsione dello spazio di indirizzamento (segmenti o aree).
Normalmente le aree contengono una i codici e le istruzioni, un'altra parte i dati statici, un'altra è lo spazio di indirizzamento per lo heap, lo stack ecc.
Ci sono poi le variabili d'ambiente e anche se non fa parte dlelo spazio di indirizzamento è comunque riguardante il processo l' insieme delle risorse del processo.
Parte della zona specifica di ogni processo, è la parte che contiene gli handle del processo che puo usare.
Ci sono i file col file mapping, ma anche gli altri oggetti kernel manipolati dagli handle fanno parte del processo.

LA caratteristica principale di un thread, è che condividono tutte le variabili globali, d'ambiente e le risorse. Non condividono: 
	- lo stack (potrebbe essere un punto critico se non si comportano in modo coerente fra loro);
	- Thread Local Storage (TLS): è una parte che rivedremo in Unix/Linux, svolge un ruolo particolare. Svolge la parte di spazio di indirizzamento che permette ad ogni thread di avere variabili globali cioè accessibili da qualunque parte del processo ma privato al thread. Si vedranno meglio più avanti.

creazione di processi
==========
Si usa una funzione fondamentale  per la gestione dei processi che è la CreateProcess. Ha molti parametri ç_ç
In realtà la CreateProcess svolge la funzione svolta da due primitive in Unix: 

	- la fork, che crea un nuovo processo;
	- la exec, che permette di avere in esecuzione un programma diverso da quello che è stato duplicato.
Intanto non abbiamo una duplicazione del programma di partenza (non genitore perchè non esiste in win). Altra differenza, nel caso di unix con la fork il processo figlio, parte dal punto in cui è stata fatta la fork.
Quando vengono fatti i thread singoli, si possono specificare ( si fa sempre) un punto di partenza per un Thread (non riinizia l' esecuzione).
Quello che viene creato, è a sua volta un oggetto Kernel -> perciò và manipolato con gli handle. Perciò questo deve essere ritornato (in un certo modo opportuno), torna anche l' equivalente che è un id numerico.
Alcune operazioni si possono fare con l'handle, mentre altre si fanno usando l' id numerico.
Queste informazioni vengono tornate tramite la struttura contiene il pid associato al processo del MainThread appena creato.

Alcuni argomenti della CreateProcess:
	- lpApplicationName: La applicationName può essere null, e in quel caso conta solo la commandline. Contiene il nome del comando e i suoi eventuali argomenti.
		Se non è null, specifica il comando da mandare in esecuzone. Se specifico application name è come se il path non esistesse. Se no vale il primo argomento della command line -> segue delle regole diverse da unix/linux.
	- LPTSTR lpCommandLine : comando del programma da eseguire. 
	
Caso semplice: entrambi hanno un FullPath. O esiste il programma a quel fullpath o errore.
Caso particolare: non fullpath, ma path relativo. Se si usa questo, su UnixLinux si cerca solo nel PATH. Su Windows, succede che viene eseguito questo inseime di regole:
	1. La directory corrente del processo invocante (WD).
	2. La Windows System Directory (ottenibile con GetSystemDirectory)
	3. La Windows Directory (ottenibile con GetWindowsDirectory)
	4. La directory specificate nella variabile di ambiente PATH;

Su Unix posso fare in mdoo che non venga trovato nessun programma, annullando il valore della variabile PATH (unset(PATH)).
Sotto windows, non c'è perchè esiste questo ordinamento. E' molto importante il punto 1, che normalmente su unix non c'è, perchè se esistono nomi duplicati allora va a prendere quello contenuto nella wd.
In Unix si può aggiungere la wd con un punto (.). Però viene comunque aggiunto alla fine del PATH quindi le cose vengono rigirate.

Creando il processo si possono creare dei flag:
 1. DETACHED_PROCESS : (Deamon: Processo eseguito in background, in maniera trasparente all' utente). SottoWindows: è simile al deamon (anche se l'ambiente è diverso). DETACHED: vuol dire che è scollegato dal terminale. Sotto Unix invece si fa quando il processo è già stato creato.
 2. CREATE_SUSPENDED : Uno dei problemi classici del modello UnixLinux è chi per primo va in reale esecuzione. Con Windows si può creare il processo e mandarlo direttamente nello stato suspended. Questa caratteristica si applica anche ai thread in Windows.
	
C'è la possibilità di specificare un blocco di variabili d'ambiente. Normalmente su Unix vengono ereditate, si possono anche aggiungere variabili d'ambiente. Su Unix posso aggiungere variabili d'ambiente (getenv ecc).
Un processo può ottenere la propria command line usando argv oppure GetCommandLine (-> torna in una stringa tutta la lista dei comandi).

Trasmissione degli handle
====
La trasmissione degli handle quando si crea un processo non è così naturale come con Unix. Questo perchè non stiamo facendo le stesse operazioni, stiamo creando un nuovo processo con immagine diversa che quindi può e non ha senso trasmettere tutti gli handle del processo che ha fatto la CreateProcess. Posso specificare gli handle che devono essere trasmessi.

Passi da seguire:
 1. Settare flagbInheritHandles=TRUE;
 2. Per ogni singolo handle che voglio trasmettere, devo aggiungere il flag nella struttura SECURITY_ATTRIBUTES di quell' handle specifico.
 3. Una volta attivato il flag, devo effettivamente trasmetterlo esplicitamente. Ho tre modi:
	1. IPC mechanism,
	2. Ridirezione dello standard I/O e associo l'handle che deve essere ereditato dagli handle standard,
	3. Converto l'handle in testo come fosse una stringa, e lo passo o come una linea comando, o in una variabile d'ambiente.
 
Gli handle in questo modo vengono ereditati, ma sono copie distinte.
Posso duplicare un handle usando una funzione  DuplicateHandle.
Col meccanismo di ereditarietà posso passare un handle ad un' altro processo. In questo caso ho delle copie dell' handle, che sono diverse (infatti ho un file pointer diverso). Se invece voglio proprio duplicare un handle, devo usare la primitiva DuplicateHandle.
E' interessante perchè permette di ottenere una duplicazione dell' handle. Inoltre, posso specificare un target di processo sul quale fare una duplicazione. Usando la DuplicateHandle, copio un handle, e lo associo ad un altro processo.
C'è comunque una regolazione di questa associazione, perchè di fatto vado a modificare un'altro processo. 
CreateProcess(
	Handle hSourceseProcessHandle, handle del processo sorgente
	Handle hSourceHandle, handle da copiare
	Handle hTargetProcessHandle, handle del processo a cui voglio associare l'handle.
	LPHANDLE lpTargetHandle, pointer al nuovo handle.
	...
	)
Il file pointer è proprio lo stesso del primo handle.

Aprire un processo
===
Primitiva che mi permette di aprire un procsso ottenendo un handle che poi utilizzo per fare altre operazioni su quel processo:
OpenProcess
Sotto unix posso sapere al massimo il pid, e mandare i segnali.
Sotto windows invece posso avere un accesso che mi permette di fare diverse operazioni ad un altro processo, controllato dall' ID.
Specifico il processo uando l' id, e mi ritorna un handle.
Usando dwDesideredAccess posso richeidere varie cose:
 - SYNCHRONIZE: Sincronizzarmi col processo
 - PROCESS_QUERY_INFORMATION : richiedere informazioni sul processo

Esempio di utilizzo:
VOID ExitProcess (UINT uExitCode); Metodo con cui un processo si suicida, specificando un valore di ritorno (0 andato a buon fine.).

In Unix solo il genitore può sapere il valore di uscita di un processo. Due processi scollegati non possono sapere il rispettivo valore di uscita.
Su Windows invece, un altro processo può invocare la:
GetExitCodeProcess(HANDLE hProcess, LPDWORD lpExitCode) per chiedere il valore di uscita di un processo. Il valore di ritorno è specificato in lpExitCode.
Devo avere l' HANDLE, per averlo uso la OpenProcess. Se il processo è ancora attivo, GetExitCode ritorna false.
D'altro canto windows è costretto a tenere informazioni (es sull' exit code), finchè c'è almeno un altro processo che ha un handle aperto su quel processo.

BOOL TerminateProcess( HANDLE hProcess, UINT uExitCode): Per terminare un altro processo. Il processo terminate specifica l' exit code.
Su unix non si può settare l'exit code di un altro processo, perchè i processi sono molto più isolati (apparte rapporti familiari) un processo scollegato non può conoscere il valore di uscita e non può influenzarlo, e chiuderlo. 

Interazione con altri processi
====

Primitiva molto importante, sopratutto nei meccanismi di sincronizzazione, è la WaitForSingleObject.
DWORD WaitForSingleObject ( HANDLE hHandle, DWORD dwMilliseconds): attende per un temo che succeda qualcosa su un certo oggetto.
dwMilliseconds: E' un tempo max di attesa. Di solito coi processi, aspetto che un processo termini. Se impostato a 0, eseguo il polling. Non resto in attesa. In altri casi la applico ad esempio che un certo meccanismo di sincronizzazione mi dia l'accesso.

Ci sono casi in cui devo attendere eventi per più entità. Potrei fare dei loop, ma per una serie di ragioni esiste la variante che permette di specificare un gruppo di oggetti (sempre con timeout, che vale per ognuno), e specificando però un comportamento che dice o attenderli tutti (Es max 4 processi) oppure per la terminazione di almeno 1.
DWORD WaitForMultipleObjects(DWORD cObjects, LPHANDLE lphObjects, BOOL fWaitAll, DWORD dwMilliseconds);

Sotto a unix, c'è la wait:
wait (int *status) non specifica un timeout, ma ritorna dentro al puntatore il valore di uscita del processo di cui attendo la terminazione.
Nella sua forma classica, può essere utilizzata solo per attendere la terminazione di un figlio da parte del padre.
esiste la variante waitpid (pid_t wpid, int *status, int option) he specifica il pid, in status ho l'exit code e con le options posso specificare se è bloccante o meno (WNOHANG). La wait semplice è bloccante.

Variabili d'ambiente
===

Per ottenere una variabile d'ambiente si usa:

DWORD GetEnvironmentVariable(
	LPCTSTR lpName,
	LPTSTR lpBuffer,
	DWORD nSize);

Su unix invece si può usare la primitiva getenv(char *string),
Se vado a chiedere un valore di una variabile d'ambiente e la vado a mettere in un buffer non abbastanza capiente, accade un BufferOverflow.
Su unix getenv semplicemente, continua a scrivere nel buffer (causando anche problemi di sicurezza).
Su windows invece, si specifica nSize che sarebbe la dimensione del buffer che viene passato. Ritorna la lunghezza della stringa valore.
Se la lunghezza del valore di ritorno è più grande di nSize, allora windows si ferma ad nSize. Di solito chiamo la funzione specificando la lunghezza zero. Mi viene tornata la dimensione di cui ho bisogno, alloco lo spazio necessario ed invoco la funzione sapendo lo spazio necessario.

DWORD SetEnvironmentVariable(
	LPCTSTR lpName, //nome
	LPTSTR lpValue //valore
	);









