#Lezione 10 os

Tempo di esecuzione di un programma
===
Finchè esiste un processo che ha un handle verso un processo, il so deve mantenere le informazioni anche di quel processo.

    GetProcessTimes

Esempio: _CHAPTR07/TIMEP.C_

__Uptime__: Tempo che un processo è stato in esecuzione (Diviso in un tempo passato modalità kernel e utente)

__Elapsed time__: è il tempo di orologio o calendario, perciò rimane più tempo in esecuzione rispetto al tempo in cui viene eseguito. Questo a causa dello scheduling.

Generazione di eventi di controllo
===
Il cntrl-c è un esempio di categoria di eventi che sono meccanismi di interazione con un processo. Sono simili (ma diversi) dai segnali di Linux/Unix.

Gli eventi, come i segnali, si possono applicare ad un gruppo di processi. Questi eventi possono essere generati in maniera interattiva dall' utente, ma sotto hanno un meccanismo di sistema che genera l'evento e scatena il meccanismo di comunicazione.

Per generare gli *eventi* si usa questa primitiva:

    BOOL GenerateConsoleCtrlEvent(
        DWORD dwCtrlEvent,
        DWORD dwProcessGroupId
    }
    
**CTRL_C** e **CTRL_C_BREAK** sono gli eventi più usati. E' possibile specificare un processo leader di un gruppo durante la CreateProcess settando il flag CREATE_NEW_PROCESS_GROUP <- da verificare.



Terminale di controllo (in Linux), in Windows si parlerebbe di consolle. Parlerebbe perchè c'è una differenza fra i processi gestiti dalla UI e i processi che inveec sono gestiti direttamente nel caso da linea comandi o da equivalenti funzionalità messi a disposizione dalla ui.

Il terminale di controllo sono le entità a cui sono collegati i descrittori del mondo unix, handler in windows, che permettono di avere STDIN, STDOUT e STDERR. La consolle è dove viene inviato lo standard output di default.

Proprietà ereditate in unix da un processo figlio
===
Il processo figlio è una copia del padre che ricomincia da dove è stata chiamata la fork, le proprietà ereditate sono:

 * i descrittori (socket, segmenti di memoria condivisa ecc) 
 * Identificatori (utente processo ecc)
 * Informazioni di lavoro (cwd)
 * Ereditati i meccanismi di gestione dei segnali
 * Risorse ecc

Differenza fra processo padre e figlio
---
I lock non sono ereditati, 


Proprietà ereditate tramite ExecXX
---
 - PID e PPID,
 - Segnali Pending,
 - SE un processo fa fork, il processo che viene generato mantenga un lock altrimenti ci sarebbero due processi con un lock. In realtà però se il processo fa exec, in alcuni casi potrebbe non essere necesario che il processo che fa exec mantenga il lock.
A volte però è necesario che mantenga il lock. Esegue un lock, cambia contesto, ma deve mantenere la lock

Process Group e sessioni
---
Un process group è una collezione di uno o più processi (un gruppo di processi). Questi processi però vengono in realtà messi in relazione fra di loro. La più semplice è quella che si ottiene mettendo con la pipe in relazione lo stdin lo stdout di due processi.

*Esempio*:

    proc1 | proc2 &
    proc3 | proc4 | proc5
    
Questi due processi distinti, fanno parte di una sessione. Una sessione è una collezione di vari gruppi di processi, 

Normalmente tutti i processi fatti partire da una shell interattiva appartengono alla stessa sessione.

I deamon normalmente creano delle sessioni separate.

---

Gestione delle eccezioni
===
Meccanismo messo a disposizione a livello di sistema operativo. E' da certi punti di vista più generale, (si potrebbero applicare in altri contesti), ha una differenza fondamentale perchè non è gestito dal linguaggio.
Il linguaggio richiede la disponibilità di keyword che definiscono il blocco che può lanciare l'eccezione e il blocco che la gestisce, ma la gestione viene fatta a livello di sysop.

**Structured Exception Handling** sono le condizioni in cui l'app dovrebbe gestire le situazioni di errore, ma demanda la responsabilità al sistema.
Un esempio semplice è quello in cui si cerca di deferenziare un puntatore a null. 
Quello che si può ottenere è che venga chiamato una exception handlere prima della terminazione dell' applicazione.

L' eccezione è un concetto più generale di semplice condizione di errore, un errore è un problema locale: 
> Ho una funzione che per un qualsiasi motivo mi torna un valore inaspettato, o comunque tale che possa essere considerat un errore. Controlloquindi se c'è una condizione d'errore.
E' localizzato e sincrono. Ottengo un errore in quella funzione.

L' eccezione include la condizione di errore, ma può avvenire in punti inaspettati e sopratutto in momenti impredicibili. Ovvero non vado a controllare ed ottengo un eccezione, ma viene semplicemente lanciata.

Prima di tutto deve essere indivudato blocco di codice da controllare. 
Il blocco viene inserito in

    _try{
        /* blocco di codice da controllare*/
    }

L'eccezione viene gestita da un'altro blocco:

    _except(filter expression)
    {
        /* codice di gestione dell' eccezione */
    }

**Nota: Questa funzione non worka con MinGW! Accetta le keyword ma non le usa!**

Nel caso in cui ho una eccezione, il codice contenuto nel blocco di codice _viene mandato in esecuzione_.

Una volta che l' handler completa l'esecuzione, il controllo passa all' istruzione successiva al blocco dell' handler.

Il `filter expression` è valutata mmediatamente dopo l'eccezione. Può essere una costante, una espressione condizionale, o una chiamata ad una funzione.

Questa funzionenon è generica ma deve tornare uno di questi tre valori:

 - `EXECPTION_EXECUTE_HANDLER` : Esegui l'handler.
 - `EXECPTION_CONTINUE_SEARCH` : Segnale di saltare e andare al livello successivo nella gestione delle eccezioni. Meccanismo per gestire un annidamento di eccezioni. 
 - `EXECPTION_CONTINUE_EXECUTION` : Continua l'esecuzione dal punto in cui è avvenuta l'eccezione (es. `floating point exception`: cerco di fare la radice quadrata di un numero negativo. Normalmente vengono ignorate.)

Se ho provato a scrivere su una locazione di memoria non mia, non ha molto senso continuare l'esecuzione.

Per sapere chi ha scatenato l'eccezione, uso la funzione `GetExceptionCode()` viene usata all' interno dell' espressione filtro. Questa funzione può ritorna diversi valori, fra cui:
 - EXCEPTION_ACCESS_VIOLATION ...
 - ..

Un'altra caratteristica di questo meccanismo, non molto utilizzata, è la possibilità di _sollevare un'eccezione_. E' possibile sollevarla tramite la funzione `RaiseException`:

    void RaiseException(
        DWORD 
        DWORD 
        DWORD 
        const ULONG_PTR* lpArguments; //contatore di argomenti dell' eccezione
    );

Il meccanismo delle eccezioni ha un parente stretto, che è il gestore di terminazione.

Handler per la terminazione in Win32
---
Un termination handler, è simile ad un gestore di eccezioni ma viene eseguito sia quando un thrad lascia un blocco al termine della regolare esecuzione sia quando viene sollevata un' eccezione.
L' idea di questa variante è il blocco di codice finally verrà sempre eseguito:

    __try {
        /*blocco di codice */
    }
    __finally {
        /* handler di terminazione */
    }

Gestisce eccezioni ma non è _filtrabile_. Perciò è un meccanismo di gestione simile ma non uguale a quello di prima.

Il finally viene _sempre e comunque eseguito_, come se fosse la naturale prosecuzione del blocco precedente.

Non può essere usato assieme al gestore di eccezioni, ma si possono annidare.

Se ho una sorta di conclusione naturale, ci casco dentro (*falling throught*). Altrimenti è l' equivalente del blocco di eccezione

Esempio di annidamento:

    __try
    {
        while(..) __try
        {
            if(...) __try
            {
            
            }
            __except()
            {
            
            }
        }
        __finally 
        {
        }
    }
    __except() 
    {
    }
        
I termination handler non sono eseguiti se un processo  thread termina (via ExitXX o TerminateXXX). 

La funzione `atexit` è una funzione di Unix/Linux che permette di ottenere un effetto di conclusione del processo, controllato.
Dentro l'`atexit`,posso mettere una funzione che viene passato come argomento (prende in input un puntatore a funzione) che viene invocato quando esce. Questa funzione contiene tipicamente _una serie di manipolazioni che servono a manipolare le risorse ecc_

Questo è un meccanism generale del *processo*. Solo quando viene *terminato* viene attivato. La atexit deve essere registrata. Se il processo termina prima di aver invocato l'atexit la funzione invocata non viene mandata in esecuzione.

Se utilizziamo c++ sotto java, microsoft sconsiglia di cercare di combinare i due meccanismi di eccezioni di java e c++. Apparentemente funziona, ma le modalità di funzionamento entrano in conflitto di solito.

Sotto linux non c'è una gestione delle eccezioni, a volte vengono sfruttati i segnali come funzione di gestione degli errori. Si associa un handler ad un segnale, in modo che in caso avvenga una certa azione che provochi un segnale che quindi avvierà una funzione. Non esiste comunque un meccanismo a livello di sistema operativo.

Annidamento di eccezioni
---
Una singola eccezione può essere gestita da un solo handler. Una volta gestita, i blocchi che contengono l'annidamento che contiene tutti i livelli non viene più "onorato". Il primo handler che gestisce le eccezioni prende il controllo e non manda avanti la ricerca.
Si può comunque andare avanti nella gerarchia usando il valore `EXCEPTION_CONTINUE_SEARCH`. Se si raggiunge la fine, viene ignorata e viene usato il livello più esterno.

Eccezioni in C++ e Java
---
Il c++ offre tre operatori per la gestione delle eccezioni: `try`, `throw`, `catch`. 

L' utilizzo è il seguente:

    try
    {
        //blocco di codice
        throw exception;
    }
    catch( type exception)
    {
        //codice da eseguire in caso di eccezione
    }

Quì è il programma che solleva l' eccezione. A livello di sistema, è il sistema che richiede, a fronte di un' eccezione, che venga eseguito un blocco di codice. A livello di linguaggi di programmazione invece, è il linguaggio che se ne accorge.

catch può essere overloaded in modo da accettare diversi tipi come parametri. E' possibile definire un blocco catch che gestisce qualunque tipo di throw non importa il tipo, usando: `catch(...)`.

In Java le keywords sono le stesse, in java per gestire le eccezioni possiamo usare classi e sotto classi. Normalmente c'è una gerarchia che gestisce dalla class geenrale exception, ed anche in questo caso uno solo viene eseguito.

Console Control Handler
===
E' possibile generare gli eventi, sotto windows abbiamo per eventi come quello dell' invio di un comando come cntrl-c, è possibile associare dei gestori simili agli handler che si hanno sotto unixlinux.
Si specifica un puntatore a funzione, e tramite:

    BOOL SetConsoleCtrlHandler (
    PHANDLER_ROUTINE HandlerRoutine,
    BOOL Add
    );

Quando arriva il segnale, invoca l' handler. Però in questo caso, è possibile avere più handler. In Unix/Linux abbiamo un handler per segnale, ma non possiamo avere due handler per lo stesso segnale.

Sotto Windows invece, possiamo avere un accodamento di handler: se specifico `Add` a TRUE, posso invocare più handler. Annido l' esecuzione di handler, e vengono chiamai in ordine inverso in base a come sono inseriti.
Questi handler, vengono eseguiti in thread separati.





Gestione dei segnali Unix/Linux
===
La gestione dei segnali in Unix
