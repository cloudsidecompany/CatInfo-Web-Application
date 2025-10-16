# Esercizio Pratico: CatInfo Web Application

## Obiettivo
Realizzare una web application completa per la consultazione di informazioni sulle razze di gatti, utilizzando un'architettura a microservizi deployata su Kubernetes.

**Tempo stimato:** 3 giorni  
**Difficoltà:** Intermedio

---

## 1. Panoramica del Progetto

### 1.1 Descrizione Funzionale
CatInfo è una semplice applicazione web che permette agli utenti di:
- Inserire il nome di una razza di gatto
- Ottenere una descrizione della razza (caratteristiche, temperamento, origini)
- Visualizzare lo storico delle ricerche effettuate

### 1.2 Stack Tecnologico
- **Frontend:** HTML/CSS/JavaScript vanilla + Bootstrap (opzionale)
- **Backend:** Spring Boot (Java 17+)
- **Database:** PostgreSQL 15+
- **Containerizzazione:** Docker
- **Orchestrazione:** Kubernetes (cluster Rancher)
- **Package Manager:** Helm 3
- **Infrastructure as Code:** Terraform

### 1.3 Architettura
```
┌─────────────────────────────────────────────────────────┐
│                    GitHub Repository                     │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                  Kubernetes Cluster                      │
│  ┌───────────────────────────────────────────────────┐  │
│  │              Helm Chart: catinfo                   │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌──────────┐  │  │
│  │  │  Frontend   │  │   Backend   │  │   PSQL   │  │  │
│  │  │   Service   │─▶│   Service   │─▶│  (Helm)  │  │  │
│  │  │  Deployment │  │  Deployment │  │          │  │  │
│  │  └─────────────┘  └─────────────┘  └──────────┘  │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
         ▲
         │ Provisioned by
         │
┌────────┴────────┐
│   Terraform     │
│  (Rancher API)  │
└─────────────────┘
```

---

## 2. Requisiti Dettagliati

### 2.1 Frontend Service
**Path:** `/frontend`

**Responsabilità:**
- Servire file statici (HTML, CSS, JS)
- Interfaccia utente per ricerca razze
- Chiamate REST API al backend

**Tecnologie:**
- Spring Boot (per servire static resources)
- HTML5, CSS3, JavaScript (vanilla)
- Fetch API per chiamate HTTP

**Endpoints esposti:**
- `GET /` - Homepage
- `GET /static/*` - Risorse statiche

**Porta:** 8080

**Docker:**
- Base image: `eclipse-temurin:17-jre-alpine`
- Exposed port: 8080

---

### 2.2 Backend Service
**Path:** `/backend`

**Responsabilità:**
- Gestione logica di business
- API REST per frontend
- Persistenza dati su PostgreSQL

**Tecnologie:**
- Spring Boot 3.x
- Spring Data JPA
- Spring Web
- PostgreSQL Driver

**API Endpoints:**

| Method | Endpoint | Descrizione | Request Body | Response |
|--------|----------|-------------|--------------|----------|
| GET | `/api/breeds` | Lista tutte le razze | - | `[{id, name, description}]` |
| GET | `/api/breeds/{name}` | Dettagli razza | - | `{id, name, description, temperament, origin}` |
| POST | `/api/breeds` | Crea nuova razza | `{name, description, temperament, origin}` | `{id, ...}` |
| GET | `/api/searches` | Storico ricerche | - | `[{id, breedName, timestamp}]` |
| POST | `/api/searches` | Registra ricerca | `{breedName}` | `{id, ...}` |

**Porta:** 8081

**Docker:**
- Base image: `eclipse-temurin:17-jre-alpine`
- Multi-stage build (Maven build + runtime)
- Exposed port: 8081

**Configurazione:**
```yaml
spring:
  datasource:
    url: jdbc:postgresql://postgresql:5432/catinfo
    username: ${DB_USER}
    password: ${DB_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
```

---

### 2.3 Database (PostgreSQL)
**Deployment:** Helm Chart ufficiale o Kubernetes Operator

**Opzioni:**
1. **Helm Chart Bitnami:** `bitnami/postgresql`
2. **CloudNativePG Operator:** Operator per PostgreSQL cloud-native

**Configurazione minima:**
- Database: `catinfo`
- User: `catinfo_user`
- Password: da Secret Kubernetes
- Storage: PersistentVolume 1Gi
- Replica: 1 (single instance)

**Schema Database:**

```sql
-- Table: breeds
CREATE TABLE breeds (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL,
    description TEXT,
    temperament VARCHAR(255),
    origin VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Table: searches
CREATE TABLE searches (
    id SERIAL PRIMARY KEY,
    breed_name VARCHAR(100) NOT NULL,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Dati di esempio
INSERT INTO breeds (name, description, temperament, origin) VALUES
('Persiano', 'Gatto a pelo lungo con muso schiacciato, elegante e tranquillo.', 'Calmo, affettuoso, docile', 'Persia (Iran)'),
('Siamese', 'Gatto snello con occhi blu intensi e pelo color point.', 'Vocale, socievole, intelligente', 'Thailandia'),
('Maine Coon', 'Una delle razze più grandi, pelo semi-lungo e coda folta.', 'Amichevole, giocoso, adattabile', 'Stati Uniti'),
('Bengala', 'Gatto dal mantello maculato che ricorda un leopardo.', 'Energico, curioso, atletico', 'Stati Uniti'),
('Ragdoll', 'Gatto di taglia grande che si rilassa completamente quando sollevato.', 'Docile, affettuoso, tranquillo', 'Stati Uniti');
```

---

## 3. Infrastructure as Code

### 3.1 Terraform
**Path:** `/terraform`

**Obiettivo:** Provisioning di un cluster Kubernetes downstream su Rancher

**Provider:** Rancher2 Provider

**Risorse da creare:**
1. Cluster downstream (1 nodo custom)
2. Node registration command
3. Kubeconfig output

**File structure:**
```
terraform/
├── main.tf
├── variables.tf
├── outputs.tf
├── terraform.tfvars.example
└── README.md
```

**Requisiti implementativi:**
- Configurare provider Rancher2 puntando a `rancher-240.local`
- Creare un cluster custom con RKE2
- Configurare un machine pool per 1 nodo (control-plane + etcd + worker)
- Generare comando di registrazione per aggiungere il nodo custom
- Output del kubeconfig per accesso al cluster
- Parametrizzare token, versione Kubernetes e configurazioni

**Note importanti:**
- Il cluster è di tipo **custom** (non provisioned automaticamente)
- Il nodo VM esiste già, va solo registrato al cluster
- Usare `insecure = true` per ambiente di sviluppo locale
- Salvare token Rancher in variabile o tfvars (non hard-coded)

---

### 3.2 Helm Chart
**Path:** `/helm/catinfo`

**Obiettivo:** Creare un Helm chart che pacchettizzi frontend e backend, utilizzando PostgreSQL tramite chart/operator esterno.

**Requisiti:**
- Chart.yaml con metadata e dependency PostgreSQL (Bitnami o CloudNativePG)
- values.yaml parametrizzato per configurare immagini, repliche, risorse
- Templates per:
  - Deployment frontend e backend
  - Service per frontend e backend
  - ConfigMap per configurazioni non sensibili
  - Secret per credenziali database
  - (Opzionale) Ingress per esposizione esterna
- Helpers in _helpers.tpl per riutilizzo logica
- NOTES.txt con istruzioni post-installazione

**Note importanti:**
- Parametrizzare tutto ciò che può variare tra ambienti (immagini, repliche, risorse)
- Usare templating Helm per nomi dinamici e labels
- Implementare probes (liveness/readiness) nei Deployment
- Gestire credenziali database tramite Secret Kubernetes
- Dependency PostgreSQL deve essere configurabile (enabled/disabled)

---

## 4. Struttura Repository GitHub

```
catinfo-project/
├── README.md                          # Documentazione principale
├── ARCHITECTURE.md                    # Documento architetturale
├── .gitignore
├── docker-compose.yml                 # Per sviluppo locale (opzionale)
│
├── frontend/
│   ├── Dockerfile
│   ├── pom.xml
│   └── src/
│       └── main/
│           ├── java/
│           │   └── com/company/catinfo/frontend/
│           │       └── FrontendApplication.java
│           └── resources/
│               ├── application.yml
│               └── static/
│                   ├── index.html
│                   ├── style.css
│                   └── app.js
│
├── backend/
│   ├── Dockerfile
│   ├── pom.xml
│   └── src/
│       └── main/
│           ├── java/
│           │   └── com/company/catinfo/backend/
│           │       ├── BackendApplication.java
│           │       ├── controller/
│           │       │   ├── BreedController.java
│           │       │   └── SearchController.java
│           │       ├── model/
│           │       │   ├── Breed.java
│           │       │   └── Search.java
│           │       ├── repository/
│           │       │   ├── BreedRepository.java
│           │       │   └── SearchRepository.java
│           │       └── service/
│           │           └── BreedService.java
│           └── resources/
│               ├── application.yml
│               └── data.sql (optional init script)
│
├── terraform/
│   ├── README.md
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── terraform.tfvars.example
│
├── helm/
│   └── catinfo/
│       ├── Chart.yaml
│       ├── values.yaml
│       ├── values-dev.yaml
│       ├── values-prod.yaml
│       ├── README.md
│       └── templates/
│           ├── _helpers.tpl
│           ├── frontend-deployment.yaml
│           ├── frontend-service.yaml
│           ├── backend-deployment.yaml
│           ├── backend-service.yaml
│           ├── configmap.yaml
│           ├── secret.yaml
│           └── NOTES.txt
│
└── docs/
    ├── images/
    │   └── architecture-diagram.png
    └── deployment-guide.md
```

---

## 5. Workflow di Sviluppo e Deploy

### 5.1 Fase 1: Sviluppo Locale (Giorno 1)
1. **Setup repository:**
   - Clone repository
   - Creazione struttura base

2. **Sviluppo Backend:**
   - Setup progetto Spring Boot
   - Implementazione entities e repositories
   - Creazione REST controllers
   - Test con H2 in-memory o PostgreSQL locale

3. **Sviluppo Frontend:**
   - Setup progetto Spring Boot statico
   - Creazione interfaccia HTML
   - Implementazione chiamate API con JavaScript
   - Test comunicazione con backend

4. **Containerizzazione:**
   - Creazione Dockerfile per frontend
   - Creazione Dockerfile per backend
   - Build immagini Docker
   - Test con docker-compose (opzionale)

### 5.2 Fase 2: Infrastructure as Code (Giorno 2 mattina)
1. **Terraform:**
   - Configurazione provider Rancher
   - Creazione cluster downstream
   - Registrazione nodo custom
   - Export kubeconfig

2. **Verifica cluster:**
   ```bash
   export KUBECONFIG=./terraform/kubeconfig.yaml
   kubectl get nodes
   kubectl get pods -A
   ```

### 5.3 Fase 3: Packaging e Deploy (Giorno 2 pomeriggio - Giorno 3)
1. **Creazione Helm Chart:**
   - Inizializzazione chart
   - Configurazione templates
   - Setup PostgreSQL dependency
   - Configurazione values

2. **Push immagini Docker:**
   ```bash
   docker tag catinfo-frontend:latest registry.example.com/catinfo-frontend:1.0.0
   docker tag catinfo-backend:latest registry.example.com/catinfo-backend:1.0.0
   docker push registry.example.com/catinfo-frontend:1.0.0
   docker push registry.example.com/catinfo-backend:1.0.0
   ```

3. **Deploy su Kubernetes:**
   ```bash
   # Aggiunta repo Bitnami per PostgreSQL
   helm repo add bitnami https://charts.bitnami.com/bitnami
   helm repo update
   
   # Deploy applicazione
   helm install catinfo ./helm/catinfo \
     --namespace catinfo \
     --create-namespace \
     -f helm/catinfo/values-dev.yaml
   
   # Verifica deployment
   kubectl get all -n catinfo
   kubectl logs -n catinfo -l app=catinfo-backend
   ```

4. **Testing:**
   - Port-forward per accesso locale
   - Test funzionalità complete
   - Verifica persistenza dati

### 5.4 Fase 4: Documentazione (Giorno 3)
1. **README.md principale:**
   - Overview progetto
   - Prerequisites
   - Quick start
   - Deployment instructions

2. **ARCHITECTURE.md:**
   - Diagramma architettura
   - Descrizione componenti
   - Flusso dati
   - Decision log

3. **Deployment guide:**
   - Step-by-step deployment
   - Troubleshooting
   - Configurazioni avanzate

---

## 6. Requisiti Tecnici Specifici

### 6.1 Docker
- Dockerfile multi-stage per ottimizzazione dimensioni
- Health checks nei container
- Non eseguire container come root
- .dockerignore appropriato

### 6.2 Kubernetes
- Resource limits e requests definiti
- Liveness e readiness probes
- ConfigMaps per configurazione
- Secrets per credenziali
- Labels e selectors corretti

### 6.3 Helm
- Templating corretto con values
- Helpers in _helpers.tpl
- NOTES.txt con informazioni post-install
- Dependency management per PostgreSQL

### 6.4 Terraform
- Variabili parametrizzate
- Output significativi
- State management locale (per semplicità)
- Documentazione inline

---

## 7. Testing e Validazione

### 7.1 Checklist Pre-Deploy
- [ ] Le immagini Docker buildano correttamente
- [ ] Backend si connette al database locale
- [ ] Frontend comunica con backend
- [ ] Helm chart passa `helm lint`
- [ ] Terraform plan esegue senza errori

### 7.2 Checklist Post-Deploy
- [ ] Tutti i pod sono in stato Running
- [ ] Backend può connettersi a PostgreSQL
- [ ] Frontend è accessibile via Service
- [ ] API backend restituiscono dati corretti
- [ ] Le ricerche vengono salvate nel database
- [ ] Logs non mostrano errori critici

### 7.3 Test Funzionali
1. **Test ricerca razza esistente:**
   - Input: "Persiano"
   - Output: Descrizione completa della razza

2. **Test ricerca razza non esistente:**
   - Input: "RazzaInesistente"
   - Output: Messaggio appropriato

3. **Test storico ricerche:**
   - Eseguire 3-5 ricerche
   - Verificare visualizzazione storico

4. **Test persistenza:**
   - Riavviare pod backend
   - Verificare che i dati persistano

---

## 8. Documentazione da Produrre

### 8.1 README.md
```markdown
# CatInfo - Cat Breeds Information System

## Overview
Applicazione web per la consultazione di informazioni sulle razze di gatti.

## Architecture
[Diagramma architettura]

## Prerequisites
- Docker 24+
- Kubernetes cluster (Rancher)
- Helm 3.x
- Terraform 1.x
- Java 17+
- Maven 3.8+

## Quick Start
[Comandi per build e deploy]

## Development
[Istruzioni per setup ambiente locale]

## Deployment
[Guida passo-passo deployment su K8s]

## API Documentation
[Elenco endpoints con esempi]

## Troubleshooting
[Problemi comuni e soluzioni]
```

### 8.2 ARCHITECTURE.md
Deve contenere:
1. **Context Diagram:** Sistema nel suo ambiente
2. **Container Diagram:** Componenti principali e comunicazioni
3. **Component Diagram:** Struttura interna backend
4. **Deployment Diagram:** Topologia Kubernetes
5. **Sequence Diagram:** Flusso ricerca razza
6. **Technology Stack:** Giustificazione scelte tecnologiche
7. **Design Decisions:**
   - Perché microservizi separati per frontend/backend
   - Scelta Spring Boot per frontend statico
   - Gestione stato con PostgreSQL
   - Pattern di comunicazione REST

---

## 9. Criteri di Valutazione

L'esercizio sarà valutato secondo i seguenti criteri:

### 9.1 Funzionalità (30%)
- [ ] Applicazione funzionante end-to-end
- [ ] Tutte le API implementate
- [ ] Frontend interattivo e usabile
- [ ] Persistenza dati corretta

### 9.2 Containerizzazione (20%)
- [ ] Dockerfile ottimizzati
- [ ] Immagini buildano correttamente
- [ ] Best practices Docker rispettate
- [ ] Health checks implementati

### 9.3 Kubernetes/Helm (25%)
- [ ] Deployment completo funzionante
- [ ] Helm chart ben strutturato
- [ ] Resource management appropriato
- [ ] Probes configurati correttamente

### 9.4 Infrastructure as Code (15%)
- [ ] Terraform provisiona cluster correttamente
- [ ] Codice pulito e parametrizzato
- [ ] Kubeconfig generato utilizzabile

### 9.5 Documentazione (10%)
- [ ] README completo e chiaro
- [ ] ARCHITECTURE.md dettagliato
- [ ] Diagrammi presenti e leggibili
- [ ] Istruzioni deployment funzionanti

---

## 10. Suggerimenti e Best Practices

### 10.1 Sviluppo
- Usare Spring Initializr per bootstrap progetti
- Testare ogni componente in isolamento prima dell'integrazione
- Usare variabili d'ambiente per configurazioni sensibili
- Committare frequentemente con messaggi descrittivi

### 10.2 Docker
- Usare .dockerignore per escludere file inutili
- Multi-stage build per ridurre dimensione immagini
- Non includere secrets nelle immagini
- Taggare le immagini con versioni semantiche

### 10.3 Kubernetes
- Usare namespace dedicato per l'applicazione
- Definire sempre resource limits
- Usare ConfigMap per configurazione non sensibile
- Usare Secret per password e token

### 10.4 Helm
- Parametrizzare tutto ciò che può variare tra ambienti
- Usare helper functions per evitare ripetizioni
- Testare chart con `helm template` prima del deploy
- Documentare values.yaml con commenti

### 10.5 Git
- Struttura commit logica e atomic
- Branch strategy: main + feature branches
- .gitignore per escludere file temporanei
- Non committare credenziali o secrets

---

## 11. Troubleshooting Comune

### 11.1 Database Connection Failed
**Problema:** Backend non riesce a connettersi a PostgreSQL

**Soluzioni:**
- Verificare che il service PostgreSQL sia running: `kubectl get svc -n catinfo`
- Controllare secret con credenziali: `kubectl get secret -n catinfo`
- Verificare logs backend: `kubectl logs -n catinfo -l app=catinfo-backend`
- Controllare network policies o firewall rules

### 11.2 Frontend Cannot Reach Backend
**Problema:** JavaScript fetch fallisce

**Soluzioni:**
- Verificare URL backend configurato in app.js
- Controllare che backend service sia accessibile
- Verificare CORS configuration in Spring Boot
- Testare backend con curl da dentro cluster

### 11.3 Terraform Rancher Connection Issues
**Problema:** Terraform non riesce a comunicare con Rancher

**Soluzioni:**
- Verificare token Rancher valido
- Controllare URL Rancher corretto
- Verificare certificato SSL (usare insecure=true per dev)
- Testare connessione con curl/API diretta

### 11.4 Helm Dependency Issues
**Problema:** PostgreSQL chart non viene scaricato

**Soluzioni:**
- Eseguire `helm dependency update` nella directory chart
- Verificare connessione internet
- Controllare repo Bitnami aggiunto: `helm repo list`
- Verificare sintassi Chart.yaml dependencies

---

## 12. Risorse Utili

### Documentazione Ufficiale
- [Spring Boot](https://spring.io/projects/spring-boot)
- [Docker Documentation](https://docs.docker.com/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Helm Documentation](https://helm.sh/docs/)
- [Terraform Rancher2 Provider](https://registry.terraform.io/providers/rancher/rancher2/latest/docs)

### Tutorial e Guide
- [Spring Boot REST API Tutorial](https://spring.io/guides/gs/rest-service/)
- [Kubernetes Best Practices](https://kubernetes.io/docs/concepts/configuration/overview/)
- [Helm Chart Development Guide](https://helm.sh/docs/chart_template_guide/)

### Tools
- [Spring Initializr](https://start.spring.io/)
- [Docker Hub](https://hub.docker.com/)
- [Bitnami Helm Charts](https://github.com/bitnami/charts)

---

## 13. Deliverables Finali

Al termine dell'esercizio, la repository GitHub deve contenere:

1. **Codice sorgente completo:**
   - Frontend Spring Boot con static resources
   - Backend Spring Boot con REST API
   - Dockerfile per entrambi i servizi

2. **Infrastructure as Code:**
   - Configurazione Terraform per cluster Rancher
   - Helm chart completo con templates

3. **Documentazione:**
   - README.md principale
   - ARCHITECTURE.md con diagrammi
   - Guide di deployment

4. **Scripts (opzionali ma consigliati):**
   - build.sh - Build immagini Docker
   - deploy.sh - Deploy automatico su K8s
   - cleanup.sh - Rimozione risorse

---

## Conclusioni

Questo esercizio integra tutti gli elementi del path formativo, dalla programmazione Java all'orchestrazione Kubernetes, passando per containerizzazione e Infrastructure as Code. L'obiettivo è consolidare le competenze acquisite attraverso un progetto pratico e realistico.

**Buon lavoro!** 🚀🐱

Per qualsiasi dubbio o chiarimento, riferirsi al team di formazione.
