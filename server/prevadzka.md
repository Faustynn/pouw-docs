---
title: Prevádzka
parent: Server
nav_order: 3
---

# Prevádzka

Bežná práca so staging serverom a repozitárom `pouw-deploy`. Príkazy na serveri
spúšťaj ako `ubuntu` v `~/pouw/pouw-deploy`.

## Stav a logy

```bash
make ps                              # kontajnery a ich zdravie
make logs SERVICE=jenkins            # logy jednej služby: jenkins, proxy, grafana, indexer, ...
make logs SERVICE=indexer SINCE=1h   # iba posledná hodina
tail deploy-history.log              # nasadenia a automatické rollbacky
cat versions.env                     # verzie služieb z posledného nasadenia
systemctl status pouw-secrets
```

Bez SSH prístupu: GitHub → Actions → **Deploy** → akcia `status`.

Docker drží z každého kontajnera najviac 3 súbory logov po 10 MB a najstarší
maže (`x-logging` v `compose.yml`), takže logy disk nezaplnia. Staršie záznamy
sa tým strácajú; čo treba uchovať, patrí do metrík alebo do databázy.

## Alerty

Prometheus vyhodnocuje pravidlá v `monitoring/prometheus/rules/` a Alertmanager
posiela alerty e-mailom na `ALERT_EMAIL` cez ten istý SMTP účet ako Jenkins
(časť SMTP v [inštalácii](instalacia.md)). Jeden e-mail na alert 30 s po
začiatku, pripomienka každých 12 hodín, kým trvá, a jeden, keď sa vyrieši.
Alertmanager je na serveri na `127.0.0.1:9093` (cez SSH tunel).

| Alert | Kedy | Čo robiť |
| --- | --- | --- |
| `NodeDown` | Prometheus sa 2 minúty nedostane na uzol | `make logs SERVICE=node-1`; ak nepomôže, Actions → Deploy → `restart` |
| `ServiceDown` | to isté pre indexer alebo generátor | `make logs SERVICE=indexer` (alebo `workload`) |
| `ChainStalled` | výška žiadneho uzla sa 10 minút nezmenila | uzol neťaží alebo stojí: `make logs SERVICE=node-1`, Grafana → PoUW · Network |
| `IndexerLagging` | indexer je 5 minút viac ako 10 blokov za uzlom | `make logs SERVICE=indexer`; po reštarte chvíľu dobieha, to je v poriadku |
| `DiskAlmostFull` | disk servera je 10 minút plný na viac ako 80 % | `docker system df`, cache buildov: `docker compose exec jenkins-docker docker system prune -af` |
| `ContainerRestarting` | kontajner sa za 15 minút reštartoval viac ako dvakrát | `make logs SERVICE=<kontajner bez pouw- a -1>`; pri `backup` zlyháva záloha |

```bash
make test-alert    # skúšobný alert celou cestou až po e-mail
# stíšenie počas plánovanej práce
docker compose exec alertmanager amtool silence add alertname=NodeDown \
  --duration=2h --comment="výmena uzla" --alertmanager.url=http://localhost:9093
```

Nové pravidlo: `monitoring/prometheus/rules/pouw.yml` a k nemu test v
`monitoring/prometheus/tests/` (prípad, keď alert príde, aj keď nie). `make
check` testy spustí, Prometheus pravidlá načíta pri každom `make up`.

## Zálohy a obnova

Služba `backup` každú noc o 02:00 UTC uloží dump databázy indexera do
`~/pouw/pouw-deploy/backups/` (`BACKUP_DIR`), mimo zväzkov Dockeru, a zmaže
dumpy staršie ako 7 dní. Ak je najnovší starší ako 20 hodín, urobí jeden hneď
po štarte. Keď dump zlyhá, kontajner skončí, Docker ho dookola reštartuje a
príde alert `ContainerRestarting`. V databáze sú aj metadata Hasury. Grafana,
Prometheus ani Jenkins sa nezálohujú: dashboardy a konfigurácia sú v gite,
metriky sú dočasné.

```bash
ls -lt backups/        # dumpy, najnovší prvý
make backup            # dump hneď, napríklad pred rizikovou zmenou
make restore-test      # skúšobná obnova najnovšieho dumpu
```

**Skúšobná obnova** (`make restore-test`, iný dump cez `DUMP=pouw-<čas>.dump`)
obnoví dump do novej databázy `pouw_restore` vedľa živej, vypíše tabuľky s
počtom riadkov v oboch, overí, že sedia tabuľky, a `pouw_restore` zmaže. Živú
databázu nemení, spustiť sa dá kedykoľvek. To isté robí každý `make test`, takže
večerný build spadne, keď sa záloha nedá obnoviť.

**Obnova živej databázy**, napríklad po poškodení zväzku `postgres-data`:

1. Ak zväzok zmizol celý, `make up` najprv vytvorí prázdnu databázu.
2. Vyber dump: `ls -lt backups/`.
3. `./scripts/restore.sh --replace pouw-<čas>.dump` a potvrď `yes`. Skript
   zastaví indexer a Hasuru, zmaže databázu `pouw`, obnoví ju z dumpu (celú
   alebo nič) a služby znova spustí.
4. `make ps`: `indexer` a `hasura` sú `(healthy)`. Indexer pokračuje od
   posledného bloku v dumpe a novšie si stiahne z uzla.

Bez zálohy vie indexer databázu naplniť z uzla aj sám, len to trvá dlhšie:

```bash
docker compose stop indexer hasura
docker compose exec postgres dropdb -U pouw --force pouw
docker compose exec postgres createdb -U pouw pouw
make up
```

## Nasadenie a rollback

Na serveri nikdy neupravuj súbory ani nespúšťaj `git pull` ručne. Každá zmena
ide cez `main` a workflow Deploy, ktorý ju zapíše do `deploy-history.log`.

| Cieľ | Ako |
| --- | --- |
| Nasadiť posledný otestovaný stav | nič, urobí to večerný build, ak prejdú testy |
| Nasadiť `main` hneď | Actions → Deploy → akcia `deploy`, `sha` prázdne |
| Rollback | Actions → Deploy → akcia `deploy`, `sha` = starší commit z `main` (pozri `deploy-history.log`), `versions` = verzie z toho istého riadku, ak chceš vrátiť aj služby |
| Reštartovať všetko | Actions → Deploy → akcia `restart` (načíta aj heslá a konfiguráciu Jenkinsu) |
| Zistiť, čo beží | Actions → Deploy → akcia `status` |

`remote.sh` prijme iba celý SHA commitu, ktorý je v `origin/main`. Ak sa nová
verzia nenaštartuje (`make up` zlyhá, napríklad na healthchecku), vráti
predchádzajúci commit aj verzie a beh skončí chybou.

## Zmena konfigurácie Jenkinsu

| Zmena | Kde | Kedy začne platiť |
| --- | --- | --- |
| Pluginy | `jenkins/plugins.txt` | po nasadení (`make up` prestavia obraz) |
| Agent, credentials, e-mail, prihlásenie | `jenkins/casc/*.yaml` | po nasadení a Deploy → `restart`; CasC sa číta iba pri štarte Jenkinsu |
| Pre ktoré repozitáre sú joby | `jenkins/casc/jobs/github.groovy` | po nasadení a `restart` |
| Pipeline, čas večerného buildu, príjemca e-mailov | `Jenkinsfile` | pri ďalšom builde, Jenkins ho číta z gitu |
| Správcovia, adresy | `.env` na serveri | `make up` na serveri |

Zmeny urobené v UI Jenkinsu sa pri ďalšom reštarte prepíšu.

Zmenu Jenkinsu vyskúšaš pred merge lokálne: v `.env` nastav
`COMPOSE_PROFILES=jenkins` a `JENKINS_LOCAL_BRANCH=<tvoja vetva>`, potom
`make up`. Vetva musí byť pushnutá, Jenkins číta `Jenkinsfile` z GitHubu.
Lokálne sa nedá vyskúšať iba prihlásenie cez GitHub a priečinok organizácie.

## Heslá

Všetky heslá sú v `secrets.list`. Prečo práve `pass`, je v
[ADR-0003](../decisions/0003-hesla-v-pass.md).

| Druh | Kto ho vytvorí | Príklady |
| --- | --- | --- |
| `random` | `pass-init.sh`, nové aj `secrets.sh` pri nasadení | admin Grafany, používateľ Grafany v databáze, admin Jenkinsu, secret webhookov |
| `manual` | ty, `pass insert -m` | GitHub App, OAuth App, SMTP |
| `ssh-key` | `secrets.sh` pri každom štarte, nie je v `pass` | kľúč Jenkins → agent |

Kontajnery dostanú heslá ako súbory v `/run/secrets/<meno>`, nikdy cez
premenné prostredia. Na serveri sú súbory v `/run/pouw-secrets` (tmpfs,
po vypnutí zmiznú a pri štarte ich znova vytvorí `pouw-secrets.service`).

**Zmena hesla**

```bash
pass insert -m -f pouw/<cesta>    # alebo: pass generate -f --no-symbols pouw/<cesta> 32
make restart                      # prepíše /run/pouw-secrets a znova vytvorí kontajnery
```

Samotné `make up` nestačí: kontajner drží starý súbor, kým ho niekto znova
nevytvorí.

**Nové heslo**: riadok v `secrets.list` → položka v `secrets:` v `compose.yml`
(na najvyššej úrovni aj pri službe) → nasadenie. Heslo druhu `random`, ktoré
ešte nie je v `pass`, vygeneruje `secrets.sh` pri nasadení sám; heslo druhu
`manual` treba vložiť cez `pass insert -m`.

**Záloha úložiska.** Úložisko je git repozitár zašifrovaný pre kľúč servera a
kľúče správcov. Skopíruj ho na bezpečné miesto, napríklad do súkromného
repozitára:

```bash
pass git remote add origin git@github.com:<súkromný-repozitár>.git
pass git push -u origin main
```

Obnova potrebuje kľúč, pre ktorý je úložisko zašifrované. Preto spusti
`pass-init.sh` aspoň s jedným GPG kľúčom správcu.

## Nový test siete

Testy celej siete sú v `e2e/smoke.sh`, jedna kontrola na riadok:

```bash
check "<popis>" <služba> '<shell príkaz, ktorý beží v kontajneri služby>'
```

Príkaz beží cez `docker exec` v kontajneri služby, takže nepotrebuje porty na
hostiteľovi a funguje lokálne, v Jenkinse aj na stagingu. Kontrola služby z
voliteľného profilu (`network`, `host-metrics`), ktorá nebeží, sa preskočí;
služba bez profilu, ktorá nebeží, je chyba. Každá kontrola je jeden test v
záložke **Tests** v Jenkinse. Večerné nasadenie prebehne, iba ak prejdú všetky.

Smoke testy sa pýtajú iba „beží to a je to prepojené“. Správanie celého
devnetu testuje `e2e/test_devnet.py` (pytest, výsledky v suite `devnet`):
uzly majú spoločný genesis a vidia sa, `node-1` ťaží, indexer je menej ako
5 blokov za uzlom, Hasura vracia najnovší blok uzla, subscription dostane nový
blok a explorer odpovedá. Test, že všetky tri uzly majú rovnaký tip, je
označený ako očakávané zlyhanie (`xfail`), kým uzly nevymieňajú bloky
(pouw-node#86, #90). Keď prejde, značku zmaž.

Ktorý druh testu kam patrí (unit, integračné, smoke, e2e) a ako ich Jenkins
nájde, popisuje README repozitára `pouw-deploy`, časť *Tests*.

Lokálne ich vyskúšaš cez `make dev-test` (postaví služby z tvojich pracovných
kópií) alebo `make images test IMAGE_TAG=local` (z `main`), potom `make test-down`.
Proti bežiacej sieti: `./e2e/smoke.sh pouw` a `./e2e/run.sh pouw`.

## Devnet

Sieť na stagingu aj v každej testovacej kópii je devnet s tromi uzlami
`node-1`, `node-2` a `node-3`. Každý má vlastný zväzok (RocksDB a kľúč uzla).
`node-1` je bootnode a jediný ťažiar, `node-2` a `node-3` sa naň pripoja
(`NODE_BOOTNODES`) a cez neho nájdu aj seba navzájom. Indexer a generátor
čítajú z `node-1`, Prometheus zbiera metriky všetkých troch.

Genesis vytvorí služba `genesis` raz, keď je zväzok `devnet` prázdny: zo
šablóny `devnet/chainspec.json` (kópia `pouw-node/chainspec/devnet.json`) s
aktuálnym časom ako `genesis.timestamp`. Plán obtiažnosti sa počíta od tohto
času, takže reťazec spustený dni po svojom genesis by najprv vyťažil nával
ľahkých blokov.

**Nový reťazec** (nový genesis, prázdne uzly aj databáza indexera):

```bash
./scripts/devnet-reset.sh    # pýta sa, potom make up
```

Treba ho po zmene `devnet/chainspec.json` (drž ju rovnakú ako v pouw-node) a
raz po nasadení, ktoré mení genesis: uzol svoju databázu pod iným genesis
odmietne a indexer prestane synchronizovať reťazec, s ktorým nemá spoločný ani
genesis. `make seed` do databázy s devnetom nikdy nepúšťaj, z rovnakého dôvodu.

## Nová služba

Príklad: `pouw-node` dostane `Dockerfile`.

1. V repozitári služby: `Dockerfile`. `docker.yml` potom pri každom pushi do
   `main` publikuje `ghcr.io/pouw-zk/pouw-node:main` a `:sha-<7>`. Server
   sťahuje bez prihlásenia, takže balíček v GHCR musí byť verejný
   (organizácia → Packages → pouw-node → Package settings → visibility).
2. Lokálne over, že sa služba postaví a nabehne spolu so sieťou: `make dev`
   v `pouw-deploy` (postaví ju z tvojej pracovnej kópie).
3. Keď je obraz v GHCR, v `compose.yml` pri službe zmaž riadok
   `profiles: [network]`. Odvtedy ju spúšťa `make up`, večerný test aj staging.
   Nová služba dostane `image: ghcr.io/pouw-zk/<repo>:${<MENO>_VERSION:-main}`
   a healthcheck. Porty na hostiteľovi iba ak sú naozaj potrebné; vtedy na
   `127.0.0.1` a v `compose.test.yml` ich vypni cez `ports: !reset []`.
4. Nové heslá: pozri časť Heslá vyššie.
5. Metriky: scrape job v `monitoring/prometheus/prometheus.yml`.
6. Testy: kontroly v `e2e/smoke.sh` alebo `e2e/test_*.py` v repozitári služby.

`scripts/build-images.sh`, `Makefile` (`IMAGE_TAG`) a `remote.sh` už poznajú
`pouw-node`, `pouw-indexer`, `pouw-workload` a `pouw-dashboard`. Službu s novým
menom treba pridať do všetkých troch.

## Kapacita

Agent má 2 executory (`numExecutors` v `jenkins/casc/jenkins.yaml`), takže
naraz bežia najviac dva buildy, každý s celou testovacou kópiou siete.
Orientačne podľa pamäte servera: 8 GB a menej → 1, 16 GB → 2, 32 GB a viac → 3
až 4. Zníž, ak buildy spomaľujú staging; zvýš, ak buildy čakajú vo fronte.

Izolovaný Docker drží cache v zväzku `jenkins-docker-data`, nepoužívané obrazy
sa mažú po každom builde. Ak sa disk aj tak zaplní:

```bash
docker compose exec jenkins-docker docker system prune -af
```

## Bezpečnosť

- **Zvonka dostupné:** iba proxy na 80/443 (HTTPS, Let's Encrypt). Všetky
  ostatné porty sú na `127.0.0.1`.
- **Prístup do Jenkinsu:** prihlásenie cez GitHub, členovia PoUW-ZK môžu joby
  pozerať a spúšťať, `JENKINS_ADMINS` spravujú Jenkins. Anonym vidí iba stav
  buildu (odznak v README).
- **Buildy:** bežia na agentovi ako používateľ `jenkins` proti izolovanému
  Dockeru. Nevidia kontajnery stagingu ani `/run/pouw-secrets`. PR z forkov sa
  nestavajú.
- **Kľúč na nasadenie:** forced command (`remote.sh`) v `authorized_keys`.
  Uniknutý kľúč vie iba nasadiť commit, ktorý už je v `main`, reštartovať a
  ukázať stav.
- **Heslá:** zašifrované v `pass`, dešifrované iba do tmpfs, do kontajnerov ako
  súbory. GitHub má iba kľúč na nasadenie, kľúč GitHub App je v `pass` na serveri.
- **Skenovanie:** Trivy a actionlint pri každom PR (workflow `security`).
  `jenkins/agent.Dockerfile` zámerne končí `USER root` (potrebuje ho sshd,
  buildy bežia ako `jenkins`), preto má `# trivy:ignore:DS-0002`. Tento riadok
  nemaž.

## Riešenie problémov

| Príznak | Príčina | Riešenie |
| --- | --- | --- |
| `make up`: `Missing in pass: pouw/smtp/...` | SMTP nie je nastavené | v poriadku, ak je to zámer; pozri časť SMTP v [inštalácii](instalacia.md) |
| Jenkins nenabehne, v `make logs SERVICE=jenkins` je `ConfiguratorException` | chybná hodnota v `jenkins/casc` alebo v hesle | oprav YAML alebo heslo; `pass show pouw/github/app-key \| head -1` musí byť `-----BEGIN PRIVATE KEY-----` |
| Prihlásenie cez GitHub sa zacyklí alebo zlyhá | callback URL v OAuth App nesedí s `JENKINS_URL` | callback musí byť `https://<host>/securityRealm/finishLogin` |
| Priečinok PoUW-ZK je prázdny | GitHub App nie je nainštalovaná alebo nemá práva | Install App → All repositories; priečinok → Scan Organization Now |
| PR nespúšťajú buildy | zlá URL alebo secret webhooku | GitHub App → Advanced → Recent Deliveries ukáže chybu |
| Bez certifikátu, prehliadač hlási chybu TLS | zatvorené porty 80/443 alebo DNS neukazuje na server | `make logs SERVICE=proxy`, zvonka `nc -vz <ip> 443` |
| Večerné nasadenie: `Deploy workflow run not found` | GitHub App nemá Actions: Read and write | nastavenia aplikácie → Permissions, potom potvrď zmenu v inštalácii |
| Beh Deploy čaká (`Waiting for a runner`) | runner na serveri nebeží | `sudo systemctl status 'actions.runner.*'` na serveri; znova `sudo ./scripts/runner-setup.sh <token>` |
| Beh Deploy zlyhá na SSH | zlé hodnoty `DEPLOY_*` (host je `127.0.0.1`) alebo chýba riadok v `authorized_keys` | pozri časť 7 v [inštalácii](instalacia.md) |
| Beh Deploy: `Commit ... is not in main` | `sha` je z inej vetvy | použi commit z `main` |
| Workflow Security padá na DS-0002 | z `agent.Dockerfile` zmizol riadok `trivy:ignore` | vráť ho nad `USER root` |
| `pass` hlási `Please tell me who you are` | stará verzia `pass-init.sh` bez git identity | `pass git config user.name "PoUW server"`, `pass git config user.email pouw@$(hostname)`, `pass git add -A && pass git commit -m "Add secrets"` |
| Po reštarte servera kontajnerom chýbajú heslá | zlyhala `pouw-secrets.service` | `systemctl status pouw-secrets`, `journalctl -u pouw-secrets` |
| Lokálny Jenkins: `No rule to make target 'check'` | job vytvorený v UI s vloženým skriptom nemá checkout | používaj job `pouw-deploy` z CasC, job z UI zmaž |
| Lokálny Jenkins sa nedostane na GitHub | pokazená sieť Docker Desktop | reštartuj Docker Desktop; `docker run --rm curlimages/curl -sI https://github.com` |
| `make test`: `port is already allocated` | nová služba publikuje port na hostiteľovi | pridaj ju do `compose.test.yml` s `ports: !reset []` |
| Uzol sa nenaštartuje (exit 1 alebo 2), v logu iný genesis | databáza uzla alebo `devnet/chainspec.json` patrí k inému genesis | `./scripts/devnet-reset.sh`; po zmene šablóny skontroluj, či sa zhoduje s `pouw-node/chainspec/devnet.json` |
| Indexer: `the node's chain shares no block with the database, not even the genesis` | devnet dostal nový genesis alebo niekto spustil `make seed` | `./scripts/devnet-reset.sh` |
| `make dev`: `no ../pouw-xxx, skipped`, potom `pull access denied` | repozitár služby nie je naklonovaný vedľa `pouw-deploy` | `.github/scripts/sync-repos.sh` |
