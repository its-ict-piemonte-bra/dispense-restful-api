# Autenticazione e sicurezza

## Prerequisiti
- Conoscere la struttura di richiesta e risposta HTTP.
- Sapere cosa sono gli header HTTP e i codici di stato 401 e 403.
- Avere familiarità con il formato JSON.

## Obiettivi
- Distinguere l'autenticazione dall'autorizzazione.
- Conoscere gli schemi di autenticazione HTTP più comuni.
- Capire ruoli, scope e flussi del framework OAuth 2.0.
- Comprendere la struttura e la natura di un token JWT.
- Riconoscere le vulnerabilità e i limiti più noti di questi meccanismi.

## 1. Autenticazione e autorizzazione
Sono due concetti diversi e complementari.

L'**autenticazione** verifica *chi* è un utente.
L'**autorizzazione** stabilisce *cosa* quell'utente può fare.

In REST/HTTP la sicurezza si basa su header dedicati e codici di stato appositi.
Il codice `401 Unauthorized` indica che il client non è autenticato: non si sa chi sia.
Il codice `403 Forbidden` indica che il client è autenticato ma non ha i permessi per l'operazione.

```mermaid
flowchart TD
    A[Richiesta in arrivo] --> B{"Chi sei?<br/>(autenticazione)"}
    B -->|Identità non valida| E401[401 Unauthorized]
    B -->|Identità valida| C{"Puoi farlo?<br/>(autorizzazione)"}
    C -->|Permesso negato| E403[403 Forbidden]
    C -->|Permesso concesso| OK[200 OK]
```

## 2. Gli header dell'autenticazione
Due header gestiscono l'autenticazione in HTTP.

- `Authorization`: contiene le credenziali che l'utente invia per autenticarsi.
- `WWW-Authenticate`: il server lo usa per indicare il tipo di autenticazione richiesto.

I valori più importanti dello schema di autenticazione sono due: Basic e Bearer.

## 3. Schemi di autenticazione HTTP
### Basic Authentication
La **Basic Authentication** è lo schema meno sicuro ed è considerato legacy.
Si prende la stringa `username:password` e si applica la codifica Base64.
La codifica Base64 non è cifratura: chiunque può decodificarla.
Per questo Basic va usato solo su canali HTTPS.

### Bearer Token
Lo schema **Bearer** richiede di fornire un *bearer token*.
Un bearer token è letteralmente un "token che porta con sé" i permessi necessari.
Rientrano in questa categoria le **API Key** fornite da un provider.
Un token Bearer può anche essere rilasciato da un endpoint di login dopo l'invio di username e password.

Esempio di header con token Bearer:

```http
Authorization: Bearer <token>
```

## 4. Il framework OAuth 2.0
OAuth 2.0 è un framework di **autorizzazione**, definito dall'RFC 6749.

### Scopo di OAuth 2.0
Lo scopo è permettere a un'applicazione terza di ottenere un accesso **limitato** a risorse protette.
L'accesso avviene per conto del proprietario della risorsa.
Il proprietario non deve condividere le proprie credenziali (la password) con l'applicazione terza.
OAuth 2.0 separa quindi l'identità dell'utente dall'autorizzazione delegata, gestita tramite token.

### Autorizzazione, non autenticazione
OAuth 2.0 è nativamente ed esclusivamente un framework di autorizzazione.
Rilascia permessi tramite l'**Access Token**.
Non è nato come protocollo di autenticazione dell'identità.
Per gestire anche l'identità dell'utente è stato creato uno strato aggiuntivo: **OpenID Connect (OIDC)**.
OIDC si appoggia sopra OAuth 2.0.

### I quattro ruoli
OAuth 2.0 definisce quattro ruoli che interagiscono nel sistema:

- **Resource Owner**: l'utente proprietario dei dati.
- **Client**: l'applicazione terza che chiede l'accesso.
- **Resource Server**: il server che ospita le risorse protette.
- **Authorization Server**: il server che emette i token.

```mermaid
flowchart LR
    RO["Resource Owner<br/>(utente)"] -->|autorizza| C["Client<br/>(app terza)"]
    C -->|richiede token| AS["Authorization Server"]
    AS -->|rilascia Access Token| C
    C -->|usa Access Token| RS["Resource Server<br/>(dati protetti)"]
    RS -->|risorsa protetta| C
```

### Lo scope
Il parametro **scope** specifica i permessi dettagliati richiesti dal client.
Definisce il livello di accesso, per esempio sola lettura o anche scrittura.
Lo scope delimita l'ampiezza dell'autorizzazione concessa dall'utente.

### Client pubblici e client riservati
OAuth 2.0 distingue due categorie di client.

Un **Confidential Client** (client riservato) può custodire in modo sicuro un segreto statico.
Un **Public Client** (client pubblico) non può custodire un segreto in modo sicuro.

Una Single Page Application (SPA) eseguita nel browser è un client pubblico.
Anche un'app mobile è un client pubblico.
Il loro codice e i dati locali sono esposti all'utente finale.
Per questo non possono conservare in sicurezza un `client_secret`.

### I flussi di autorizzazione
OAuth 2.0 prevede diversi flussi, detti *grant*, adatti a scenari diversi.

L'**Authorization Code Grant** è il più sicuro per le applicazioni web lato server.
Un codice intermedio transita nel browser (front-channel).
L'Access Token vero e proprio viene poi scambiato tramite una comunicazione server-to-server (back-channel).
In questo modo il token non viene mai esposto direttamente nel browser.

```mermaid
sequenceDiagram
    participant U as Resource Owner
    participant B as Browser
    participant C as Client (backend)
    participant AS as Authorization Server
    U->>B: Avvia accesso
    B->>AS: Richiesta di autorizzazione
    AS->>U: Richiede login e consenso
    U->>AS: Login + consenso
    AS-->>B: Authorization Code (front-channel)
    B->>C: Consegna il code
    C->>AS: Code + client_secret (back-channel)
    AS-->>C: Access Token
    Note over C,AS: Il token viaggia solo server-to-server,<br/>mai esposto nel browser
```

Il **Client Credentials Grant** si usa nelle interazioni Machine-to-Machine (M2M).
È adatto quando un backend protetto deve comunicare con un'altra API senza alcun utente presente.
Il client si autentica con le proprie credenziali, cioè `client_id` e `client_secret`.

L'**Implicit Grant** è oggi deprecato.
Era usato dalle SPA, ma esponeva l'Access Token negli URL e nella cronologia del browser.
Le attuali best practice lo sconsigliano in favore dell'Authorization Code Flow combinato con PKCE.

### Il Refresh Token
Il **Refresh Token** serve a ottenere un nuovo Access Token quando quello corrente scade.
Non viene mai inviato al Resource Server.
Non accompagna le normali richieste API.
Viene inviato esclusivamente all'Authorization Server, solo quando l'Access Token scade.
Questo evita di costringere l'utente a rifare il login.

## 5. JSON Web Token (JWT)
Un **JWT** è un token compatto usato per trasportare informazioni in modo verificabile.

### Struttura
Un JWT è composto da tre sezioni codificate in Base64URL.
Le tre sezioni sono separate dal carattere punto `.`.
La forma standard è `header.payload.signature`.

- **Header**: contiene il tipo di token (tipicamente JWT) e l'algoritmo di firma (per esempio HS256 o RS256).
- **Payload**: contiene i dati, detti *claim*.
- **Signature** (firma): garantisce l'integrità del token.

L'header serve a istruire il server su come verificare la firma.

```mermaid
flowchart LR
    JWT["JWT<br/>header.payload.signature"] --> H["Header<br/>tipo + algoritmo<br/>(es. HS256)"]
    JWT --> P["Payload<br/>claim (es. sub, exp)"]
    JWT --> S["Signature<br/>firma di integrità"]
```

### I claim del payload
Un **claim** è un'informazione contenuta nel payload.
Alcuni claim sono standard e registrati.

- `exp` (expiration): indica la data e l'ora di scadenza del token. È un timestamp Unix Epoch, cioè i secondi trascorsi dal 1° gennaio 1970.
- `sub` (subject): identifica in modo univoco l'entità a cui si riferisce il token, per esempio l'ID dell'utente.

Oltre la scadenza indicata da `exp`, il token deve essere rifiutato.

### I dati sono codificati, non cifrati
In un JWT standard firmato (JWS) il payload è solo **codificato in Base64URL**.
Non è crittografato.
Chiunque può decodificare e leggere i dati del payload.
La chiave segreta serve solo a validare l'integrità tramite la firma, non a nascondere i dati.

### Natura stateless
L'autenticazione basata su JWT è **stateless**.
Tutte le informazioni dell'utente sono codificate nel token, conservato sul client.
Il server deve solo verificare la firma.
Non deve interrogare un database di sessioni né mantenere uno stato in memoria.
Questo garantisce una grande scalabilità orizzontale, utile nei microservizi.

### Il problema della revoca
La natura stateless ha uno svantaggio.
Un JWT standard non può essere revocato istantaneamente con facilità.
Una volta emesso, resta valido fino alla scadenza indicata da `exp`.
Per invalidarlo prima serve una blacklist centralizzata, per esempio su Redis.
Una blacklist però riduce la natura puramente stateless del sistema.

### La vulnerabilità dell'algoritmo "none"
In passato molte librerie JWT soffrivano di una grave vulnerabilità legata al parametro `alg` dell'header.
Un attaccante poteva impostare `alg` su `none`.
Questo forzava il server ad accettare il token come valido, bypassando il controllo della firma.
Il payload poteva così essere manomesso a piacere.
Le librerie aggiornate rifiutano il valore `none`.

### Trasmissione sicura del token
Il metodo standard per trasmettere un JWT è l'header `Authorization` con schema Bearer.
Non si deve inserire il token nei parametri dell'URL (query string).
Un token nella query string resta visibile nei log dei server, nella cronologia del browser e nei proxy.

## 6. Esempio guidato
Vediamo un login che restituisce un token e una richiesta autenticata successiva.

### Request (login)
```http
POST /login HTTP/1.1
Host: api.example.com
Content-Type: application/json

{"username":"mara","password":"segreta"}
```

### Response (login)
```http
HTTP/1.1 200 OK
Content-Type: application/json

{"access_token":"eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxMjMiLCJleHAiOjE3MDB9.abc123","token_type":"Bearer"}
```

### Request (risorsa protetta)
```http
GET /users/123/orders HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxMjMiLCJleHAiOjE3MDB9.abc123
```

### Response (risorsa protetta)
```http
HTTP/1.1 200 OK
Content-Type: application/json

{"orders":[{"id":987,"totale":42.50}]}
```

Se il token mancasse, il server risponderebbe `401 Unauthorized`.
Se il token fosse valido ma senza permessi sufficienti, risponderebbe `403 Forbidden`.

## 7. Errori comuni
- Errore: considerare la Basic Authentication sicura perché "codificata" in Base64.
  Correzione: Base64 non è cifratura; usare Basic solo su HTTPS, preferendo schemi più robusti.
- Errore: salvare dati sensibili nel payload di un JWT pensando che siano cifrati.
  Correzione: il payload è solo codificato e leggibile da chiunque; non inserire segreti.
- Errore: passare il JWT nella query string dell'URL.
  Correzione: inviarlo nell'header `Authorization: Bearer`.
- Errore: trattare OAuth 2.0 come un sistema di autenticazione dell'identità.
  Correzione: OAuth 2.0 autorizza; per l'identità usare OpenID Connect.
- Errore: usare un `client_secret` statico dentro una SPA.
  Correzione: una SPA è un client pubblico; usare Authorization Code Flow con PKCE.
- Errore: inviare il Refresh Token a ogni richiesta verso il Resource Server.
  Correzione: il Refresh Token va solo all'Authorization Server, solo alla scadenza dell'Access Token.

## Riepilogo
- L'autenticazione verifica l'identità, l'autorizzazione stabilisce i permessi; 401 e 403 li distinguono.
- Gli schemi HTTP principali sono Basic (legacy) e Bearer (token), gestiti dagli header `Authorization` e `WWW-Authenticate`.
- OAuth 2.0 è un framework di autorizzazione con quattro ruoli, scope e flussi diversi; per l'identità si aggiunge OIDC.
- Un JWT ha forma `header.payload.signature`, è codificato in Base64URL ma non cifrato, ed è stateless.
- I limiti chiave sono la difficoltà di revoca del JWT e la storica vulnerabilità dell'algoritmo `none`.
