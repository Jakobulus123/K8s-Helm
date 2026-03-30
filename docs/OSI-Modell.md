# Das OSI-Modell — und wo Kubernetes, Traefik & ArgoCD drin stecken

> Das OSI-Modell erklärt WARUM Netzwerke so funktionieren wie sie es tun.
> Wenn du verstehst auf welcher Schicht ein Problem liegt, weißt du sofort
> welches Tool du brauchst um es zu debuggen.

---

## Das Modell auf einen Blick

```
┌─────┬──────────────────┬───────────────────────────────────────────────────────┐
│ Nr. │ Schicht          │ Frage die sie beantwortet                             │
├─────┼──────────────────┼───────────────────────────────────────────────────────┤
│  7  │ Application      │ Was bedeuten die Daten?                               │
│  6  │ Presentation     │ In welchem Format sind die Daten?                     │
│  5  │ Session          │ Wie wird eine Verbindung aufgebaut und gehalten?       │
│  4  │ Transport        │ Wie kommen Datenpakete zuverlässig ans Ziel?          │
│  3  │ Network          │ Wie findet ein Paket seinen Weg durch Netzwerke?      │
│  2  │ Data Link        │ Wie kommunizieren Geräte im selben Netzwerksegment?   │
│  1  │ Physical         │ Wie werden Bits physisch übertragen?                  │
└─────┴──────────────────┴───────────────────────────────────────────────────────┘
```

**Merkhilfe (von oben nach unten):**
> **A**lle **P**riester **S**aufen **T**equila **N**ach **D**er **P**redigt
> Application → Presentation → Session → Transport → Network → Data Link → Physical

---

## Schicht 1 — Physical (Bitübertragung)

### Was passiert hier?

Rohe Bits (0 und 1) werden physisch übertragen. Nichts anderes.

```
Bit: 1  →  Spannung: +5V
Bit: 0  →  Spannung:  0V

Oder bei Glasfaser:
Bit: 1  →  Lichtpuls
Bit: 0  →  kein Licht
```

### Komponenten

| Komponente | Beschreibung |
|------------|-------------|
| Netzwerkkabel (Cat5e, Cat6) | Kupferkabel, überträgt elektrische Signale |
| Glasfaserkabel | Lichtpulse, höhere Bandbreite, längere Strecken |
| WLAN (Radio Waves) | Elektromagnetische Wellen statt Kabel |
| Netzwerkkarte (NIC) | Wandelt digitale Bits in physische Signale um |
| Hub | Verteilt Signale stur an alle Ports (veraltet) |
| Repeater | Verstärkt Signal damit es weiter kommt |

### Kubernetes-Relevanz

```
Server (Node) ─── Kabel ─── Switch ─── Router ─── Internet
     │
     └── NIC überträgt Ethernet-Frames als elektrische Signale
```

Wenn ein Kubernetes-Pod auf Node 1 mit einem Pod auf Node 2 kommuniziert,
landen die Pakete irgendwann hier — als Bits auf dem Kabel zwischen den Servern.

**Debugging:** Kein Ping möglich, obwohl Kabel drin? → Layer 1 Problem.
```bash
ethtool eth0         # NIC-Status prüfen
ip link show eth0    # Interface up/down?
```

---

## Schicht 2 — Data Link (Sicherungsschicht)

### Was passiert hier?

Bits werden zu **Frames** gruppiert. Geräte im **selben Netzwerksegment**
(gleicher Switch) finden sich gegenseitig über **MAC-Adressen**.

```
Frame-Struktur:
┌──────────────┬──────────────┬──────┬─────────────┬─────┐
│ Ziel-MAC     │ Quell-MAC    │ Typ  │   Nutzdaten  │ FCS │
│ (6 Bytes)    │ (6 Bytes)    │(2B)  │  (46-1500B)  │(4B) │
└──────────────┴──────────────┴──────┴─────────────┴─────┘
```

### MAC-Adressen

**MAC** (Media Access Control) = Hardware-Adresse der Netzwerkkarte.
Fest eingebrannt, weltweit eindeutig:
```
00:1A:2B:3C:4D:5E
│  │  │  │  │  │
└──┴──┴──┘  └──┴──┴──┘
 Hersteller    Geräte-ID
```

### ARP — Wer hat welche IP?

Das Problem: Ich kenne die IP (Layer 3) des Ziels, brauche aber die MAC-Adresse
(Layer 2) um das Frame zu bauen.

```
Ich: "Wer hat IP 192.168.1.1? Bitte melde dich!"  (ARP Request — Broadcast an alle)
      ↓ (alle Geräte im Segment hören zu)
Router: "Ich! Meine MAC ist 00:1A:2B:3C:4D:5E"   (ARP Reply — direkt zurück)
      ↓
Ich speichere: 192.168.1.1 → 00:1A:2B:3C:4D:5E  (ARP-Cache)
      ↓
Jetzt kann ich Frames an den Router schicken
```

### Komponenten

| Komponente | Beschreibung |
|------------|-------------|
| Switch | Lernt welche MAC hinter welchem Port ist, leitet gezielt weiter |
| Bridge | Verbindet zwei Netzwerksegmente auf Layer 2 |
| VLAN | Logische Trennung auf Layer 2 (gleicher Switch, getrennte Netze) |
| Ethernet | Das bekannteste Layer-2-Protokoll |
| WiFi (802.11) | Wireless Layer-2-Protokoll |

### Kubernetes-Relevanz

```
Node 1 (MAC: AA:BB:CC:DD:EE:01)
    │
    └── Will zu Node 2 (MAC: AA:BB:CC:DD:EE:02)
            ↓
        ARP: "Wer hat 192.168.1.11?"
            ↓
        Switch lernt: Port 3 = Node 2
            ↓
        Frame geht direkt zu Node 2 (nicht Broadcast)
```

**VXLAN (Flannel/Calico)** arbeitet auf Layer 2/3:
```
Pod-zu-Pod über Nodes = Layer-2-Frame wird in UDP-Paket verpackt (VXLAN)
→ Das nennt man "Overlay Network" — Layer 2 über Layer 3 tunneln
```

**Debugging:**
```bash
ip neigh show          # ARP-Tabelle (welche IP hat welche MAC?)
arp -n                 # Alternativ
bridge fdb show        # Forwarding-Tabelle des Linux-Bridges
```

---

## Schicht 3 — Network (Vermittlungsschicht)

### Was passiert hier?

**IP-Pakete** werden durch Netzwerke geroutet. Geräte in **verschiedenen Netzwerken**
finden sich über **IP-Adressen**.

```
IP-Paket-Struktur:
┌──────────┬──────────┬─────┬────────────┬────────────┬─────────────┐
│ Version  │  TTL     │ Pro-│ Quell-IP   │  Ziel-IP   │  Nutzdaten  │
│ IHL, TOS │ (Hops)   │tokol│ (4 Bytes)  │  (4 Bytes) │             │
└──────────┴──────────┴─────┴────────────┴────────────┴─────────────┘
```

### IP-Adressen und Subnetting

```
IP-Adresse:  192.168.1.100
Subnetzmaske: 255.255.255.0  (= /24)

CIDR-Notation: 192.168.1.0/24
               └────────┘└─┘
               Netzwerk  Prefix-Länge (24 Bits = Netzwerkteil)

/24 bedeutet: 256 Adressen, davon 254 nutzbar
  .0   = Netzwerkadresse
  .255 = Broadcast
  .1 - .254 = Hosts
```

### Routing

Router entscheiden anhand der **Routing-Tabelle** wohin ein Paket geht:

```
Routing-Tabelle auf Node 1:
  10.244.2.0/24 → via 192.168.1.11  (Node 2 — dort liegen diese Pod-IPs)
  10.244.3.0/24 → via 192.168.1.12  (Node 3)
  192.168.1.0/24 → direkt (gleicher Switch)
  0.0.0.0/0    → 192.168.1.1        (Default Gateway = Router ins Internet)

Paket für 10.244.2.8:
  → Passt zu 10.244.2.0/24
  → Sende an 192.168.1.11 (Node 2)
  → Flannel auf Node 2 entpackt und liefert an Pod
```

### NAT — Network Address Translation

```
Privates Netzwerk (intern):  192.168.1.0/24
Öffentliche IP (extern):     89.167.116.105

Paket von Pod (10.244.1.5) ins Internet:
  Original:  Src: 10.244.1.5  → Dst: 8.8.8.8
  Nach NAT:  Src: 89.167.116.105 → Dst: 8.8.8.8
  (Router merkt sich die Übersetzung)

Antwort kommt zurück:
  Original:  Src: 8.8.8.8  → Dst: 89.167.116.105
  Nach NAT:  Src: 8.8.8.8  → Dst: 10.244.1.5
```

**DNAT** (Destination NAT) = kube-proxy nutzt das für Services:
```
Request an Service-IP 10.96.42.100:80
DNAT-Regel: Ändere Ziel zu 10.244.2.5:8080 (echter Pod)
```

### Komponenten in Kubernetes auf Layer 3

| Was | Layer-3-Konzept | Beschreibung |
|-----|-----------------|-------------|
| **Pod-IP** | IP-Adresse | Jeder Pod bekommt eine eigene IP aus dem Pod-CIDR |
| **Node-IP** | IP-Adresse | Physische IP des Servers |
| **ClusterIP** | Virtuelle IP | Service-IP — existiert nur in iptables |
| **Pod-CIDR** | Subnetz | z.B. `10.244.0.0/16` — Bereich für alle Pod-IPs |
| **Service-CIDR** | Subnetz | z.B. `10.96.0.0/12` — Bereich für ClusterIPs |
| **CNI Plugin** | Routing | Flannel/Calico erstellt Routen zwischen Nodes |
| **kube-proxy** | NAT/Routing | iptables-DNAT für Service-zu-Pod-Weiterleitung |
| **VXLAN** | Tunnel | Flannel verpackt Pod-Pakete für Node-zu-Node |

### Die 3 IP-Bereiche im Cluster

```
┌─────────────────────────────────────────────────────────┐
│  Node-Netzwerk:  192.168.1.0/24                         │
│  Echte physische IPs der Server                         │
│  ─────────────────────────────────────────────────────  │
│  Node 1: 192.168.1.10                                   │
│  Node 2: 192.168.1.11                                   │
│  Node 3: 192.168.1.12                                   │
├─────────────────────────────────────────────────────────┤
│  Pod-Netzwerk:  10.244.0.0/16                           │
│  Virtuelle IPs — vom CNI-Plugin verwaltet               │
│  ─────────────────────────────────────────────────────  │
│  Node 1 Pods: 10.244.1.0/24                             │
│  Node 2 Pods: 10.244.2.0/24                             │
│  Node 3 Pods: 10.244.3.0/24                             │
├─────────────────────────────────────────────────────────┤
│  Service-Netzwerk:  10.96.0.0/12                        │
│  Virtuelle IPs — nur in iptables, kein echtes Interface  │
│  ─────────────────────────────────────────────────────  │
│  kubernetes.default:  10.96.0.1                         │
│  coredns:             10.96.0.10                        │
│  traefik:             10.96.x.x                         │
└─────────────────────────────────────────────────────────┘
```

**Debugging:**
```bash
ip route show                    # Routing-Tabelle der Node
ip addr show                     # IP-Adressen der Interfaces
iptables -t nat -L -n --line-numbers  # NAT-Regeln (kube-proxy)
kubectl get nodes -o wide        # Node-IPs
kubectl get pods -o wide         # Pod-IPs und auf welcher Node
```

---

## Schicht 4 — Transport (Transportschicht)

### Was passiert hier?

**Daten werden zuverlässig** (TCP) oder **schnell** (UDP) zwischen Prozessen übertragen.
Ports identifizieren den Prozess auf einem Host.

```
IP-Adresse = welcher Host
Port       = welcher Prozess auf dem Host

10.244.1.5:8080  →  Pod, der auf Port 8080 lauscht
10.96.0.1:443    →  kube-apiserver auf Port 443
```

### TCP vs. UDP

| Merkmal | TCP | UDP |
|---------|-----|-----|
| Verbindung | Verbindungsorientiert (Handshake) | Verbindungslos |
| Zuverlässigkeit | Ja — jedes Paket wird bestätigt | Nein — Fire and forget |
| Reihenfolge | Garantiert | Nicht garantiert |
| Fehlerkorrektur | Ja — Retransmission | Nein |
| Geschwindigkeit | Langsamer (Overhead) | Schneller |
| Verwendung | HTTP, HTTPS, SSH, kubectl | DNS, VXLAN, Video-Streaming |

### TCP Three-Way Handshake

Bevor irgendein HTTP-Byte fließt, muss TCP eine Verbindung aufbauen:

```
Client                              Server
  │                                   │
  │──── SYN (Seq=100) ───────────────►│  "Ich will reden, meine Seq ist 100"
  │                                   │
  │◄─── SYN-ACK (Seq=200, Ack=101) ──│  "OK, meine Seq ist 200, ich erwarte 101"
  │                                   │
  │──── ACK (Ack=201) ───────────────►│  "Verstanden, ich erwarte 201"
  │                                   │
  │        Verbindung steht           │
  │──── HTTP GET / ─────────────────►│
```

### Ports in Kubernetes

```
Bekannte Ports (well-known):
  80   = HTTP
  443  = HTTPS
  22   = SSH
  5432 = PostgreSQL
  6379 = Redis

Kubernetes-spezifische Ports:
  6443  = kube-apiserver (HTTPS API)
  2379  = etcd client
  2380  = etcd peer
  10250 = kubelet API
  10256 = kube-proxy health check
  30000-32767 = NodePort-Bereich

Traefik:
  80    = web EntryPoint
  443   = websecure EntryPoint
  8080  = Traefik Dashboard (intern)
  9000  = Traefik Metrics
```

### Layer 4 in Kubernetes — kube-proxy im Detail

kube-proxy arbeitet auf Layer 3/4 — es manipuliert IP-Pakete anhand von IP+Port:

```
iptables-Regel (vereinfacht) für Service "meine-app":

-A KUBE-SERVICES -d 10.96.42.100/32 -p tcp --dport 80
   -j KUBE-SVC-MEINE-APP

-A KUBE-SVC-MEINE-APP -m statistic --mode random --probability 0.33333
   -j KUBE-SEP-POD1
-A KUBE-SVC-MEINE-APP -m statistic --mode random --probability 0.50000
   -j KUBE-SEP-POD2
-A KUBE-SVC-MEINE-APP
   -j KUBE-SEP-POD3

-A KUBE-SEP-POD1 -p tcp -j DNAT --to-destination 10.244.1.5:8080
-A KUBE-SEP-POD2 -p tcp -j DNAT --to-destination 10.244.2.9:8080
-A KUBE-SEP-POD3 -p tcp -j DNAT --to-destination 10.244.3.2:8080
```

Ergebnis: Traffic an `10.96.42.100:80` wird zufällig (33/33/33%) auf die 3 Pods verteilt.

### NetworkPolicy — Firewall auf Layer 3/4

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: nur-traefik-darf-rein
spec:
  podSelector:
    matchLabels:
      app: meine-app
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: traefik      # Nur Traefik-Pods dürfen rein
    ports:
    - protocol: TCP
      port: 8080            # Nur auf diesem Port
```

**Debugging:**
```bash
kubectl get services -A                       # Services und ihre Ports
kubectl get endpoints -A                      # Pod-IPs hinter Services
ss -tlnp                                      # Lauschende Ports auf der Node
iptables -t nat -L KUBE-SERVICES -n           # kube-proxy Regeln
```

---

## Schicht 5 — Session (Sitzungsschicht)

### Was passiert hier?

**Verbindungen werden aufgebaut, verwaltet und beendet.** Die Session-Schicht
stellt sicher dass Kommunikation koordiniert stattfindet.

In der Praxis ist Layer 5 oft in Layer 4 (TCP) und Layer 7 (HTTP) integriert —
eine strikte Trennung gibt es im TCP/IP-Stack nicht.

### TLS — Das wichtigste Session-Protokoll

TLS (Transport Layer Security) sitzt zwischen Layer 4 (TCP) und Layer 7 (HTTP).
Es gehört konzeptuell zu Layer 5/6.

```
TLS Handshake (vereinfacht — TLS 1.3):

Client                                    Server (Traefik)
  │                                          │
  │── ClientHello ──────────────────────────►│
  │   - TLS Version (1.3)                    │
  │   - Cipher Suites (AES-256-GCM, etc.)   │
  │   - Random-Bytes                         │
  │   - Key Share (Diffie-Hellman Public Key)│
  │                                          │
  │◄─ ServerHello ───────────────────────────│
  │   - Gewählte Cipher Suite               │
  │   - Key Share (DH Public Key)           │
  │   - Zertifikat (von Let's Encrypt)      │
  │   - CertificateVerify                   │
  │   - Finished (MAC)                      │
  │                                          │
  │   [Beide berechnen Session Keys         ]│
  │   [aus DH Key Exchange]                  │
  │                                          │
  │── Finished ─────────────────────────────►│
  │                                          │
  │   === Ab jetzt alles verschlüsselt ===   │
  │── HTTP GET / (encrypted) ───────────────►│
```

### Diffie-Hellman Key Exchange (vereinfacht)

```
Ziel: Beide Seiten einigen sich auf einen geheimen Schlüssel
      OHNE ihn übers Netzwerk zu schicken.

Analogie mit Farben:
  Gemeinsame Farbe: Gelb (öffentlich bekannt)
  Client Geheimnis: Rot   → mischt zu Orange → schickt Orange
  Server Geheimnis: Blau  → mischt zu Grün   → schickt Grün

  Client nimmt Grün + Rot  = Braun
  Server nimmt Orange + Blau = Braun
  → Beide haben Braun (den Session Key) ohne Braun je zu senden!
```

### Session-Konzepte in Kubernetes

| Konzept | Beschreibung |
|---------|-------------|
| **TLS-Terminierung bei Traefik** | Traefik beendet TLS-Session, leitet HTTP intern weiter |
| **TLS-Passthrough** | Traefik leitet verschlüsselt durch, App terminiert selbst |
| **Session Affinity** | Service schickt Client immer zum selben Pod (Sticky Sessions) |
| **Keepalive** | TCP-Verbindung bleibt offen für mehrere Requests (HTTP/1.1, HTTP/2) |

### TLS-Secret in Kubernetes

cert-manager speichert das Zertifikat als Secret:
```yaml
apiVersion: v1
kind: Secret
type: kubernetes.io/tls
metadata:
  name: meine-app-tls
data:
  tls.crt: LS0tLS1CRUdJTi... # base64(Zertifikat + Chain)
  tls.key: LS0tLS1CRUdJTi... # base64(Private Key)
```

Traefik liest dieses Secret und nutzt es für den TLS Handshake mit dem Browser.

---

## Schicht 6 — Presentation (Darstellungsschicht)

### Was passiert hier?

**Daten werden in ein Format gebracht** das beide Seiten verstehen.
Verschlüsselung, Komprimierung, Zeichenkodierung.

In der Praxis: TLS-Verschlüsselung, gzip-Komprimierung, Zeichensätze.

### Relevante Konzepte

| Konzept | Schicht 6 Aufgabe |
|---------|-------------------|
| **TLS-Verschlüsselung** | Daten werden ver-/entschlüsselt |
| **gzip / brotli** | HTTP-Response wird komprimiert |
| **Base64** | Binäre Daten als Text kodiert (K8s Secrets!) |
| **JSON / YAML** | Serialisierungsformat für Kubernetes-API |
| **Protobuf** | Kubernetes API-Server nutzt intern Protobuf statt JSON |
| **UTF-8** | Zeichenkodierung für Text |

### Warum Base64 in Kubernetes Secrets?

```
Problem: YAML ist textbasiert — binäre Daten (Zertifikate, Schlüssel)
         können nicht direkt rein

Lösung: Base64-Kodierung
  Binary: 0x48 0x65 0x6C 0x6C 0x6F
  Base64: SGVsbG8=

kubectl create secret generic test --from-literal=pass=geheim
→ K8s speichert: Z2VoZWlt  (= base64("geheim"))

Wichtig: Base64 ist KEINE Verschlüsselung — es ist nur Kodierung!
         Jeder kann es dekodieren: echo "Z2VoZWlt" | base64 -d
```

### HTTP/2 vs. HTTP/1.1

```
HTTP/1.1 (Layer 6/7):
  - Eine Anfrage pro TCP-Verbindung (oder pipelining — selten)
  - Textbasiertes Protokoll
  - Header werden jedes Mal komplett gesendet

HTTP/2 (Layer 6/7):
  - Mehrere Anfragen gleichzeitig über eine TCP-Verbindung (Multiplexing)
  - Binäres Protokoll (schneller zu parsen)
  - Header-Komprimierung (HPACK)
  - Server Push möglich

Traefik und kube-apiserver sprechen beide HTTP/2.
```

---

## Schicht 7 — Application (Anwendungsschicht)

### Was passiert hier?

**Die eigentliche Anwendungslogik** — HTTP, DNS, SMTP, etc.
Das ist die Schicht mit der Entwickler täglich arbeiten.

### HTTP — Das wichtigste Protokoll

```
HTTP Request:
  GET /api/users HTTP/1.1
  Host: meine-app.beispiel.de
  Authorization: Bearer eyJhb...
  Content-Type: application/json
  Accept: application/json

HTTP Response:
  HTTP/1.1 200 OK
  Content-Type: application/json
  Content-Length: 1234

  {"users": [...]}
```

**HTTP-Methoden:**
```
GET     = Daten lesen (idempotent)
POST    = Neue Ressource erstellen
PUT     = Ressource komplett ersetzen (idempotent)
PATCH   = Ressource teilweise ändern
DELETE  = Ressource löschen (idempotent)
```

**HTTP-Statuscodes:**
```
2xx = Erfolg
  200 OK
  201 Created
  204 No Content

3xx = Weiterleitung
  301 Moved Permanently
  302 Found (temporär)
  307 Temporary Redirect
  308 Permanent Redirect

4xx = Client-Fehler
  400 Bad Request
  401 Unauthorized (nicht eingeloggt)
  403 Forbidden (eingeloggt, aber keine Rechte)
  404 Not Found
  429 Too Many Requests

5xx = Server-Fehler
  500 Internal Server Error
  502 Bad Gateway  ← Traefik sieht diesen wenn Pod nicht antwortet
  503 Service Unavailable
  504 Gateway Timeout
```

### DNS — Layer 7 Protokoll

DNS läuft über UDP Port 53 (Layer 4), aber das Protokoll selbst ist Layer 7:

```
DNS Query:
  ID: 1234
  Type: A (IPv4 Adresse)
  Name: meine-app.beispiel.de

DNS Response:
  ID: 1234
  Name: meine-app.beispiel.de
  Type: A
  TTL: 300 (Sekunden)
  Address: 89.167.116.105
```

**DNS-Record-Typen:**
```
A      = Domain → IPv4-Adresse
AAAA   = Domain → IPv6-Adresse
CNAME  = Domain → andere Domain (Alias)
MX     = Mail-Server für Domain
TXT    = Freier Text (z.B. für ACME DNS-01 Challenge)
NS     = Nameserver für Domain
```

**CoreDNS in Kubernetes (Layer 7):**
```
Pod fragt:  meine-app.default.svc.cluster.local → Typ A
CoreDNS:    Schaut in Kubernetes API nach Service "meine-app" in Namespace "default"
Antwort:    10.96.42.100  (ClusterIP des Services)
```

### Layer 7 in Kubernetes

| Was | Protokoll | Beschreibung |
|-----|-----------|-------------|
| **kube-apiserver** | HTTPS + JSON/Protobuf | Alle kubectl-Befehle |
| **ArgoCD Server** | HTTPS + gRPC | Web UI und CLI |
| **ArgoCD Git-Sync** | HTTPS / SSH | Cloned Git-Repositories |
| **Traefik Dashboard** | HTTP | Monitoring-UI |
| **Traefik Routing** | HTTP/HTTPS | Wertet Host-Header aus → Layer 7! |
| **cert-manager ACME** | HTTPS | Kommunikation mit Let's Encrypt |
| **kubelet API** | HTTPS | Health Checks, Metrics |
| **Prometheus/Metrics** | HTTP | `/metrics` Endpoint |

### Traefik als Layer-7-Proxy

Das ist entscheidend: Traefik ist ein **Layer-7-Proxy**.

```
Layer-4-Load-Balancer (z.B. HAProxy TCP mode):
  Sieht nur IP + Port
  Kann nicht nach Hostname unterscheiden
  Leitet blind weiter

Layer-7-Proxy (Traefik):
  Sieht den kompletten HTTP-Request
  Kann nach Host-Header routen: "Host: app-a.beispiel.de" → App A
  Kann nach Pfad routen: "/api/" → Backend, "/" → Frontend
  Kann Header lesen/schreiben
  Kann Auth prüfen (Middleware)
  Kann Rate Limiting anwenden
```

**Traefik muss TLS terminieren** um in den HTTP-Request schauen zu können:
```
Verschlüsselt (TLS):    Traefik sieht nur IP:Port — kein Hostname erkennbar
TLS terminiert (HTTP):  Traefik sieht Host-Header → kann routen
```

Deshalb läuft die Kette so:
```
Browser (HTTPS) → Traefik terminiert TLS → HTTP intern zum Pod
                  (liest Host-Header und routet)
```

---

## Alle Schichten zusammen — Eine HTTPS-Anfrage durch den Stack

Wenn du `https://meine-app.beispiel.de/api/data` aufrufst:

```
┌─────┬──────────────┬─────────────────────────────────────────────────────────────┐
│ OSI │ Schicht      │ Was passiert                                                │
├─────┼──────────────┼─────────────────────────────────────────────────────────────┤
│  7  │ Application  │ Browser erstellt HTTP-Request:                              │
│     │              │ GET /api/data HTTP/2                                         │
│     │              │ Host: meine-app.beispiel.de                                 │
│     │              │ Authorization: Bearer ...                                   │
│     │              │                                                             │
│     │              │ DNS-Auflösung:                                              │
│     │              │ meine-app.beispiel.de → 89.167.116.105                      │
├─────┼──────────────┼─────────────────────────────────────────────────────────────┤
│  6  │ Presentation │ HTTP-Request wird für TLS vorbereitet                       │
│     │              │ Komprimierung (gzip) wird ausgehandelt                      │
│     │              │ HTTP/2 Binary-Framing                                       │
├─────┼──────────────┼─────────────────────────────────────────────────────────────┤
│  5  │ Session      │ TLS Handshake:                                              │
│     │              │ - Zertifikat austauschen (Let's Encrypt)                   │
│     │              │ - Diffie-Hellman Key Exchange                               │
│     │              │ - Session Keys ableiten                                     │
│     │              │ - Alles ab jetzt verschlüsselt                             │
├─────┼──────────────┼─────────────────────────────────────────────────────────────┤
│  4  │ Transport    │ TCP Three-Way Handshake zu 89.167.116.105:443               │
│     │              │ Port 443 = HTTPS                                            │
│     │              │ TCP garantiert Reihenfolge und Zuverlässigkeit              │
│     │              │                                                             │
│     │              │ Im Cluster: kube-proxy DNAT                                 │
│     │              │ 10.96.42.100:80 → 10.244.2.5:8080 (Pod)                    │
├─────┼──────────────┼─────────────────────────────────────────────────────────────┤
│  3  │ Network      │ IP-Routing:                                                 │
│     │              │ Internet-Router leiten Pakete zu 89.167.116.105             │
│     │              │                                                             │
│     │              │ Im Cluster (Flannel):                                       │
│     │              │ 10.244.2.5 liegt auf Node 2 (192.168.1.11)                  │
│     │              │ VXLAN-Tunnel: Paket wird in UDP verpackt                    │
│     │              │ Node1→Node2: 192.168.1.10 → 192.168.1.11                    │
├─────┼──────────────┼─────────────────────────────────────────────────────────────┤
│  2  │ Data Link    │ Ethernet Frames:                                            │
│     │              │ MAC: 00:AA:BB:CC → 00:DD:EE:FF                              │
│     │              │ ARP: "Wer hat 192.168.1.11?"                                │
│     │              │ Switch leitet Frame zu richtigem Port                       │
├─────┼──────────────┼─────────────────────────────────────────────────────────────┤
│  1  │ Physical     │ Elektrische Signale / Lichtpulse auf dem Kabel              │
│     │              │ NIC wandelt Bits in physische Signale                       │
└─────┴──────────────┴─────────────────────────────────────────────────────────────┘
```

---

## Kubernetes-Komponenten nach OSI-Schicht

```
┌─────┬──────────────┬─────────────────────────────────────────────────────┐
│ OSI │ Schicht      │ Kubernetes-Komponenten                              │
├─────┼──────────────┼─────────────────────────────────────────────────────┤
│  7  │ Application  │ kube-apiserver (REST/HTTPS)                         │
│     │              │ ArgoCD Server (HTTPS/gRPC)                          │
│     │              │ CoreDNS (DNS)                                       │
│     │              │ Traefik (HTTP-Routing nach Host/Pfad)               │
│     │              │ cert-manager (ACME HTTPS)                           │
│     │              │ kubelet API (HTTPS)                                 │
│     │              │ Prometheus Metrics (/metrics HTTP)                  │
├─────┼──────────────┼─────────────────────────────────────────────────────┤
│  6  │ Presentation │ TLS-Zertifikate (cert-manager, Secrets)            │
│     │              │ Base64-Kodierung (K8s Secrets)                      │
│     │              │ JSON/YAML Serialisierung (API-Objekte)              │
│     │              │ Protobuf (interne API-Kommunikation)                │
│     │              │ gzip (HTTP-Komprimierung)                           │
├─────┼──────────────┼─────────────────────────────────────────────────────┤
│  5  │ Session      │ TLS Handshake (Traefik ↔ Browser)                  │
│     │              │ TLS Terminierung (Traefik)                          │
│     │              │ Session Affinity (K8s Service)                     │
│     │              │ SSH-Sessions (ArgoCD Git-Zugriff)                  │
├─────┼──────────────┼─────────────────────────────────────────────────────┤
│  4  │ Transport    │ kube-proxy (iptables — TCP/UDP Ports)              │
│     │              │ Service Ports (ClusterIP:Port → PodIP:Port)        │
│     │              │ NodePort (30000-32767)                              │
│     │              │ NetworkPolicy (TCP/UDP Filterung)                  │
│     │              │ Liveness/Readiness Probes (TCP-Check)              │
├─────┼──────────────┼─────────────────────────────────────────────────────┤
│  3  │ Network      │ Pod-IPs (CNI Plugin — Flannel/Calico)              │
│     │              │ ClusterIP (virtuelle Service-IPs)                  │
│     │              │ DNAT/SNAT (kube-proxy iptables)                    │
│     │              │ VXLAN-Tunnel (Flannel — Pod-zu-Pod über Nodes)     │
│     │              │ Pod-CIDR, Service-CIDR                             │
│     │              │ Node-Routing-Tabellen                              │
├─────┼──────────────┼─────────────────────────────────────────────────────┤
│  2  │ Data Link    │ Ethernet (Node-zu-Node Kommunikation)              │
│     │              │ veth pairs (virtuelle Ethernet-Kabel zu Pods)      │
│     │              │ Linux Bridge / OVS (Pod-Netzwerk)                  │
│     │              │ ARP (IP → MAC Auflösung)                           │
│     │              │ VLAN (Netzwerk-Segmentierung)                      │
├─────┼──────────────┼─────────────────────────────────────────────────────┤
│  1  │ Physical     │ Netzwerkkarten (NIC) der Nodes                     │
│     │              │ Kabel zwischen Nodes und Switch                    │
│     │              │ Switch-Hardware im Rechenzentrum                   │
└─────┴──────────────┴─────────────────────────────────────────────────────┘
```

---

## Wie ein Pod ans Netzwerk kommt — Layer 1 bis 7

Wenn ein neuer Pod gestartet wird, richtet das CNI-Plugin das Netzwerk ein:

```
1. [L1/L2] veth-Pair erstellen:
   containerd/CNI erstellt ein virtuelles Ethernet-Kabel:
   - Ein Ende: eth0 im Pod (bekommt die Pod-IP)
   - Anderes Ende: vethXXXXX auf der Node (wie ein Kabelende aus dem Pod)

2. [L2] Linux Bridge verbinden:
   vethXXXXX wird mit der cni0-Bridge verbunden
   (Bridge = virtueller Switch im Linux-Kernel)

3. [L3] IP-Adresse zuweisen:
   Pod eth0 bekommt IP aus dem Node-CIDR: z.B. 10.244.1.15/24
   Default-Route: 0.0.0.0/0 → 10.244.1.1 (Bridge-IP)

4. [L3] Route auf der Node eintragen:
   "10.244.1.15/32 → vethXXXXX"
   (Traffic für diese Pod-IP geht in die veth)

5. [L3] Cluster-weite Route (Flannel):
   Andere Nodes wissen: "10.244.1.0/24 ist auf Node 1"
   → VXLAN-Tunnel oder direkte Route

6. [L4] kube-proxy iptables:
   Service-Regeln werden für neue Pod-IP angepasst
   Endpoints-Objekt wird updated

7. [L7] Readiness Probe:
   kubelet macht HTTP GET /healthz zum Pod
   Erst wenn 200 OK → Pod bekommt Traffic
```

---

## Debugging nach OSI-Schichten

Wenn etwas nicht funktioniert, arbeite dich von unten nach oben:

```
Problem: "Ich kann meine-app.beispiel.de nicht erreichen"

L1 - Physical:
  ✓ Sind die Server überhaupt hochgefahren?
  ✓ Sind die Netzwerkkarten aktiv?
  → ip link show

L2 - Data Link:
  ✓ Sind ARP-Einträge vorhanden?
  ✓ Ist der Pod mit der Bridge verbunden?
  → ip neigh show
  → bridge fdb show

L3 - Network:
  ✓ Hat der Pod eine IP?
  ✓ Kann der Pod die Node anpingen?
  ✓ Kann Node 1 Node 2 anpingen?
  ✓ Sind die Routen korrekt?
  → kubectl get pod -o wide
  → ping <node-ip>
  → ip route show

L4 - Transport:
  ✓ Ist der Port offen?
  ✓ Sind die kube-proxy Regeln korrekt?
  ✓ Lauscht der Service auf dem richtigen Port?
  → kubectl get service
  → kubectl get endpoints
  → ss -tlnp (auf der Node)
  → iptables -t nat -L -n

L5 - Session:
  ✓ Ist das TLS-Zertifikat gültig?
  ✓ Ist es noch nicht abgelaufen?
  ✓ Passt die Domain?
  → kubectl get certificate
  → kubectl describe certificate <name>
  → openssl s_client -connect beispiel.de:443

L6 - Presentation:
  ✓ Ist das Secret korrekt base64-kodiert?
  ✓ Stimmt das JSON/YAML-Format?
  → kubectl get secret <name> -o yaml
  → echo "<base64>" | base64 -d

L7 - Application:
  ✓ Antwortet der Pod auf HTTP?
  ✓ Ist die URL korrekt?
  ✓ Stimmt der DNS-Name?
  ✓ Routet Traefik korrekt?
  → kubectl logs <pod>
  → curl -v https://meine-app.beispiel.de
  → kubectl exec -it <pod> -- curl localhost:8080/healthz
  → nslookup meine-app.default.svc.cluster.local
```

---

## Zusammenfassung in einem Satz pro Schicht

| OSI | Schicht | Ein-Satz-Zusammenfassung | Kubernetes-Beispiel |
|-----|---------|--------------------------|---------------------|
| 7 | Application | Protokolle die Menschen und Apps direkt nutzen | HTTP-Request an kube-apiserver |
| 6 | Presentation | Daten werden formatiert und verschlüsselt | Base64 in Secrets, TLS-Verschlüsselung |
| 5 | Session | Verbindungen werden aufgebaut und verwaltet | TLS Handshake mit Traefik |
| 4 | Transport | Zuverlässige Übertragung zwischen Ports | kube-proxy leitet TCP-Traffic weiter |
| 3 | Network | Pakete finden den Weg durch Netzwerke | Pod-IP, ClusterIP, VXLAN-Tunnel |
| 2 | Data Link | Frames bewegen sich im lokalen Netzwerksegment | veth pairs, ARP, Linux Bridge |
| 1 | Physical | Bits reisen als Signale über physische Medien | Netzwerkkabel zwischen Nodes |

---

*Zum besseren Verständnis: Das OSI-Modell ist ein Referenzmodell — in der Praxis
verschwimmen die Grenzen zwischen Schichten. TLS sitzt zwischen L4 und L7.
kube-proxy arbeitet auf L3 und L4. Traefik ist ein L7-Proxy der aber L4-Sessions
verwaltet. Das Modell hilft beim Denken und Debuggen — nicht als starre Grenze.*
