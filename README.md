**Progettazione e implementazione di una rete aziendale enterprise-grade**
con sicurezza multi-livello, segmentazione VLAN e architettura DMZ a doppio firewall.

</div>

---

## 📋 Indice

- [Panoramica](#-panoramica)
- [Architettura di Rete](#-architettura-di-rete)
- [Subnetting VLSM](#-subnetting-vlsm)
- [Segmentazione VLAN](#-segmentazione-vlan)
- [Infrastruttura Server](#-infrastruttura-server)
- [Sicurezza e Firewall](#-sicurezza-e-firewall)
- [Connettività WAN](#-connettivit-wan)
- [Sicurezza delle Comunicazioni](#-sicurezza-delle-comunicazioni)
- [VPN IPsec](#-vpn-ipsec)

---

## 🔍 Panoramica

Questa rete aziendale è stata progettata seguendo i principi di **sicurezza a più livelli**, **ottimizzazione delle risorse** e **scalabilità futura**. L'intera infrastruttura è stata simulata e configurata tramite **Cisco Packet Tracer**, adottando tecnologie e standard di settore utilizzati in ambienti di produzione reali.

```
                        ┌─────────────────────────────────────┐
                        │            INTERNET                  │
                        └──────────────┬──────────────────────┘
                                       │
                               ┌───────▼────────┐
                               │   Router ISP   │
                               └───────┬────────┘
                                       │  WAN
                               ┌───────▼────────┐
                               │   Router LAN   │  ← SFP · 802.11be · NAT/PAT
                               └───────┬────────┘
                                       │
                               ┌───────▼────────┐
                               │  ASA Esterno   │  ← Prima linea di difesa
                               └───────┬────────┘
                                       │
                        ┌──────────────▼──────────────────┐
                        │          Switch DMZ              │
                        │  ┌────────┐ ┌─────┐ ┌────────┐  │
                        │  │  DNS   │ │MAIL │ │  FTP   │  │
                        │  └────────┘ └─────┘ └────────┘  │
                        └──────────────┬──────────────────┘
                                       │
                               ┌───────▼────────┐
                               │  ASA Interno   │  ← Seconda linea di difesa
                               └───────┬────────┘
                                       │
                               ┌───────▼────────┐
                               │   SW3 Core     │  ← Collapsed Core
                               └───────┬────────┘
                          ┌────────────┼────────────┐
                       ┌──▼──┐     ┌──▼──┐      ┌──▼──┐
                       │ SW1 │     │ SW2 │  ... │ SW7 │
                       └──┬──┘     └──┬──┘      └──┬──┘
                        LAN         LAN            LAN
```

---

## 🏗️ Architettura di Rete

### Topologia Collapsed Core

L'intera rete LAN è organizzata attorno a un **Multilayer Switch centrale (SW Core)** al quale si collegano tutti gli switch di distribuzione/accesso. Questo approccio garantisce:

- ✅ **Eliminazione dei loop** — nessun link trasversale tra switch, STP semplificato
- ✅ **Scalabilità** — nuovi switch si aggiungono senza modificare il core
- ✅ **Gestione STP ottimizzata** — il core è il Root Bridge unico e designato
- ✅ **Single point of management** — tutta la configurazione inter-VLAN centralizzata

### Architettura DMZ a Doppio Firewall

La zona DMZ è protetta da **due firewall Cisco ASA in cascata**, realizzando la cosiddetta *zona cuscinetto*:

| Firewall | Ruolo | Interfacce |
|---|---|---|
| **ASA Esterno** | Prima linea — filtra traffico Internet | OUTSIDE → Router, DMZ → Switch DMZ |
| **ASA Interno** | Seconda linea — protegge LAN interna | OUTSIDE → Switch DMZ, INSIDE → SW Core |

> Un attaccante che compromette il primo firewall si trova nella DMZ isolata, non nella LAN interna. Deve bucare un secondo firewall, configurato con regole indipendenti, per raggiungere le risorse aziendali.

---

## 📐 Subnetting VLSM

Il subnetting è stato progettato con tecnica **VLSM (Variable Length Subnet Masking)** per ottimizzare l'utilizzo degli indirizzi IP, allocando esattamente le dimensioni necessarie per ogni reparto con margine per espansioni future.

| Reparto | Hosts richiesti | Subnet assegnata | Hosts disponibili |
|---|:---:|---|:---:|
| 🎨 Progettazione | 241 | `/24` | 254 |
| 🌐 Sviluppo Web | 150 | `/24` | 254 |
| 💻 Sviluppo Software | 75 | `/25` | 126 |
| 🔒 Cybersecurity | 53 | `/26` | 62 |
| 🗂️ Amministrazione | 30 | `/27` | 30 |
| 👔 Dirigenza | 10 | `/28` | 14 |
| 🖧 Server Farm | — | `/29` | 6 |
| 🌍 DMZ Server | — | `/29` | 6 |

> **Principio VLSM**: ogni subnet è dimensionata sulla potenza di 2 immediatamente superiore al numero di host richiesti, evitando sprechi tipici del subnetting classico (FLSM).

---

## 🔀 Segmentazione VLAN

La rete è segmentata in **VLAN distinte per reparto**, ottenendo:

- 📉 **Riduzione del dominio di broadcast** — il traffico broadcast rimane confinato alla singola VLAN
- 🔐 **Isolamento logico** — i reparti sono separati a livello Data-Link (Layer 2)
- 🚀 **Scalabilità** — nuovi dispositivi si aggiungono alla VLAN corretta senza modifiche strutturali
- 🔧 **Manutenzione semplificata** — i guasti possono essere isolati per reparto

| VLAN ID | Nome | Reparto |
|:---:|---|---|
| 10 | `Progettazione` | 🎨 Team design e grafica |
| 20 | `Sviluppo_Web` | 🌐 Frontend e backend web |
| 30 | `Sviluppo_Software` | 💻 Sviluppo applicazioni |
| 40 | `Cybersecurity` | 🔒 Team sicurezza informatica |
| 50 | `Amministrazione` | 🗂️ Uffici amministrativi |
| 60 | `Dirigenza` | 👔 Management aziendale |
| 70 | `Server_Farm` | 🖧 Server interni aziendali |

Tutti i collegamenti tra switch sono configurati in modalità **trunk** (IEEE 802.1Q), trasportando tutte le VLAN simultaneamente. Le porte verso i dispositivi finali sono in modalità **access**, assegnate alla VLAN del rispettivo reparto.

Il **routing inter-VLAN** è gestito centralmente dallo Switch Core tramite **interfacce VLAN (SVI)**, eliminando la necessità di un router dedicato per il traffico interno.

---

## 🖥️ Infrastruttura Server

Tre server dedicati forniscono i servizi fondamentali della rete:

### 🔵 Server 1 — DHCP & NTP
- **DHCP**: assegna automaticamente indirizzi IP a tutti i dispositivi della LAN, con pool dedicato per ogni VLAN. Le richieste broadcast vengono inoltrate tramite `ip helper-address` configurato su ogni interfaccia VLAN degli switch.
- **NTP**: sincronizzazione dell'orologio di rete per tutti i dispositivi, fondamentale per la correlazione dei log di sicurezza e il corretto funzionamento dei certificati TLS.

### 🟢 Server 2 — DNS *(in DMZ)*
- Risoluzione dei nomi per tutti i client interni e per i servizi esposti verso internet.
- Posizionato nella **DMZ** per essere raggiungibile sia dall'interno che dall'esterno, con accesso filtrato dal firewall su porta `53 UDP/TCP`.

### 🟡 Server 3 — MAIL & FTP *(in DMZ)*
- **MAIL**: gestione della posta elettronica aziendale (SMTP porta 25, IMAP/POP3).
- **FTP**: trasferimento file sicuro con accesso controllato dalle ACL del firewall.
- Collocato in DMZ per consentire l'accesso esterno senza esporre la LAN interna.

---

## 🔒 Sicurezza e Firewall

### Firewall Stateful ASA

Entrambi i firewall Cisco ASA operano in modalità **stateful**, ispezionando il traffico fino al **livello Transport (Layer 4)**:

- Mantengono una **tabella delle connessioni attive** — il traffico di ritorno è permesso automaticamente senza regole esplicite
- Applicano **regole ACL** (Access Control List) sia standard che extended
- Ogni interfaccia ha un **security-level** che determina il flusso di traffico permesso di default

### Regole ACL

Le ACL seguono il principio **Default Deny**: tutto il traffico non esplicitamente permesso viene bloccato dall'*implicit deny* finale.

```
! Esempio — traffico LAN verso DMZ
access-list INSIDE_TO_DMZ permit tcp any host [IP_DNS] eq 53
access-list INSIDE_TO_DMZ permit udp any host [IP_DNS] eq 53
access-list INSIDE_TO_DMZ permit tcp any host [IP_MAIL] eq 25
! implicit deny ip any any  ← tutto il resto viene bloccato
```

| Flusso | Politica |
|---|---|
| Internet → DMZ | ✅ Solo porte specifiche (53, 25, 80, 443) |
| LAN → DMZ | ✅ Controllato per host e porta |
| DMZ → LAN | ❌ Bloccato completamente |
| LAN → Internet | ✅ Tramite NAT/PAT |

---

## 🌍 Connettività WAN

Il router aziendale di ultima generazione integra:

| Caratteristica | Dettaglio |
|---|---|
| **Interfaccia SFP** | Collegamento WAN in fibra ottica — multimodale per brevi distanze, monomodale per lunghe distanze |
| **Porte LAN** | 1 Gbps e 10 Gbps per connessioni ad alta velocità verso switch e dispositivi |
| **Porta Console** | Cavo RS-232 per configurazione/manutenzione diretta |
| **Access Point integrato** | Connettività wireless con standard **Wi-Fi 802.11be** (Wi-Fi 7) per le massime performance |
| **Porta WAN** | Connessione internet alternativa in caso di mancato utilizzo della fibra SFP |

### NAT/PAT

La navigazione internet è resa possibile tramite **NAT** e più precisamente **PAT (Port Address Translation)**:

- Un **unico IP pubblico** assegnato dall'ISP (statico o dinamico in base al contratto) rappresenta l'intera rete aziendale verso internet
- Il PAT differenzia le connessioni di migliaia di dispositivi interni tramite le **porte sorgente**, permettendo la condivisione del singolo IP pubblico
- Il router mantiene una **tabella di traduzione** che associa ogni connessione interna alla porta PAT corrispondente

---

## 🔐 Sicurezza delle Comunicazioni

Tutte le comunicazioni in rete sono protette tramite **crittografia ibrida**:

```
┌─────────────────────────────────────────────────────┐
│                    TLS 1.3                          │
│                                                     │
│  ┌─────────────────┐    ┌──────────────────────┐   │
│  │   RSA            │    │   AES                │   │
│  │                  │    │                      │   │
│  │  Scambio chiavi  │───▶│  Cifratura dati      │   │
│  │  asimmetrico     │    │  simmetrica          │   │
│  │                  │    │                      │   │
│  │  Protegge da     │    │  Veloce ed           │   │
│  │  Man-in-the-     │    │  efficiente per      │   │
│  │  Middle          │    │  grandi volumi       │   │
│  └─────────────────┘    └──────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

- **RSA** — algoritmo asimmetrico per lo scambio sicuro delle chiavi di sessione, prevenendo attacchi **Man-in-the-Middle**
- **AES** — algoritmo simmetrico per la cifratura rapida ed efficiente di tutti i dati trasmessi
- **TLS 1.3** — protocollo che integra entrambe le funzionalità, eliminando algoritmi obsoleti e vulnerabili presenti nelle versioni precedenti

---

## 🔑 VPN IPsec

Per i dipendenti in **lavoro remoto** è disponibile un servizio VPN basato su protocollo **IPsec**:

- 🕵️ **Anonimato** — l'indirizzo IP reale del dipendente viene oscurato
- 🔐 **Cifratura end-to-end** — tutto il traffico tra il dispositivo remoto e la rete aziendale è cifrato
- 🏢 **Accesso sicuro** — il dipendente remoto opera come se fosse fisicamente in azienda
- 🛡️ **Tunnel protetto** — i dati transitano su internet pubblico in modo completamente sicuro

---
