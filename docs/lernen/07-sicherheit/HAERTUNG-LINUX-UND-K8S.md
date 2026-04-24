# Härtung: Linux + Kubernetes von 0 auf 100 %

> Pädagogischer Leitfaden. Erklärt, **wie man einen Server und einen
> Kubernetes-Cluster absichert** — in Stufen, von "gar nichts getan" bis
> "produktionsreif". Der konkrete Audit aus
> [`PORT-AUDIT.md`](PORT-AUDIT.md) dient als laufendes Beispiel und wird
> am Ende auf dieser Skala eingeordnet.

---

## Inhaltsverzeichnis

1. [Mental Model: Was heißt "absichern"?](#1-mental-model-was-heißt-absichern)
2. [Die drei Fragen, die jedes Audit beantwortet](#2-die-drei-fragen-die-jedes-audit-beantwortet)
3. [Skala: Was bedeuten 1 %, 50 %, 100 %?](#3-skala-was-bedeuten-1--50--100-)
4. [Teil A — Linux-Basis (0–40 %)](#4-teil-a--linux-basis-0-40-)
5. [Teil B — Kubernetes-Basis (40–70 %)](#5-teil-b--kubernetes-basis-40-70-)
6. [Teil C — Fortgeschrittene Kubernetes-Härtung (70–100 %)](#6-teil-c--fortgeschrittene-kubernetes-härtung-70-100-)
7. [Wo steht unser Cluster heute?](#7-wo-steht-unser-cluster-heute)
8. [Wie oft härtet man nach?](#8-wie-oft-härtet-man-nach)

---

## 1. Mental Model: Was heißt "absichern"?

Absichern ist **nicht** "eine Sache einschalten, fertig". Es ist das
systematische Reduzieren von Angriffsfläche auf mehreren Ebenen:

```
┌─────────────────────────────────────────────────┐
│  Ebene 4: Anwendung — eure Pods, euer Code      │
│  Ebene 3: Kubernetes — RBAC, PSA, NetworkPolicy │
│  Ebene 2: Cluster-Komponenten — etcd, kubelet   │
│  Ebene 1: Linux-Host — sshd, UFW, User          │
│  Ebene 0: Netzwerk/Provider — VPC, Firewall     │
└─────────────────────────────────────────────────┘
```

**Defense in Depth**: jede Ebene schützt unabhängig. Wenn Ebene 4 fällt,
hält Ebene 3. Wenn 3 fällt, hält 2. Wer nur eine Ebene hat, verliert
komplett beim ersten Bug.

**Least Privilege**: jeder Prozess, User, Pod bekommt nur genau die
Rechte, die er braucht — nicht mehr. Ein kompromittierter Nginx-Pod
darf nicht das ganze Cluster sehen.

**Zero Trust by Default**: alles ist zunächst verboten. Zugriffe werden
explizit erlaubt. Nicht "ich blockiere die bösen Dinge", sondern "ich
erlaube nur die guten".

---

## 2. Die drei Fragen, die jedes Audit beantwortet

Bei einem unbekannten Server oder Cluster fragst du immer drei Dinge:

### Frage 1: Was lauscht?

Welche Dienste hören auf Ports? Auf welchen Interfaces?

```bash
ss -tlnp                         # TCP listen
ss -ulnp                         # UDP listen
ps auxf                          # welcher Prozess überhaupt
```

### Frage 2: Was filtert?

Welche Firewall steht davor? Welche Regeln?

```bash
sudo ufw status verbose
sudo iptables -L -n --line-numbers
sudo nft list ruleset
```

Und: was filtert der **Provider/Hoster/Cloud** davor? Ein "offener" Port
lokal kann extern geschlossen sein (Cloud Security Group), oder umgekehrt.

### Frage 3: Was authentifiziert?

Wer darf was, mit welchem Beweis?

- SSH: Key oder Passwort?
- Kubernetes: RBAC wie konfiguriert?
- Services: eigene Auth?
- etcd: Client-Cert?

**Erst wenn du alle drei Fragen beantworten kannst, verstehst du den
Sicherheitsstand.** Das Audit in `PORT-AUDIT.md` ist genau die Antwort
auf Frage 1 und 2 für unseren Master-Node.

---

## 3. Skala: Was bedeuten 1 %, 50 %, 100 %?

Pragmatische Orientierung, kein Industriestandard:

| Reifegrad | Charakter                                                |
|-----------|----------------------------------------------------------|
|  0 %      | Frisch gebootet, Default-Passwörter, alles offen         |
| 10 %      | SSH absichern, Updates, Firewall aktiv                   |
| 30 %      | Dienste auf Loopback, minimaler User-Set, fail2ban       |
| 50 %      | K8s Control Plane sauber gebunden, kubelet auth, UFW     |
| 70 %      | RBAC least-privilege, NetworkPolicies, Secrets encrypted |
| 90 %      | Pod Security Admission, seccomp, Image-Signierung        |
| 100 %     | CIS Benchmark grün, Runtime-Security, SBOM, Audit-Logs   |

Die meisten Hobbyprojekte bleiben bei 30–50 %. Produktion sollte
mindestens 70 % erreichen. 100 % ist ein bewegliches Ziel.

---

## 4. Teil A — Linux-Basis (0–40 %)

Die Linux-Schicht ist dein Fundament. Wenn ein Angreifer hier Root
bekommt, ist jede darüberliegende Ebene egal.

### 4.1 Stufe 5 % — SSH absichern

Der häufigste Angriffsvektor bei jedem Internet-Server. Default-sshd
akzeptiert Passwörter — Bots probieren 24/7 durch.

**Schritte in `/etc/ssh/sshd_config`:**

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AllowUsers admin           # nur explizit erlaubte User
Port 22                    # optional auf unübliche Zahl ändern (Obfuskation, kein Schutz)
```

Danach `sudo systemctl reload sshd`. Und **vorher** einen funktionierenden
Key-Login testen — sonst sperrst du dich aus.

### 4.2 Stufe 10 % — Firewall mit Deny-by-Default

Zero Trust auf Netz-Ebene. Standardmäßig verbietet die Firewall alles,
du erlaubst explizit was rein darf.

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp                 # SSH
sudo ufw allow 80/tcp                 # HTTP (Ingress)
sudo ufw allow 443/tcp                # HTTPS (Ingress)
sudo ufw allow from <deine IP> to any port 6443  # K8s API nur für dich
sudo ufw enable
```

**Testen mit `nmap` von extern**, ob die Erwartung stimmt. Interne
Kubernetes-Ports (etcd, kubelet, CNI) dürfen hier **nicht** aufgelistet
werden — die gehören auf Loopback oder private IPs (siehe Teil B).

### 4.3 Stufe 15 % — Automatische Updates

Bekannte CVEs sind der zweite häufige Angriffsweg. Wer nicht patcht,
wird irgendwann durch eine Standard-Schwachstelle genommen.

```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

In `/etc/apt/apt.conf.d/50unattended-upgrades` Security-Updates
aktivieren. Reboots nach Kernel-Update nicht vergessen — `needrestart`
hilft.

### 4.4 Stufe 20 % — User & Sudo sauber

```bash
# Root direkt sperren (passwortlos)
sudo passwd -l root

# Sudo-Gruppe minimal halten
getent group sudo
```

Niemand sollte als Root arbeiten. Jeder Admin hat seinen eigenen User
mit `sudo`. Service-Accounts für Apps: eigener User ohne Login-Shell
(`/usr/sbin/nologin`).

### 4.5 Stufe 25 % — Dienste minimieren

```bash
systemctl list-unit-files --state=enabled | grep service
```

Alles abschalten, was nicht gebraucht wird. Cups, Avahi, snap, was
immer der Installer mitbrachte. **Jeder lauschende Port ist Angriffs-
fläche** — auch wenn du ihn nicht nutzt.

### 4.6 Stufe 30 % — fail2ban

Bots, die SSH durchprobieren, werden per IP-Ban temporär gesperrt.

```bash
sudo apt install fail2ban
sudo systemctl enable --now fail2ban
```

Default-Jails decken SSH ab. Weitere Jails für Nginx, Apache, Postfix
je nach Workload.

### 4.7 Stufe 35 % — Audit & Logging

```bash
sudo apt install auditd
# Logins loggen:
journalctl -u ssh --since "1 week ago"
# Sudo-Nutzung:
journalctl _COMM=sudo
```

Ohne Logs weißt du nie, ob du kompromittiert wurdest. Minimum: SSH-Logs
+ sudo-Logs an einen Remote-Syslog, falls der Host selbst übernommen
wird.

### 4.8 Stufe 40 % — Disk-Encryption / Backup-Integrität

Disks verschlüsselt (LUKS), Backups signiert und getrennt aufbewahrt.
Bei Cloud-Hostern meistens eh aktiv; bare metal nachrüsten.

### Linux-Ergebnis bei 40 %

- sshd nur mit Key, nur erlaubte User
- UFW mit Deny-by-Default
- Auto-Updates aktiv
- Keine unnötigen Services
- fail2ban schützt SSH
- Logs gehen irgendwohin

Das reicht für einen Hobby-Server. **Für Kubernetes fangen wir jetzt
erst an.**

---

## 5. Teil B — Kubernetes-Basis (40–70 %)

Kubernetes fügt eine eigene Angriffsfläche hinzu: die Control-Plane-
Komponenten sind selber Netzwerk-Services mit eigener Auth.

### 5.1 Stufe 45 % — Control-Plane-Bindungen

Diese Dienste **dürfen nie öffentlich sein**:

| Komponente         | Soll-Bind                      |
|--------------------|--------------------------------|
| etcd (2379/2380)   | `127.0.0.1` oder Node-Private-IP |
| kubelet healthz    | `127.0.0.1`                    |
| controller-manager | `127.0.0.1`                    |
| scheduler          | `127.0.0.1`                    |

Bei kubeadm-Clustern steht der Bind in
`/etc/kubernetes/manifests/*.yaml` (statische Pods) — Änderung = Pod
restart = sofort wirksam.

**Der apiserver (6443) muss erreichbar bleiben**, wenn du extern
`kubectl` nutzt. Dafür Firewall-Whitelist (siehe 4.2).

### 5.2 Stufe 50 % — kubelet-Authentifizierung

In `/var/lib/kubelet/config.yaml`:

```yaml
authentication:
  anonymous:
    enabled: false        # WICHTIG
  webhook:
    enabled: true
authorization:
  mode: Webhook           # nicht AlwaysAllow
```

Ohne das kann jeder mit Netzzugriff auf Port 10250 Pods starten und
Container-Code ausführen. Eine der am häufigsten missbrauchten
K8s-Schwachstellen.

### 5.3 Stufe 55 % — Audit Logging im apiserver

Im `/etc/kubernetes/manifests/kube-apiserver.yaml` ergänzen:

```yaml
- --audit-log-path=/var/log/kubernetes/audit.log
- --audit-policy-file=/etc/kubernetes/audit-policy.yaml
- --audit-log-maxage=30
- --audit-log-maxsize=100
- --audit-log-maxbackup=10
```

Audit-Policy definiert, was geloggt wird. Minimum: alle Writes auf
Secrets, RBAC-Änderungen, Pod-Exec.

### 5.4 Stufe 60 % — RBAC Least Privilege

Default-Setup nutzt oft zu weite Rollen. Regel:

1. **Kein `cluster-admin`** für Apps/Users, nur für dich als Human-Admin.
2. **Pro Namespace eigene ServiceAccounts** statt `default`.
3. **Roles und RoleBindings** so eng wie möglich:

```yaml
kind: Role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]     # nicht "*"
```

Prüfen:
```bash
kubectl auth can-i --list --as=system:serviceaccount:myns:mysa
```

### 5.5 Stufe 65 % — ServiceAccount-Token-Mounting aus

Pods bekommen per Default das SA-Token gemountet. Wenn der Pod's App
keinen API-Zugriff braucht:

```yaml
spec:
  automountServiceAccountToken: false
```

Ein kompromittierter Pod kann dann nicht automatisch mit der K8s-API
reden.

### 5.6 Stufe 70 % — NetworkPolicies einführen

Default in K8s: **jeder Pod kann mit jedem Pod reden**. Das willst du
nicht.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: myapp
spec:
  podSelector: {}
  policyTypes: ["Ingress", "Egress"]
```

Danach explizit erlauben, was geredet werden darf (Frontend → Backend,
Backend → DB, Backend → DNS, etc.).

⚠️ Setzt einen CNI voraus, der Policies umsetzt (Calico, Cilium).
Flannel allein macht das **nicht**.

### Kubernetes-Basis-Ergebnis bei 70 %

- Control-Plane-Ports intern gebunden
- kubelet nicht anonym
- apiserver loggt Audits
- RBAC minimal
- Pods haben nur das Token, das sie brauchen
- Default-Deny-Netzwerk

Das ist der Sockel, auf dem Prod-Workloads stehen dürfen.

---

## 6. Teil C — Fortgeschrittene Kubernetes-Härtung (70–100 %)

### 6.1 Stufe 75 % — Secrets-at-Rest-Encryption

Default: K8s-Secrets liegen **base64-encoded, nicht verschlüsselt** in
etcd. Wer etcd liest, hat alle Credentials.

Fix: EncryptionConfiguration mit KMS oder aescbc-Provider:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources: ["secrets"]
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: <base64-32byte>
  - identity: {}
```

An apiserver als `--encryption-provider-config=...` übergeben. Bestehende
Secrets danach neu schreiben: `kubectl get secrets -A -o json | kubectl
replace -f -`.

### 6.2 Stufe 80 % — Pod Security Admission

Namespace-Label setzt das Security-Level für alle Pods darin:

```yaml
metadata:
  labels:
    pod-security.kubernetes.io/enforce: restricted
```

`restricted` = kein privileged, kein hostNetwork, readOnlyRootFilesystem,
runAsNonRoot. Pods, die das verletzen, werden gar nicht erst
geschedult.

### 6.3 Stufe 85 % — seccomp, capabilities, SecurityContext

Pro Container:

```yaml
securityContext:
  runAsNonRoot: true
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: ["ALL"]
  seccompProfile:
    type: RuntimeDefault
```

Verhindert, dass ein Container-Escape auf den Node durchschlägt.

### 6.4 Stufe 90 % — Image-Sicherheit

- **Minimal Base Images** (distroless, alpine, scratch)
- **Image-Scanning in CI** (Trivy, Grype) — keine kritischen CVEs ins Image
- **Signierte Images** (cosign) und Policy im Cluster (Kyverno, OPA)
- **Image Pull Policy** `Always` bei Tags, besser: per Digest pinnen

### 6.5 Stufe 93 % — Runtime Security

Falco oder Tetragon beobachtet Syscalls in Containern und alarmiert bei
Anomalien (`/etc/shadow` gelesen, Shell in unerwartetem Pod, etc.).

### 6.6 Stufe 96 % — Supply Chain

- **SBOM** (Software Bill of Materials) pro Image erzeugen und signieren
- **Sigstore/cosign** verifiziert Signatur beim Pull
- **Dependency-Scanning** der CI-Pipelines selbst

Ziel: bei jedem laufenden Container weißt du, **welcher Commit welchen
Codes** von **welcher Person** hereinkam.

### 6.7 Stufe 100 % — Benchmark, Pen-Test, Incident Response

- [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes)
  mit `kube-bench` durchlaufen — grün werden
- **Regelmäßiger Pen-Test** von extern
- **Incident Response Plan**: wer wird wie alarmiert, wie isoliert man
  einen Node, wo sind Backups
- **Chaos-Drills**: regelmäßig Ausfälle simulieren

100 % ist ein **Zustand, den man aufrechterhält**, nicht ein Knopf, den
man einmal drückt.

---

## 7. Wo steht unser Cluster heute?

Basierend auf [`PORT-AUDIT.md`](PORT-AUDIT.md) vom 2026-04-24:

| Level  | Bereich                              | Status |
|--------|--------------------------------------|--------|
| 5 %    | SSH absichern                        | ⚠️ zu prüfen — nutzt dieser Host Password-Auth? |
| 10 %   | UFW Deny-by-Default                  | ❌ UFW `inactive` |
| 15 %   | Auto-Updates                         | ⚠️ zu prüfen |
| 20 %   | User & sudo                          | ⚠️ zu prüfen |
| 30 %   | fail2ban                             | ⚠️ zu prüfen |
| 45 %   | Control-Plane-Bindungen              | ❌ etcd auf öffentlicher IP |
| 50 %   | kubelet auth                         | ⚠️ zu prüfen (`/var/lib/kubelet/config.yaml`) |
| 55 %   | apiserver audit log                  | ⚠️ zu prüfen |
| 60 %   | RBAC least privilege                 | ⚠️ miniapp nutzt SA + RoleBinding (schon etwas) |
| 70 %   | NetworkPolicy default-deny           | ❌ Flannel ohne Policy-Support |

**Geschätzter Reifegrad: 15–25 %.** Die harten Kanten (etcd public, UFW
aus, Python-Server auf 8000) ziehen den Gesamtstand nach unten, auch
wenn einzelne Apps (miniapp, RBAC) schon sauberer sind.

**Der realistische nächste Schritt** ist, Teil A vollständig auf 40 %
zu bringen (1–2 Stunden Arbeit: sshd config, UFW aktivieren, Updates,
fail2ban) und dann Teil B anzufangen (etcd-Bind fixen ist die
wichtigste Einzeländerung).

---

## 8. Wie oft härtet man nach?

| Ereignis                          | Re-Audit nötig?        |
|-----------------------------------|------------------------|
| Neuer Service eingebaut           | Ja, Port & RBAC prüfen |
| Neue K8s-Minor-Version            | Ja, Manifeste + CIS    |
| Neuer Cluster-Node                | Ja, Linux-Basis + kubelet |
| Quartalsweise (auch ohne Änderung) | Ja, Baseline halten   |
| Nach Incident                     | Komplett, auch Ebenen, die "nicht betroffen" schienen |

**Sicherheit ist kein Projekt, sondern ein Prozess.** Einmal aufbauen
reicht nicht — man bleibt nur auf 70 %, wenn man alle paar Monate
nachzieht.

---

## Weiterführend

- `PORT-AUDIT.md` — der konkrete Ausgangs-Audit
- [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes)
- [NSA/CISA Kubernetes Hardening Guide](https://media.defense.gov/2022/Aug/29/2003066362/-1/-1/0/CTR_KUBERNETES_HARDENING_GUIDANCE_1.2_20220829.PDF)
- [Kubernetes Security Checklist (offiziell)](https://kubernetes.io/docs/concepts/security/security-checklist/)
