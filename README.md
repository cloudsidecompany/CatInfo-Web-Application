
# CatInfo Web Application

**CatInfo** è un’applicazione web per la gestione e la visualizzazione di informazioni sulle razze di gatti.
Il progetto è composto da un **backend Spring Boot**, un **frontend web statico** e un’infrastruttura **containerizzata con Docker**, **deployata su Kubernetes tramite Terraform e Rancher**.

---

##  Architettura del progetto

```
catinfo-project/
├── backend/                  # Applicazione Spring Boot (API REST)
├── frontend/                 # Interfaccia utente statica (HTML, CSS, JS)
├── terraform/                # Configurazione Terraform per Rancher/Kubernetes
├── docker-compose.yml        # Avvio locale con Docker
└── README.md
```

---

##  Tecnologie utilizzate

* **Java 21 / Spring Boot 3.5**
* **PostgreSQL**
* **HTML / JavaScript (Fetch API)**
* **Docker / Docker Hub**
* **Kubernetes**
* **Terraform + Rancher2 Provider**

---

##  Avvio in locale (Docker Compose)

Per eseguire il progetto in locale:

```bash
cd catinfo-project
docker-compose up --build
```

L’applicazione sarà disponibile su:

* **Frontend:** [http://localhost:8085](http://localhost:8085)
* **Backend:** [http://localhost:8080/api/breeds](http://localhost:8080/api/breeds)

---

##  Deploy su Kubernetes

###  Costruisci e pubblica le immagini Docker

Dalla root dei moduli `backend` e `frontend`:

```bash
# Backend
docker build -t selenepuzio/catinfo-backend:v4 .
docker push selenepuzio/catinfo-backend:v4

# Frontend
docker build -t selenepuzio/catinfo-frontend:v4 .
docker push selenepuzio/catinfo-frontend:v4
```

> 🔁 Ogni nuova modifica al codice comporta **un nuovo tag** (es. `v5`, `v6`, ecc.).
> Il tag `latest` è sconsigliato perché può creare conflitti con Terraform/Kubernetes.

---

###  Aggiorna le immagini in Kubernetes

```bash
kubectl set image deployment/catinfo-backend catinfo-backend=selenepuzio/catinfo-backend:v4 -n catinfo
kubectl set image deployment/catinfo-frontend catinfo-frontend=selenepuzio/catinfo-frontend:v4 -n catinfo
kubectl rollout status deployment/catinfo-backend -n catinfo
kubectl rollout status deployment/catinfo-frontend -n catinfo
```

---

### Verifica lo stato dei Pod

```bash
kubectl get pods -n catinfo -o wide
kubectl logs <nome-pod> -n catinfo
```

---

## Deploy con Terraform + Rancher

Nella cartella `terraform/`:

```bash
cd terraform
terraform init -upgrade
terraform plan
terraform apply
```

Assicurati che nel file `terraform.tfvars` siano presenti:

```hcl
rancher_url      = "https://<IP_RANCHER>"
rancher_token    = "token-xxxx:yyyy"
cluster_name     = "catinfo-cluster"
node_template_id = "nt-abc123"
```

---

##  Configurazione CORS (Backend)

Il backend consente richieste dal frontend tramite un filtro CORS globale:

```java
@Configuration
public class CorsConfig {
    @Bean
    public CorsFilter corsFilter() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("*"));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("*"));
        config.setAllowCredentials(false);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return new CorsFilter(source);
    }
}
```

---

## Troubleshooting

| Problema                                         | Causa                                  | Soluzione                                        |
| ------------------------------------------------ | -------------------------------------- | ------------------------------------------------ |
| `ImagePullBackOff`                               | Tag immagine non trovato su Docker Hub | Verifica `docker push` e `kubectl describe pod`  |
| `CORS policy: No 'Access-Control-Allow-Origin'`  | Configurazione CORS mancante           | Verifica `CorsConfig.java` nel backend           |
| `duplicate key value violates unique constraint` | Dati duplicati nel DB                  | Pulisci la tabella o disattiva script `data.sql` |
| `terraform apply` bloccato                       | Rancher non raggiungibile              | Verifica `curl -k https://<rancher_ip>/v3`       |

---

## API Principali

| Endpoint             | Metodo | Descrizione                             |
| -------------------- | ------ | --------------------------------------- |
| `/api/breeds`        | GET    | Lista di tutte le razze di gatti        |
| `/api/breeds/{name}` | GET    | Dettagli di una razza specifica         |
| `/api/breeds`        | POST   | Aggiunge una nuova razza (se abilitato) |

---



Vuoi che lo adatti anche per **GitHub** (con badge, emoji e sezioni colorate tipo “Getting Started”, “Contributing”, ecc.) oppure vuoi mantenerlo **più semplice e pulito per consegna interna**?
