Gestione Memoria
===

`flushviewoffile`, permette di trasferire da memoria a disco le modifiche che ho fatto.

Se non lo utilizzo, ho una condiziione di apparrente inconsistenza. SE vado a vedere una locazione di memoria corrispondente al file, potrebbe avere un contenuto diverso rispetto a quello che ha il file su disco.

Prendono tutte le pagine di memoria modificate e le scrive sul disco. Non è una buona idea mischiare le due modalità di accesso ad un file.

Se uso il mapping in memoria, in quello stesso file dovrei evitare di fare anche operazioni di input output tradizionali. Questo perchè per 
essere sicuro della consistenza dovrei usare ogni volta il flushview.
Se uso il __file mapping__ dovrei evitare di usare primitive di i/o.

Condivisione di memoria
====
Avere due processi(o più), distinti (uguale sia win che unix). Gli indirizzi di uno non hanno senso per l'altro (a causa del sistema di sicurezza dell' os). Mi impedisce però di condividere informazioni.

Possono condividere parte di uno spazio di indirizzamento che credono appartenergli.
(Si potrebbe creare un problema di consistenza).
Uno dei modi per realizzarlo è usando il filemapping.

Si prende un file, si usa come base per un file mapping (posso usare un file reale, sia su win o unix un file relativamente irreale). Su win posso usare il file di paging, e su unix c'è n file particolare.

Creo il file mapping, e lo associo ad entrambi i processi. Ognuno dei due carica l'associazione nel proprio spazio di indirizzamento e ora il file appartiene ad entrambi gli spazio di indirizzamento, possono quindi leggere e scrivervi anche se in realtà vanno a leggere le stesse locazioni di memoria fisica.

Anche se probabilmente l' indirizzo virtuale è diverso nei due processi.

Due indirizzi di due spazi di indirizzamento diversi.Anche se diversi puntano alla stessa pagina di memoria fisica. E' la pagina che contiene parte del mapping di un file che si trova in un disco.

Bisogna anche tenere in considerazione la sincronizzazione di lettura e scrittura di due processi.


Sotto Unix/Linux
=======
L' idea è la stessa. Si prende un file, rappresentato da un descrittore. Questo file di cui ho il descrittore viene caricato in memoria tramite la mmap. Vi permetterebbe scegliere dove associare il file mapping anche se di solito è sempre NULL.

Ci ritornerà un indirizzo che decide il sistema operativo.
Su win nonc'è proprio la possibilità. 

Per liberare la parte di spazio di indirizzamento si utilizza la unmap. Per liberarlo, si deve specificare esattamente il puntatore che era stato tornato dalla mmap.

Per forzare l'aggiornamento si può utilizzare la msync.

Sotto unix, a differenza di win, bisogna distinguere il caso in cui si fa fork() o exec. Se al processo viene associato un file mapping, facendo la fork, il file mapping viene ereditato (come tutto lo spazio di indirizzamento) dal figlio. Quindi contiene implicitamente il file mapping.

Se un processo esegue  l'exec, non viene ereditato il file mapping. Exec non modifica ilpid, ma cambia il contesto anche i contenuti dello spazio di indirizzamento.

Posso utilizzare uno pseudo file /dev/zero, posso usare anonumous memory mapping che permette di specificare come descrittore -1 che mi permette di evitare di avere un file reale a cui associare un file mapping.

I file mappati in memoria si trovano in mezzo allo stack e allo heap

Utilizzo pratico delle DLL
====
Le librerie dinamiche che vengono utilizate per caricare delle funzionalità ed utility. Esistono dll di sistema e delle singole apps, associate a certi device ecc.
Ci interessano perchè sono un esempio di utilizzo pratico del file mapping.
Differenza fra libreria dinamica e statica

Librerie statiche:Normalmente abbiamo i singoli oggetti di una libreria che vengono utilizzati in forma statica.Ogni volta che la libreria viene usata dal programma, abbiamo una copia della libreria.
Questo costa spazio in memoria. PErmettono portabilità perchè compilo e poinonmi servono più

Librerie dinamiche: Funzionano diveramente. I singoli oggetti della libreria non sono caricati nell' eseguibile, ma sono soltanto indicati come locazione. Come se si facesse un caricamento ritardato rispetto all' effettiva esecuzione.

#Vantaggi:

* Non dovendo incldere i singoli oggetti abbiamo eseguibili di dimensione minore
* Se ho un'app di cui lancio molte istanze (o molti processi che la usano) posso caricare una volta la libreria in memoria e tutti quanti i processi che vogliono utilizzare uqelle libreri possono usarle dalla memoria e trovandole già caricate.
 * Se voglio aggiornare una libreria statica aggiornata, devo rifare il link e ricompilare tutte le apps.
 * Se ho una libreria dinamica, aggiorno la libreria e viene caricata aggironata. Tutti i programmi che vogliono usare la libreria la trovano già aggiornata.
 * Tutte le librerie di sistema sono librerie dinamiche.

Sotto windows abbiamo librerie basi, utilizzate da tutte le applicazioni e una marea di dll usate da sottoinsieme di applicazioni.

Uno dei probemi che si pone quando si usa una libreria condivisa, è la protezione dell' accesso del codice alla libreria da parte di più thread.

Se parlo di accesso da più processi distinti attraverso al file mapping non ho problemi. Quando ho un'accesso tramite dei thread.

I processi sono entità eseguibili con spazio di indirizzamento distinti e protetti
I thread sono flussi di esecuzione che condividono lo spazio di indirizzamento. Per questo motivo è facile capire che ci potrebbero essere dei problemi di accesso concorrente.

Le DLL dovrebbero tenere conto di qeusto potenziale problema e dovrebbe essere scritti in maniera thread safe.

#Implicit linking in Win32
File mapping con le dll, anche come si fa a scrivere una libreria dinamica per Win

Esistono due possibili alternative la prima, più utilizzata, implicit linking. Viene creata la libreria e caricata in maniera implicita quando parte il task che ha bisogno della libreria. Esiste anche una maniera esplicita ma è meno utilizzata.

Implicit linking: dipende dal compilatore che deve creare gli oggetti in modo particolare (la vediamo in microsoft c ma vale anche con mingw).
Tutti i simboli che io voglio i simboli che voglio includere nella libreria devono essere dichiarati usando ddl export.

Ho una funzione con un certo valore di ritorno, la devo modificare aggiungendo ddlexport

crea due file di output. Uno dll e no di estensione .lib. Quesllo con .ib viene utilizzato durante il link della applicazione, ma non è quello che poi viene caricato in memoria che invece è quello con estensione .dll.

L' applicazione che deve utilizzare l' entrypoint che fa parte della lbreria dinamica cdeve definire quell' entry point importato usando dllimport.

A runtime succede che il sistema quando deve far partire il nuovo processo che contiene dei simboli che si sa che devono essere importati, controlla se i simboli sono già stati caricati in memoria magari da altri proessi che lo hanno già utilizzati.

Se è la prima volta, il sistema <-, apre il file della libreria (CreateFile) lo mappa in memoria, associa il file mapping al processo che sta andando in esecuzione ( il file mapping entra a far parte dello spazio di indirizzamento del processo in esecuzione).

Arriva unnuovo processo che ha bisogno degli stessi simboli della stessa libreria, a questo punto il sys non ha più bisogno di caricare la libreria, ma usa il file mapping e lo associa al secondo processo e così via.

La conseguenza èche ognuno dei processi si porta la libreria nel proprio spazio di indirizzamento implicitamente, ma in realtà è sempre la stessa libreria.
Un'altra variante, si può utilizzare questo meccanismo anche per le variabili ma è pochissimo utilizzato a causa della consistenza rispetto all' accsso delle variabili.

# E' possibile anche farlo in maniera esplicita.
Linking esplicito. Non è più un modulo del sistema operativo che si occupa di caricare il file mapping inmemoria ecc,ma è la libreria che si occupa di caricare il file.

Per fare questosi può usare la primitiva load library: prende il nome del file della libreria).
Viene ritornato un oggetto HISTANCE che è un handle delle librerie (non è però un oggetto handle così). Può essere utilizzato per andare ad accedere a parti della libreria.
PEr eliminarla si può usare la FreeLibrary

Come si utilizza l' handle della libreria, si utilizza attraverso GetProcAddress. Prende un handle della libreria (a cui voglio fare accesso) ed un entry point. E' il nome della funzione che fa parte della libreria di cuivoglio l'indirizzo.
Se questa funzione è presente, getprocaddress ritorna l'indirizzo con farproc (processo lontano). Puntatore alla funzione.

Esempio atouel.c : fa una conversione andando a prendere un entry point da una libreria
	//Carico la libreria
	hDLL = LoadLibrary (argv [LocDLL]);
	if (hDLL == NULL)
		ReportError (_T ("Failed loading DLL."), 4, TRUE);

	/*  Get the entry point address. */
	
	//Dentro la libreria cerco l'entry point Asc2Un
	//pA2U è un puntatore
	
	pA2U = GetProcAddress (hDLL, "Asc2Un"); 
	if (pA2U == NULL) 
		ReportError (_T ("Failed of find entry point."), 5, TRUE);
	//Lo associo ad un altro puntatore,
	Asc2Un = (BOOL (*)(LPCTSTR, LPCTSTR, BOOL)) pA2U;

	/*  Call the function. */
	//Infine richiamo la funzione
	if (!Asc2Un (argv [LocFileIn], argv [LocFileOut], FALSE)) {
		FreeLibrary (hDLL);
		ReportError (_T ("ASCII to Unicode failed."), 6, TRUE);
	}

# Sotto a linux invece:
	dlopen
torna un puntatore generico (tipo handle), ad un entità di primitive della stessa famiglia.
usiamo dlsym a cui passiamo il puntatore ritornato. RTLD_LAZY (viene caricata la libreria solo quando faccio la ricerca del simbolo che viene effettivamente utilizzato)




