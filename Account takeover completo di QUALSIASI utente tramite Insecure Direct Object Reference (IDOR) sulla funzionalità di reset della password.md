Salve, gente! Sono Samuele Gugliotta, Offensive Security Researcher & Bug Bounty Hunter, probabilmente meglio conosciuto come indevi0us. Non c'è niente di meglio che iniziare un nuovo anno con il piede giusto e, infatti, questa volta è per presentare una vulnerabilità critica nel mio primo writeup di questo 2022! È stata recentemente trovata in un programma privato sulla mia piattaforma di bug bounty preferita: Hackrate. Quindi, bando alle ciance, andiamo al sodo.

## **Account takeover completo di QUALSIASI utente tramite Insecure Direct Object Reference (IDOR) sulla funzionalità di reset della password!**

### Sommario
Come potete vedere nel banner di cui sopra, si tratta di un Insecure Direct Object Reference (IDOR), che è in realtà una tra le più impattanti e sottovalutate vulnerabilità lato *access control*. Si verifica quando un'applicazione utilizza input forniti dall'utente per accedere direttamente agli oggetti.

Nel mio caso, mi avrebbe permesso di reimpostare la password letteralmente di qualsiasi utente presente nell'applicazione semplicemente facendo *guessing* degli identificativi numerici incrementali associati ad essi. Questa condizione, combinata con un altro comportamento anomalo rilevato, ovvero che l'applicazione memorizzava dettagli aggiuntivi e li abbinava alle voci del database rilevanti nell'HTML esterno delle pagine (ad esempio, `<label class="label is-marginless">E-mail:</label><p class="control">indevi0us@letmehack.it</p><input type="hidden" name="id" value="57">`), mi ha permesso di ottenere il pieno controllo dell'account impattato.

### Descrizione
Dopo aver creato l'account utente e navigato tra le funzionalità autenticate dell'applicazione web su `<redacted.com>`, mi sono concentrato in particolare sul modulo di reimpostazione della password, poiché è una funzione notoriamente abusata per ovvie ragioni. Dopo aver interagito con il modulo richiedendo il link di conferma per reimpostare la mia password via e-mail, ho notato che agli utenti registrati venivano assegnati identificativi numerici di tipo incrementale. Questo era evidente dalla URL ricevuta, che era qualcosa del tipo `https://<redacted.com>/reset-password/0-57-axq24ncy32bh1hc8nrp78xyz`, in cui era possibile notare l'identificativo numerico `57` tra `0-` e il token. Per confermare questa evidenza, ho creato un secondo account che avrei usato come utente vittima, e ripetendo la stessa procedura di reimpostazione della password ho ottenuto un link simile a questo: `https://<redacted.com>/reset-password/0-58-bce64t3ane46tt9nr6ce2kk33`. Inevitabilmente, quel `58` ha attirato tutta la mia attenzione.

![](https://media.tenor.com/PVlkN3u4KNsAAAAC/varg-smiling.gif)

Quindi, sono andato a recuperare la richiesta POST di reimpostazione della password inviata in precedenza per il primo utente con `id=57`, dalla cronologia del proxy del mio BurpSuite Professional che intanto stava ascoltando passivamente. Ho scoperto che lo stesso ID veniva passato come parametro nel corpo della richiesta POST.

![](https://blog.hckrt.com/static/eeeec38782b8bc660d7639ed72d29dd5/07af8/request.webp)

Supponendo che il `_token` fosse stato già utilizzato, ho ripetuto la richiesta di reimpostazione della password per il primo utente, che ho chiamato *attaccante*, ma questa volta intercettandola con web application proxy. Quindi ho cambiato il parametro `id` nel corpo di quella richiesta da `id=57` a `id=58` e ho scelto una password diversa prima di inoltrarla. L'applicazione ha risposto con un popup che mi informava che la password era stata cambiata con successo. E infatti, non ero più in grado di effettuare l'autenticazione con l'utente *vittima* utilizzando la password scelta durante la prima registrazione, ma quella inviata nella richiesta POST malformata dall'utente *attaccante*, funzionava perfettamente. ^^

### Impatto
Compromissione completa dell'account di qualsiasi utente registrato a sistema.

Nel peggiore dei casi, lo stesso attacco potrebbe essere ripetuto su tutti gli altri utenti con id `<57` e ovviamente su quelli successivi che sarebbero stati creati.

### Ringraziamenti

Vorrei ringraziare il Team di [Hackrate](hckrt.com) per avermi dato l'opportunità di partecipare a questo programma privato e per aver effettuato il triage della segnalazione, nonché le fasi successive, in modo serio, professionale e utile in termini di tempistica sia per me, ma soprattutto per il Team che si è occupato della fix. Ringrazio anche il Team del Cliente per aver preso in considerazione il mio report e per aver risolto la vulnerabilità in modo magistrale.

È tutto per ora.

*indevi0us*
