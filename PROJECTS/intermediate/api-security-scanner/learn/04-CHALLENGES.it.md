# 04-CHALLENGES.md

# Challenge di estensione

Hai costruito il progetto base. Ora rendilo tuo estendendolo con nuove funzionalità.

Queste challenge sono ordinate per difficoltà. Inizia con quelle più facili per acquisire sicurezza, poi affronta quelle più difficili quando vuoi approfondire.

## Challenge facili

### Challenge 1: Aggiungi la detection delle misconfiguration CORS

**Cosa costruire:**
Uno scanner che testa le policy CORS (Cross-Origin Resource Sharing) eccessivamente permissive. Verifica se l'API accetta richieste da qualsiasi origin, riflette l'header Origin, o consente le credenziali con origin wildcard.

**Perché è utile:**
Le misconfiguration CORS permettono agli attaccanti di rubare dati agli utenti autenticati. Se un'API accetta `Origin: https://evil.com` e risponde con `Access-Control-Allow-Origin: https://evil.com`, l'attaccante può fare richieste dal suo sito e leggere le risposte. È così che funzionava la vulnerabilità di information disclosure di PayPal (2020).

**Cosa imparerai:**
- Analisi degli header HTTP e pattern matching
- Tecniche di manipolazione dell'header Origin
- Comprensione del modello di sicurezza CORS
- Come i browser applicano la same-origin policy

**Suggerimenti:**
- Guarda `backend/scanners/auth_scanner.py` per esempi di manipolazione degli header
- Lo scanner deve fare richieste con header `Origin` diversi
- Controlla nella risposta `Access-Control-Allow-Origin`, `Access-Control-Allow-Credentials`
- Casi di test: wildcard (`*`), origin riflesso, origin `null`

**Verifica che funzioni:**
```python
# In scanners/cors_scanner.py
origins_to_test = [
    "https://evil.com",
    "null",
    "https://attacker.com",
]

for origin in origins_to_test:
    response = self.make_request("GET", "/", headers={"Origin": origin})
    
    allow_origin = response.headers.get("Access-Control-Allow-Origin")
    allow_creds = response.headers.get("Access-Control-Allow-Credentials")
    
    if allow_origin == origin or allow_origin == "*":
        # Found CORS misconfiguration
```

Verifica scansionando un endpoint con CORS permissivo. Dovrebbe rilevare `Access-Control-Allow-Origin: *` come vulnerabile.

### Challenge 2: Aggiungi il report sulla robustezza delle password

**Cosa costruire:**
Potenzia la registrazione per controllare la robustezza della password e fornire feedback. Testa contro le liste di password comuni (le prime 10000 di rockyou.txt), controlla i pattern (keyboard walk come "qwerty") e calcola l'entropia.

**Perché è utile:**
Le password deboli sono la causa n. 1 di account takeover. La breach di LinkedIn (2012) ha esposto 117 milioni di password, molte delle quali erano "123456" e "password". Un feedback in tempo reale aiuta gli utenti a creare password più robuste.

**Cosa imparerai:**
- Calcolo dell'entropia delle password (bit di casualità)
- Rilevamento di pattern nelle stringhe
- Lavorare con wordlist e dataset
- Bilanciare sicurezza e usabilità

**Suggerimenti:**
- Modifica `backend/schemas/user_schemas.py:26-38` (il validator della password)
- Scarica le liste di password comuni da SecLists su GitHub
- Calcola l'entropia: `log2(character_space ^ length)`
- Controlla caratteri sequenziali, caratteri ripetuti, parole del dizionario

**Approccio all'implementazione:**

1. **Aggiungi un'utility per il controllo delle password** in `backend/core/security.py`:
```python
def check_password_strength(password: str) -> dict[str, Any]:
    score = 0
    feedback = []
    
    # Length check
    if len(password) >= 12:
        score += 2
    elif len(password) >= 8:
        score += 1
    else:
        feedback.append("Password should be at least 12 characters")
    
    # Character variety
    if re.search(r"[A-Z]", password):
        score += 1
    if re.search(r"[a-z]", password):
        score += 1
    if re.search(r"[0-9]", password):
        score += 1
    if re.search(r"[^A-Za-z0-9]", password):
        score += 1
    
    # Check against common passwords
    common_passwords = load_common_passwords()  # Load from file
    if password.lower() in common_passwords:
        score = 0
        feedback.append("This is a commonly used password")
    
    # Patterns
    if re.search(r"(.)\1{2,}", password):  # Repeated chars
        score -= 1
        feedback.append("Avoid repeated characters")
    
    strength = "weak" if score < 3 else "medium" if score < 5 else "strong"
    
    return {
        "strength": strength,
        "score": score,
        "feedback": feedback,
    }
```

2. **Aggiorna l'endpoint di registrazione** per restituire le informazioni sulla robustezza (facoltativo, solo informativo).

**Testa i casi limite:**
- `password123` - Pattern comune
- `P@ssw0rd` - Soddisfa i requisiti ma è comunque debole
- `correct-horse-battery-staple` - Passphrase lunga (buona)
- `aaaaAAAA1111!!!!` - Soddisfa i requisiti ma contiene pattern

### Challenge 3: Aggiungi una dashboard di monitoraggio dei tempi di risposta

**Cosa costruire:**
Traccia e visualizza le metriche di performance dello scanner. Memorizza i tempi di risposta di ogni test, calcola i percentili (p50, p95, p99) e rileva quando i target rallentano.

**Perché è utile:**
I dati sulle performance aiutano a regolare i timeout dello scanner e a individuare problemi. Se la detection time-based dello scanner SQLi mostra un'alta varianza, sai che la baseline non è affidabile. Se tutti i test contro un target improvvisamente richiedono 10 volte di più, il target potrebbe star applicando un rate limiting nei tuoi confronti.

**Cosa imparerai:**
- Raccolta e aggregazione di metriche
- Calcolo dei percentili (non solo le medie)
- Visualizzazione di dati time-series
- Definizione di una baseline di performance

**Suggerimenti:**
- `BaseScanner.make_request()` traccia già `request_time` alla riga 117
- Memorizza i dati di timing in un nuovo modello `ScanMetrics`
- Calcola i percentili con `numpy.percentile()`
- Il frontend può usare Chart.js o Recharts per la visualizzazione

**Punti extra:**
Aggiungi un alerting quando il tempo di risposta p95 supera una soglia. Se una scansione media dura 30s ma il p95 è 120s, alcune scansioni vanno spesso in timeout.

## Challenge intermedie

### Challenge 4: Implementa la detection dello Stored XSS

**Cosa costruire:**
Estendi il testing XSS oltre il reflected XSS. Invia payload via POST, poi recupera la risorsa via GET per verificare se il payload persiste. Testa i campi dei commenti, i profili utente, qualsiasi dato che viene memorizzato e visualizzato.

**Perché è utile:**
Lo Stored XSS è più pericoloso del reflected perché colpisce tutti gli utenti, non solo la vittima che clicca su un link malevolo. Il worm Samy di MySpace (2005) usò lo stored XSS per infettare oltre 1 milione di profili in 20 ore.

**Applicazione nel mondo reale:**
Qualsiasi API con contenuti generati dagli utenti ha bisogno di testing per lo stored XSS. Forum, sistemi di commenti, pagine di profilo, upload di file con anteprima.

**Cosa imparerai:**
- Workflow di testing multi-step (invia poi recupera)
- Variazioni di encoding dei payload (URL encoding, entità HTML, Unicode)
- Detection XSS context-aware (contesto JavaScript vs contesto HTML)
- Riduzione dei falsi positivi nel fuzzy matching

**Approccio all'implementazione:**

1. **Crea lo scanner per lo stored XSS** in `backend/scanners/stored_xss_scanner.py`:
```python
class StoredXSSScanner(BaseScanner):
    def scan(self) -> TestResultCreate:
        # Generate unique marker
        marker = f"XSS_{uuid.uuid4().hex[:8]}"
        payload = f"<script>alert('{marker}')</script>"
        
        # Step 1: Submit payload
        submit_result = self._submit_payload(payload, marker)
        if not submit_result["submitted"]:
            return self._safe_result("Could not submit payload")
        
        # Step 2: Retrieve and check
        retrieve_result = self._retrieve_and_check(marker)
        if retrieve_result["vulnerable"]:
            return self._vulnerable_result(payload, marker, retrieve_result)
        
        return self._safe_result("No stored XSS detected")
    
    def _submit_payload(self, payload: str, marker: str) -> dict[str, Any]:
        # Try common endpoints
        endpoints = ["/comments", "/api/posts", "/api/profile"]
        
        for endpoint in endpoints:
            try:
                response = self.make_request(
                    "POST",
                    endpoint,
                    json={"content": payload, "text": payload, "bio": payload}
                )
                
                if response.status_code in (200, 201):
                    return {"submitted": True, "endpoint": endpoint}
            except Exception:
                continue
        
        return {"submitted": False}
    
    def _retrieve_and_check(self, marker: str) -> dict[str, Any]:
        # Retrieve content to see if payload persists
        response = self.make_request("GET", "/")
        
        if marker in response.text:
            # Check if it's encoded
            if f"&lt;script&gt;" in response.text:
                return {"vulnerable": False, "encoded": True}
            
            # Check if CSP would block it
            csp = response.headers.get("Content-Security-Policy", "")
            if "script-src 'none'" in csp or "script-src 'self'" in csp:
                return {"vulnerable": False, "csp_protected": True}
            
            return {"vulnerable": True, "marker": marker}
        
        return {"vulnerable": False}
```

2. **Aggiungi un cleanup** per rimuovere i payload di test dopo la scansione (buona cittadinanza).

3. **Testa i casi limite:**
- Il payload viene codificato in HTML (sicuro)
- Il payload viene memorizzato ma la CSP ne impedisce l'esecuzione (segnala comunque come stored XSS)
- Il payload compare in una risposta JSON (il contesto conta)

**Insidie:**
- Non lasciare payload di test nei sistemi di produzione (fai sempre il cleanup)
- Alcuni sistemi ritardano il rendering (cache, elaborazione asincrona) quindi il marker potrebbe non comparire subito
- Fai attenzione all'attribuzione utente - non associare i payload di test a utenti reali

### Challenge 5: Aggiungi il testing del bypass del rate limit con rotazione degli header

**Cosa costruire:**
Estendi il testing del bypass del rate limit con tecniche più sofisticate. Testa la rotazione dello User-Agent, la manipolazione dell'ID di sessione, il fuzzing dei timestamp e le variazioni dell'header Origin.

**Perché è impegnativa:**
I rate limiter moderni usano più segnali (IP, user agent, sessione, fingerprint). Devi testare le combinazioni in modo sistematico.

**Cosa imparerai:**
- Tecniche avanzate di evasione del rate limiting
- Manipolazione degli header HTTP su larga scala
- Analisi statistica dell'efficacia del rate limit
- Progettazione di matrici di test (combinatorial testing)

**Approccio all'implementazione:**

1. **Estendi lo scanner del rate limit** in `backend/scanners/rate_limit_scanner.py`:
```python
def _test_header_rotation_bypass(self) -> dict[str, Any]:
    """
    Test if rotating headers bypasses rate limits
    """
    user_agents = RateLimitBypassPayloads.USER_AGENT_ROTATION
    
    # Establish that rate limit exists
    for _ in range(20):
        response = self.make_request("GET", "/")
        if response.status_code == 429:
            break
    else:
        return {"bypass_successful": False, "reason": "No rate limit found"}
    
    # Try bypassing with User-Agent rotation
    success_count = 0
    for i in range(20):
        ua = user_agents[i % len(user_agents)]
        
        response = self.make_request(
            "GET", "/",
            headers={"User-Agent": ua}
        )
        
        if response.status_code != 429:
            success_count += 1
        else:
            break
    
    if success_count == 20:
        return {
            "bypass_successful": True,
            "bypass_method": "User-Agent Rotation",
            "requests_completed": success_count,
        }
    
    return {"bypass_successful": False}
```

2. **Testa le combinazioni**:
```python
combinations = [
    {"User-Agent": ua, "X-Forwarded-For": ip},
    {"User-Agent": ua, "Origin": origin},
    # etc
]
```

3. **Aggiungi un'analisi del timing** per rilevare i limiti soft (degradati ma non bloccati).

**Risorse:**
- Leggi "Bypassing Rate Limits" sul blog PortSwigger Research
- Studia la documentazione sul rate limiting di Cloudflare per capire contro cosa ti stai misurando

### Challenge 6: Implementa la detection dell'XML External Entity (XXE)

**Cosa costruire:**
Testa le vulnerabilità XXE nelle API che accettano XML. Invia payload con riferimenti a entità esterne e verifica la disclosure di file, l'SSRF o il denial of service.

**Perché è difficile:**
L'XXE richiede la comprensione dei parser XML, della sintassi DTD e dei diversi vettori di attacco (file disclosure vs SSRF vs billion laughs).

**Cosa imparerai:**
- Vulnerabilità del parsing XML
- Tecniche di esfiltrazione dei dati out-of-band
- Sfruttamento SSRF tramite XML
- Sicurezza della configurazione dei parser

**Implementazione:**

Crea `backend/scanners/xxe_scanner.py`:

```python
    def scan(self) -> TestResultCreate:
        # Test file disclosure
        file_disclosure = self._test_file_disclosure()
        if file_disclosure["vulnerable"]:
            return self._vulnerable_result(
                "XXE file disclosure detected",
                file_disclosure,
                Severity.CRITICAL
            )
        
        # Test SSRF
        ssrf_test = self._test_ssrf_xxe()
        if ssrf_test["vulnerable"]:
            return self._vulnerable_result(
                "XXE SSRF detected",
                ssrf_test,
                Severity.HIGH
            )
        
        return self._safe_result()
    
    def _test_file_disclosure(self) -> dict[str, Any]:
        # Payload to read /etc/passwd
        payload = """<?xml version="1.0" encoding="UTF-8"?>
        <!DOCTYPE foo [
          <!ENTITY xxe SYSTEM "file:///etc/passwd">
        ]>
        <data>&xxe;</data>"""
        
        response = self.make_request(
            "POST",
            "/",
            data=payload,
            headers={"Content-Type": "application/xml"}
        )
        
        # Check if file contents leaked
        if "root:" in response.text or "bin/bash" in response.text:
            return {
                "vulnerable": True,
                "payload": payload,
                "leaked_data": response.text[:200]
            }
        
        return {"vulnerable": False}
    
    def _test_ssrf_xxe(self) -> dict[str, Any]:
        # Test if parser makes external requests
        # Use Burp Collaborator or similar out-of-band detection
        
        collaborator_url = "http://burpcollaborator.net/unique-id"
        
        payload = f"""<?xml version="1.0" encoding="UTF-8"?>
        <!DOCTYPE foo [
          <!ENTITY xxe SYSTEM "{collaborator_url}">
        ]>
        <data>&xxe;</data>"""
        
        response = self.make_request(
            "POST",
            "/",
            data=payload,
            headers={"Content-Type": "application/xml"}
        )
        
        # Check if request was made (need out-of-band detection)
        # This is simplified - real implementation needs callback server
        
        return {"vulnerable": False, "note": "Manual verification required"}
```

**Criteri di successo:**
- Rileva l'XXE negli endpoint XML
- Testa più tipi di entità (file, http, parameter entity)
- Gestisce le diverse risposte del parser (messaggi di errore, timeout, dati)
- Riporta la severità in base all'impatto (file disclosure = CRITICAL, SSRF = HIGH)

## Challenge avanzate

### Challenge 7: Costruisci un sistema di plugin per gli scanner

**Cosa costruire:**
Crea un'architettura a plugin che permetta agli utenti di scrivere scanner personalizzati senza modificare il codice core. Gli scanner dovrebbero essere caricabili da una directory `plugins/`, con discovery e registrazione automatiche.

**Perché è difficile:**
Richiede il caricamento dinamico dei moduli, contratti di interfaccia, isolamento degli errori (un plugin rotto non deve mandare in crash lo scanner) e documentazione per gli sviluppatori di plugin.

**Cosa imparerai:**
- Introspezione dei moduli Python e import dinamici
- Abstract base class e design delle interfacce
- Pattern di architettura a plugin
- Sandboxing e isolamento degli errori

**Modifiche all'architettura necessarie:**

```
Current:
scanner_mapping = {
    TestType.SQLI: SQLiScanner,  # Hardcoded
    ...
}

New:
scanner_registry = ScannerRegistry()
scanner_registry.load_builtin_scanners()
scanner_registry.discover_plugins("plugins/")

scanner_mapping = scanner_registry.get_all_scanners()
```
**Passi di implementazione:**

1. **Definisci l'interfaccia dei plugin** in `backend/scanners/plugin_interface.py`:
```python
from abc import ABC, abstractmethod
from typing import Any

class ScannerPlugin(ABC):
    """
    Base class for scanner plugins
    
    All plugins must inherit from this and implement required methods
    """
    
    @property
    @abstractmethod
    def name(self) -> str:
        """Unique scanner name (e.g., 'custom_xxe')"""
        pass
    
    @property
    @abstractmethod
    def version(self) -> str:
        """Plugin version (semver: '1.0.0')"""
        pass
    
    @property
    @abstractmethod
    def test_type(self) -> str:
        """Test type identifier (must be unique)"""
        pass
    
    @abstractmethod
    def scan(self, target_url: str, **kwargs: Any) -> dict[str, Any]:
        """
        Execute scan and return results
        
        Returns:
            dict with keys: vulnerable, details, evidence, recommendations
        """
        pass
    
    def validate(self) -> bool:
        """
        Validate plugin configuration
        Override this to add custom validation
        """
        return True
```

2. **Crea il plugin loader** in `backend/scanners/plugin_loader.py`:
```python
import os
import importlib.util
from pathlib import Path
from typing import Type

class PluginLoader:
    def __init__(self, plugin_dir: str = "plugins"):
        self.plugin_dir = Path(plugin_dir)
        self.loaded_plugins: dict[str, Type[ScannerPlugin]] = {}
    
    def discover_plugins(self) -> list[Type[ScannerPlugin]]:
        """
        Discover and load all plugins from plugin directory
        """
        if not self.plugin_dir.exists():
            return []
        
        plugins = []
        
        for file in self.plugin_dir.glob("*.py"):
            if file.stem.startswith("_"):
                continue  # Skip __init__.py, _template.py
            
            try:
                plugin_class = self._load_plugin_from_file(file)
                if plugin_class:
                    plugins.append(plugin_class)
            except Exception as e:
                print(f"Failed to load plugin {file}: {e}")
                continue  # Don't let broken plugins crash the scanner
        
        return plugins
    
    def _load_plugin_from_file(self, filepath: Path) -> Type[ScannerPlugin] | None:
        """
        Dynamically load plugin class from Python file
        """
        spec = importlib.util.spec_from_file_location(
            f"plugins.{filepath.stem}",
            filepath
        )
        
        if spec is None or spec.loader is None:
            return None
        
        module = importlib.util.module_from_spec(spec)
        spec.loader.exec_module(module)
        
        # Find ScannerPlugin subclass
        for attr_name in dir(module):
            attr = getattr(module, attr_name)
            
            if (isinstance(attr, type) and 
                issubclass(attr, ScannerPlugin) and 
                attr is not ScannerPlugin):
                
                # Instantiate to validate
                instance = attr()
                if instance.validate():
                    return attr
        
        return None
```

3. **Crea un plugin di esempio** in `plugins/example_scanner.py`:
```python
from scanners.plugin_interface import ScannerPlugin
from scanners.base_scanner import BaseScanner

class ExampleScanner(ScannerPlugin, BaseScanner):
    """
    Example scanner plugin
    
    Copy this to create your own scanners
    """
    
    @property
    def name(self) -> str:
        return "example_scanner"
    
    @property
    def version(self) -> str:
        return "1.0.0"
    
    @property
    def test_type(self) -> str:
        return "example"
    
    def scan(self, target_url: str, **kwargs) -> dict[str, Any]:
        """
        Your scanner logic here
        """
        self.target_url = target_url
        
        response = self.make_request("GET", "/")
        
        # Implement your detection logic
        vulnerable = self._check_for_vulnerability(response)
        
        return {
            "vulnerable": vulnerable,
            "details": "Example vulnerability found" if vulnerable else "Safe",
            "evidence": {"status_code": response.status_code},
            "recommendations": ["Fix the issue"] if vulnerable else [],
        }
    
    def _check_for_vulnerability(self, response) -> bool:
        # Your detection logic
        return False
```

**Strategia di testing:**
- Unit test della plugin discovery (crea plugin di test in una directory temporanea)
- Testa l'isolamento degli errori (un plugin rotto non manda in crash lo scanner)
- Testa il versioning dei plugin (gestisci più versioni dello stesso plugin)

**Sfide note:**

1. **Conflitti di naming dei plugin**
   - Problema: Due plugin registrano lo stesso `test_type`
   - Suggerimento: Usa il namespacing oppure first-come-first-served con warning

2. **Sicurezza dei plugin**
   - Problema: Plugin malevoli potrebbero accedere al database e al file system
   - Suggerimento: Esegui i plugin in modalità ristretta, limita gli import, usa l'isolamento tramite subprocess

**Risorse:**
- Leggi del modulo `importlib` di Python
- Studia il sistema di estensioni di Flask per ispirazione
- Guarda l'architettura dei plugin di pytest

### Challenge 8: Implementa la detection del Blind SSRF con canali Out-of-Band

**Cosa costruire:**
Rileva le vulnerabilità di Server-Side Request Forgery anche quando le risposte non fanno trapelare dati. Usa callback DNS, callback HTTP o attacchi basati sul timing per confermare l'SSRF.

**Perché è impegnativa:**
Il Blind SSRF non mostra i dati della risposta. Serve un'infrastruttura (un callback server) per rilevare quando il target effettua richieste.

**Tempo stimato:**
8-12 ore per un'implementazione di base, più di 20 ore per una versione robusta da produzione.

**Prerequisiti:**
Dovresti aver completato prima le challenge SQLi e Auth perché questa si basa sull'analisi del timing delle richieste e sulla manipolazione degli header.

**Cosa imparerai:**
- Tecniche di detection delle vulnerabilità out-of-band
- Esfiltrazione DNS e meccanismi di callback
- Detection blind time-based con significatività statistica
- Costruzione dell'infrastruttura di supporto (callback server)

**Pianificare questa funzionalità:**

Prima di scrivere codice, rifletti su:
- Come funziona il callback server? (DNS? HTTP?)
- E se il target ha un egress filtering? (Non può fare richieste in uscita)
- Come associ i callback alle scansioni? (Identificatori univoci)
- Qual è il tuo tasso di falsi positivi? (Rumore di rete, prefetching delle CDN)

**Architettura ad alto livello:**

```
Scanner                    Callback Server               Target API
   |                              |                            |
   |--SSRF Payload-------------->|                            |
   |  (http://callback.io/abc)   |                            |
   |                              |                            |
   |                              |<---HTTP Request------------|
   |                              |   (GET /abc)               |
   |                              |                            |
   |<--Confirmation---------------|                            |
   |   (Received request abc)     |                            |
```

**Fasi di implementazione:**

**Fase 1: Costruisci il Callback Server** (3-4 ore)

Crea `backend/ssrf_callback_server.py`:

```python
from fastapi import FastAPI, Request
from datetime import datetime, timedelta
import asyncio

app = FastAPI()

# Store received callbacks
callbacks_received: dict[str, dict] = {}

@app.get("/callback/{identifier}")
async def handle_callback(identifier: str, request: Request):
    """
    Receive SSRF callback
    """
    callbacks_received[identifier] = {
        "timestamp": datetime.utcnow(),
        "headers": dict(request.headers),
        "client_ip": request.client.host,
    }
    
    return {"status": "received"}

@app.get("/check/{identifier}")
async def check_callback(identifier: str):
    """
    Check if callback was received
    """
    callback = callbacks_received.get(identifier)
    
    if callback:
        # Clean up old callback
        del callbacks_received[identifier]
        return {"received": True, "data": callback}
    
    return {"received": False}
```

Esegui questo su un server pubblico (DigitalOcean, AWS) con un dominio che punta a esso.

**Fase 2: Implementa lo scanner SSRF** (3-4 ore)

Crea `backend/scanners/ssrf_scanner.py`:

```python
import uuid
import time

class SSRFScanner(BaseScanner):
    def __init__(self, target_url: str, callback_server: str, **kwargs):
        super().__init__(target_url, **kwargs)
        self.callback_server = callback_server  # http://callback.io
    
    def scan(self) -> TestResultCreate:
        # Test URL parameters
        url_param_test = self._test_url_parameters()
        if url_param_test["vulnerable"]:
            return self._vulnerable_result(url_param_test)
        
        # Test headers
        header_test = self._test_headers()
        if header_test["vulnerable"]:
            return self._vulnerable_result(header_test)
        
        return self._safe_result()
    
    def _test_url_parameters(self) -> dict[str, Any]:
        # Generate unique identifier
        identifier = str(uuid.uuid4())
        
        # Payloads to test
        params = ["url", "callback", "webhook", "redirect", "link"]
        
        for param in params:
            callback_url = f"{self.callback_server}/callback/{identifier}"
            
            # Submit SSRF payload
            response = self.make_request(
                "GET",
                f"/?{param}={callback_url}"
            )
            
            # Wait for callback
            time.sleep(5)
            
            # Check if callback was received
            check_response = requests.get(
                f"{self.callback_server}/check/{identifier}"
            )
            
            if check_response.json()["received"]:
                return {
                    "vulnerable": True,
                    "parameter": param,
                    "callback_data": check_response.json()["data"],
                }
        
        return {"vulnerable": False}
    
    def _test_headers(self) -> dict[str, Any]:
        """
        Test headers like Referer, X-Forwarded-Host for SSRF
        """
        identifier = str(uuid.uuid4())
        callback_url = f"{self.callback_server}/callback/{identifier}"
        
        headers_to_test = [
            "Referer",
            "X-Forwarded-Host",
            "X-Original-URL",
            "Host",
        ]
        
        for header in headers_to_test:
            response = self.make_request(
                "GET",
                "/",
                headers={header: callback_url}
            )
            
            time.sleep(5)
            
            check_response = requests.get(
                f"{self.callback_server}/check/{identifier}"
            )
            
            if check_response.json()["received"]:
                return {
                    "vulnerable": True,
                    "header": header,
                    "callback_data": check_response.json()["data"],
                }
        
        return {"vulnerable": False}
```

**Fase 3: Aggiungi un'alternativa con callback DNS** (2-3 ore)

Alcuni ambienti bloccano HTTP ma consentono il DNS. Usa i callback DNS:

```python
def _test_dns_callback(self) -> dict[str, Any]:
    """
    Use DNS exfiltration for SSRF detection
    """
    identifier = uuid.uuid4().hex[:8]
    dns_domain = f"{identifier}.callback.io"  # Your DNS server
    
    # Submit payload that triggers DNS lookup
    payloads = [
        f"http://{dns_domain}/",
        f"https://{dns_domain}/",
        f"//{dns_domain}/",
    ]
    
    for payload in payloads:
        self.make_request("GET", f"/?url={payload}")
    
    time.sleep(5)
    
    # Check DNS logs for lookup
    # (Requires DNS server that logs queries)
    
    return {"vulnerable": False}  # Implement DNS checking
```

**Fase 4: Aggiungi la detection time-based** (3-4 ore)

Per il Blind SSRF completo (nessun callback possibile), usa il timing:

```python
def _test_timing_based_ssrf(self) -> dict[str, Any]:
    """
    Detect SSRF via timing differences
    
    Internal IPs respond fast, external IPs are slower
    """
    # Baseline with external IP (slow)
    baseline_times = []
    for _ in range(5):
        start = time.time()
        self.make_request("GET", "/?url=http://example.com/")
        baseline_times.append(time.time() - start)
    
    baseline_avg = statistics.mean(baseline_times)
    
    # Test with internal IPs (should be faster if SSRF exists)
    internal_ips = [
        "http://127.0.0.1/",
        "http://localhost/",
        "http://192.168.1.1/",
    ]
    
    for ip in internal_ips:
        test_times = []
        for _ in range(5):
            start = time.time()
            self.make_request("GET", f"/?url={ip}")
            test_times.append(time.time() - start)
        
        test_avg = statistics.mean(test_times)
        
        # If internal IP is significantly faster, SSRF likely exists
        if test_avg < (baseline_avg * 0.5):
            return {
                "vulnerable": True,
                "technique": "timing-based",
                "baseline_time": baseline_avg,
                "internal_ip_time": test_avg,
            }
    
    return {"vulnerable": False}
```

**Testare lo scanner:**

Imposta un'API di test vulnerabile:
Set up a vulnerable test API:
```python
# test_api.py
from flask import Flask, request
import requests

app = Flask(__name__)

@app.route("/")
def ssrf_vulnerable():
    url = request.args.get("url")
    if url:
        # Vulnerable - makes request to user-supplied URL
        requests.get(url)
    return "OK"

if __name__ == "__main__":
    app.run(port=5000)
```

Esegui lo scanner contro di essa:

Run scanner against it:
```bash
# Should detect SSRF and receive callback
python -m backend.scanners.ssrf_scanner http://localhost:5000
```

**Criteri di successo:**
- [ ] Rileva l'SSRF nei parametri URL
- [ ] Rileva l'SSRF negli header
- [ ] Usa i callback DNS quando HTTP è bloccato
- [ ] Ricade sulla detection time-based
- [ ] Gestisce con grazia i ritardi di rete
- [ ] Fa il cleanup dei callback di test

### Challenge 9: Costruisci un generatore completo di report di vulnerabilità

**Cosa costruire:**
Genera report PDF professionali dei risultati della scansione con riassunto esecutivo, dettagli tecnici, passi di remediation e scoring CVSS.

**Tempo stimato:**
10-15 ore per un'implementazione completa con styling e grafici.

**Cosa imparerai:**
- Generazione di PDF con ReportLab o WeasyPrint
- Calcolo del punteggio CVSS
- Visualizzazione dei dati (matplotlib, plotly)
- Templating e styling dei report
- Standard di documentazione professionale

**Fasi di implementazione:**

**Fase 1: Struttura dei dati del report** (2-3 ore)

Crea `backend/services/report_service.py`:

```python
from dataclasses import dataclass
from typing import List
import matplotlib.pyplot as plt
from reportlab.lib.pagesizes import letter
from reportlab.pdfgen import canvas

@dataclass
class VulnerabilitySummary:
    critical: int
    high: int
    medium: int
    low: int
    info: int
    
    @property
    def total(self) -> int:
        return self.critical + self.high + self.medium + self.low + self.info

class ReportGenerator:
    def __init__(self, scan_id: int, db: Session):
        self.scan = ScanRepository.get_by_id(db, scan_id)
        self.db = db
    
    def generate_pdf(self, output_path: str) -> str:
        """
        Generate comprehensive PDF report
        """
        c = canvas.Canvas(output_path, pagesize=letter)
        width, height = letter
        
        # Cover page
        self._add_cover_page(c, width, height)
        c.showPage()
        
        # Executive summary
        self._add_executive_summary(c, width, height)
        c.showPage()
        
        # Vulnerability details
        self._add_vulnerability_details(c, width, height)
        
        c.save()
        return output_path
    
    def _add_cover_page(self, c, width, height):
        c.setFont("Helvetica-Bold", 24)
        c.drawString(100, height - 100, "Security Scan Report")
        
        c.setFont("Helvetica", 14)
        c.drawString(100, height - 150, f"Target: {self.scan.target_url}")
        c.drawString(100, height - 180, f"Date: {self.scan.scan_date}")
        
        # Add severity chart
        self._add_severity_chart(c, width, height - 400)
    
    def _add_severity_chart(self, c, x, y):
        """
        Create pie chart of vulnerabilities by severity
        """
        summary = self._calculate_summary()
        
        # Create matplotlib chart
        fig, ax = plt.subplots(figsize=(4, 4))
        
        labels = ["Critical", "High", "Medium", "Low", "Info"]
        sizes = [summary.critical, summary.high, summary.medium, summary.low, summary.info]
        colors = ["#dc2626", "#ea580c", "#f59e0b", "#3b82f6", "#6b7280"]
        
        ax.pie(sizes, labels=labels, colors=colors, autopct="%1.1f%%")
        
        # Save to temp file and embed in PDF
        chart_path = "/tmp/severity_chart.png"
        plt.savefig(chart_path)
        plt.close()
        
        c.drawImage(chart_path, x, y, width=300, height=300)
```

**Fase 2: Scoring CVSS** (2-3 ore)

Aggiungi il calcolo del punteggio CVSS:

```python
class CVSSCalculator:
    """
    Calculate CVSS v3.1 scores for vulnerabilities
    """
    
    def calculate_score(
        self,
        test_result: TestResult
    ) -> dict[str, Any]:
        """
        Calculate CVSS score based on test result
        """
        # Base metrics
        av = self._attack_vector(test_result.test_name)  # Network
        ac = self._attack_complexity(test_result)  # Low/High
        pr = self._privileges_required(test_result)  # None/Low/High
        ui = self._user_interaction(test_result)  # None/Required
        s = "U"  # Scope: Unchanged
        c = self._confidentiality_impact(test_result)  # High/Low/None
        i = self._integrity_impact(test_result)
        a = self._availability_impact(test_result)
        
        # Calculate base score using CVSS formula
        # (Simplified - full implementation in CVSS spec)
        
        impact = 1 - ((1 - c) * (1 - i) * (1 - a))
        exploitability = 8.22 * av * ac * pr * ui
        
        if impact <= 0:
            base_score = 0
        else:
            base_score = min(
                (impact + exploitability) * 1.08,
                10.0
            )
        
        return {
            "score": round(base_score, 1),
            "severity": self._score_to_severity(base_score),
            "vector": f"CVSS:3.1/AV:{av}/AC:{ac}/PR:{pr}/UI:{ui}/S:{s}/C:{c}/I:{i}/A:{a}",
        }
```

**Fase 3: Guida alla remediation** (2-3 ore)

Aggiungi una remediation dettagliata per ogni tipo di vulnerabilità:

```python
REMEDIATION_GUIDES = {
    TestType.SQLI: {
        "title": "SQL Injection Remediation",
        "steps": [
            "1. Use parameterized queries (prepared statements) for all database operations",
            "2. Never concatenate user input into SQL strings",
            "3. Use ORMs like SQLAlchemy that handle escaping automatically",
            "4. Implement input validation to reject suspicious patterns",
            "5. Use least privilege database accounts",
        ],
        "code_example": """
# Before (Vulnerable)
query = f"SELECT * FROM users WHERE email = '{email}'"

# After (Safe)
query = db.query(User).filter(User.email == email).first()
        """,
        "references": [
            "OWASP SQL Injection Prevention Cheat Sheet",
            "CWE-89: Improper Neutralization of Special Elements",
        ],
    },
    # ... other test types
}
```

**Criteri di successo:**
- [ ] Genera PDF dall'aspetto professionale
- [ ] Include un riassunto esecutivo (finding di alto livello)
- [ ] Mostra la suddivisione delle vulnerabilità per severità
- [ ] Calcola i punteggi CVSS
- [ ] Fornisce passi di remediation specifici
- [ ] Include esempi di codice
- [ ] Grafici e visualizzazioni

## Mix and Match

Combina le funzionalità per progetti più grandi:

**Idea di progetto 1: Piattaforma completa di sicurezza API**
- Combina la Challenge 7 (sistema di plugin) + la Challenge 9 (generazione report) + la Challenge 8 (detection SSRF)
- Aggiungi una web UI per la pianificazione delle scansioni
- Aggiungi notifiche via email quando vengono trovate vulnerabilità
- Risultato: Testing continuo della sicurezza API pronto per la produzione

**Idea di progetto 2: Integrazione della sicurezza in CI/CD**
- Combina la Challenge 4 (stored XSS) + la Challenge 6 (XXE) + la Challenge 2 (robustezza delle password)
- Costruisci una GitHub Action che esegue le scansioni a ogni commit
- Fai fallire le build se vengono trovate vulnerabilità critiche
- Risultato: Testing di sicurezza nella pipeline di sviluppo

## Challenge di integrazione nel mondo reale

### Integra con Slack per le notifiche

**L'obiettivo:**
Invia messaggi Slack quando le scansioni si completano o vengono trovate vulnerabilità.

**Cosa ti servirà:**
- Un workspace Slack con accesso admin
- Una Slack App con un incoming webhook
- Comprensione del Block Kit di Slack per i messaggi ricchi

**Piano di implementazione:**

1. **Crea una Slack App** su https://api.slack.com/apps
2. **Abilita gli Incoming Webhook** e ottieni l'URL del webhook
3. **Aggiungi alla configurazione** in `.env`:
```bash
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/WEBHOOK/URL
```

4. **Create notification service** in `backend/services/notification_service.py`:
```python
import requests
from typing import List

class SlackNotifier:
    def __init__(self, webhook_url: str):
        self.webhook_url = webhook_url
    
    def send_scan_complete(self, scan: Scan):
        """
        Send notification when scan completes
        """
        summary = self._calculate_summary(scan)
        
        blocks = [
            {
                "type": "header",
                "text": {
                    "type": "plain_text",
                    "text": "🔒 Security Scan Complete"
                }
            },
            {
                "type": "section",
                "fields": [
                    {"type": "mrkdwn", "text": f"*Target:*\n{scan.target_url}"},
                    {"type": "mrkdwn", "text": f"*Date:*\n{scan.scan_date}"},
                ]
            },
            {
                "type": "section",
                "fields": [
                    {"type": "mrkdwn", "text": f"*Critical:* {summary.critical}"},
                    {"type": "mrkdwn", "text": f"*High:* {summary.high}"},
                    {"type": "mrkdwn", "text": f"*Medium:* {summary.medium}"},
                ]
            }
        ]
        
        if summary.critical > 0 or summary.high > 0:
            blocks.append({
                "type": "section",
                "text": {
                    "type": "mrkdwn",
                    "text": "⚠️ *Critical vulnerabilities found!* Review immediately."
                }
            })
        
        requests.post(self.webhook_url, json={"blocks": blocks})
```

5. **Collega al servizio di scan** alla fine di `ScanService.run_scan()`:
```python
# After saving results
if settings.SLACK_WEBHOOK_URL:
    notifier = SlackNotifier(settings.SLACK_WEBHOOK_URL)
    notifier.send_scan_complete(scan)
```

**Attenzione a:**
- Rate limit sui webhook Slack (1 messaggio al secondo)
- Limiti di dimensione dei messaggi (3000 caratteri per i text block)
- Gestione degli errori (l'URL del webhook potrebbe non essere valido)

### Esegui il deploy in produzione (AWS/DigitalOcean)

**L'obiettivo:**
Far funzionare questo progetto in produzione su infrastruttura reale.

**Cosa imparerai:**
- Deploy Docker sul cloud
- Gestione delle variabili d'ambiente in produzione
- Setup del certificato SSL/TLS
- Backup e manutenzione del database

**Passi:**

1. **Fai il provisioning del server** (DigitalOcean Droplet, AWS EC2)
   - Ubuntu 24.04 LTS
   - Minimo 2GB di RAM (4GB raccomandati)
   - Porte aperte: 80 (HTTP), 443 (HTTPS), 22 (SSH)

2. **Installa Docker** sul server:
```bash
ssh root@your-server-ip

curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh

apt install docker-compose-plugin
```

3. **Clona il progetto** e configuralo:
```bash
git clone https://github.com/yourusername/api-security-scanner.git
cd api-security-scanner

cp .env.example .env
nano .env  # Edit production settings
```

4. **Imposta le variabili d'ambiente di produzione**:
```bash
# .env for production
SECRET_KEY=$(openssl rand -hex 32)
DEBUG=false
DATABASE_URL=postgresql://...
CORS_ORIGINS=https://yourdomain.com
```

5. **Configura SSL con Let's Encrypt**:
```bash
# Install certbot
apt install certbot python3-certbot-nginx

# Get certificate
certbot --nginx -d yourdomain.com
```

6. **Aggiorna la configurazione nginx** in `conf/nginx/prod.nginx` per usare SSL:
```nginx
server {
    listen 443 ssl http2;
    server_name yourdomain.com;
    
    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;
    
    # ... rest of config
}
```

7. **Fai il deploy**:
```bash
docker compose -f compose.yml up -d --build
```

**Checklist di produzione:**
- [ ] Hai cambiato `SECRET_KEY` con un valore casuale
- [ ] Hai impostato `DEBUG=false`
- [ ] Hai configurato un database reale (non localhost)
- [ ] Hai configurato i certificati SSL
- [ ] Hai configurato il firewall (ufw allow 80,443,22)
- [ ] Hai configurato i backup (dump del database su S3/Spaces)
- [ ] Hai configurato il monitoring (Prometheus, Grafana, o il cloud provider)
- [ ] Hai configurato il logging (log centralizzati, rotazione dei log)

## Challenge di performance

### Challenge: Gestire 1000 scansioni concorrenti

**L'obiettivo:**
Scalare il sistema per gestire 1000 scansioni in esecuzione simultanea senza crash o rallentamenti.

**Collo di bottiglia attuale:**
Le scansioni vengono eseguite in modo sincrono nel request handler. Le connessioni al database sono limitate a 5 di default. L'utilizzo della memoria cresce linearmente con le scansioni concorrenti.

**Approcci di ottimizzazione:**

**Approccio 1: Task Queue con Celery**
- Come: Sposta le scansioni su worker in background con Celery + Redis
- Guadagno: API non bloccante, scaling orizzontale, logica di retry
- Compromesso: Complessità aggiuntiva, serve un'infrastruttura Redis

Implementazione:
```bash
# Install Celery
pip install celery redis

# backend/celery_app.py
from celery import Celery

celery = Celery(
    "scanner",
    broker="redis://localhost:6379/0",
    backend="redis://localhost:6379/0"
)

@celery.task
def run_scan_async(scan_id: int):
    """Run scan in background"""
    db = SessionLocal()
    scan = ScanRepository.get_by_id(db, scan_id)
    # ... execute scan
```

**Approccio 2: Aumenta le connessioni al database**
- Come: Configura il connection pooling in SQLAlchemy
- Guadagno: Più operazioni concorrenti sul database
- Compromesso: Maggiore utilizzo della memoria, limiti di connessione di Postgres


```python
# core/database.py
engine = create_engine(
    settings.DATABASE_URL,
    pool_size=20,  # Increased from 5
    max_overflow=40,  # Up to 60 total connections
    pool_pre_ping=True,
)
```

**Fai il benchmark:**
```bash
# Load testing with Apache Bench
ab -n 1000 -c 100 http://localhost:8000/scans/ \
   -H "Authorization: Bearer TOKEN" \
   -p scan_request.json
```

Metriche obiettivo:
- Throughput: 50+ richieste/secondo
- Latenza p95: <2 secondi (per la creazione della scansione, non l'esecuzione)
- Tasso di errore: <1%

### Challenge: Riduci l'utilizzo della memoria dello scanner

**L'obiettivo:**
Riduci l'utilizzo della memoria del 50% quando esegui 100 scansioni concorrenti.

**Fai prima il profiling:**
```python
# Add memory profiling
import tracemalloc

tracemalloc.start()

# Run scan
scanner.scan()

current, peak = tracemalloc.get_traced_memory()
print(f"Current: {current / 1024 / 1024:.2f} MB")
print(f"Peak: {peak / 1024 / 1024:.2f} MB")

tracemalloc.stop()
```

**Aree comuni di ottimizzazione:**
- Memorizzare i corpi delle risposte in memoria (le risposte grandi consumano RAM)
- Oggetti sessione non ripuliti
- Il JSON delle evidenze memorizza intere risposte

Correzione:
```python
# Instead of storing full response
evidence = {"response": response.text}  # 100KB+

# Store summary
evidence = {
    "status_code": response.status_code,
    "length": len(response.text),
    "excerpt": response.text[:500],  # Just first 500 chars
}
```

## Challenge di sicurezza

### Challenge: Aggiungi la verifica della firma dei webhook

**Cosa implementare:**
Quando invii i risultati della scansione ai webhook, firma il payload così i destinatari possono verificare che provenga dal tuo scanner.

**Threat model:**
Questo protegge contro:
- Un attaccante che invia risultati di scansione falsi al webhook
- Manomissione man-in-the-middle dei dati del webhook

**Implementazione:**

```python
import hmac
import hashlib

class WebhookSigner:
    def __init__(self, secret: str):
        self.secret = secret.encode()
    
    def sign(self, payload: str) -> str:
        """
        Create HMAC-SHA256 signature of payload
        """
        signature = hmac.new(
            self.secret,
            payload.encode(),
            hashlib.sha256
        ).hexdigest()
        
        return signature
    
    def verify(self, payload: str, signature: str) -> bool:
        """
        Verify signature is valid
        """
        expected = self.sign(payload)
        return hmac.compare_digest(expected, signature)
```

Utilizzo:
```python
# When sending webhook
signer = WebhookSigner(settings.WEBHOOK_SECRET)
payload_json = json.dumps(scan_data)
signature = signer.sign(payload_json)

requests.post(
    webhook_url,
    json=scan_data,
    headers={"X-Signature": signature}
)

# Receiver verifies
received_signature = request.headers["X-Signature"]
if not signer.verify(payload_json, received_signature):
    raise HTTPException(401, "Invalid signature")
```

### Challenge: Supera la conformità OWASP Top 10

**L'obiettivo:**
Rendere questo progetto conforme a OWASP Top 10 2021.

**Lacune attuali:**
- A01:2021-Broken Access Control: controlli IDOR implementati ✓
- A02:2021-Cryptographic Failures: Bcrypt per le password ✓, ma nessuna applicazione forzata di HTTPS
- A03:2021-Injection: detection della SQL injection ✓, ma si potrebbe aggiungere la command injection
- A07:2021-Identification and Authentication Failures: validazione JWT ✓
- A09:2021-Security Logging and Monitoring Failures: nessun audit logging ❌

**Remediation:**

Aggiungi l'audit logging:
```python
# models/AuditLog.py
class AuditLog(BaseModel):
    __tablename__ = "audit_logs"
    
    user_id = Column(Integer, ForeignKey("users.id"))
    action = Column(String(100))  # "scan_created", "login_success"
    ip_address = Column(String(45))  # Support IPv6
    user_agent = Column(String(255))
    details = Column(JSON)

# Log every important action
def log_action(user_id: int, action: str, request: Request, **details):
    log = AuditLog(
        user_id=user_id,
        action=action,
        ip_address=request.client.host,
        user_agent=request.headers.get("User-Agent"),
        details=details,
    )
    db.add(log)
    db.commit()
```

## Idee per contribuire

Hai finito una challenge? Condividila:

1. **Fai il fork del repo** su github.com/yourusername/api-security-scanner
2. **Implementa la tua estensione** in un feature branch (`git checkout -b feature/xxe-scanner`)
3. **Documentala** - Aggiungi alla cartella learn, aggiorna il README
4. **Invia una PR** con:
   - Codice dell'implementazione
   - Unit test (coverage minima dell'80%)
   - Test di integrazione
   - Documentazione (docstring, aggiornamenti al README)
   - Esempio di utilizzo

Le estensioni migliori potrebbero essere integrate nel progetto principale.

## Sfida te stesso ancora di più

### Costruisci qualcosa di nuovo

Usa i concetti che hai imparato qui per costruire:

- **Scanner di sicurezza GraphQL** - Testa i leak di introspection, i limiti di profondità delle query, gli attacchi di batching
- **Tester di sicurezza WebSocket** - Testa l'injection, l'autenticazione, il rate limiting nelle connessioni WebSocket
- **Scanner per API cloud** - Testa le API AWS, Azure, GCP per le misconfiguration (bucket S3 pubblici, database aperti)

### Studia implementazioni reali

Confronta la tua implementazione con i tool di produzione:

- **Burp Suite** - Studia come il loro active scanner rileva la SQLi (tecniche multiple, testing adattivo)
- **OWASP ZAP** - Guarda l'architettura a plugin del loro scanner
- **Nuclei** - Dai un'occhiata al loro sistema di template basato su YAML per controlli personalizzati

Leggi il loro codice (molti sono open source), comprendi i loro compromessi, ruba le loro buone idee.

### Scrivine

Documenta la tua estensione:

- Un post sul blog che spiega cosa hai costruito e perché ("Adding XXE Detection to an API Scanner")
- Un tutorial che altri possano seguire ("How to Build a CORS Misconfiguration Scanner")
- Un confronto con approcci alternativi ("Callback-based SSRF Detection vs Timing-based")

Insegnare agli altri è il modo migliore per verificare di aver capito davvero.

## Ottenere aiuto

Bloccato su una challenge?

1. **Fai debug in modo sistematico**
   - Cosa ti aspettavi che succedesse?
   - Cosa è successo davvero?
   - Qual è il caso di test più piccolo che lo riproduce?
   - Cosa hai già provato?

2. **Leggi il codice esistente**
   - Lo scanner SQLi fa detection time-based (simile al timing SSRF)
   - Lo scanner Auth fa testing multi-step (simile allo stored XSS)
   - Il base scanner ha una logica di retry che puoi riusare

3. **Cerca problemi simili**
   - Stack Overflow con i tag: python, fastapi, security-testing
   - Issue GitHub in progetti simili (ZAP, Nuclei, SQLMap)
   - Forum InfoSec come /r/netsec, /r/AskNetsec

4. **Chiedi aiuto con il contesto**
   - Posta nelle discussion o issue del progetto
   - Includi: cosa stai cercando di costruire, cosa hai provato, cosa è successo, cosa ti aspettavi
   - Fornisci snippet di codice e messaggi di errore
   - Non limitarti a incollare i messaggi di errore senza spiegazione

## Tracker di completamento delle challenge

Traccia i tuoi progressi:

- [ ] Challenge facile 1: Detection CORS
- [ ] Challenge facile 2: Robustezza delle password
- [ ] Challenge facile 3: Monitoraggio dei tempi di risposta
- [ ] Challenge intermedia 4: Stored XSS
- [ ] Challenge intermedia 5: Bypass avanzato del rate limit
- [ ] Challenge intermedia 6: Detection XXE
- [ ] Challenge avanzata 7: Sistema di plugin
- [ ] Challenge avanzata 8: Blind SSRF
- [ ] Challenge expert 9: Generazione report

**Challenge bonus:**
- [ ] Integrazione Slack
- [ ] Deploy in produzione
- [ ] Performance: 1000 scansioni concorrenti
- [ ] Sicurezza: firme dei webhook
- [ ] Conformità: OWASP Top 10

Le hai completate tutte? Hai padroneggiato il testing della sicurezza delle API. È il momento di costruire qualcosa di nuovo, contribuire a tool di sicurezza open source, o applicare queste competenze professionalmente in ruoli di penetration testing o security engineering.
