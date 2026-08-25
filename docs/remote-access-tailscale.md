# Accesso remoto Phoenix con Tailscale

Questa pagina documenta la configurazione Tailscale validata sulla Sovol SV08 **Phoenix** il 25 agosto 2026.

L'obiettivo è ottenere accesso remoto a **SSH** e **Mainsail/Moonraker** senza aprire porte sul router e senza trasformare il CB1 in subnet router o exit node.

## Baseline validata

Sistema Phoenix al momento del test:

- BTT CB1;
- Debian 11 Bullseye, arm64;
- kernel `5.16.17-sun50iw9`;
- NetworkManager attivo;
- collegamento IP tramite `wlan0`;
- `systemd-resolved` disabilitato;
- `/etc/resolv.conf` gestito direttamente da NetworkManager;
- supporto TUN presente (`CONFIG_TUN=m`, `/dev/net/tun` disponibile).

Durante la validazione Phoenix era collegata a Internet tramite hotspot Android. Questo non ha impedito il funzionamento di Tailscale.

## Architettura scelta

Phoenix è configurata come normale nodo Tailscale.

Non vengono usati:

- subnet routing;
- exit node;
- route pubblicizzate;
- gestione DNS Tailscale sul CB1.

La scelta è intenzionalmente conservativa: la rete locale già funzionante deve restare indipendente da Tailscale.

## Installazione

È stato usato il repository APT ufficiale Tailscale per Debian Bullseye, quindi installato il pacchetto `tailscale` per arm64.

Versione validata:

```text
1.102.3
```

Il servizio `tailscaled` viene installato e abilitato tramite systemd.

## Avvio e DNS

Poiché Phoenix usa NetworkManager con un `/etc/resolv.conf` tradizionale e `systemd-resolved` è inattivo, Tailscale viene avviato senza assumere la gestione DNS:

```bash
sudo tailscale up --accept-dns=false
```

Dopo l'autenticazione, le preferenze validate mostravano:

```text
RouteAll: false
CorpDNS: false
WantRunning: true
AdvertiseRoutes: null
```

La default route locale e i resolver configurati da NetworkManager sono rimasti invariati.

## Moonraker e rete Tailscale

Tailscale assegna gli indirizzi IPv4 del tailnet nello spazio `100.64.0.0/10`.

Moonraker, nella configurazione Phoenix iniziale, considerava trusted solo loopback e reti RFC1918. Di conseguenza Mainsail veniva caricato dal web server ma il WebSocket verso Moonraker falliva con `HTTP 401 Unauthorized` quando il client arrivava tramite Tailscale.

La correzione validata è aggiungere la rete Tailscale a `[authorization]` in `moonraker.conf`:

```ini
[authorization]
trusted_clients:
    127.0.0.0/8
    10.0.0.0/8
    172.16.0.0/12
    192.168.0.0/16
    100.64.0.0/10
    ::1/128
    fe80::/10
```

Dopo il riavvio del solo servizio Moonraker, il blocco `Trusted Clients` caricato riportava correttamente `100.64.0.0/10`.

## Test validati

Sono stati completati con successo:

- autenticazione del nodo Phoenix nel tailnet;
- raggiungibilità IPv4 Tailscale dal PC Kubuntu;
- SSH verso Phoenix usando l'indirizzo Tailscale;
- caricamento di Mainsail tramite indirizzo Tailscale;
- WebSocket Mainsail → Moonraker dopo l'aggiunta di `100.64.0.0/10` ai trusted clients;
- persistenza della default route locale;
- persistenza dei DNS locali con `--accept-dns=false`.

Nel test tramite hotspot mobile, dopo il primo pacchetto di assestamento il ping dal PC a Phoenix è sceso nell'ordine di circa 10–20 ms.

## Test ancora da completare

Il test del 25 agosto è stato eseguito con Phoenix e il PC collegati alla stessa infrastruttura Internet tramite hotspot mobile, pur usando gli indirizzi Tailscale per SSH e Mainsail.

Resta quindi da validare esplicitamente il caso realmente remoto con i due dispositivi su reti Internet differenti, ad esempio:

- Phoenix sulla rete di casa;
- PC o telefono su rete mobile o altra rete esterna.

Non sono previste modifiche di configurazione per questo test: Tailscale dovrebbe mantenere lo stesso nodo e selezionare automaticamente il percorso disponibile.

## Note di sicurezza

- Non esporre Moonraker o SSH con port forwarding pubblico solo per ottenere accesso remoto: Tailscale elimina questa necessità nel modello adottato da Phoenix.
- `100.64.0.0/10` viene aggiunto ai `trusted_clients` perché l'accesso è limitato ai peer ammessi dal tailnet. Le policy ACL/grants del tailnet restano comunque il livello corretto per restringere ulteriormente chi può raggiungere Phoenix.
- Non attivare subnet routing o exit-node se non esiste un requisito concreto.
- Prima di modificare `moonraker.conf`, creare sempre un backup verificato.
