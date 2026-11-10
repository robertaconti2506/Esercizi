# Esercizio AWX → Jenkins

AWX preleva un sorgente Java (`example.java`) da una VM e, tramite un webhook,
avvia una pipeline Jenkins che lo compila con `javac` e ne archivia il risultato.

## Obiettivo

1. Creare un Job Template AWX che preleva un dato da una VM (`example.java`)
2. Creare un Webhook su AWX che si interfaccia a Jenkins creando un artefatto con il dato prelevato
3. Triggerare da AWX una pipeline Jenkins che usa l'artefatto per una build (`javac example.java`)
4. Il `Jenkinsfile` della pipeline deve essere preso da GitHub

## Architettura

```
┌──────────────────────── Mac (Intel, 8 GB RAM) ────────────────────────┐
│                                                                       │
│  Docker Desktop                        VirtualBox                     │
│  └─ minikube (profilo "awx")           └─ VM Vagrant "vm-sorgente"    │
│      └─ AWX 24.6.1                         IP 192.168.56.13           │
│         (AWX Operator 2.19.1)              ├─ /home/vagrant/example.java
│                                            └─ Jenkins (Podman rootless)
│                                               porta 8080              │
└───────────────────────────────────────────────────────────────────────┘
                     │                          │
                     └────────► GitHub ◄────────┘
                       playbook.yaml    Jenkinsfile
```

| Componente | Dove gira | Ruolo |
|---|---|---|
| AWX | minikube su Docker Desktop | preleva il file e lo invia a Jenkins |
| VM Vagrant | VirtualBox | contiene il sorgente e ospita Jenkins |
| Jenkins | container Podman rootless nella VM | compila il sorgente |
| GitHub | online | ospita playbook e Jenkinsfile |

## Flusso

```
AWX Job Template "preleva-sorgente"
  1. fetch      → copia example.java dalla VM via SSH
  2. set_stats  → pubblica il file (base64) negli artifacts del job
        │
        ▼  job riuscito
AWX Notification Template "jenkins-build-example" (Webhook)
  3. POST JSON {nome_file, sorgente_b64, awx_job_id} + header token
        │
        ▼
Jenkins job "build-example-awx" (Generic Webhook Trigger)
  4. checkout del Jenkinsfile da GitHub
  5. decodifica il base64 → example.java
  6. javac example.java → java example
  7. archivia example.java ed example.class come artefatti
```

## Struttura del repository

```
awx/
├── Vagrantfile          VM sorgente (Debian 12, IP 192.168.56.13)
├── k8s/
│   ├── kustomization.yaml   AWX Operator 2.19.1
│   └── awx.yaml             istanza AWX
├── playbook.yaml        preleva il sorgente e lo pubblica come artifact
├── Jenkinsfile          pipeline: ricostruisce, compila, esegue, archivia
└── README.md
```

## Passaggi svolti

1. **VM Vagrant**: Debian 12, IP fisso, `example.java` creato dal provisioning
2. **Jenkins**: container Podman rootless con volume, avviato al boot da systemd (linger)
3. **AWX**: minikube con driver Docker, AWX Operator installato con Kustomize
4. **Repository GitHub**: `Jenkinsfile` e `playbook.yaml`
5. **Jenkins**: plugin Generic Webhook Trigger, credenziale `awx-webhook-token`,
   job Pipeline "from SCM"
6. **AWX**: Credential SSH, Project, Inventory, Job Template
7. **Webhook**: Notification Template collegato al Job Template sull'evento *Success*

## Scelte tecniche

- **File in base64**: il webhook trasporta solo testo JSON
- **Generic Webhook Trigger**: `buildWithParameters` non legge un corpo JSON
- **minikube su Docker**: il driver VirtualBox non è supportato su macOS 13+
- **Token del webhook come credenziale Jenkins**: non compare nel repository pubblico