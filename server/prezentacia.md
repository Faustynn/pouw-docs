---
title: Prezentácia (demo)
parent: Server
nav_order: 6
---

# Prezentácia 30. 11.: kto čo hovorí a ukazuje

Každý člen tímu predstaví svoju časť: čo robí, ako to funguje a naživo to
ukáže na stagingu. Celkový čas je približne 30 minút. Poradie sleduje cestu
dát: uzol → indexer → API → explorer, potom záťaž a prevádzka. Pojmy sú
vysvetlené v [Ako to funguje](ako-to-funguje.md), záložný plán v
[Demo 30. 11.](demo.md).

| # | Časť | Kto | Čas |
| --- | --- | --- | --- |
| 1 | Úvod a architektúra | Nazar Meredov | 3 min |
| 2 | Uzol, sieť a ľahký uzol | Ilia Poliak | 6 min |
| 3 | Indexer a databáza | Kostiantyn Cherniakov | 4 min |
| 4 | Webové API (Hasura GraphQL) | Vladimir Riazantsev | 4 min |
| 5 | Explorer | Yaroslav Marochok | 5 min |
| 6 | Generátor záťaže | Nikita Lovkin | 5 min |
| 7 | Nasadenie, testy a prevádzka | Nazar Meredov | 4 min |
| 8 | Čo ďalej | všetci | 1 min |

## Príprava (hodinu pred demom)

- VPN školy zapnutá, `ssh pouw` funguje, v termináli `cd ~/pouw/pouw-deploy`.
- `make ps`: všetky služby `(healthy)`.
- Otvorené záložky:

| Čo | Adresa | Prihlásenie |
| --- | --- | --- |
| Explorer | <https://explorer.147-175-151-18.sslip.io> | bez prihlásenia |
| Grafana | <https://grafana.147-175-151-18.sslip.io> | `admin`, `pass show pouw/grafana/admin` |
| Alertmanager | <https://alerts.147-175-151-18.sslip.io> | `admin`, `pass show pouw/alertmanager/password` |
| Jenkins | <https://jenkins.147-175-151-18.sslip.io> | GitHub |
| GitHub Actions | pouw-deploy → Actions → Deploy | GitHub |

- V Grafane otvorené dashboardy PoUW · Network, PoUW · Services a PoUW · Load test.
- V deň dema nič nenasadzovať ani nereštartovať.

Pomocná funkcia do terminálu, používajú ju ukážky nižšie:

```bash
q() { docker compose exec -T "$1" curl -s -H 'Accept: application/json' "localhost:8000$2"; echo; }
```

---

## 1. Úvod a architektúra (Nazar, 3 min)

**Čo poviem**

- Staviame vlastný L1 blockchain, ktorý funguje ako Bitcoin: ťažiari hashujú
  hlavičky blokov a súťažia o odmenu, používatelia si posielajú mince.
- Cieľom projektu je Proof of Useful Work (dôkazy pre klientov namiesto
  zbytočných hashov). Na deme ukazujeme hotovú L1 so všetkým okolo, dôkazy sú
  ďalšia fáza.
- Celý systém má 6 repozitárov a beží v Dockeri na jednom serveri:

```text
generátor záťaže ──transakcie──▶ uzly 1, 2, 3 (ťažia) ◀──▶ ľahký uzol
                                      │
                                   indexer ──▶ PostgreSQL ──▶ Hasura (GraphQL) ──▶ explorer
                                      │
                Prometheus ◀── metriky všetkých služieb ──▶ Grafana, Alertmanager (e-mail)
```

**Ukážka:** schéma zo stránky [Ako to funguje](ako-to-funguje.md#celý-obraz) a
`make ps` v termináli: 16 služieb, všetky `(healthy)`.

---

## 2. Uzol, sieť a ľahký uzol (Ilia, 6 min)

**Čo poviem**

- Uzol je napísaný v Ruste. Blok má hlavičku so 168 bajtmi, ťažiar mení
  nonce a hashuje ju (SHA-256), kým hash nie je pod cieľom. Cieľ sa sám
  upravuje, aby blok vznikol približne každých 10 sekúnd.
- Účty ako v Ethereu: každá adresa má zostatok a nonce, transakcia je v
  borsh a podpísaná kľúčom Ed25519.
- Sieť cez libp2p: gossipsub pre nové bloky a transakcie, request-response
  pre synchronizáciu. Nový uzol sťahuje najprv hlavičky, potom telá blokov
  (headers-first initial block download).
- Ochrana: limity spojení a zoznam zakázaných uzlov. Dáta v RocksDB, REST API
  na porte 8000, metriky na 9100.
- Na deme ťažia všetky 3 uzly a súťažia o každý blok. Keď dvaja nájdu blok
  naraz, vyhrá reťaz s väčšou prácou (reorg).
- Ľahký uzol drží iba hlavičky a pýta sa plných uzlov; cez neho číta
  generátor záťaže.

**Ukážka**

1. Stav uzlov, všetky majú rovnakú výšku a 2 spojenia:

   ```bash
   for n in node-1 node-2 node-3; do q $n /v1/node; done
   ```

2. Posledný blok: hlavička s ťažiarom, cieľom a nonce:

   ```bash
   h=$(q node-1 /v1/node | grep -oE '"height":[0-9]+' | cut -d: -f2)
   q node-1 /v1/blocks/$h
   ```

3. Ľahký uzol má rovnakú výšku ako plné uzly:

   ```bash
   q light-node /v1/light/head
   ```

4. Grafana → PoUW · Network: *Height per node* (tri čiary spolu), *Blocks per
   miner* (bloky nachádzajú všetci traja), *Peers*.
5. *(voliteľne)* Výpadok uzla: `docker compose restart node-3`, po 30 s znova
   krok 1, node-3 dobehol ostatných.

**Otázky, ktoré môžu prísť**

- *Čím sa líši od Bitcoinu?* Účty namiesto UTXO, blok každých 10 s, v ďalšej
  fáze dôkazy v bloku (miesto pre ne už v bloku je: `proof_txs`).
- *Čo keď dvaja ťažiari nájdu blok naraz?* Na chvíľu sú dve vetvy, vyhrá tá s
  väčšou prácou, ostatné uzly sa prepnú.

---

## 3. Indexer a databáza (Kostiantyn, 4 min)

**Čo poviem**

- Uzol vie odpovedať na jednoduché otázky („blok číslo 100“), ale nie „všetky
  transakcie adresy“ alebo „koľko blokov vyťažil ktorý ťažiar“. Indexer (Rust)
  preto prepisuje reťaz do PostgreSQL.
- Bloky a účty číta z uzla 1 a pamätá si, kde skončil, takže po reštarte
  pokračuje od posledného zapísaného bloku.
- Pri reorgu označí bloky zahodenej vetvy ako `canonical = false`.
- Schéma vzniká migráciami pri štarte, raz za minútu počíta štatistiky siete
  pre Grafanu (`network_stats`).

**Ukážka**

1. Indexer stíha uzol, oneskorenie 0 blokov:

   ```bash
   docker compose exec -T indexer curl -s localhost:8000/metrics | grep -E '^pouw_indexer_(indexed_height|node_height|lag_blocks)'
   ```

2. Čo je v databáze: bloky hlavnej reťaze aj zahodené pri reorgoch, prevody:

   ```bash
   docker compose exec -T postgres psql -U pouw -d pouw -c \
     "select canonical, count(*) from blocks group by canonical;" -c \
     "select count(*) as transfers from transfers;"
   ```

3. Štatistiky za posledné minúty:

   ```bash
   docker compose exec -T postgres psql -U pouw -d pouw -c \
     "select minute, blocks, transfers, avg_block_time_s from network_stats order by minute desc limit 5;"
   ```

4. Grafana → PoUW · Services: *Height: node / indexer* a *Indexing delay*.

**Otázky, ktoré môžu prísť**

- *Prečo nečíta explorer priamo z uzla?* Uzol nie je vystavený verejne a
  databáza vie rýchlo odpovedať aj na zložité otázky.
- *Čo keď indexer spadne?* Po štarte pokračuje od posledného bloku, ktorý
  zapísal.

---

## 4. Webové API: Hasura GraphQL (Vladimir, 4 min)

**Čo poviem**

- Hasura z databázy automaticky robí GraphQL API pre explorer.
- Nevidí tabuľky indexera, iba pohľady (views) v schéme `api`: tie z bajtov
  robia identifikátory ako `blk:…`, pridávajú súčty a stránkovanie. Keď sa
  zmení tabuľka indexera, zmenia sa iba pohľady, API ostane rovnaké.
- Bezpečnosť: rola `anonymous` smie iba čítať, najviac 100 riadkov, a iba
  vopred schválené dotazy (allow-list z kontraktu `web-api.graphql`).
- Okrem dotazov posiela zmeny naživo (subscriptions cez WebSocket).

**Ukážka**

1. Schválený dotaz z kontraktu, posledné bloky:

   ```bash
   curl -s https://explorer.147-175-151-18.sslip.io/v1/graphql -H 'Content-Type: application/json' -d '{"query":"query LatestBlocks($limit: Int = 10) { blocks(where: {canonical: {_eq: true}}, order_by: {height: desc}, limit: $limit) { ...BlockRow } } fragment BlockRow on blocks { hash height time miner transfer_count registration_count proof_count fee_total }","variables":{"limit":2}}'
   ```

2. Neschválený dotaz Hasura odmietne:

   ```bash
   curl -s https://explorer.147-175-151-18.sslip.io/v1/graphql -H 'Content-Type: application/json' -d '{"query":"{ blocks { hash } }"}'
   ```

   Odpoveď: `query is not allowed`.

3. Konzola a metadata Hasury nie sú zvonka dostupné:

   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' https://explorer.147-175-151-18.sslip.io/v1/metadata
   ```

   Odpoveď: `404`.

**Otázky, ktoré môžu prísť**

- *Prečo GraphQL a nie REST?* Explorer si vyberie presne tie polia, ktoré
  zobrazuje, a subscriptions dávajú živé dáta bez vlastného servera.

---

## 5. Explorer (Yaroslav, 5 min)

**Čo poviem**

- Webová aplikácia v Reacte (Vite), beží v nginx za HTTPS proxy.
- Je ako blockchain.com pre Bitcoin: posledné bloky, detail bloku a prevodu,
  adresy so zostatkom, ťažiari, vyhľadávanie.
- Živé dáta cez GraphQL subscriptions (WebSocket s obnovením spojenia), bez
  obnovovania stránky.
- Vyhľadávanie rozpozná výšku, hash bloku, `txid` aj adresu, aj v rôznych
  formátoch (`blk:…`, `0x…`, holý hash).
- Stránky pre dôkazy (circuits, proofs) sú na demo skryté, patria do ďalšej
  fázy.

**Ukážka**

1. **Home**: nové bloky pribúdajú samy, štatistiky siete.
2. **Blocks** → detail bloku: ťažiar, čas, prevody, poplatky.
3. Adresa ťažiara `addr:8a88e3dd7409f195fd52db2d3cba5d72ca6709bf1d94121bf3748801b40f6f5c`:
   zostatok a história.
4. **Mining**: podiel troch ťažiarov.
5. Vyhľadávanie: zadať výšku bloku, potom hash bloku.
6. Spolu s Nikitom (časť 6): počas záťaže bloky s desiatkami prevodov.

**Otázky, ktoré môžu prísť**

- *Čo keď spadne spojenie?* Explorer ukáže stav spojenia a sám sa pripojí
  znova.

---

## 6. Generátor záťaže (Nikita, 5 min)

**Čo poviem**

- Hrá úlohu mnohých používateľov naraz, napísaný v Jave.
- Scenár v YAML určuje trvanie, rýchlosť a mix požiadaviek. Scenáre
  `transfers-steady` (2 prevody/s), `transfers-spike` (nárazy) a `transfers`.
- Pri štarte vytvorí peňaženky (Ed25519), pošle im mince z kľúča genesis,
  potom posiela podpísané prevody (borsh) striedavo do všetkých troch plných
  uzlov. Nonce a čítania berie cez ľahký uzol.
- Open loop: požiadavky posiela podľa plánu, aj keď sieť nestíha, takže meria
  skutočné oneskorenie. Meria p50, p95 a p99 a posiela metriky do Prometheu.

**Ukážka**

1. Scenáre, ktoré generátor pozná:

   ```bash
   docker compose exec -T workload curl -s localhost:8000/scenarios
   ```

2. Záťaž (2 minúty):

   ```bash
   make load SCENARIO=transfers-steady
   ```

3. Počas behu: Grafana → PoUW · Load test (požiadavky za sekundu, oneskorenie
   p95, chyby) a v exploreri bloky s desiatkami prevodov.
4. Po behu: tabuľka v termináli, koľko prevodov a čítaní prešlo a s akým
   oneskorením.

**Otázky, ktoré môžu prísť**

- *Koľko transakcií sieť zvládne?* Ukázať číslo z behu `transfers-spike`
  (10 prevodov/s v nárazoch) a ako sa mení oneskorenie.

---

## 7. Nasadenie, testy a prevádzka (Nazar, 4 min)

**Čo poviem**

- Každá služba má Dockerfile, celá sieť je v jednom `compose.yml`. Obrazy
  stavia GitHub Actions do GHCR.
- Pull request potrebuje zelenú kontrolu `ci-ok` (lint, testy, bezpečnosť) a
  schválenie iného člena tímu.
- Každý večer o 21:00 Jenkins postaví celú sieť od nuly v oddelenej kópii a
  pustí testy celej siete: genesis, spojenia uzlov, rovnaký vrchol, viac
  ťažiarov, indexer, Hasura, subscription, skutočný prevod až do explorera,
  obnova zálohy. Iba keď prejdú, nová verzia ide na staging; ak nenaštartuje,
  sama sa vráti predchádzajúca.
- Monitoring: Prometheus, Grafana, Alertmanager posiela alerty e-mailom.
  Každú noc sa zálohuje databáza.
- Heslá iba v `pass`, zvonka iba HTTPS (Caddy, certifikáty sa vydávajú automaticky).

**Ukážka**

1. Jenkins: posledný nočný build `pouw-deploy/main`, kroky a záložka Tests.
2. GitHub → Actions → Deploy a história nasadení:

   ```bash
   tail -5 deploy-history.log
   ```

3. Alertmanager s prihlásením a skúšobný alert, ktorý príde e-mailom:

   ```bash
   make test-alert
   ```

4. Zálohy a skúšobná obnova (živú databázu nemení):

   ```bash
   ls -lt backups | head -3
   make restore-test
   ```

5. Grafana → PoUW · Server: CPU, pamäť a disk servera.

---

## 8. Čo ďalej (všetci, 1 min)

- Proof of Useful Work: dôkazy pre klientov v blokoch, skutočný Groth16 a
  ďalšie dôkazové systémy.
- Rýchlejšia synchronizácia nového uzla od nuly pri dlhej reťazi.
- Ľahký uzol ako klient pre používateľov (vlastné peňaženky).

## Keď niečo nepôjde

| Problém | Čo urobiť |
| --- | --- |
| Uzol zaostáva | `docker compose restart node-N`, počkať 30 s |
| Prevod je `pending` | počkať 1–2 bloky (asi 30 s) |
| Explorer sa neobnovuje | F5, explorer ukáže stav spojenia |
| Služba nebeží | `make ps`, potom Actions → Deploy → `restart` |
| Server je nedostupný | záložný notebook podľa [Demo 30. 11.](demo.md#záložný-plán) |
