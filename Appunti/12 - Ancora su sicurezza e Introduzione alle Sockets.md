##Lettura ed aggiornamento dei security descriptor
Si possono leggere gli attributi di un file uso `GetFileSecurity`
Per associare gli attributi di sicurezza uso `SetFileSecurity`.

Esistono funzioni equivalenti da applicare a degli oggetti kernel, getsetKernelObjectSecurity.
I private object sono una variante di oggetti kernel, e sono oggetti kernel non nativi di windows ma inseriti successivamente come addon.
> Ad esempio, i socket sono private object.

Windows ci mette a disposizione altre routine di utilità:

 - Per l' owner : GetSecurityDescriptorOwner
 - AcL: getSecurityDescriottDacl
 - ...

Uso getAce per accedere alle singole ace, dato una ACL (GetAclInformation).

##Modello di sicurezza di Unix/Linux
Modello non discrezionale, ed è più o meno lo stesso dall' epoca di Unix (fine anni 60); è quindi un modello pone dei problemi.
Ogni processo ha due utenti associati:

 - Real user id
 - Effective user id : è quello che realmente viene utilizzato per stabilire se un processo ha i privilegi o no per fare una certa azione. (ad esempio nell' accesso a file di sistema).

Nella maggior parte dei casi, i due id coincidono. Nel caso in cui sia necessario usare dei privilegi diversi da quelli dell' utente o privilegi particolari, si attiva il meccanismo usando set user id, e il processo prende l' id del proprietario del file.
Quello che è interessando, è considerare ocsa succede in casi particolari:

 Cosa succede quando parliamo di permesso di esecuzione per quanto riguarda una directory. (esecuzione su file, può creare un processo).
 Ma su directory?
 Dà il significato di attraversabilità: possiamo attraversare una directory solo se quella ha il bit di eseguibilità.
 SE voglio raggiungere un certo file che stà in una cartella annidata quattro livelli sotto la home di un utente, devo avere il permesso sia come utente proprietario sia come utente reale, devo avere il permesso di eseguibilità per tutte le cartelle e sottocartelle.
Per rendere un file condivisibile ad altri utenti nella stessa macchina (multiutente), si deve usare chmod e dare il permesso di eseguibilità.
E' importante anche per le cartelle il permesso di scrivere in una cartella, oppure posso cancellare una file in una cartella (cancellare associazione fra nome inode file).

`chmod` si applica al file e `fchmod` si applica al descrittore del file.
Per permettere che l'operazone vada a buon fine, solo l'utente privilegiato(può modificare i permessi di qualcunque file di qualunque utente) o il proprietario può modificare i permessi del file.

umask è un meccanismo che serve a definire dei modi di default con cui viene creato un file.
Se specifichiamo una umask 022 indichiamo che permessi di scrittura sono disabilitati per group e other.

Se un utente crea un file, e l'utente viene eliminato, il file rimane senza proprietario.
Se un utente crea un file, non può cambiare il proprietario di un file. Questo perchè il proprietario del file è responsabile di questo. Solo il superutente può cambiare l'ownership di un file (usando `chown`).

#Lo sticky bit
L' ultimo bit speciale nella struttura dati che descrive i permessi di un file è quello sticky bit il cui signficato dipende dal tipo di file.
Di solito viene semplicemente ignorato, ma ha importanza a volte.
Ha un significato diverso a seconda venga applicato ad un file o una directory.
Se associato ad un file regolare, storicamente aveva un senso che una volta che quel file veniva caricato in memoria doveva rimanerci anche se magari non era utilizzato (prima veniva utilizzato per programmi eseguibili).
Ad una **directory** invece, applicando lo sticky bit implica che per quella directory un utente può cancellare un file soltanto se è il proprietario del file, se è il proprietario della directory oppure se è il SU. Viene usato per esempio all' interno della directory **tmp**, **mailbox** (sono directory che devono essere lette e scritte dagli utenti che la utilizzano).

##Modificare l' uid
setuid è una primitiva che ci pemrette di modificare il real e l' effective UID.

#Socket
Un meccanismo di comunicazione, proposto in BSD Unix 4.2 e sono ormai suppportate ormai da tutti i SO compreso Windows, che all' inizio li supportavano in maniera maccaniosa e inaffidabile mentre adesso è ottima e standard (tranne una piccola eccezione).

I socket sono una API, un insieme di primitive che permette di interagire con meccanismi di comunicazione regolate dai protocolli.
La pipe è una IPC che però non è regolato da un protocollo: c'è un canale di comunicazione in cui uno scrive e un'altro legge. Non c'è un protocollo. A livello applicativo, posso definire un protocollo, ma non c'è a livello di sistema.

> Un protocollo è un insieme di regole che definisce, ad esempio, come devono comunicare due entità.

Nella pratica, il protocllo corrisponde ad un blocco di solito dentro al kernel che implementa l'enforcement ovvero fa in modo che le regole vengono rispettate.
I blocchi software che implementano i protocolli tcp/ip vivono dentro al kernel (come regola generale). E' possibile comunque implementare il protocollo tcp/ip a livello utente (ma non lo fa nessuno).
I protocolli sono implementati a livello kernel nei SO.
I socket sono una interfaccia che permettono alle applicazioni di comunicare usando le regole che di quei protocolli e che devono rispettare.
Il socket è quindi un modo di parlare col protocollo tcp/ip, e nascono i socket per tcp/ip originariamente.

> Sincrona: voglia comunicare e finchè non ho finito la comunicazione non faccio altro. Attendo finchè la comunicazione non è terminata.
> Asincrono: inizio una comunicazione, ma non attendo la terminazione (tipicamente mi metto a fare altro) e ad un certo punto :o controllo che la comunicazione è terminata, o mi viene segnalato che è terminata.

Il ruolo principale dei socket è fare in modo che il dominio applicativo ,fortemente sincrono, interagisca al meglio col dominio del sistema operativo dove vivono i protocolli che invece è fortemente asincrono.
Nella stragrande maggioranza dei casi la comunicazone a livello applicativo è sincrona.
A livello kernel, la comunicazione è fortemente asincrona: questo per ragioni di efficienza. Se io ho su un sistema più entità che devono comunicare se io facessi comunicazione sincrona a livello kernel rallenterei in una maniera micidiale.
La separazione fra questi due modi di comunicare è resa ragionevolmente accettabile proprio grazie alle sockets.

Questo si basa su meccanismi di buffering e quindi accumulo di dati. Quando comunichiamo con sockets abbiamo buffer di memoria che accumulano dati che devono essere inviati o ricevuti a livello di applicazione.

Originariamente i socket erano proprio delle syscall, mentre ora sono implementati a livello di libreria utente che si và poi ad interfacciare con il SO.
Normalmente i socket vengono pensati come comunicazione fra due sistemi distinti, ma in verità possono funzionare anche sulla stessa macchina.
Possono essere usati come per far comunicare due entità distribuite ma sulla stessa macchina, oppure possono comunicare come se sapessero che si trovano nella stessa macchina.
Questo dipende dal dominio di comunicazione: può essere internet (quindi cambia l'indirizzo) o dominio Unix.
Il dominio specifica dove posso comunicare (unix = locale).
Esso poi deve essere utilizzato secondo regole specifiche di quel dominio: nel caso di internet, si utilizza indirizza ip e le porte.
La porta è una traduzione pessima dell' inglese port.
Nel dominio unix, si usano nomi equivalenti al nome di file. Un esempio con l' x window: esiste un x server e un x client, che comunicano usando una socket unix.
Il dominio Unix e internet usano sia la  comunicazione a datagramma, che a flusso:

 - Stream socket : a flusso
 - Datagram socket : a pachetti, 
 - Row socket: pacchetti di livello più basso, permettono di saltare layer dello stack