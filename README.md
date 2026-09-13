# WireGuard + AdGuard Home

Configurazione Docker Compose per creare una VPN WireGuard con **wg-easy** e
**AdGuard Home** sulla stessa rete Docker privata.

L'obiettivo è:

- esporre pubblicamente solo WireGuard e la WebUI di wg-easy;
- mantenere la WebUI di AdGuard Home non pubblicata sull'host;
- utilizzare AdGuard Home come DNS per i client WireGuard;
- poter raggiungere AdGuard Home tramite il suo IP Docker dalla VPN;
- rendere la configurazione facilmente replicabile su un altro server.

## Architettura

```text
                         Internet
                            |
                 +----------+----------+
                 |                     |
            UDP :51820             TCP :51821
                 |                     |
                 v                     v
          +-------------+       +-------------+
          |   WireGuard |       |   wg-easy   |
          |    :51820   |       |   WebUI     |
          +------+------+       +-------------+
                 |
                 | VPN
                 v
        +----------------------------------+
        |     Docker: wireguard-adguard    |
        |           10.0.0.0/24            |
        |                                  |
        |  10.0.0.10       10.0.0.11       |
        |  wg-easy   <----> AdGuard Home   |
        |                    :3000          |
        |                    :53            |
        +----------------------------------+
```

## Indirizzi e porte

| Servizio | IP Docker | Porta | Accesso |
|---|---:|---:|---|
| wg-easy | `10.0.0.10` | `51820/udp` | Pubblico, VPN |
| wg-easy WebUI | `10.0.0.10` | `51821/tcp` | Pubblico |
| AdGuard Home | `10.0.0.11` | `3000/tcp` | Solo tramite rete Docker/VPN |
| AdGuard DNS | `10.0.0.11` | `53` | DNS dei client WireGuard |

> **Nota:** `expose` non pubblica una porta sull'host. Serve a dichiarare che
> la porta è disponibile per la comunicazione tra container/reti Docker
> pertinenti. La porta `3000` di AdGuard non viene quindi aggiunta a `ports`.

---

# 1. Prerequisiti

Sul server devono essere disponibili:

- Docker
- Docker Compose Plugin (`docker compose`)

Verificare:

```bash
docker --version
docker compose version
```

Creare la directory del progetto:

```bash
mkdir -p ~/adguard
cd ~/adguard
```

---

# 2. Struttura del progetto

La struttura prevista è:

```text
~/adguard/
├── docker-compose.yml
├── .gitignore
├── README.md
├── adguard/
│   ├── conf/
│   └── work/
└── wg-easy/
```

Le directory `adguard/` e `wg-easy/` contengono dati persistenti e
configurazioni generate dai container, quindi **non devono essere versionate
in Git**.

## `.gitignore`

```gitignore
adguard/
wg-easy/
```

---

# 3. Docker Compose

Creare `docker-compose.yml`:

```yaml
services:
  wg-easy:
    image: ghcr.io/wg-easy/wg-easy:15
    container_name: wg-easy
    restart: unless-stopped
    environment:
      - INSECURE=true
      - DISABLE_IPV6=true
    volumes:
      - ./wg-easy:/etc/wireguard
      - /lib/modules:/lib/modules:ro
    ports:
      - "51820:51820/udp"
      - "51821:51821/tcp" # WebUI
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1
      - net.ipv6.conf.all.disable_ipv6=1
    networks:
      wireguard-adguard:
        ipv4_address: 10.0.0.10

  adguardhome:
    image: adguard/adguardhome:latest
    container_name: adguardhome
    restart: unless-stopped
    volumes:
      - ./adguard/work:/opt/adguardhome/work
      - ./adguard/conf:/opt/adguardhome/conf
    expose:
      - "3000" # WebUI, accessibile dalla rete Docker
    networks:
      wireguard-adguard:
        ipv4_address: 10.0.0.11

networks:
  wireguard-adguard:
    name: wireguard-adguard
    driver: bridge
    ipam:
      config:
        - subnet: 10.0.0.0/24
```

Avviare i container:

```bash
docker compose up -d
```

Verificare:

```bash
docker compose ps
```

e:

```bash
docker network inspect wireguard-adguard
```

---

# 4. Accesso a wg-easy

La WebUI di wg-easy è pubblicata sulla porta TCP `51821`.

Dal browser:

```text
http://SERVER_IP:51821
```

La porta UDP `51820` viene invece utilizzata dai client WireGuard:

```text
SERVER_IP:51820/udp
```

Assicurarsi che il Security Group/firewall del server consenta:

```text
UDP 51820
TCP 51821
```

Non è necessario pubblicare la porta `3000` di AdGuard sull'host.

---

# 5. Configurazione iniziale di AdGuard Home

AdGuard Home utilizza due directory persistenti:

```text
./adguard/work
./adguard/conf
```

Queste directory vengono montate rispettivamente in:

```text
/opt/adguardhome/work
/opt/adguardhome/conf
```

La WebUI iniziale di AdGuard Home utilizza la porta `3000`.

Poiché la porta non è presente in `ports:`, non è esposta direttamente
sull'IP pubblico del server.

L'indirizzo interno del container è:

```text
10.0.0.11
```

Quindi la WebUI è:

```text
http://10.0.0.11:3000
```

L'accesso deve avvenire da una macchina che abbia raggiungibilità verso la
rete Docker `10.0.0.0/24`, ad esempio tramite il percorso configurato dalla
VPN.

---

# 6. Utilizzare AdGuard come DNS di WireGuard

Il punto fondamentale della configurazione è utilizzare:

```text
10.0.0.11
```

come server DNS nei profili WireGuard generati da wg-easy.

Nel profilo client WireGuard:

```ini
[Interface]
DNS = 10.0.0.11
```

In questo modo il client, quando è connesso alla VPN, utilizza AdGuard Home
come resolver DNS.

Il flusso diventa:

```text
Client
  |
  | WireGuard
  v
wg-easy
  |
  | Docker network: wireguard-adguard
  v
AdGuard Home
10.0.0.11:53
```

La WebUI di AdGuard è invece disponibile su:

```text
http://10.0.0.11:3000
```

---

# 7. Perché non usare `ports` per AdGuard

Non utilizzare:

```yaml
ports:
  - "3000:3000"
```

se l'obiettivo è mantenere la WebUI accessibile solo attraverso la rete
privata/VPN.

Con `ports`, Docker pubblicherà la porta sull'host.

La configurazione scelta utilizza invece:

```yaml
expose:
  - "3000"
```

Questo mantiene la porta fuori dall'esposizione pubblica dell'host.

---

# 8. Verifica della rete Docker

Controllare che entrambi i container siano sulla rete:

```bash
docker network inspect wireguard-adguard
```

Gli indirizzi previsti sono:

```text
wg-easy       -> 10.0.0.10
adguardhome   -> 10.0.0.11
```

Da un container della stessa rete è possibile verificare la raggiungibilità
di AdGuard:

```bash
docker exec -it wg-easy sh
```

e, se disponibili gli strumenti necessari:

```bash
ping 10.0.0.11
```

oppure verificare direttamente la porta:

```text
10.0.0.11:3000
```

---

# 9. Persistenza dei dati

I dati non vengono salvati solamente all'interno dei container.

Sono montati sul filesystem del server:

```text
./wg-easy
./adguard/work
./adguard/conf
```

Questo permette di ricreare i container senza perdere configurazioni e dati.

Per esempio:

```bash
docker compose down
docker compose up -d
```

non elimina i dati delle directory persistenti.

**Attenzione:** non eseguire `docker compose down -v` aspettandosi che sia
sempre equivalente, perché la gestione dei volumi e dei dati dipende da come
sono stati definiti i mount. In questa configurazione, i bind mount principali
sono comunque directory locali del progetto.

---

# 10. Backup

Per replicare questa configurazione su un altro server, salvare almeno:

```text
docker-compose.yml
.gitignore
adguard/
wg-easy/
```

Le directory `adguard/` e `wg-easy/` contengono dati/configurazioni e devono
essere trattate come dati sensibili.

Un backup può essere creato, ad esempio:

```bash
tar -czf adguard-wg-backup.tar.gz   docker-compose.yml   adguard/   wg-easy/
```

Conservare il backup in un luogo sicuro.

---

# 11. Git

Il repository deve contenere solamente i file necessari a ricreare
l'infrastruttura, non i dati generati dai container.

Esempio:

```text
.gitignore
README.md
docker-compose.yml
```

Il `.gitignore` deve contenere:

```gitignore
adguard/
wg-easy/
```

Quindi:

```bash
git status
```

dovrebbe mostrare le directory `adguard/` e `wg-easy/` come ignorate.

---

# 12. Comandi operativi

## Avviare

```bash
docker compose up -d
```

## Fermare

```bash
docker compose down
```

## Riavviare

```bash
docker compose restart
```

## Aggiornare le immagini

```bash
docker compose pull
docker compose up -d
```

## Visualizzare i container

```bash
docker compose ps
```

## Visualizzare i log

```bash
docker compose logs -f
```

Solo wg-easy:

```bash
docker compose logs -f wg-easy
```

Solo AdGuard:

```bash
docker compose logs -f adguardhome
```

---

# 13. Firewall / Security Group

Sul server devono essere consentite almeno:

| Porta | Protocollo | Utilizzo |
|---|---|---|
| `51820` | UDP | WireGuard |
| `51821` | TCP | WebUI wg-easy |

La porta:

```text
3000/tcp
```

**non deve essere aperta pubblicamente** se l'obiettivo è utilizzare la
WebUI di AdGuard esclusivamente tramite VPN.

---

# 14. Considerazioni di sicurezza

La configurazione attuale contiene:

```yaml
- INSECURE=true
```

Questa impostazione consente l'utilizzo della WebUI di wg-easy senza HTTPS.
È importante comprenderne le implicazioni prima di esporre `51821` a Internet.

Per un'installazione destinata a rimanere online a lungo termine, è
preferibile proteggere la WebUI con HTTPS e/o limitarne l'accesso tramite
firewall, reverse proxy o VPN.

Inoltre, non versionare mai in Git:

```text
wg-easy/
adguard/
```

perché possono contenere configurazioni, chiavi e altri dati sensibili.

---

# 15. Procedura di replica da zero

Quando sarà necessario ricreare l'infrastruttura su un nuovo server:

### 1. Installare Docker

Verificare:

```bash
docker --version
docker compose version
```

### 2. Creare il progetto

```bash
mkdir -p ~/adguard
cd ~/adguard
```

### 3. Ripristinare

Copiare:

```text
docker-compose.yml
adguard/
wg-easy/
```

nella directory del progetto.

### 4. Verificare il compose

```bash
docker compose config
```

### 5. Avviare

```bash
docker compose up -d
```

### 6. Verificare la rete

```bash
docker network inspect wireguard-adguard
```

Controllare che:

```text
wg-easy     = 10.0.0.10
adguardhome = 10.0.0.11
```

### 7. Configurare il DNS WireGuard

Nei profili client:

```ini
[Interface]
DNS = 10.0.0.11
```

### 8. Testare

Connettere il client alla VPN e verificare:

```text
http://10.0.0.11:3000
```

Per il DNS, verificare che le query del client vengano ricevute da
AdGuard Home.

---

# Riepilogo

La configurazione finale è basata su tre elementi principali:

```text
Internet
   |
   +-- UDP 51820 --> WireGuard
   |
   +-- TCP 51821 --> wg-easy WebUI

VPN Client
   |
   +-- DNS --> 10.0.0.11:53
   |
   +-- WebUI --> 10.0.0.11:3000
                         |
                         v
                   AdGuard Home
```

La rete Docker dedicata è:

```text
wireguard-adguard
10.0.0.0/24
```

con:

```text
wg-easy       10.0.0.10
adguardhome   10.0.0.11
```

La scelta progettuale fondamentale è **non pubblicare AdGuard Home
sull'host**: il servizio rimane sulla rete Docker e viene raggiunto dai
client tramite il percorso VPN configurato.
