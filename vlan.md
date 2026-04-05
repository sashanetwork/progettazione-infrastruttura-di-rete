
## 📋 Indice

- [Panoramica](#-panoramica)
- [Architettura DMZ e Doppio Firewall](#-architettura-dmz-e-doppio-firewall)
- [Tabella VLAN](#-tabella-vlan)
- [Configurazione CLI](#-configurazione-cli)
  - [Inserimento VLAN nel Database](#1-inserimento-vlan-nel-database-degli-switch-l3)
  - [Configurazione SVI](#2-configurazione-delle-svi-inter-vlan-routing-e-dhcp-relay)
  - [Configurazione Porte](#3-configurazione-delle-porte)
- [Schema Riassuntivo Collegamenti](#-schema-riassuntivo-collegamenti)

---

## 🔍 Panoramica

Per la segmentazione logica della rete sono state create **7 VLAN**, una per ogni reparto aziendale, escludendo la DMZ in quanto già isolata fisicamente e logicamente dalla LAN interna grazie all'architettura a doppio firewall — il cosiddetto sistema **"zona cuscinetto"**.

La segmentazione VLAN garantisce:

- 📉 **Riduzione del dominio di broadcast** — il traffico broadcast rimane confinato alla singola VLAN, evitando congestioni sull'intera rete
- 🔐 **Isolamento logico** — i reparti sono separati a livello Data-Link (Layer 2), un dispositivo di un reparto non può comunicare direttamente con un altro senza passare dal router/firewall
- 🚀 **Scalabilità** — nuovi dispositivi si aggiungono alla VLAN corretta senza modifiche strutturali alla rete
- 🔧 **Manutenzione semplificata** — i guasti possono essere isolati per reparto, facilitando l'individuazione e la risoluzione dei problemi

---

## 🛡️ Architettura DMZ e Doppio Firewall

La DMZ non utilizza VLAN in quanto è già isolata fisicamente e logicamente tramite due firewall Cisco ASA in cascata:

```
        INTERNET
            │
     ┌──────▼───────┐
     │  Router LAN  │   NAT/PAT · SFP · 802.11be
     └──────┬───────┘
            │  access
     ┌──────▼───────┐
     │ ASA Esterno  │   ← Prima linea di difesa
     └──────┬───────┘
            │  access
     ┌──────▼───────┐
     │  Switch DMZ  │   Nessuna VLAN — subnet dedicata
     │  ┌─────────┐ │
     │  │   DNS   │ │
     │  │ MAIL/FTP│ │
     │  └─────────┘ │
     └──────┬───────┘
            │  access
     ┌──────▼───────┐
     │ ASA Interno  │   ← Seconda linea di difesa
     └──────┬───────┘
            │  trunk (VLAN 10-70)
     ┌──────▼───────┐
     │   SW Core    │   Inter-VLAN Routing · SVI
     └──────┬───────┘
            │  trunk (VLAN 10-70)
       ┌────┴────┐
      SW1  ...  SW7
       │          │
    access      access
    (per VLAN)  (per VLAN)
```

## 📊 Tabella VLAN

| VLAN ID | Nome | Reparto | Host richiesti | Subnet | Gateway |
|:---:|---|---|:---:|---|---|
| 10 | `Progettazione` | 🎨 Team design e grafica | 241 | `172.16.0.0/24` | `172.16.0.1` |
| 20 | `Sviluppo_Web` | 🌐 Frontend e backend web | 150 | `172.16.1.0/24` | `172.16.1.1` |
| 30 | `Sviluppo_Software` | 💻 Sviluppo applicazioni | 75 | `172.16.2.0/25` | `172.16.2.1` |
| 40 | `Cybersecurity` | 🔒 Team sicurezza informatica | 53 | `172.16.2.128/26` | `172.16.2.129` |
| 50 | `Amministrazione` | 🗂️ Uffici amministrativi | 30 | `172.16.2.192/26` | `172.16.2.193` |
| 60 | `Dirigenza` | 👔 Management aziendale | 10 | `172.16.3.0/28` | `172.16.3.1` |
| 70 | `Server_Farm` | 🖧 Server interni aziendali | — | `172.16.3.16/29` | `172.16.3.17` |

---

## ⚙️ Configurazione CLI

### 1. Inserimento VLAN nel Database degli Switch L3

Le VLAN vengono create nel database di ogni switch L3 della LAN. Il comando `vlan database` permette di inserire più VLAN in un unico blocco:

```
vlan database
 vlan 10 name Progettazione
 vlan 20 name Sviluppo_Web
 vlan 30 name Sviluppo_Software
 vlan 40 name Cybersecurity
 vlan 50 name Amministrazione
 vlan 60 name Dirigenza
 vlan 70 name Server_Farm
exit
```

> Questo blocco va eseguito su **tutti gli switch L3 della LAN**. Lo Switch DMZ non necessita di VLAN.

---

### 2. Configurazione delle SVI, Inter-VLAN Routing e DHCP Relay

Il comando `ip routing` viene abilitato **globalmente** una sola volta su ogni switch L3, permettendo il routing inter-VLAN tramite le SVI (Switch Virtual Interface) senza necessità di un router dedicato.

Per ogni VLAN viene creata una **SVI** con:
- `ip address` — indirizzo IP che funge da **gateway** per i dispositivi di quel reparto e da **giaddr** nelle richieste DHCP
- `ip helper-address` — necessario perché il protocollo DHCP usa messaggi **broadcast** che non attraversano i router. L'helper-address converte automaticamente il broadcast in **unicast** verso il server DHCP (`172.16.3.18`), il quale identifica il pool corretto leggendo l'IP della SVI mittente nel campo **giaddr** del pacchetto
- `no shutdown` — attiva l'interfaccia VLAN (di default è in stato shutdown)

```
ip routing

interface vlan 10
 ip address 172.16.0.1 255.255.255.0
 ip helper-address 172.16.3.18
 no shutdown

interface vlan 20
 ip address 172.16.1.1 255.255.255.0
 ip helper-address 172.16.3.18
 no shutdown

interface vlan 30
 ip address 172.16.2.1 255.255.255.128
 ip helper-address 172.16.3.18
 no shutdown

interface vlan 40
 ip address 172.16.2.129 255.255.255.192
 ip helper-address 172.16.3.18
 no shutdown

interface vlan 50
 ip address 172.16.2.193 255.255.255.192
 ip helper-address 172.16.3.18
 no shutdown

interface vlan 60
 ip address 172.16.3.1 255.255.255.240
 ip helper-address 172.16.3.18
 no shutdown

interface vlan 70
 ip address 172.16.3.17 255.255.255.248
 ip helper-address 172.16.3.18
 no shutdown
```

> **Come funziona il giaddr**: quando un PC della VLAN 10 richiede un IP, lo switch riceve il broadcast sulla SVI della VLAN 10 (IP `172.16.0.1`) e lo inoltra al server DHCP inserendo `172.16.0.1` nel campo giaddr. Il server legge questo campo e assegna un IP dal pool `172.16.0.0/24` — quello corretto per la VLAN Progettazione.

---

### 3. Configurazione delle Porte

Le porte degli switch sono configurate in due modalità in base al dispositivo collegato:

#### Porte Trunk — tra switch e verso il firewall

Le porte trunk trasportano **tutte le 7 VLAN simultaneamente** tramite il protocollo **IEEE 802.1Q**, che aggiunge un tag di 4 byte ad ogni frame per identificarne la VLAN di appartenenza:

```
interface gigabitEthernet x/x
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,50,60,70
 no shutdown
```

#### Porte Access — verso i dispositivi finali

Le porte access trasportano il traffico di **una sola VLAN**, quella del reparto a cui appartiene il dispositivo collegato. Il dispositivo non è consapevole della VLAN — riceve frame normali senza tag:

```
interface gigabitEthernet x/x
 switchport mode access
 switchport access vlan 10   ← sostituire con la VLAN corretta
 no shutdown
```

#### Porte nella zona DMZ — access senza VLAN

I collegamenti nella zona DMZ non utilizzano VLAN — sono connessioni punto-punto semplici tra firewall, switch DMZ e server:

```
interface gigabitEthernet x/x
 switchport mode access
 no shutdown
```

---
