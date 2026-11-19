# Candidate Submission - DevOps Technical Challenge (60 minutes)

**Candidate Name:** Manuel Bajescu  
**Date & Time:** 19.11.2026  

---

## Task 1: Code Review & Anti-Pattern Analysis (15 min)

> **Requirement:** Identify a minimum of 10 distinct issues across `Dockerfile`, `deployment.yaml`, `values.yaml`, and `pipeline.yaml` (bonus: `argocd/application.yaml`).  
> For each issue, provide:
> 1. **Why it is a problem** (*De ce este o problemă*)
> 2. **Impact** (*Care este impactul*)
> 3. **Remediation** (*Cum ai remedia-o*)

---

### Issue #1
- **File & Line(s):** ./Dockerfile Line 5
- **Category:** `[Security]`
- **De ce este o problemă:** DATABASE_URL connection strig la baza cu parola, user admin si baza de date scrise in clar. 
- **Care este impactul:** poate fi citita/folosita de persoane neautorizate
- **Cum ai remedia-o:** definire de Environment secrets in repository si apel in cod la acest secret; exemplu: "${{secrets.DATABASE_URL}}" 

---

### Issue #2
- **File & Line(s):** ./Dockerfile Line 6
- **Category:** `[Security]`
- **De ce este o problemă:** JWT_SECRET="unhashed-jwt-production-secret-token" secret expus in cod
- **Care este impactul:** si in acest caz este nevoie sa fie folosit secrets din repository si apel in cod la variabile de environment secrets

- **Cum ai remedia-o:** definire de secret in repository sau mai bine in environment secrets si apel in cod la acest secret; exemplu: "${{ secrets.JWT_SECRET}}" 
---

### Issue #3
- **File & Line(s):** ./Dockerfile Line 7
- **Category:** `[CI/CD | GitOps]`
- **De ce este o problemă:** ENV ENVIRONMENT="production". environment hardcodat.
- **Care este impactul:** pentru CI/CD este nevoie sa fie defint mediu in Git, in felul acesta codu poate folosit fara modificari pentru toate mediile din CI/CD
- **Cum ai remedia-o:** as defini mediu in repository si as face apel la el in cod

---

### Issue #4
- **File & Line(s):** ./Dokerfile Line 10
- **Category:** `[Security]`
- **De ce este o problemă:** download and executa imediat cu userul de root de la un vendor untrusted; avand in vedere ca vorbim de productie este o problema
- **Care este impactul:** fisierul poate contine comenzi neautorizate care pot compromite productia
- **Cum ai remedia-o:** se copiaza fisierul in repo si apoi se valideaza daca este ok, dupa care se ruleaza separat, teortic in prod nu trebuie sa se descarca la build un sh.  Deasemena trebuie creat un user si nu sa ruleze cu root. 


---

### Issue #5
- **File & Line(s):** ./Dockerfile Line 2
- **Category:** `[Reliability]`
- **De ce este o problemă:** python:latest nu se poate reproduce, nu este o versiune stabila
- **Care este impactul:** daca se schimba versiunea imaginii poate da eroare build-ul. 
- **Cum ai remedia-o:** teoretic se trece versiunea agreata si pe care este testata aplicatia; 

---

### Issue #6
- **File & Line(s):** ./Dokerfile Line 25
- **Category:** `[Security | Reliability | Kubernetes Best Practice | CI/CD | GitOps]`
- **De ce este o problemă:** flag-ul --reload este pentru dezvoltare si nu pentru productie 
- **Care este impactul:** adauga overhead si productia devine mai putin predictibila
- **Cum ai remedia-o:** trebuie eliminat acest flag din productie

---

### Issue #7 
- **File & Line(s):** ./Dokerfile Line 16
- **Category:** `[Reliability]`
- **De ce este o problemă:** Contine librarii/module necesare pentru compilarea pachetelor la build nu si cand ruleaza (gcc, build-esential, libpq-dev); pip nu verifica ce instaleaza;
- **Care este impactul:** functioneaza insa sunt mai multe probleme: incarca cu cateva 100MB imaginea, ofera posibiltate de atac 
- **Cum ai remedia-o:** multi-stage build, se compileaza intr-o prima faza apoi imaginea finala primeste rezultatul; requirements.txt cu hash-uri (--require-hashes)

# ---- build: aici avem compilatorul ----
FROM python:3.13-slim AS builder

RUN apt-get update \
 && apt-get install -y --no-install-recommends build-essential libpq-dev \
 && rm -rf /var/lib/apt/lists/*

RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH" \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

COPY requirements.txt .
RUN pip install --require-hashes -r requirements.txt

# ---- runtime: doar ce trebuie ca să ruleze ----
FROM python:3.13-slim

RUN apt-get update \
 && apt-get install -y --no-install-recommends libpq5 \
 && rm -rf /var/lib/apt/lists/* \
 && useradd --create-home --uid 1000 app

ENV PATH="/opt/venv/bin:$PATH" \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

COPY --from=builder /opt/venv /opt/venv

WORKDIR /app
COPY --chown=app:app . .

USER app
EXPOSE 8000

---

### Issue #8
- **File & Line(s):** ./deployment.yaml Lines 43
- **Category:** `[Security]`
- **De ce este o problemă:** credetiale/parola si date de conectare scrise in clar in manifest.
- **Care este impactul:** pot fi vazute de persoane neautorizate si folosite in alte scopuri
- **Cum ai remedia-o:** Datele sensibile se pun in secret Kubernetes sau intr-o aplicatie externa de secrete si folsita cu secretKeyRef. User nu trebuie sa fie admin si un user dedicat cu permisiuni minime necesare. Limitarea prin RBAC la secretul definit.  

---

### Issue #9
- **File & Line(s):** ./deployment.yaml Line 20 
- **Category:** `[Kubernetes Best Practice]`
- **De ce este o problemă:** eticheta tier de la template nu se potriveste cu tier-ul din selector, spec.selector.matchLabels cere tier: api-backend iar in template tier: wrong-backend. 
- **Care este impactul:** API server din Kubernetes respinge fisierul deployment. 
- **Cum ai remedia-o:** este nevoie sa fie acceasi valoare a etichetei, api-backend 

---

### Issue #10
- **File & Line(s):** ./deployment.yaml Lines 22-26
- **Category:** `[Kubernetes Best Practice | CI/CD | GitOps]`
- **De ce este o problemă:** security context trebuie setat la nivelul deploymentului si nu la nivelul template-ului
- **Care este impactul:** API server respinge fisierul 
- **Cum ai remedia-o:** privileged, allowPriviledgeEscalation, readOnlyFilesystem si rootAsNonRoot se seteaza la spec.securityContext de deployment 

---

### Issue #11
- **File & Line(s):** ./values.yaml Lines 17
- **Category:** `[Security]`
- **De ce este o problemă:** parola scrisa in clar; user cu permisiuni mai multe decat necesar
- **Care este impactul:** credentialele pot fi vazut de catre cineva neautorizat si folosite neadecvat 
- **Cum ai remedia-o:** crearea de secret Kubernetes sau intr-o aplicatie externa de secrete si folsita cu secretKeyRef  

---

### Issue #12
- **File & Line(s):** ./values.yaml Lines 52
- **Category:** `[Kubernetes]`
- **De ce este o problemă:** targetCPUUtilizationPercentage: 1 target nerealist
- **Care este impactul:** va declansa scalarea instant 
- **Cum ai remedia-o:** setarea undeva la 60-70% si limitarea la replici, maxReplicas  

---

### Issue #13
- **File & Line(s):** ./values.yaml Lines 21-26
- **Category:** `[Kubernetes]`
- **De ce este o problemă:** simultan serviciu LoadBalancer si ingress active duc al o expunere foarte larga
- **Care este impactul:** expunerea larga sursa de vulnerabilitati 
- **Cum ai remedia-o:**  daca traficul trece prin Ingress, serviciul poate fi ClusterIP. Host-ul * expune ruta pentru orice domeniniu directionat catre controller. 

---

### Issue #14
- **File & Line(s):** ./values.yaml Line 30
- **Category:** `[Kubernetes]`
- **De ce este o problemă:** ssl-redirect: false si tls: [] permit trafic necriptat
- **Care este impactul:** trafic necriptat, exista posibilitatea de a capta informatia de catre persoane neautorizate 
- **Cum ai remedia-o:**  activare tls si redirectarea http catre https   

---

### Issue #15
- **File & Line(s):** ./pipeline.yaml Line 19
- **Category:** `[CI/CD]`
- **De ce este o problemă:** token scris in clar in cod
- **Care este impactul:** poate fi captat si utilizat de persoane neautorizate 
- **Cum ai remedia-o:**  setare in secrets la repository si apelarea ca variabila 
password: ${{ secrets.DOCKERHUB_TOKEN }}

---

### Issue #16
- **File & Line(s):** ./pipeline.yaml Line 7
- **Category:** `[CI/CD]`
- **De ce este o problemă:** permissions: write-all permisiuni/acces prea largi 
- **Care este impactul:** recomandarea este ca accesul acordat sa fie in concordanta cu nevoia. 
- **Cum ai remedia-o:**  se acorda permisuni minime gradual de la content: read catre alte permisiuni necesare

---

### Issue #17
- **File & Line(s):** ./pipeline.yaml Line 15
- **Category:** `[CI/CD]`
- **De ce este o problemă:** actions/checkout@master nu este setat la o versiune stabila. 
- **Care este impactul:** se schimba versiunea si poate da fail; 
- **Cum ai remedia-o:**  este nevoie de versiune explicita actions/checkout@v4 

---

### Issue #18
- **File & Line(s):** ./pipeline.yaml Line 7
- **Category:** `[CI/CD]`
- **De ce este o problemă:** continue-on-error: true duce la rularea pipeline chiar daca testele esueaza. 
- **Care este impactul:** se duce in productie cod cu probleme 
- **Cum ai remedia-o:**  trebuie eliminat ca astfel la esuarea testeleor pipeline-ul sa se opreasca. 

---

### Issue #19
- **File & Line(s):** ./pipeline.yaml Line 39
- **Category:** `[CI/CD]`
- **De ce este o problemă:** se apeleaza kubectl dar runner-ul nu are configurat context Kubernets si credentiale valide. 
- **Care este impactul:** nu va functiona 
- **Cum ai remedia-o:**  trebuie configurat accesul in cluster cu OIDC sau GitHub secrets  

---

### Issue #20
- **File & Line(s):** ./pipeline.yaml Line 39
- **Category:** `[CI/CD]`
- **De ce este o problemă:** --validate=false dezactiveaza verifciarile fisierelor si manifestelor  
- **Care este impactul:** daca fisierel nu sunt valide, pipeline-ul esueaza 
- **Cum ai remedia-o:**  trebuie eliminat flag-ul si remediat orice eroare apare.   

### Bonus Issues (Optional)
*(Add any additional findings here)*

- **File & Line(s):** ./argocd/application.yaml Line 26
- **Category:** `[CI/CD]`
- **De ce este o problemă:** --validate=false dezactiveaza verificarile fisierelor si manifestelor  
- **Care este impactul:** daca fisierel nu sunt valide, pipeline-ul esueaza 
- **Cum ai remedia-o:**  trebuie eliminat flag-ul si remediat orice eroare apare.  

- **File & Line(s):** ./argocd/application.yaml Line 22-23
- **Category:** `[CI/CD]`
- **De ce este o problemă:** prune: false si selfHeal: false  
- **Care este impactul:** resursele șterse din Git nu vor fi șterse automat din cluster; schimbările făcute direct în cluster nu vor fi corectate automat la următorul sync 
- **Cum ai remedia-o:** as folosi prune: true si selfHeal: true.  


