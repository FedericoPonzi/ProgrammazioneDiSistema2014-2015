Lezione 11 - Programmazione di sistema

Gestione dei segnali sotto unix/linux
===
A differenza di Windows, non esiste un effettivo sistema di gestione delle eccezioni su Linux. Quello che si può fare è gestire i segnali in Unix che, anche se sono supportati dal sistema operativo, viene in gran parte gestito nella libreria C.
La funzionalità più simile ai segnali Unix in Win32 sono __console control handlers__.
In Unix, la gestione del segnale non avviene a livello di thread, ma a livello di processo.
Esistono diversi tipi di segnali, la maggior parte sono associati a degli errori (o comunque non normalità dell' esecuzone). In verità comunque il segnale è un meccanismo di comunicazione ( IPC) con cui appunto si invia una informazione. La ricezione corrisponde all' esecuzione dell' handler.
Esistono __32 segnali__ di Unix/Linux, per un buon motivo che vedremo più avanti.
Esistono segnali che invia il sistema operativo, e segnali che vengono inviati dai processi (o comunque non dal sistema operativo).
Alcuni *segnali Unix* corrispondono ad *eccezioni in Win32*:

 - SIGILL - EXEPTION_PRIV_INSTRUCTION - Tentativo di effettuare un' azione privilegiata da parte di un processo che non ha i privilegi.
 - SIGSEGV - EXCEPTION_ACCESS_VIOLATION. Segnale di violazione di memoria (accesso).
 - SIGUSR1 e SIGUSR2 - Eccezioni generate dall' utente. Sono proprio due segnali usati per comunicare. 

Win32 non genera SIGILL, SIGSEGV, SIGTERM e non support SIGINT.

Gestione dei segnali in Unix
=
Il segnale è un sistema di comunicazione. In generale può essere sincrona o asincrona.
 - Sincrona: Ci sono due entità che devono comunicare, una entità comunica e l'altra deve essere disponibile a ricevere il contenuto della comunicazione. Quindi si sincronizzano sulla comunicazione.
 - Asincrona: Ci sono due entità che devono comunicare, una entità comincia a comunicare inviando un' informazione. Il destinatario, può non ricevere subito l' informazione. Ad un certo punto il destinatario riceve il messaggio, e lo va a ricevere.

Questo implica due cose: chi invia non sa se/quando riceve l' informazione.

<h4>Che succede quando viene inviato un segnale? </h4>
Esempio: una cpu, un processo A vuole comunicare col processo B.
Per farlo, ha bisogno del controllo della CPU. Vuol dire che manda il segnale quando ha il controllo della cpu.
Il processo B non è in esecuzione, quindi la ricezione non può essere sincrona. Se i processi sono 150, non è neanche scontato che il processo B venga eseguito proprio dopo il processo A.

Quando il processo B entra in esecuzione, può ricevere il segnale. Dove viene quindi memorizzato questo segnale?
Viene scritto in una _maschera_ (o buffer di informazione), che è parte della struttura che rappresenta un processo.
Una parte della pcb è la maschera dei segnali, quando si invia un segnale si imposta un bit a 1.

<h4>Come si controlla questa informazione?</h4> 
Il processo, legge la sua maschera quando esce dallo stato kernel per entrare nello stato utente. Tutti i processi fanno questo passaggio prima o poi.
In quel momento il processo può di fatto saltare all' handler.
Più avanti vedremo che succede quando c'è più di un segnale da gestire.

Gestione dei segnali
=
I segnali hanno delle **azioni**. L' azione di default è quella di terminare il processo.
Alcuni segnali (pochi) hanno come azione quella di essere *ignorati* (SIGCHILD). Perchè il processo non li deve gestire, in caso deve installare un handler.

Tramite la funzione **signal** o **sigaction**, si richiama una funzione. Esiste una funzione `signal` per gestire uno o più segnale.
Quando il segnale arriva, viene invocata la funzione. Quando termina la funzione associata come handler, il processo prosegue l'esecuzione come se fosse stato invocato il segnale immediatamente (sincrono).

<h4>Cosa si fa nell' handler?</h4>
Si possono fare diverse azioni, a seconda del segnale. Ad esempio, in un segnale per un errore critico l'handler si limita a dare delle informazioni per capire meglio il problema e terminare il processo.

In altri casi, come in `sigusr1` o `sigusr2` potrebbe fare diverse azioni. Una volta che l' handler termina il processo continua la propria esecuzione.

Un esempio di handler reale: i deamon di solito hanno file di configurazione. Si invia un segnale al processo, questo ha come handler associato al segnale la funzione di aprire il file di configurazione e rileggere i contenuti.
Quale segnale dipende ovviamente da deamon a deamon. Potrebbe inoltre fare diverse configurazioni (es: un webserver deve avviare istanze particolari).  Come handler termina la sua esecuzione e riparte aggiornato.

Come viene gestito l'handler
=
Originariamente, esisteva un comportamento di questo tipo: quando si installava un handler, la corrispondenza valeva una volta sola. Inviato un segnale, invocato l'handler, rinviato il segnale l'handler non era più valido e veniva avviata l' azione di default.
Quello che si faceva come soluzione: la prima azione che faceva l' handler era reinstallare se stesso.
La `signal` moderna evita questa necessità: nonostante questo, comunque da molti anni la signal non viene molto utilizzata. Viene usata di più la `sigaction.`
Oltre a non aver bisogno di reinstallarsi, ha altre caratteristiche diverse dalla signal che vedremo dopo.
Quando l' handler di un certo segnale è in esecuzione, quello stesso segnale viene bloccato dal sysop. Se quello stesso segnale viene invocato mentre l' handler è in esecuzione, non viene reinvocato l' handler. Il segnale è bloccato, come lo sono anche gli altri segnali specificati in una certa maschera.
Esiste una maschera generale che contiene un bit per ogni segnale.
Questa è la maschera dei segnali *pending*. Se non ha nessun bit impostato, significa che nessun segnale è pending.
Ci possono essere più segnali pending.
Oltre a questa maschera, c'è la maschera dei segnali bloccati che dice quali segnali non devono essere onorati/gestiti, per le più svariate ragioni (di solito per non interrompere l'esecuzione di un altro handler).
Quest'ultima maschera, può avere un qualunque numero di bit impostati nonostante segnali pending.
La maschera dei segnali bloccati, c'è sempre il segnale associato all' handler in esecuzione.
> Es: sigusr1, viene avviato l' handler, mentre è in esecuzione rimando il segnale, questo viene bloccato comunque perchè è in esecuzione.
Mentre questo handler è in esecuzione, posso decidere di bloccare altri segnali (es sigusr2).

Sul singolo segnale, non possiamo accodare eventi. Su segnali diversi si, sul singolo no. Il singolo segnale è rappresentato da un bit, quindi se è pending vale _1_.

In *POSIX*, non c'è un ordine specificato per la gesitone dei segnali. 

>Se ho 4 segnali pending, non esiste una regola che mi dice quale dei 4 verrà eseguito prima.


Signal handling

	struct sigaction {
		void (*sa_handler) (int) //puntatore a funzione che fa da handler
		siget_t sa_mask; //maschera Segnali da bloccare
		int sa_flags;
		void (*sa_restorer)(void);
	}
	
	int sigaction (int signum, const struct sigaction * act, struct sigatcion *oldact);

Esiste una azione di default, e la possibilità di ignoare il segnale usando la macro SIG_IGN.
Esistono due eccezioni a questa regola (dell' handler): `SIGSTOP` e `SIGKILL` non possono essere intercettati.

 - SIGKILL: termina il processo.
 - SIGSTOP: ferma il processo. Rimane vivo ma non può più andare in esecuzione (come se fosse tolto da Runnable). Finchè non arriva SIGCONT (che lo rimette nella lista dei Runnable).

Alcuni flags:
 - SA_SIGINFO: vi permette di avere informazioni aggiuntive, all' handler viene passato un puntatore ad una struttura chiamata sign _info che contiene alcun informazioni. La più comune è trovare l' indirizzo di una accesso fuorimemoria.
 - SA_ RESTART: Interoperabilità delle syscall. Una syscall che viene interrotta. Potrebbe essere lasciata proseguire nella sua esecuzione come se non fosse arrivato il segnale, altrimenti con questo flag posso richiedere che le syscall vengano fatte ripartire.

I segnali sono di un certo numero, almeno 32, possono essere trovati in un file chiamato signum.

Altre funzioni:

 - `int sigprocmask ( int how, const sigset_t *set, sigset_t *oldset)` la maschera dei segnali bloccati è manipolata tramite questa funzione. Si possono fare diverse azioni.

 - `int sigpending(sigset_t *set);` Se un processo vuole sapere se ha segnali pending, può usare la funzione   ha senso solo per il processo stesso. Può sapere i suoi segnali pending. Non è molto utile ma è comunque una funzione disponibile.

 - `int sigsuspend ( const_sigset_t * maskj)`; funzione d'emergenza di sigprocmask.
 - `unsigned int alarm( unsigned int seconds)` : Funzione che fa da sveglia. Mi permette di passare un numero di secondi e, dopo quel numero di secondi il sistema manda un segnale al processo (`SIGALARM`).
E' un modo per effettuare un'attivazione a distanza di tempo di una certa funzionalità. Alarm ovviamente torna immediatamente. Dopo seconds, arriva il segnale sigalarm.
Un difetto di questa primitiva è la scarsa granularità, perchè utilizza il secondo. Non posso usare alarm per dimensioni < dei secondi. Altro difetto è la scarsa precisione: 

>5 secondi, sono 5 secondi con granularità del clock del sistema. Quindi 5 secondi col delta dato dal clock del sistema. Questa delta vale circa 1/100 di secondo.

Esistono altri meccanismi che lavorano col timer, ma usano meccanismi simili alla Sleep. 

Come inviare un segnale
=
Alcuni processi sono inviati dal sysop, alcuni da processi, ma in tutti i casi vengono inviati usando la funzone kill che invia un segnale ad un processo o ad un gruppo di processi.
`int kill (pid_t pid, int signo)` Prende il pid del processo, ed il segnale che deve essere inviato.
Se voglio segnalare più processi devo inviare un segnale per processo.
Il pid_t ha diverse interpretazioni (o diversi effetti a seconda del valore).

 - `pid>0` : Il segnale è inviato al processo pid (cioè che ha esattamente quel pid).
 - `pid ==0` : Il segnale è inviato a tutti i processi che fanno parte del gruppo a cui appartiene il processo invocante (pid==0 _teoricamente_ non esiste). A tutti quanti viene inviato lo stesso segnale.
 - `pid==-1` : Il segnale viene inviato a tutti i processi tranne quello con pid 1 (int). E' un modo per uccidere tutti i processi tranne l'1.
 - `pid<-1` : Il segnale è inviato ad ogni processo nel gruppo -pid.

Si può inviare un segnale 0 (che non esiste), che viene usato per controllare se un processo è ancora attivo tramite il valore di ritorno della kill.
Se voglio mandare un segnale a se stesso (tipo alarm, ma lo fa ritardato nel tempo), posso usare la funzione `raise` (non viene molto utilizzato).

I segnali possono sempre venire invocati da processi ownati dallo stesso utente,  per poter inviare i segnali deve essere l' effective o real uid dell' utente del real o saved id del ricevente.
L' utente root non ha questa forma di controllo. Un processo avviato come root può inviare segnali a qualunque processo.

`goto` non locale
-
Un goto locale avviene all' interno della stessa funzione. Quello che adesso vedremo è un goto non locale: posso andare dentro ad un'altra funzione, purchè abbia fatto alcune operazioni (è un goto all' indietro in un certo senso).
Il linguaggio C non permette un goto ad un punto (label) appartenente ad un' altra funzione.
Questo tipo di salto incondizionato è reso possibile dalle funzioni setjmp o longjmp.

Setjmp(jmp_buf env) è la funzione invocata per definire un punto di ritorno, è una sorta di marcatore. All' interno della struttura salva delle informazioni generalmente di stack che mi permette di tornare ad un certo punto del flusso di esecuzione.
Quando voglio effettuare il goto non locale, invoco longjump(); che mi riporta al punto che avevo salvato con setjump.

>Es: gestione degli errori in situazioni in cui c'è una esecuzione molto annidata (voglio avere una gestione degli errori centralizzata). Usando questo unito ai segnali, ho una specie di gestione delle eccezioni.

Come funzionano i valori di ritorno di setjump:

    #include <setjmp.h>
    int setjmp(jmp_buf env); //Se va a buon fine, mi ritorna 0.
    
    void longjmp (jmp_buf env, int val);

Quando si fa il goto non locale, il valore di ritorno non può essere 0 perchè non so se sono tornato lì a seguito di un jump oppure se ho impostato con successo l'etichetta.

	#include	<setjmp.h>
	#include	"ourhdr.h"

	static void	f1(int, int, int);
	static void	f2(void);

	static jmp_buf	jmpbuffer;

	int
	main(void)
	{
		int				count;
		register int	val;
		volatile int	sum;

		count = 2; val = 3; sum = 4;
		if (setjmp(jmpbuffer) != 0) //Ho impostato il punto di ritorno
		{
			//Non entro in questo blocco la prima volta
			
			printf("after longjmp: count = %d, val = %d, sum = %d\n",
					count, val, sum);
			exit(0);
		}
		count = 97; val = 98; sum = 99;
					/* changed after setjmp, before longjmp */
		f1(count, val, sum);		/* never returns */
	}

	static void
	f1(int i, int j, int k)
	{
		printf("in f1(): count = %d, val = %d, sum = %d\n", i, j, k);
		f2();
	}

	static void
	f2(void)
	{
		//longjmp(jmpbuffer, 0); //Non funziona perchè se torna zero, quando entra nella condizione dell' if torna 0 e quindi prosegue.
		longjmp(jmpbuffer, 1);
	}
