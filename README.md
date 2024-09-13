# Applicazione-Bancaria
Applicazione web realizzata con spring boot ed esposizione API. Questo progetto simula una realtà di applicazione bancaria, nella quale vengono gestiti i vari utenti, i loro conti, le loro carte, prestiti, trasferimenti di denaro. 

La parte del client è stata sviluppata con html e thymeleaf, mentre la parte dell’admin con angular che utilizza le funzionalità esposte dal rest controller sviluppato tramite spring boot.

- Progetto spring boot
- Database: MySQL 8.4 e MongoDB 7.0
- Front end: html 5, css, Angular 17, PrimeFaces, PrimeNG, PrimeIcons, Bootstrap 3(solo lato client), JavaScript
- Server: Tomcat 9
- Test: JUnit 5

- ## Dependecy Utilizzate:

- Spring Data JPA
- Spring Data MongoDB
- Spring DataRest
- Spring Thymeleaf
- Spring Security
- Spring Lombok
- Spring Web MVC
- Spring Mail Service
- JWT API
- ITextPdf

## Dettagli implementazioni:

- Implementazione dei model sia per le tabelle MySQL sia per quelle di Mongo.
- Annotation di mongo:

```java
@Data
@Document(collection = "audit_mongo")
```

- Tra le tabelle di mongo rientrano due in particolare che tengono traccia degli accessi dell’admin e delle azioni che effettua l’admin sui clienti

```java
package com.tas.applicazionebancaria.businesscomponent.model;

import java.io.Serializable;
import java.util.Date;

import org.springframework.data.mongodb.core.mapping.Document;

import jakarta.persistence.Id;
import lombok.Data;

@Document(collection = "log_accessi_admin")
@Data
public class LogAccessiAdmin implements Serializable{

	private static final long serialVersionUID = -9053244362811124361L;
	
	@Id
	private String codAdmin;
	
	private Date data;
	
	private String dettagli;
}

```

```java
package com.tas.applicazionebancaria.businesscomponent.model;

import java.io.Serializable;
import java.util.Date;

import org.springframework.data.mongodb.core.mapping.Document;

import com.tas.applicazionebancaria.businesscomponent.model.enumerations.TipoModificaAudit;

import jakarta.persistence.Id;
import lombok.Data;

@Data
@Document(collection = "audit_mongo")
public class AuditLog implements Serializable{
	
	private static final long serialVersionUID = -8110110095719109493L;
	
	@Id
	private String codAdmin;
	
	private Date data;
	
	private TipoModificaAudit tipo;
	
	private String dettagli;
}

```

- Implementazione delle repository dei model sia per mongo sia per mysql, facendo estendere la relativa interfaccia
- Inoltre in alcune repository è stata aggiunta l’annotation “@Hidden”, per fare in modo che Swagger non creasse in automatico ogni dettaglio sulla documentazione del progetto
- Per le query complesse delle repository di mongo è stato usato l’annotation “@Aggregation”, per indicare l’utilizzo di funzioni di aggregazione

```java
package com.tas.applicazionebancaria.repository;

import java.util.List;
import java.util.Optional;

import org.springframework.data.mongodb.repository.Aggregation;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.stereotype.Repository;

import com.tas.applicazionebancaria.businesscomponent.model.TransazioniMongo;

import io.swagger.v3.oas.annotations.Hidden;

@Hidden
@Repository("TransazioniMongoRepository")
public interface TransazioniMongoRepository extends MongoRepository<TransazioniMongo, String> {
	@Aggregation(pipeline = { "{$group: { _id: '$tipoTransazione', count: {$sum: 1} }}",
			"{$match: {_id: 'ADDEBITO'}}" })
	Optional<Long> findTotAddebiti();
		
	@Aggregation(pipeline = { "{$group: { _id: '$tipoTransazione', count: {$sum: 1} }}",
			"{$match: {_id: 'ACCREDITO'}}" })
	Optional<Long> findTotAccrediti();

	@Aggregation(pipeline = { "{ $group : { _id: '$codCliente', avgTransactions: { $avg: 1 } } }" })
	Optional<Long> transazioniMediePerCliente();

	@Aggregation(pipeline = {
	        "{ $group : { _id: {'$month': '$dataTransazione'}, importo: { $sum: '$importo' } } }"
	    })
	Optional<List<TransazioniMongo>> importoTransazioniPerMese();
}
```

- Implementazione dei service e service implementation

- Implementazione di una classe JWT , progettata per gestire la creazione e la validazione dei token JWT, che possono essere utilizzati per autenticare gli utenti in un'applicazione web. Quando un utente si autentica, viene generato un token JWT contenente le sue informazioni, che può essere verificato successivamente per garantire che l'utente sia autenticato.

```java
package com.tas.applicazionebancaria.utils;

import java.security.Key;
import java.time.Instant;
import java.util.Base64;
import java.util.Date;
import java.util.UUID;

import javax.crypto.spec.SecretKeySpec;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jws;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;

public class JWT implements Costanti {
	private static String secret = TOKEN_SECRET;
	
	public static String generate(String nome, String cognome, String email) {
	    // Crea una chiave HMAC usando il segreto codificato in Base64
	    Key hmacKey = new SecretKeySpec(Base64.getDecoder().decode(secret), SignatureAlgorithm.HS256.getJcaName());

	    // Ottiene l'istante corrente
	    Instant now = Instant.now();

	    // Costruisce il token JWT
	    String jwtToken = Jwts.builder()
	            .claim("nome", nome)                      // Aggiunge il nome come claim personalizzato
	            .claim("cognome", cognome)                // Aggiunge il cognome come claim personalizzato
	            .setSubject(email)                        // Imposta l'email come subject del token
	            .setId(UUID.randomUUID().toString())      // Imposta un ID univoco per il token
	            .setIssuedAt(Date.from(now))              // Imposta la data di emissione del token
	            .setExpiration(Date.from(now.plus(TOKEN_EXPIRATION, UNIT))) // Imposta la data di scadenza
	            .signWith(hmacKey)                        // Firma il token con la chiave HMAC
	            .compact();                               // Converte il builder in una stringa compatta (il token JWT)

	    return jwtToken;                                  // Restituisce il token JWT generato
	}

	public static Jws<Claims> validate(String jwtToken) {
	    // Crea una chiave HMAC usando il segreto codificato in Base64
	    Key hmacKey = new SecretKeySpec(Base64.getDecoder().decode(secret), SignatureAlgorithm.HS256.getJcaName());

	    // Analizza il token JWT e restituisce i claims
	    Jws<Claims> claims = Jwts.parserBuilder()
	            .setSigningKey(hmacKey)                   // Imposta la chiave di firma
	            .build()                                  // Costruisce il parser
	            .parseClaimsJws(jwtToken);                // Analizza il token e verifica la firma

	    return claims;                                    // Restituisce i claims contenuti nel token
	}

}
```

- Implementazione dell’Email Service utilizzabile ovunque all’interno dell’applicazione per tenere traccia dei vari eventi, come ad esempio l’invio dell’email di benvenuto appena dopo la registrazione dell’utente.

```java
package com.tas.applicazionebancaria.utils;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.mail.SimpleMailMessage;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.stereotype.Component;

@Component
public class EmailService {

    @Autowired
    private JavaMailSender mailSender;
    
    @Value("${spring.mail.username}")
    private String fromMail;
    /**
     * Specificare in ordine: email mittente, oggetto e corpo
     */
    public void sendEmail(String to, String oggetto, String body) {
    	System.out.println("sto inviando un email");
        SimpleMailMessage message = new SimpleMailMessage();
        message.setFrom(fromMail);
        message.setTo(to);
        message.setSubject(oggetto);
        message.setText(body);

        mailSender.send(message);
    }
}
```

Implementazione della sicurezza attraverso:

### 1. Classe `JwtAuthFilter`

Questa classe è un filtro di sicurezza personalizzato che estende `OncePerRequestFilter`. Viene utilizzata per esaminare ogni richiesta HTTP e determinare se il richiedente è autenticato tramite un JWT.

### Funzioni principali:

- **`getCookieByName`**: Questo metodo aiuta a recuperare un cookie specifico dalla richiesta HTTP. Viene utilizzato per estrarre il cookie che contiene il JWT.
- **`doFilterInternal`**: Questo è il metodo principale del filtro, eseguito per ogni richiesta HTTP. Esegue le seguenti operazioni:
    - **Recupero del JWT**: Ottiene il JWT dal cookie specificato.
    - **Validazione del JWT**: Se il token è valido, carica i dettagli dell'utente tramite il servizio `AdminDetailsService` e imposta l'autenticazione nel contesto di sicurezza (`SecurityContextHolder`).
    - **Prosegue la catena di filtri**: Alla fine del metodo, la richiesta viene passata al prossimo filtro nella catena.
        - **Gestione del JWT scaduto**: Se il token è scaduto (`ExpiredJwtException`), recupera le informazioni dell'amministratore da un altro cookie (`admin`), rigenera un nuovo token JWT e lo aggiorna nel contesto di sicurezza.

### 2. Classe `SecurityConfig`

Questa classe configura la sicurezza per l'applicazione Spring Boot. Viene utilizzata per definire come le richieste HTTP devono essere gestite in termini di autenticazione e autorizzazione.

### Funzioni principali:

- **`adminAuthenticationProvider`**: Configura un provider di autenticazione DAO che utilizza `AdminDetailsService` per caricare i dettagli dell'utente e `BCryptPasswordEncoder` per confrontare le password durante l'autenticazione.
- **`passwordEncoder`**: Definisce un encoder per le password, utilizzando `BCryptPasswordEncoder`, che è uno dei metodi più sicuri per la crittografia delle password.
- **`authenticationManager`**: Crea un `AuthenticationManager`, che è il componente principale di Spring Security per gestire il processo di autenticazione.
- **`filterChain`**: Configura la catena dei filtri di sicurezza:
    - **CSRF disabilitato**: Disabilita la protezione CSRF (Cross-Site Request Forgery), che può essere comune in API stateless.
    - **Autorizzazione delle richieste**: Configura l'accesso alle risorse API (`/api/**`) solo agli utenti autenticati. Tutte le altre richieste sono permesse senza autenticazione.
    - **Integrazione del filtro JWT**: Aggiunge il filtro `JwtAuthFilter` prima del filtro standard di Spring Security (`UsernamePasswordAuthenticationFilter`).
    - **Disabilitazione del logout**: Disabilita la funzionalità di logout.

- Implementazione del servizio angular `ApiService` che centralizza tutte le chiamate HTTP verso un backend, rendendo più semplice la gestione delle operazioni RESTful come recuperare dati, creare, aggiornare o eliminare risorse.

### Struttura e Funzioni della Classe `ApiService`

### Attributi Privati

- **`cliente`**: Un oggetto della classe `Cliente`, che rappresenta il cliente attualmente gestito nel contesto dell'applicazione. Viene utilizzato per mantenere informazioni sul cliente nel servizio.
- **`basePath`**: La URL base del backend, usata come prefisso per tutte le chiamate API.
- **`httpOptions`**: Un oggetto di configurazione per le richieste HTTP. Definisce gli header usati, in particolare il `Content-Type` e la proprietà `withCredentials: true` per includere i cookie nelle richieste.

## Le funzioni all’interno usano sempre Observable, subscribe e pipe

### Observable

Un **`Observable`** è un costrutto che rappresenta una sequenza di eventi asincroni che possono essere osservati. Può emettere tre tipi di notifiche:

- **`next`**: Emesso quando c'è un nuovo valore disponibile.
- **`error`**: Emesso quando si verifica un errore.
- **`complete`**: Emesso quando la sequenza termina.

Un `Observable` non inizia a emettere valori finché non viene sottoscritto con `subscribe`. Questo modello è utile per gestire dati asincroni, come le risposte HTTP o gli eventi.

### Pipe

**`pipe`** è un metodo degli `Observable` che permette di combinare operatori funzionali per trasformare, filtrare, o gestire i dati emessi dall'Observable. Gli operatori di RxJS come `map`, `filter`, `catchError`, ecc., vengono passati all'interno del `pipe`.

### Subscribe

Il metodo **`subscribe`** viene utilizzato per "iscriversi" agli aggiornamenti di un `Observable`. Quando ti iscrivi a un `Observable`, gli fornisci tre possibili callback:

- **`next`**: Funzione chiamata ogni volta che l'Observable emette un nuovo valore.
- **`error`**: Funzione chiamata se l'Observable emette un errore.
- **`complete`**: Funzione chiamata quando l'Observable completa l'emissione di tutti i suoi valori.

- Implementazione dell’”Authentication Service” di angular che, permette di gestire le operazioni di login e logout con notifiche utente, mantenendo la sessione attraverso l'uso dei cookie.

- La parte amministrativa, realizzata su angular, permette attraverso i servizi messi a disposizione di visualizzare i clienti, i loro conti e gestire il tutto, perfino bloccarli
- Inoltre vengono mostrate delle statistiche importanti sfruttando dei grafici a torta forniti da Prime NG riguardante le operazioni dei clienti, ad esempio l’importo delle transazioni per mese.

- Le transazioni possono essere scaricate in un file pdf apposito.

## Quello che ho fatto io:

- Parte della registrazione client, bug fixes e login client
- Controllo tentativi errati client
- Email Service
- Richiesta prestiti,conferma e rifiuto prestiti admin
- Cambio password admin
- Audit admin
- Log accessi e logout admin
- Test con mock mvc e mockito
