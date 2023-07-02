Ehilà, lettori e bug bounty hunters! Sono Samuele 'indevi0us' Gugliotta, e questa volta presenterò una vulnerabilità applicativa trovata all'interno di un programma privato di bug bounty su [Hackrate](https://hckrt.com/), intorno alla metà dell'anno scorso, nel 2022.
Tagliamo corto con i convenevoli e passiamo subito nel vivo dell'azione.

## **Esfiltrazione del Database PostgreSQL attraverso l'abuso di richieste PostgREST!**

### Sommario
La vulnerabilità descritta nel report a cui si fa riferimento in questo articolo avrebbe permesso l'esfiltrazione del database, attraverso l'abuso e la malformazione di alcune richieste applicative direttamente collegate all'utilizzo del componente esterno PostgREST, utilizzato come server web standalone, nonché come alternativa alle operazioni CRUD manuali.

### Descrizione
Come target del suddetto Bug Bounty Program (BBP) privato, vi era un'applicazione web, accessibile previa autorizzazione tramite [HTTP Basic Authentication](https://developer.mozilla.org/en-US/docs/Web/HTTP/Authentication), in cui venivano immagazzinati, elaborati ed infine modificati dati di natura prevalentemente finanziaria inerenti alle organizzazioni registrate nell'applicativo stesso.
L'utenza fornita per le attività di testing poteva dunque accedere a tali dati attraverso alcune dashboard, comprensive di tabelle che venivano caricate sul front-end attraverso una serie di richieste GET seguendo un pattern abbastanza comune. In particolare, veniva passata la URI `/pg/<nome_tabella>`, seguita dall'aggiunta del parametro GET `select` che risultava contenere sempre il valore `*`.

Esaminando queste richieste nella loro interezza, sembravano indubbiamente un po' sospette e meritavano sicuramente un minimo di approfondimento:
```
/pg/vw_contract_with_orgs?select=*
/pg/vw_funding_with_orgs?select=*
```

Quindi, ho avviato il mio web application proxy preferito, [Burp Suite](https://portswigger.net/burp), per intercettare le richieste prima che le tabelle all'interno delle dashboard, ed il relativo contenuto, venissero fetchate.
Non c'era molto da aspettarsi da una semplice richiesta GET, giusto? Sbagliato. In molti contesti, soprattutto nel bug bounty, l'attenzione ai dettagli è fondamentale. Controllare il contenuto dei cookie o delle intestazioni (header) HTTP passati all'interno di determinate richieste applicative, può portare all'apertura di diverse strade che, seguite nel modo più opportuno, con il giusto approccio ed una buona dose di creatività, possono condurre alla scoperta di alcuni dei bug più interessanti.

In questo caso specifico, l'header HTTP `X-Client-Info` si è rivelato contenere un'informazione cruciale, in quanto mi ha permesso di comprendere appieno la logica dietro l'applicazione e il motivo per cui queste richieste venissero passate proprio in quel determinato modo, confermando i precedenti sospetti.

Conteneva difatti la seguente stringa:
```
X-Client-Info: postgrest-js/0.35.0
```

Che cos'è realmente?

Lasciatemi confessare che prima di quel momento non avevo mai visto applicazioni che lo utilizzassero, ed era abbastanza una novità per me, quindi mi sono armato di santa pazienza e, spinto da un naturale interesse, sono andato a leggere, e a studiare, la [documentazione ufficiale di PostgREST](https://postgrest.org/en/stable/).

![](https://media1.tenor.com/images/e976b2eeeb89ee2357c775149aa94b4f/tenor.gif?itemid=27407442f)

Non ho intenzione di citarne ogni singola sezione, quindi vi suggerisco di andare a leggerlo direttamente qualora foste interessati, altrimenti aspettatevi un *tl;dr* qui sotto.

Come si evince dalla stessa, *‟PostgREST is a standalone web server that turns your PostgreSQL database directly into a RESTful API. The structural constraints and permissions in the database determine the API endpoints and operations”*.

Non avrei potuto chiedere una rivelazione migliore! Di colpo, le richieste GET che venivano passate inizialmente presentavano una logica chiara. Per cui, smontiamole pure assieme:

* `/pg` ➔ risorsa con riferimento all'utilizzo di PostgREST, con conseguente conferma dell'esistenza di un database di tipo PostgreSQL dietro l'applicazione web;
* `vw_contract_with_orgs` e `vw_funding_with_orgs` ➔ non sono altro che due tabelle del database, certamente sanificate e private di informazioni che non dovrebbero essere visibili, ma questo non cambia la loro natura;
* `select=*` ➔ abbastanza intuibile, una semplice select di tutti i contenuti (`*`), colonne comprese, all'interno delle tabelle del database PostgreSQL.

A questo punto, avendo compreso appieno come funzionasse l'applicazione, non restava che abusare delle sue stesse funzionalità.

Al fine di mostrare l'impatto assoluto di una tale vulnerabilità, in genere un attaccante è interessato alle tabelle del database che possono rivelare contenuti sensibili. Ad esempio, la tabella `users` è spesso presa di mira per ovvi motivi, in quanto contiene informazioni quali nomi, id, indirizzi e-mail e password (hashate, si spera) di tutti gli utenti esistenti a sistema.

Penso sia opportuno, a questo punto, specificare che l'utilizzo di PostgREST in generale non costituisce assolutamente una vulnerabilità di sicurezza in sé, ma è fondamentale applicare i giusti criteri di autorizzazione e controllo degli accessi per assicurarsi che un utente, quindi nel peggiore dei casi un attaccante, non possa accedere a contenuti che non dovrebbero essere visibili by-design. Limitare l'accesso ad alcune tabelle specifiche del database, e quindi accettare query dall'applicazione solo per quelle sanificate che non contengono contenuti al di fuori di ciò a cui un utente medio dovrebbe accedere, è ciò che si sarebbe dovuto fare in una situazione ideale, ma che questa volta non è successo.

Chiarito questo, passiamo all'exploitation.

![](https://media1.tenor.com/images/7ad6e94ce87d429665b63d94505f033f/tenor.gif?itemid=27407441)

Intercettando nuovamente le richieste GET che vengono inviate dall'applicazione ogniqualvolta il contenuto delle dashboard viene fetchato, soffermiamoci sulla prima delle richieste che recuperano il contenuto dalle tabelle del database PostgreSQL. La prima era `/pg/vw_contract_with_orgs?select=*`. Dopo averla intercettata, ho inoltrato la richiesta al *Repeater* del web application proxy, per consentirmi di modificarla e testarla con rapidità.
In seguito, ho modificato la query per puntare alla tabella `users` invece che a `vw_contract_with_orgs`, in modo che la richiesta finale risultasse esattamente qualcosa come `/pg/users?select=*`. Inviando questa richiesta, mi sono trovato di fronte a un primo clamoroso epic fail. Difatti, la risposta dell'applicazione è stata:
```
{"hint":null,"details":null,"code":"42501","message":"permission denied for table users"}
```
A quanto pare, l'accesso alla tabella `users` del database PostgreSQL era stato adeguatamente protetto limitando gli utenti con permessi intrinseci. Tuttavia, non mi sono abbattuto. Piuttosto, pensiamo alla logica di quella `select` e a come dovrebbe apparire lato back-end. Avendo tutti gli elementi a portata di mano, doveva essere qualcosa di simile alla seguente query:
```
SELECT * FROM users;
```
E se non ci interessasse vedere letteralmente tutto? Cosa fareste se foste interessati a visionare soltanto qualcosa di specifico all'interno di una tabella del database?

Penso proprio che avreste fatto una query specificando le colonne a cui sareste stati interessati.

Quindi, ho modificato nuovamente la richiesta GET, fino a ottenere qualcosa di simile a:
```
/pg/users?select=id
```
E di conseguenza lato back-end sarebbe risultata:
```
SELECT id FROM users;
```
L'applicazione restituì il contenuto della colonna `id` della tabella del database `users`, che fa riferimento agli identificativi di tutti gli utenti esistenti a sistema.

![](https://media1.tenor.com/images/818a0a8c2d9c6eeaaa84e60cd21f4983/tenor.gif?itemid=27407440)

Ho continuato, quindi, a concatenare altre colonne, come `email`, `updated_at`, `created_at`, dividendole con una virgola nelle richieste applicative, in questo modo:
```
/pg/users?select=id,email,updated_at,created_at
```
E ho continuato fintanto che l'applicazione mi ha permesso di farlo da un punto di vista autorizzativo, restituendo ogni volta il contenuto richiesto attraverso quella `select` che, prendendo forma, ha iniziato a risultare qualcosa di simile a:
```
SELECT id,email,updated_at,created_at FROM users;
```
![](https://i.imgur.com/fInXcwd.png)
Tuttavia, considerando il fail dell'applicativo a livello *access control*, non avrei escluso che ci potessero essere anche altre tabelle interessanti da poter esfiltrare.

### Impatto
Possibilità di abusare dell'uso che l'applicazione fa di PostgREST per effettuare query su tabelle di database PostgreSQL che normalmente non dovrebbero essere accessibili a un utente medio, arrivando ad esfiltrare informazioni sensibili.

### Ringraziamenti
Grazie come sempre al team di HACKRATE per avermi dato l'opportunità di partecipare a un altro programma privato di bug bounty molto interessante e per aver gestito il mio report. Un sentito ringraziamento anche al team del Cliente per aver dato la giusta importanza alla mia segnalazione, avendo risolto la problematica e aver deployato una fix in tempi assolutamente rispettabili.

Per ora è tutto.

*indevi0us*

### Autore
[![Linkedin Badge](https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/samuele-gugliotta/)
[![Website Badge](https://img.shields.io/badge/website-000000?style=for-the-badge&logo=About.me&logoColor=white)](https://indevi0us.github.io)
