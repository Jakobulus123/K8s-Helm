# Port-Audit & Cluster-Härtung

> Konkreter Audit vom 2026-04-24 auf dem Kubernetes-Master (öffentliche IP
> `89.167.116.105`). Dokumentiert, welche Ports offen waren, welche davon
> riskant sind und wie man sie absichert.

---

## Inhaltsverzeichnis

1. [Wie auditiert man Ports?](#1-wie-auditiert-man-ports)
2. [Audit-Ergebnis](#2-audit-ergebnis)
3. [Was bedeutet welche Bind-Adresse?](#3-was-bedeutet-welche-bind-adresse)
4. [Risiko-Bewertung pro Port](#4-risiko-bewertung-pro-port)
5. [Fix-Reihenfolge (priorisiert)](#5-fix-reihenfolge-priorisiert)
6. [Daumenregel: Welche Ports gehören wohin?](#6-daumenregel-welche-ports-gehören-wohin)
7. [Re-Audit Checkliste](#7-re-audit-checkliste)

---

## 1. Wie auditiert man Ports?

Drei Befehle reichen für 90% der Aussage:

```bash
# Welche Ports lauschen, wer hört, auf welcher IP
ss -tlnp        # TCP listen
ss -ulnp        # UDP listen

# Was filtert die Firewall?
sudo ufw status verbose
sudo iptables -L INPUT -n --line-numbers
```

Alternativ von außen, aus einem fremden Netz:

```bash
nmap -Pn -p- 89.167.116.105
```

`nmap` von außen ist die ehrliche Antwort — `ss` zeigt nur, was lokal lauscht,
nicht was der Provider/Cloud-Firewall durchlässt.

---

## 2. Audit-Ergebnis

Auf `89.167.116.105` lauschten am 2026-04-24:

```
0.0.0.0:22                — sshd
0.0.0.0:8000              — python3 server.py (CWD /home/admin/workspace)
*:6443                    — kube-apiserver
*:10250                   — kubelet API
*:10256                   — kube-proxy healthz
89.167.116.105:2379       — etcd client
89.167.116.105:2380       — etcd peer
89.167.116.105:7946       — MetalLB memberlist
89.167.116.105:7472       — MetalLB metrics
0.0.0.0:8472 (UDP)        — Flannel/VXLAN
```

Und auf Localhost (unkritisch von außen):

```
127.0.0.1:2379, 2381, 8080, 8765, 9090, 10248, 10249, 10257, 10259, 33265
127.0.0.53/54:53          — systemd-resolved
```

**Firewall-Status:** UFW `inactive`. iptables enthält nur die von kube-proxy
verwalteten `KUBE-*`-Chains. Es gibt also keine eigene Filterung — was lauscht,
ist auch erreichbar.

---

## 3. Was bedeutet welche Bind-Adresse?

`ss -tlnp` zeigt links die Bind-Adresse. Das ist der entscheidende Indikator:

| Bind            | Bedeutung                                                     |
|-----------------|---------------------------------------------------------------|
| `127.0.0.1:X`   | Nur lokale Prozesse können verbinden. Sicher.                 |
| `89.167.116.105:X` | Nur über die öffentliche IP erreichbar (nicht über localhost-Trick). |
| `0.0.0.0:X` / `*:X` | Alle Interfaces — inklusive öffentlicher IP. Internet-exponiert. |

`0.0.0.0` und die explizite öffentliche IP sind beide aus dem Internet
erreichbar, solange keine Firewall davorsteht. Der Unterschied ist nur, ob der
Dienst zusätzlich via `localhost` antwortet.

---

## 4. Risiko-Bewertung pro Port

### Kritisch — sofort fixen

**`2379` und `2380` — etcd**

etcd ist die Single-Source-of-Truth des gesamten Clusters: alle Secrets, alle
Manifeste, alle Service-Accounts. Wer etcd lesen kann, hat den Cluster.
Standardmäßig nutzt Kubernetes Client-Cert-Auth, aber das ist nur eine
Verteidigungslinie. **etcd darf nie aus dem Internet erreichbar sein.**

Fix-Optionen:
- kubeadm-Cluster: `etcd` an `127.0.0.1` oder Node-Private-IP binden via
  `/etc/kubernetes/manifests/etcd.yaml` (`--listen-client-urls`,
  `--listen-peer-urls`).
- Notfall sofort: per `iptables` / `ufw` von außen blockieren.

**`10250` — kubelet API**

Wenn `anonymous-auth: true` oder `authorization-mode: AlwaysAllow`, kann jeder
Pods auf den Nodes starten und Code ausführen. Das ist eine der am häufigsten
ausgenutzten K8s-Schwachstellen.

Prüfen:
```bash
sudo grep -E "anonymous|authorization" /var/lib/kubelet/config.yaml
```

Soll-Werte:
```yaml
authentication:
  anonymous:
    enabled: false
authorization:
  mode: Webhook
```

### Hoch

**`8000` — python3 server.py**

Eigener Dienst (`/home/admin/workspace`). Auf `0.0.0.0` gebunden, also
internet-exponiert. Sollte entweder:
- in den Cluster wandern (als Pod hinter Ingress mit TLS),
- oder mindestens an `127.0.0.1` binden und über Traefik/Nginx mit TLS
  exponiert werden.

Niemals einen ungesicherten Dev-Server direkt ins Internet hängen.

**`6443` — kube-apiserver**

Nutzt TLS und Bearer-Token / Client-Cert-Auth. Muss aber erreichbar sein, wenn
du `kubectl` von extern nutzt. Härter machen via:
- IP-Whitelist auf Firewall-Ebene (nur deine Office-/Home-IPs).
- Audit-Logging einschalten.
- Kein `system:anonymous` Binding lassen.

### Mittel

**`7946`, `7472` — MetalLB**

MetalLB-interne Kommunikation. Sollte im Cluster-Netz bleiben, nicht öffentlich.
Auf Single-Node-Cluster nicht kritisch, aber sauberer ist Bind an Cluster-CIDR.

**`8472/UDP` — Flannel VXLAN**

Cluster-Netzwerk-Overlay. Gehört intern. Auf Single-Node nutzlos extern, aber
sollte gefiltert werden.

### Akzeptabel mit Auflagen

**`22` — SSH**

OK, wenn:
- `PasswordAuthentication no` in `/etc/ssh/sshd_config`,
- nur Key-Login,
- besser noch: fail2ban oder IP-Whitelist.

---

## 5. Fix-Reihenfolge (priorisiert)

1. **etcd absichern** (Bind ändern oder firewallen). Höchste Priorität.
2. **kubelet auth prüfen** (`anonymous.enabled: false`,
   `authorization.mode: Webhook`).
3. **UFW aktivieren** mit Whitelist:
   ```bash
   sudo ufw default deny incoming
   sudo ufw default allow outgoing
   sudo ufw allow 22/tcp        # SSH
   sudo ufw allow 80/tcp        # HTTP (Traefik)
   sudo ufw allow 443/tcp       # HTTPS (Traefik)
   sudo ufw allow 6443/tcp      # K8s API (besser: from <deine IP>)
   sudo ufw enable
   ```
   ⚠️ Vorsicht: Falls du via SSH verbunden bist und Port 22 vergisst, sperrst
   du dich aus.
4. **Python-Server umziehen** in den Cluster (Deployment + Service +
   IngressRoute mit TLS).
5. **SSH härten** (`PasswordAuthentication no`, optional fail2ban).
6. **6443 IP-Whitelist** statt komplett offen.

---

## 6. Daumenregel: Welche Ports gehören wohin?

| Komponente           | Bind sollte sein            |
|----------------------|-----------------------------|
| etcd                 | `127.0.0.1` oder Private-IP |
| kubelet healthz      | `127.0.0.1`                 |
| controller-manager   | `127.0.0.1`                 |
| scheduler            | `127.0.0.1`                 |
| kube-apiserver       | öffentlich, aber IP-Whitelist |
| App-Backends (intern)| `127.0.0.1` oder ClusterIP  |
| User-traffic (HTTP/HTTPS) | über Ingress, nicht direkt |
| SSH                  | öffentlich, key-only        |

Faustregel: **Was nicht aus dem Internet bedient werden muss, gehört nicht
ins Internet.** Default ist `127.0.0.1`, öffentlich ist eine bewusste
Entscheidung.

---

## 7. Re-Audit Checkliste

Einmal pro Quartal oder nach größeren Änderungen am Cluster:

```bash
# Was lauscht öffentlich?
ss -tlnp | grep -vE "127\.0\.0\.|::1"
ss -ulnp | grep -vE "127\.0\.0\.|::1"

# Firewall noch aktiv?
sudo ufw status verbose

# kubelet noch korrekt?
sudo grep -E "anonymous|authorization" /var/lib/kubelet/config.yaml

# Externer Blick (von einem fremden Host):
nmap -Pn -p 22,80,443,2379,2380,6443,10250 89.167.116.105
```

Wenn ein neuer öffentlicher Port auftaucht, den du nicht erwartest →
investigieren, nicht ignorieren.
