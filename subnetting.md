# 🏢 Piano di Indirizzamento della Rete Aziendale

> Progettazione di una grande rete aziendale — VLSM su blocco `172.16.0.0/16`

---

## 📋 Introduzione

Come spiegato nel README principale, è stata adottata una **subnet mask a lunghezza variabile (VLSM)** per ottimizzare l'utilizzo dello spazio di indirizzamento.

L'azienda è una big nel mercato IT, con reparti per ogni branca tecnica e amministrativa, più una server farm. Il blocco di partenza scelto è `172.16.0.0/16` — indirizzi privati di **Classe B** — in quanto una Classe C non sarebbe stata sufficiente per il numero di host presenti nell'infrastruttura.

---

## 📊 Analisi dei Reparti

I reparti sono stati ordinati in modo **decrescente per numero di host**, tecnica fondamentale nel calcolo VLSM per evitare sprechi di host e sovrapposizioni.

| Reparto | Host Richiesti |
|---|---|
| 🖥️ Progettazione | 241 |
| 🌐 Sviluppo Web | 150 |
| 💻 Sviluppo Software | 75 |
| 🔒 Cybersecurity | 53 |
| 🗂️ Amministrazione | 30 |
| 👔 Dirigenza | 10 |
| 🌍 DMZ Server | 2 |
| 🖧 Server Farm | 1 |

---

## 🗺️ Piano di Indirizzamento VLSM

| Reparto | Indirizzo di Rete | Subnet Mask | CIDR | Gateway | Broadcast | Range Host |
|---|---|---|---|---|---|---|
| Progettazione | `172.16.0.0` | `255.255.255.0` | `/24` | `172.16.0.1` | `172.16.0.255` | `172.16.0.2` — `172.16.0.254` |
| Sviluppo Web | `172.16.1.0` | `255.255.255.0` | `/24` | `172.16.1.1` | `172.16.1.255` | `172.16.1.2` — `172.16.1.254` |
| Sviluppo Software | `172.16.2.0` | `255.255.255.128` | `/25` | `172.16.2.1` | `172.16.2.127` | `172.16.2.2` — `172.16.2.126` |
| Cybersecurity | `172.16.2.128` | `255.255.255.192` | `/26` | `172.16.2.129` | `172.16.2.191` | `172.16.2.130` — `172.16.2.190` |
| Amministrazione | `172.16.2.192` | `255.255.255.192` | `/26` | `172.16.2.193` | `172.16.2.255` | `172.16.2.194` — `172.16.2.254` |
| Dirigenza | `172.16.3.0` | `255.255.255.240` | `/28` | `172.16.3.1` | `172.16.3.15` | `172.16.3.2` — `172.16.3.14` |
| Server Farm | `172.16.3.16` | `255.255.255.248` | `/29` | `172.16.3.17` | `172.16.3.23` | `172.16.3.18` — `172.16.3.22` |
| DMZ Server | `172.16.3.24` | `255.255.255.248` | `/29` | `172.16.3.25` | `172.16.3.31` | `172.16.3.26` — `172.16.3.30` |

---

## 🔗 Link Punto-Punto

| Collegamento | Indirizzo di Rete | Subnet Mask | CIDR | IP Dispositivo A | IP Dispositivo B | Broadcast |
|---|---|---|---|---|---|---|
| Router ↔ Firewall (ASA) | `172.16.3.32` | `255.255.255.252` | `/30` | `172.16.3.33` | `172.16.3.34` | `172.16.3.35` |
| Firewall (ASA) ↔ Switch DMZ | `172.16.3.36` | `255.255.255.252` | `/30` | `172.16.3.37` | `172.16.3.38` | `172.16.3.39` |
| Switch2 L3 ↔ Firewall (ASA) | `172.16.3.40` | `255.255.255.252` | `/30` | `172.16.3.41` | `172.16.3.42` | `172.16.3.43` |

---

## 📝 Precisazioni Progettuali

### Reparto Amministrazione — `/26` invece di `/27`
Un `/27` avrebbe fornito esattamente **30 host disponibili**, pari al numero richiesto, senza alcun margine per future espansioni. È stato scelto un `/26` (62 host disponibili) per garantire scalabilità futura mantenendo il principio generale adottato in tutto il piano: **lasciare sempre spazio per la crescita**.

### Server Farm — `/29` per un singolo server
La server farm ospita attualmente **un solo server** (DHCP + NTP). È stato scelto comunque un `/29` (6 host utilizzabili) per permettere l'aggiunta futura di ulteriori server interni che non offrono servizi pubblici e che quindi non necessitano di interfacciarsi direttamente con la rete esterna.

### DMZ — Separazione dei servizi pubblici
I server **DNS** e **Mail & FTP** sono stati collocati in una DMZ dedicata in quanto offrono servizi accessibili dall'esterno. La DMZ è gestita direttamente dall'ASA (Cisco ASA 5506-X) su un'interfaccia dedicata, garantendo isolamento dalla rete interna. Il server DHCP/NTP rimane nella rete interna poiché i suoi servizi sono esclusivamente ad uso locale.

