#Ancora sui sockets

I principali tipi di socket sono gli stream e datagram che si interfacciano rispettivamente con TCP e UDP.
I rawsockets sono un tipo di socket che si interfaccia al protocollo di livello più basso (network - IP).
L' interfaccia di programmazione dei sockts è molto vecchio stile: i dettagli sono molto importanti.
La primitiva fondamentale è la primitiva

    int socket(PF_INET, SOCK_STREAM, 0)

Ritorna un socket descriptor (simile a file descrpitor), sotto windows eisiste un tipo speciale chiamato **SOCKET** (int) per indirizzare i socket.
In questo caso viene creato un socket di tipo stream, nel dominio di internet, e l' ultimo argomento specifica il protocollo da associare al socket. 0 di solito, e significa una wildcard che singifica demandare al os di far scegliere il protocollo più adatto. 

>Es: sock_stream verrà scelto tcp. Potrei specificare 16 per tcp al posto di 0 ma non lo fa nessuno.

I socket di tipo stream è quello che gestisce la comunicazione a flusso che è quella associata al protocollo tcp e si può pensare all' equivalente alle pipe sotto windows, ovvero pipe fra sistemi distinti. La caratteristica principale della socket è che permette la comunicazione fra processi su sistemi distinti.
Per il datagrama è uguale, tranne che si imposta `SOCK_DGRAM`. E si imposta a 0 per far scegliere all' os il tipo di protocollo.

Il socket di tipo stream offre un meccanismo affidabile, il socket_dgram non affidabile. Nel primo caso, che il flusso di dati è inviato in ordine, non duplicato e non corrotto.

Comunicazione non affidabile significa che non è garantito che i pacchetti arrivino non in ordine, non duplicati e non corrotti a livello di **contenuto**.

Affidabile non vuol dire che se invio un messaggio sicuramente il ricevente lo riceve, è come una pipe: io scrivo e se nessuno estrae i dati questi non sono effettivamente letti.

I socket di tipo stream e datagram possono essere usati da qualunque applicazione, quelli di tipo raw possono essere usati solo da utenti di tipo root.
La sintassi è identifca tranne SOCK_RAW.

Per ora abbiamo creato solo un descrittore di socket, per comunicare abbiamo bisogno di più di informazioni: indirizzo ip sorgente/destinatario, Porta destinatario/mittente.

Per l' indirizzo del socket viene usata una struttura generica `sockaddr` che contiene questa quadrupla.
Il modo di gestire la differenza di indirizzi è poco elegante, vediamola in dettaglio

In generale esiste una struttura chiamata sockaddr:

	sockaddr{u_short sa_family; char sa_data[14];};
Questo indirizzo in questa forma non viene mai utilizzato.
Sa_family è il dominio, sa_data contiene l'ip collegato all' indirizzo.

in unix

	struct sockaddr_un{short sun_family; char sun_path[108];};
Internet:

	struct in_addr { unsigned long s_addr;}
	struct sockaddr_in { 
	short sia_family; u_short sia_port;
	struct in_addr sin_addr; char sin_zero[8];
	};
C'è sempre nuna parte generica ed una specifica di unix e di internet. Nel dominio internet la parte specifica è quella che è la parte di indirizzo internet (numero porta indirizzo ip, più un certo punto di byte di padding).

Utilizzo:

si definisce l' indirizzo che veramente serve, vado a riempire il dominio dell 'indirizzo internet
definisco ip, numero di porta, famiglia.

	memset(%server, 0, sizeof(server));
	server.sin_addr.s_addr = htonl(INADDR_ANY); //host to network long
	server.sin_family = AF_INET;
	server.sin_port = htons(s_port); //host to network short
	
	if(bind (s, (struct sockaddr *) &server, sizeof(server))<0)
	{
		...
	}

Quando vado ad usare il `bind`, faccio il cast al dominio generico, più la dimensione specifica dell' indirizzo.
htonl e htons: è un formato neutrale network (che coincide col big endian) e quando faccio operazioni di questo tipo devo usare il formato neutrale. Usato per far comunicare due sistemi con diversa endianes.

##Sequenza di chiamate dei sockets per stabilire di una connessione
Per socket di tipo stream. Poi vedremo per i datagram (dove non c'è un concetto di connessione).
Di solito in questo tipo di socket si usano i termini server e client: il server(passivo) attende la richiesta da parte di un client (attivo).

La sequenza di chiamate fra i due lati sono diverse. Una volta stabilita la connessione, questa è bidirezionale (è come una pipe bidirezionale).

Creo in entrambi i lati il descrittore usando socket(). Poi devo prendere il socket un entry point per il livello di comunicazione e gli devo assegnare un indirizzo.
Dal lato server lo facio tramite una bind.
Dal lato del client, la bind è opzionale: viene fatta se non fatta esplictiamente, quando si invoca connect.
Il server deve obbligatoriamente fare il bind ma non è tenuto.
La successiva primitiva alla bind è la listen(). Apparentemente è inutile: faccio bind e poi faccio accept, intuitivamente.
A che serve listen quindi? In realtà l' interfaccia non è particolarmente disegnata bene, il ruolo che normalmente associamo ad accept lo svolge la listen.
La listen rende il socket disponibile ad accettare richieste di connessione. Una volta che listen() torna con successo, quel socket è effettivamente pronto a ricevere una richiesta di connessione.
Invocata la listen, rendo il socket disponibile, quando arriva la richiesta il socket completa il passaggi odi informazioni per definire la connessione e a quel punto se voglio usarko per scambiare dei dati uso la primitiva accept(), che prende in input in un socket e mi restituisce un socket che mi permette di scambiare i dati.
Il socket che ho dato in input può essere utilizzato per accettare altre connessioni.

Vediamo in dettaglio:

	int listen(int s, int backlog);

Listen prende in input il socket descriptor creato e associato all' indirizzo con la bind, e prende un valore di backlog: un numero che specifica quante connessioni posso accodare.
Questo già fa capire il significato di listen: mi permette di accodare richieste alla socket, che poi gestirò con la accept.
Il limite di backlog esiste perchè potrebbe succedere è che arrivano moltissime richieste da parte di client,che vogliono cercare di connettersi a questo socket. Se non esistesse un limite tale per cui le richieste in eccesso fossero eliminate, succederebbe che si esaurirebbero le risorse del sistema molto in fretta.
La coda viene svuotata quando viene chiamato accept: si và a prendere la prima richiesta che si trova nella coda finchè non si svuota.
Finchè non viene invocata la listen, il tcp non attiva il meccanismo di 3-way handshake. 

	int accept( int s, struct sockaddr *addr, int *addrienptr);

Gli passo il descrittore, e se l'accept trova una connessione in quella coda, mi torna un nuovo descrittore. Se non ci sono richieste di connessione già completate nella coda, l'accept è bloccante.
Torno un nuovo descrittore perchè voglio comunicare con quella specifica richiesta di connessione, ma voglio mantenere il descrittore originale per accettare nuove richieste di connessione.

Non posso usare il primo descrittore per scambiare i dati, ma devo usare un descrittore definito da accept.
Abbiamo detto che c'è un indirizzo addr: questo indirizzo può, se gli passo un puntatore a sockaddr, essere usato per conoscere l'indirizzo del sistema dall'altro lato della connessione.
Potrei usare null e funziona comunque: se voglio conoscere ip e port da cui mi arriva la connessione devo inserire addr.
In addrienptr viene scritta la dimensione dell' indirizzo.

#####Ancora su listen
A livello kernel, esistono due code: la coda delle connessioni complete e pronte per l'accept, e una coda per le connessioni incomplete.
La seconda è una coda per le quali il 3WHS è in corso. Quando viene completato, si prende quella entry e si sposta nella coda delle connessioni complete.
Se si va a vedere lo stato del socket, quelli per cui il 3WHS stanno nello stato established anche se non è stata chiamata la accept.

L'argomento backlog è la *somma* delle entry nelle due code.
Se le code sono piene, non può più essere accettata una connessione e quello che si fà è ignorare la richiesta: non viene inviato un pacchetto RST se la coda è piena.

Viene inviato se viene ricevuta una richiesta di connessione e il socket non è nello stato listen (connection_refused), oppure il tcp vuole abortire una connessione esistente, o il tcp riceve un segmento per una connessione che non esiste.

Quando il 3WHS si completa, la connssione è pronta e il lato client appena il 3WHS è completa può cominciare a mandare dati, anche se dal latoserver non ho ancora usato la accept.
I dati inviati da un client, vengono accodati in un buffer specifico del socket. Ogni socket per cui è stato fatto l'accept ha dei buffer (Sia di ricezione che di invio) per rimanere in attesa.

###Operazioni tipiche nel client
Il client utilizza direttamente connect() che associa al socket l' indirizzo ip e la porta di destinazione, ma implicitamente lega l' indirizzo ip locale e la porta locale. La porta viene scelta dal sistema.
La connect permette di specificare l' indirizzo ip e la porta 

int connect (int s , struct sockaddr *addr, int addrien);

Ci permette di associare a sckaddr l' indirizo ip e porta di destinazione: in questo caso non è opzionale, ma necessario perchè altrimenti la connect non sa a chi deve connettersi. Addrien non è indirizzo ma valore di addr.
La connect scatena la 3WHS.
E' apparentemente bloccante: attiva il meccanismo, e a seconda dello stato a lato server può completarsi in maniera quasi instantanea, ma se non c'è spazio nel backlog però potrebbe prendere del tempo lungo.
In alcuni casi torna immediatmaente e con successo. In altri casi torna dopo un certo intervallo di tempo con successo, in altri casi torna dopo un intervallo di tempo con errore (tipo cerco di fare connect ad un indirizzo non disponibile, o non c'è una socket sulla porta pronta da aspettare).
La connect non torna un nuovo descrittore, la connect torna un valore che indica se l' operazione è andata o no a buon fine.
Un' altra caratteristica è che il socket che passiamo come argoment, una volta utilizzato non può più essere utilizzato in nessuna forma.
Se voglio riprovare a fare connect dopo che è fallita, `non posso` usare lo stesso descrittore. Devo creare un nuovo socket.

Si possono usare le primitive send() recv() per inviare ricevere dati, ma in lnux si può usare anche readwrite come se fosse un file.
Prima delle primtivie di invio, vediamo un esempio di connessione e come risolvere il problema seguente: dove lo trovo l' indirizzo ip?

Per quanto riguarda le porte, esistono dei numeri prefissati. Per l' ip, si può:

 - Usare il DNS,
 - Usare il file hosts.

Per richiedere l 'indirizzo ip di un host, si possono usare delle primtive ad hoc:

	struct hostent *gethostbyname(char *name);
	struct hostent *gethostbyaddr (char *, int len, type);

Ha delle limitazioni, ma è più facile. Più avanti ne vedremo un'altra più flessibile. Gli passiamo il mnemonico sito web, e lui torna un puntatore ad una struttura contenente una serie di informazioni.

	struct hostent {
		char *n_name;
		char **h_aliase;
		int h_addrtype;
		int h_lenght;
		char **h_addr_list; //Lista di indirizzi ip da usare in bind e connect
	};
A volte gli hostname ip sono mantenunti in **etc/hosts**. E' possibile anche specificare l'ordine di cercare prima in dns e poi in hosts.

##Esempio di connessione via socket
Da stevens

	#include	"unp.h"
	#include	<time.h>
	
	int
	main(int argc, char **argv)
	{
		int					listenfd, connfd;
		socklen_t			len;
		struct sockaddr_in	servaddr, cliaddr;
		char				buff[MAXLINE];
		time_t				ticks;
	
		listenfd = Socket(AF_INET, SOCK_STREAM, 0);
	
		bzero(&servaddr, sizeof(servaddr));
		servaddr.sin_family      = AF_INET;
		servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
		/* INADDR_ANY Una macro, è legato alle interfaccie disponibili sul sistema che invoca questo sistema. Chiunque usi l' interfaccia virtuale ha anche un indirizzo di loopback. Con questa mcro, posso ricevere richieste da qualunque interfaccia di rete. Potrei anche utilizzare un indirizzo locale. Nel 95% dei casi si usa questa macro. Se voglio solo connessioni locali uso 127.0.0.1.*/
		servaddr.sin_port        = htons(13);	/* daytime server */
		
		Bind(listenfd, (SA *) &servaddr, sizeof(servaddr));
		//Chiamo il listen e rendo il socket raggiungibile
		
		Listen(listenfd, LISTENQ); //LISTENQ = 1024. quindi accetta max 1024 connessioni. Definito nell' include.
		/* Il numero storico era 5 LOL */
		
		//Ciclo principale del server:
		for ( ; ; ) {
			connfd = Accept(listenfd, (SA *) NULL, NULL);
	
	        ticks = time(NULL); //prende il tempo
	        snprintf(buff, sizeof(buff), "%.24s\r\n", ctime(&ticks)); //Usa ctime per rednere il tempo leggibile
	        Write(connfd, buff, strlen(buff)); //Uso il write per scrivere sulla socket come se fosse un file.
			Close(connfd);
		}
   	}

> In questo caso, usare snprintf va bene ma in generale non è un sistema
> che funziona perchè sul socket posso inviare un flusso di byte
> generico che non ha nulla a che fare con il concetto di stringa.

Vediamo il corrispondente client.

    #include	"unp.h"
    
    int
    main(int argc, char **argv)
    {
    	int					sockfd, n;
    	char				recvline[MAXLINE + 1];
    	struct sockaddr_in	servaddr;
    
    	if (argc != 2)
    		err_quit("usage: a.out <IPaddress>");
    
    	if ( (sockfd = socket(AF_INET, SOCK_STREAM, 0)) < 0)
    		err_sys("socket error");
    
    	bzero(&servaddr, sizeof(servaddr));
    	servaddr.sin_family = AF_INET;
    	servaddr.sin_port   = htons(13);	/* daytime server */
    	if (inet_pton(AF_INET, argv[1], &servaddr.sin_addr) <= 0) //inet_pton è come gethostbyaddr
    	
    		err_quit("inet_pton error for %s", argv[1]);
	    //Quando chiamo connect, resta in attesa per il 3WHS
	    //Se va bene torna 0:
    	if (connect(sockfd, (SA *) &servaddr, sizeof(servaddr)) < 0)
    		err_sys("connect error");

	    /*
		    Per questo client, va direttamente in attesa.
		    Normalmente comunque si invia una richiesta.
    	while ( (n = read(sockfd, recvline, MAXLINE)) > 0) {
    		recvline[n] = 0;	/* null terminate */
    		if (fputs(recvline, stdout) == EOF)
    			err_sys("fputs error");
    	}
    	/* Si fa un loop finchè questa non torna ciò che mi aspetto*/
    	if (n < 0) //Se torna un numero negativo, ERRORE.
    		err_sys("read error");
    
    	exit(0);
    }

