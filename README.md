# Talos Kubernetes Cluster GitOps Repository

Dieses Repository verwaltet die Konfiguration und den Betrieb eines Kubernetes-Clusters, das auf [Talos Linux](https://www.talos.dev/) basiert.

## Übersicht

Das Repository verwendet:
- **Talos Linux** - Ein minimales, container-optimiertes Betriebssystem für Kubernetes
- **talhelper** - Tool zur Generierung von Talos-Konfigurationen
- **SOPS** - Verschlüsselung für sensible Daten (Secrets)
- **Taskfile** - Task-Runner für Automatisierung
- **GitOps** - Versionskontrolle für alle Cluster-Konfigurationen

## Voraussetzungen

Folgende Tools müssen installiert sein:

- [talosctl](https://www.talos.dev/latest/talos-guides/install/) - Talos CLI Tool
- [talhelper](https://github.com/budimanjojo/talhelper) - Konfigurationsgenerator
- [SOPS](https://github.com/getsops/sops) - Secrets-Verschlüsselung
- [Taskfile](https://taskfile.dev/) - Task-Runner
- [kubectl](https://kubernetes.io/docs/tasks/tools/) - Kubernetes CLI
- [yq](https://github.com/mikefarah/yq) - YAML Processor
- [jq](https://stedolan.github.io/jq/) - JSON Processor
- [helmfile](https://helmfile.readthedocs.io/) - Helm Chart Management (optional)

### SOPS Setup

SOPS benötigt einen AGE-Schlüssel für die Verschlüsselung. Stelle sicher, dass die Umgebungsvariable `SOPS_AGE_KEY_FILE` gesetzt ist:

```bash
export SOPS_AGE_KEY_FILE=/path/to/your/age-key.txt
```

## Repository-Struktur

```
talos-ops/
├── .sops.yaml                    # SOPS-Verschlüsselungskonfiguration
├── .taskfiles/                   # Taskfile-Module
│   ├── bootstrap/               # Bootstrap-Tasks
│   ├── talos/                   # Talos-spezifische Tasks
│   └── talos-helper/            # Helper-Tasks für Talos
├── talos/
│   ├── talconfig.yaml           # Talos-Cluster-Konfiguration (Template)
│   ├── talenv.yaml              # Talos/Kubernetes Versionen
│   ├── talsecret.sops.yaml      # Verschlüsselte Talos-Secrets
│   ├── clusterconfig/           # Generierte Cluster-Konfigurationen (gitignored)
│   │   ├── *.yaml               # Generierte Node-Konfigurationen
│   │   └── talosconfig          # Talos-Konfigurationsdatei
│   └── patches/                 # Node-spezifische Patches
│       ├── pi/                  # Raspberry Pi spezifische Patches
│       └── sysctl.yaml          # System-Parameter
├── kubeconfig                   # Kubernetes-Konfiguration (gitignored)
└── Taskfile.yaml                # Haupt-Taskfile

```

## Verfügbare Tasks

### Cluster-Bootstrap

#### `bootstrap:talos`
Bootstrappt den Talos-Cluster. Dies ist der erste Schritt nach der Installation von Talos auf den Nodes.

```bash
task bootstrap:talos
```

Führt folgende Schritte aus:
1. Erstellt Talos-Secrets (falls nicht vorhanden)
2. Generiert aktuelle Cluster-Konfigurationen
3. Wendet Konfigurationen auf alle Nodes an
4. Bootstrappt den Cluster
5. Generiert die kubeconfig

#### `bootstrap:helm:critical`
Bootstrappt kritische Netzwerk-Komponenten (z.B. Cilium).

```bash
task bootstrap:helm:critical
```

#### `bootstrap:helm:infra`
Bootstrappt Infrastruktur-Komponenten.

```bash
task bootstrap:helm:infra
```

#### `bootstrap:helm:flux`
Bootstrappt Flux für GitOps.

```bash
task bootstrap:helm:flux
```

### Konfiguration

#### `talos:talhelper:generate-config`
Generiert die Cluster-Konfigurationen aus `talconfig.yaml` und `talsecret.sops.yaml`.

```bash
task talos:talhelper:generate-config
```

#### `talos:talhelper:kubeconfig`
Regeneriert die kubeconfig-Datei.

```bash
task talos:talhelper:kubeconfig
```

#### `talos:apply-clusterconfig`
Wendet die generierten Cluster-Konfigurationen auf alle Nodes an.

```bash
task talos:apply-clusterconfig
```

#### `talos:apply-node`
Wendet die Konfiguration auf einen spezifischen Node an.

```bash
task talos:apply-node NODE=master01
```

### Upgrades

#### `talos:upgrade-node`
Upgradet Talos auf einem einzelnen Node.

```bash
task talos:upgrade-node NODE=master01
```

#### `talos:upgrade-k8s`
Upgradet Kubernetes auf allen Nodes.

```bash
task talos:upgrade-k8s
```

Die Kubernetes-Version wird aus `talos/talenv.yaml` gelesen.

### Helper-Tasks

#### `talos:helper:get-disks`
Zeigt verfügbare Disks eines Nodes an.

```bash
task talos:helper:get-disks IP=192.168.0.196
```

#### `talos:helper:get-links`
Zeigt Netzwerk-Interfaces eines Nodes an.

```bash
task talos:helper:get-links IP=192.168.0.196
```

#### `talos:helper:get-links-up`
Zeigt aktive Netzwerk-Interfaces eines Nodes an.

```bash
task talos:helper:get-links-up IP=192.168.0.196
```

### Wartung

#### `talos:reset`
Setzt Nodes zurück in den Wartungsmodus. **VORSICHT: Dies zerstört den Cluster!**

```bash
task talos:reset
```

## Workflow

### Initiales Setup

1. **Repository klonen**
   ```bash
   git clone <repository-url>
   cd talos-ops
   ```

2. **SOPS-Schlüssel konfigurieren**
   ```bash
   export SOPS_AGE_KEY_FILE=/path/to/your/age-key.txt
   ```

3. **Talos auf Nodes installieren**
   - Installiere Talos auf allen physischen/virtuellen Nodes
   - Stelle sicher, dass die Nodes erreichbar sind

4. **Cluster-Konfiguration anpassen**
   - Bearbeite `talos/talconfig.yaml` für deine Umgebung
   - Passe IP-Adressen, Hostnamen und andere Parameter an

5. **Cluster bootstrappen**
   ```bash
   task bootstrap:talos
   ```

6. **Kritische Komponenten installieren**
   ```bash
   task bootstrap:helm:critical
   ```

7. **Infrastruktur installieren**
   ```bash
   task bootstrap:helm:infra
   task bootstrap:helm:flux
   ```

### Regelmäßige Wartung

#### Konfiguration aktualisieren

1. Bearbeite `talos/talconfig.yaml` oder `talos/talenv.yaml`
2. Generiere neue Konfigurationen:
   ```bash
   task talos:talhelper:generate-config
   ```
3. Wende Konfigurationen an:
   ```bash
   task talos:apply-clusterconfig
   ```

#### Kubernetes-Upgrade

1. Aktualisiere `kubernetesVersion` in `talos/talenv.yaml`
2. Führe das Upgrade aus:
   ```bash
   task talos:upgrade-k8s
   ```

#### Talos-Upgrade

1. Aktualisiere `talosVersion` in `talos/talenv.yaml`
2. Upgrade Node für Node:
   ```bash
   task talos:upgrade-node NODE=master01
   task talos:upgrade-node NODE=master02
   task talos:upgrade-node NODE=master03
   ```

## Konfiguration

### Talos-Konfiguration (`talos/talconfig.yaml`)

Die Hauptkonfigurationsdatei definiert:
- Cluster-Name
- Talos- und Kubernetes-Versionen
- Endpoint (VIP-Adresse)
- Node-Definitionen (Hostname, IP, Installations-Disk, etc.)
- Netzwerk-Konfiguration
- System-Extensions
- Volumes und Storage

### Versionsverwaltung (`talos/talenv.yaml`)

Diese Datei wird von Renovate verwaltet und enthält:
- `talosVersion` - Talos Linux Version
- `kubernetesVersion` - Kubernetes Version

### Secrets (`talos/talsecret.sops.yaml`)

Verschlüsselte Talos-Secrets. Werden automatisch generiert, falls nicht vorhanden.

## Sicherheit

- Alle sensiblen Daten werden mit SOPS verschlüsselt
- `kubeconfig` und `talosconfig` sind in `.gitignore` ausgeschlossen
- Generierte Konfigurationsdateien werden nicht versioniert
- SOPS-Schlüsseldateien werden nicht versioniert

## Troubleshooting

### kubeconfig regenerieren

```bash
task talos:talhelper:kubeconfig
```

### Node-Status prüfen

```bash
talosctl --nodes <node-ip> health
```

### Cluster-Status prüfen

```bash
kubectl get nodes
kubectl get pods -A
```

### Konfiguration auf Node prüfen

```bash
talosctl --nodes <node-ip> get machineconfig
```

## Weitere Ressourcen

- [Talos Linux Dokumentation](https://www.talos.dev/)
- [talhelper Dokumentation](https://github.com/budimanjojo/talhelper)
- [SOPS Dokumentation](https://github.com/getsops/sops)
- [Taskfile Dokumentation](https://taskfile.dev/)

