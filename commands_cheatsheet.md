Cheatsheet


## 1. Mentalni Model Zadatka

```text
browser/client
    |
    v
nginx container :80/:443
    |
    v
java app container :8080/:9000
    |
    v
postgres container :5432
```

Cilj nije samo "popraviti", nego pokazati metodu:

1. Mapirati sto postoji.
2. Vidjeti sto radi, a sto pada.
3. Procitati logove.
4. Provjeriti konfiguraciju.
5. Testirati mrezu izmedu containera.
6. Napraviti najmanju potrebnu promjenu.
7. Validirati kroz `curl`, logove i status servisa.
8. Objasniti uzrok i predloziti poboljsanja.

Korisna recenica:

```text
Prvo cu mapirati servise i trenutno stanje, zatim pogledati logove servisa koji padaju. Nakon toga cu provjeriti env konfiguraciju, Docker network i connectivity izmedu nginx-a, aplikacije i Postgresa.
```

## 2. Brzi Redoslijed Debugiranja

```bash
pwd
ls -la
find . -maxdepth 3 -type f | sort
```

Trazi ove fileove:

```text
docker-compose.yml
compose.yml
Dockerfile
nginx.conf
default.conf
application.properties
application.yml
.env
README.md
pom.xml
build.gradle
```

Ako postoji Docker Compose:

```bash
docker compose config
docker compose ps
docker compose logs
docker compose logs -f
docker compose up -d
```

Ako je stariji setup:

```bash
docker-compose config
docker-compose ps
docker-compose logs
docker-compose up -d
```

## 3. Docker Osnove

### Status containera

```bash
docker ps
docker ps -a
docker stats
```

Sto gledati:

```text
STATUS: Up, Exited, Restarting, unhealthy
PORTS: je li servis izlozen na host
NAMES: imena containera za logs/exec
```

### Logovi

```bash
docker logs <container>
docker logs -f <container>
docker logs --tail 100 <container>
```

Za Compose:

```bash
docker compose logs <service>
docker compose logs -f <service>
docker compose logs --tail 100 <service>
```

### Ulazak u container

```bash
docker exec -it <container> sh
docker exec -it <container> bash
```

Ako `bash` ne postoji, koristi `sh`.

### Inspect

```bash
docker inspect <container>
docker inspect <container> | less
```

Korisno za:

```text
Env varijable
Network
IP adrese
Mountove/volumene
Healthcheck
Restart policy
```

### Portovi

```bash
docker port <container>
lsof -i :80
lsof -i :8080
lsof -i :5432
```

Compose razlika:

```yaml
ports:
  - "8080:8080"   # dostupno hostu

expose:
  - "8080"        # dostupno samo drugim containerima na mrezi
```

## 4. Docker Compose

### Validacija konfiguracije

```bash
docker compose config
```

Ako ovo padne, problem je u YAML-u, env varijablama ili putanjama.

### Pokretanje i restart

```bash
docker compose up -d
docker compose down
docker compose restart
docker compose restart <service>
```

### Rebuild

```bash
docker compose build
docker compose build --no-cache
docker compose up -d --build
```

### Status i logovi

```bash
docker compose ps
docker compose logs
docker compose logs -f app
docker compose logs -f nginx
docker compose logs -f postgres
```

### Tipicni Compose primjer

```yaml
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - app

  app:
    build: .
    environment:
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: appdb
      DB_USER: appuser
      DB_PASSWORD: apppass
    depends_on:
      - postgres

  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: apppass
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

## 5. Docker Networking

Unutar Docker mreze containeri se zovu po service nameu.

Dobro:

```text
postgres
app
nginx
```

Cesto krivo:

```text
localhost
127.0.0.1
```

Bitno pravilo:

```text
localhost unutar containera znaci taj container, ne host i ne drugi container.
```

### Provjera mreza

```bash
docker network ls
docker network inspect <network>
```

Za Compose mrezu:

```bash
docker compose ps
docker network ls
```

### Test connectivityja iz containera

```bash
docker exec -it <app-container> sh
```

Unutra:

```bash
getent hosts postgres
nc -vz postgres 5432
curl -v http://app:8080
```

Ako nema `nc` ili `curl`, probaj:

```bash
wget -S -O- http://app:8080
```

Ili privremeni debug container na istoj mrezi:

```bash
docker run --rm -it --network <network-name> alpine sh
apk add --no-cache curl netcat-openbsd bind-tools
```

## 6. Java App Debug

### Sto traziti u logovima

```text
Connection refused
UnknownHostException
FATAL: password authentication failed
database does not exist
relation does not exist
Flyway/Liquibase migration failed
Port already in use
OutOfMemoryError
Permission denied
```

### Spring Boot endpointi

Ako je Spring Boot Actuator ukljucen:

```bash
curl -i http://localhost:8080/actuator/health
curl -i http://localhost:8080/health
curl -i http://localhost:8080/
```

Iz nginx containera:

```bash
curl -i http://app:8080/actuator/health
```

### Tipicne Java env varijable

```text
SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/appdb
SPRING_DATASOURCE_USERNAME=appuser
SPRING_DATASOURCE_PASSWORD=apppass
SERVER_PORT=8080
SPRING_PROFILES_ACTIVE=prod
```

Primjer ispravnog JDBC URL-a u Docker Compose mrezi:

```text
jdbc:postgresql://postgres:5432/appdb
```

Cesto krivo:

```text
jdbc:postgresql://localhost:5432/appdb
```

## 7. Postgres

### Status i logovi

```bash
docker compose logs postgres
docker logs <postgres-container>
```

### Ulazak u Postgres container

```bash
docker exec -it <postgres-container> sh
psql -U <user> -d <database>
```

Ili direktno:

```bash
docker exec -it <postgres-container> psql -U <user> -d <database>
```

### Psql naredbe

```sql
\l
\c database_name
\dt
\d table_name
SELECT now();
SELECT current_database();
SELECT current_user;
```

### Test s hosta

```bash
psql -h localhost -p 5432 -U appuser -d appdb
```

Ako Postgres nije mapiran na host, ovo nece raditi, ali app container moze i dalje doci do njega preko `postgres:5432`.

### Tipicni Postgres problemi

```text
Krivi DB_HOST
Krivi username/password
Baza ne postoji
Aplikacija se digne prije baze
Stari volume ima stare credentiale
Migration nije prosao
Postgres port nije izlozen na host, ali to nije nuzno problem
```

Bitno za volume:

```text
POSTGRES_USER/POSTGRES_PASSWORD se primjenjuju samo kod prvog inicijaliziranja praznog data direktorija. Ako volume vec postoji, promjena env varijabli ne mijenja postojeceg usera/password.
```

## 8. Nginx

### Validacija konfiguracije

```bash
docker exec -it <nginx-container> nginx -t
docker compose exec nginx nginx -t
```

### Reload

```bash
docker exec -it <nginx-container> nginx -s reload
docker compose restart nginx
```

### Logovi

```bash
docker logs <nginx-container>
docker compose logs nginx
```

Unutar containera:

```bash
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log
```

### Minimalni reverse proxy

```nginx
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://app:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Cesti nginx problemi

```text
proxy_pass ide na localhost umjesto app service name
krivi upstream port
nginx nije na istoj Docker mrezi kao app
nginx config nije mountan gdje nginx ocekuje
syntax error u configu
app vraca 404, a nginx radi
nginx vraca 502 jer ne moze do app-a
```

Status kodovi:

```text
502 Bad Gateway: nginx ne moze do upstream aplikacije
404 Not Found: nginx/app radi, ali path nije dobar
500 Internal Server Error: aplikacija je primila request, ali se srusila/logicki pala
```

## 9. Curl Komande

### Osnovno

```bash
curl -i http://localhost
curl -v http://localhost
curl -sS http://localhost
```

### Health endpointi

```bash
curl -i http://localhost/health
curl -i http://localhost/actuator/health
curl -i http://localhost/api/health
```

### Direktno na app ako je port izlozen

```bash
curl -i http://localhost:8080
curl -i http://localhost:8080/actuator/health
```

### Kroz nginx

```bash
curl -i http://localhost
curl -i http://localhost/api
```

### Headeri

```bash
curl -i -H "Host: example.local" http://localhost
```

### POST request

```bash
curl -i -X POST http://localhost/api/test \
  -H "Content-Type: application/json" \
  -d '{"key":"value"}'
```

## 10. Linux Komande

### Procesi, portovi, disk

```bash
ps aux
top
df -h
du -sh *
free -m
```

Na macOS-u `free` ne postoji, ali u Linux containeru postoji ovisno o imageu.

### Portovi

```bash
lsof -i :80
lsof -i :8080
lsof -i :5432
netstat -tulpn
ss -tulpn
```

### Fileovi

```bash
ls -la
cat file
less file
tail -f file
tail -n 100 file
grep -R "DB_HOST" .
rg "DB_HOST|SPRING_DATASOURCE|postgres|proxy_pass"
```

### Permissions

```bash
ls -la
id
whoami
chmod
chown
```

## 11. Git

### Pregled

```bash
git status
git branch
git log --oneline -5
```

### Promjene

```bash
git diff
git diff --stat
git diff <file>
```

### Ako radis fix

```bash
git status
git add <file>
git diff --cached
git commit -m "Fix docker service configuration"
```

Nemoj raditi destructive stvari bez potrebe:

```bash
git reset --hard
git clean -fd
```

Najvažnije Dockerfile stvari za ovaj zadatak:

FROM eclipse-temurin:17-jre
WORKDIR /app
COPY target/app.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
Za Maven build često multi-stage:

FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /src
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

FROM eclipse-temurin:17-jre
WORKDIR /app
COPY --from=build /src/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
Što gledati u Dockerfileu:

Je li base image dobar za Java verziju?
Je li JAR stvarno kopiran na očekivanu lokaciju?
Je li ENTRYPOINT/CMD ispravan?
Je li WORKDIR smislen?
Je li port aplikacije usklađen s Compose/nginx configom?
Radi li app kao root ili non-root user?
Postoji li healthcheck?
Ima li nepotrebno velik image?
Jesu li build i runtime odvojeni?
Debug komande za Dockerfile/image:

docker build -t app-test .
docker run --rm -it app-test
docker run --rm -it app-test sh
docker image ls
docker history app-test
docker inspect app-test
Ako container odmah exit-a:

docker ps -a
docker logs <container>
docker inspect <container>
Tipični Dockerfile problemi:

COPY target/*.jar app.jar ne radi jer JAR nije buildan
kriva Java verzija, npr. app treba Java 17, image ima Java 11
ENTRYPOINT pokazuje na krivi filename
app sluša na 8080, a nginx/compose očekuje 9000
Dockerfile koristi build artifact koji ne postoji
image nema curl/nc pa je teže debugirati connectivity
permissions problem zbog non-root usera
Dakle: Compose sam naglašavao jer je najvjerojatnije “glavno bojno polje” za integraciju app-nginx-postgres, ali Dockerfile je drugi važan sloj. Dobar redoslijed je:

Dockerfile: može li se image ispravno izgraditi i pokrenuti?
Compose: mogu li servisi razgovarati međusobno?
Nginx: može li klijent doći do aplikacije?
Postgres: može li aplikacija do baze?



U pom.xml gledaj prvenstveno stvari koje utječu na build, Java verziju, packaging, dependencyje i runtime ponašanje.

Najvažnije:

<packaging>jar</packaging>
ili:

<packaging>war</packaging>
Ako je war, Dockerfile/Java pokretanje može biti drugačije. Ako je Spring Boot jar, najčešće očekuješ:

java -jar target/something.jar
Java verzija:

<properties>
    <java.version>17</java.version>
</properties>
ili:

<maven.compiler.source>17</maven.compiler.source>
<maven.compiler.target>17</maven.compiler.target>
Ovo mora odgovarati Docker imageu. Ako pom.xml traži Java 17, a Dockerfile koristi Java 11, build ili runtime može pasti.

Spring Boot plugin:

<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
</plugin>
Bitno jer radi executable/fat JAR. Bez njega možeš dobiti JAR koji nema sve dependencyje ili nema dobar manifest.

Build final name:

<build>
    <finalName>app</finalName>
</build>
Ako postoji, output može biti:

target/app.jar
Ako ne postoji, može biti nešto tipa:

target/my-service-1.0.0-SNAPSHOT.jar
To mora odgovarati Dockerfileu:

COPY target/app.jar app.jar
Dependencies za Postgres:

<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
</dependency>
Ako app treba Postgres, a drivera nema, očekuj runtime error.

Spring Data/JPA:

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
ili JDBC:

<artifactId>spring-boot-starter-jdbc</artifactId>
Migration tools:

<artifactId>flyway-core</artifactId>
ili:

<artifactId>liquibase-core</artifactId>
Ako postoje, DB se možda inicijalizira migracijama. Onda kod problema s bazom gledaj migration logove.

Actuator:

<artifactId>spring-boot-starter-actuator</artifactId>
Ako postoji, probaj:

curl http://localhost:8080/actuator/health
Profiles / resource filtering:

<profiles>
i:

<resources>
Ovo može značiti da build ovisi o Maven profilu:

mvn clean package -Pprod
ili:

mvn clean package -DskipTests
Testovi:

<plugin>
    <artifactId>maven-surefire-plugin</artifactId>
</plugin>

mvn test
mvn clean package
mvn clean package -DskipTests
Korisne komande:

mvn -version
mvn dependency:tree
mvn clean test
mvn clean package
mvn clean package -DskipTests
ls -lah target/
Praktični checklist:

[ ] Koja Java verzija?
[ ] Je li packaging jar ili war?
[ ] Koji JAR nastaje u target/?
[ ] Poklapa li se JAR ime s Dockerfile COPY naredbom?
[ ] Postoji li spring-boot-maven-plugin?
[ ] Postoji li Postgres driver?
[ ] Postoji li actuator health endpoint?
[ ] Postoje li Flyway/Liquibase migracije?
[ ] Treba li Maven profil za build?
[ ] Pada li build zbog testova, dependencyja ili Java verzije?



## 12. Tipicni Scenariji i Rjesenja

### App ne moze do Postgresa

Simptomi:

```text
Connection refused
UnknownHostException: localhost/postgres
password authentication failed
```

Provjeri:

```bash
docker compose ps
docker compose logs app
docker compose logs postgres
docker inspect <app-container>
```

Najcesci fix:

```text
DB_HOST=postgres
SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/appdb
```

Ne:

```text
DB_HOST=localhost
```

### Nginx vraca 502

Provjeri:

```bash
docker compose logs nginx
docker compose logs app
docker compose exec nginx nginx -t
docker compose exec nginx sh
curl -i http://app:8080
```

Najcesci fix:

```nginx
proxy_pass http://app:8080;
```

### App se pokrene prije baze

Simptomi:

```text
app starts, then exits
database not ready
connection refused
```

Bolje rjesenje:

```yaml
postgres:
  image: postgres:16
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U appuser -d appdb"]
    interval: 10s
    timeout: 5s
    retries: 5

app:
  depends_on:
    postgres:
      condition: service_healthy
```

Napomena:

```text
depends_on bez healthchecka kontrolira redoslijed startanja, ali ne ceka da je baza stvarno spremna.
```

### Port conflict

Simptomi:

```text
Bind for 0.0.0.0:80 failed: port is already allocated
```

Provjeri:

```bash
lsof -i :80
docker ps
```

Fix:

```yaml
ports:
  - "8081:80"
```

### Stari Postgres volume

Simptomi:

```text
promijenio sam POSTGRES_PASSWORD, ali login i dalje ne radi
```

Objasnjenje:

```text
Postgres env varijable inicijaliziraju usera/password samo prvi put kad je data directory prazan.
```

Provjeri:

```bash
docker volume ls
docker compose ps
```

Oprez:

```bash
docker compose down -v
```

Ovo brise volume i podatke. Koristi samo ako je test environment i ako si siguran da je prihvatljivo.

## 13. Security i Production Stvari Koje Vrijedi Spomenuti

Dobre opaske nakon rjesenja:

```text
Secrets ne bih drzao hardkodirane u docker-compose.yml.
Dodao bih .env.example bez stvarnih tajni.
Dodao bih healthcheckove za app i bazu.
Dodao bih structured logging i centralizirano skupljanje logova.
Za Postgres bih definirao backup/restore proceduru.
Za nginx bih provjerio TLS, security headers i limitiranje request body sizea ako je relevantno.
CI bi trebao buildati image, vrtiti testove i deployati verzionirani image, ne latest tag bez kontrole.
```

Nginx security headers primjer:

```nginx
add_header X-Content-Type-Options nosniff;
add_header X-Frame-Options SAMEORIGIN;
add_header Referrer-Policy strict-origin-when-cross-origin;
```

## 14. CI/CD Stvari Koje Moze Pitati

### Minimalni pipeline koraci

```text
checkout
install dependencies
run tests
build app
build Docker image
scan image/dependencies
push image registry
deploy to test
smoke test
deploy to prod with approval
```

### Jenkins/GitLab/GitHub pojmovi

```text
pipeline/job/workflow
stages
artifacts
cache
secrets/variables
runners/agents
environment
manual approval
rollback
```

### Dobar deployment princip

```text
Build once, promote the same artifact through environments.
```

Znaci: ne buildati novi image posebno za test i prod, nego isti image tag promovirati dalje.

## 15. Kako Objasniti Analizu

Kratki format:

```text
Problem:
Nginx je vracao 502.

Uzrok:
proxy_pass je pokazivao na localhost:8080, sto unutar nginx containera znaci sam nginx container, ne Java aplikaciju.

Rjesenje:
Promijenio sam upstream na Docker Compose service name app:8080.

Validacija:
nginx -t prolazi, curl kroz nginx vraca 200, a logovi vise ne pokazuju upstream connection error.

Poboljsanje:
Dodao bih healthcheck za aplikaciju i bolju dokumentaciju lokalnog pokretanja.
```


