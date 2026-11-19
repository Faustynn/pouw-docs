---
title: Demo 30. 11.
parent: Server
nav_order: 4
---

# Demo 30. 11.: checklist a záložný plán

Čo ukážeme, v akom poradí a čo robiť, keď niečo nepôjde. Kto čo prezentuje,
je návrh: tím ho potvrdí na stretnutí pred demom. Ako jednotlivé časti
fungujú, jednoducho vysvetlené aj s odpoveďami na otázky, je na stránke
[Ako to funguje (demo)](ako-to-funguje.md).

Adresy stagingu:

| Čo | Kde |
| --- | --- |
| Explorer a GraphQL | <https://explorer.147-175-151-18.sslip.io> |
| Grafana | <https://grafana.147-175-151-18.sslip.io> |
| Jenkins | <https://jenkins.147-175-151-18.sslip.io> |

## Poradie

| # | Čo ukážeme | Kto (návrh) | Čas |
| --- | --- | --- | --- |
| 1 | Cieľ projektu a architektúra: uzol, indexer, Hasura, explorer, generátor záťaže | Nazar | 3 min |
| 2 | Devnet s 3 uzlami: genesis, ťažba v uzle 1, uzly 2 a 3 držia rovnaký vrchol, štruktúra bloku (`txs`, `proof_txs`) | Ilia, Oleksandr | 5 min |
| 3 | Skutočná transakcia: podpísaná cez `pouw transfer`, odoslaná uzlu, v bloku a v exploreri | Kostiantyn, Vladimir | 4 min |
| 4 | Explorer: bloky, transakcie, adresy, živé aktualizácie | Yaroslav | 5 min |
| 5 | Záťažový test: scenár generátora a panely v Grafane počas behu | Nikita, Nazar | 4 min |
| 6 | Prevádzka: dashboardy, alerty, zálohy, nočný test celej siete a nasadenie | Nazar | 4 min |
| 7 | Čo nie je hotové a čo ďalej | všetci | 2 min |

Pri bode 3 a 5 pomôžu príkazy na serveri v adresári `~/pouw/pouw-deploy`:

```bash
make load SCENARIO=devnet   # 1 min čítaní, panely: Grafana → PoUW · Load test
docker compose exec node-1 curl -s -H 'Accept: application/json' localhost:8000/v1/node
```

Transakcia pre bod 3: testovací kľúč z kontraktov (ťažiar uzla 1), jeho
aktuálny nonce a nová adresa príjemcu. Odpoveď je `txid`, ten sa potom
vyhľadá v exploreri.

```bash
n() { docker compose exec -T node-1 "$@"; }
from=addr:8a88e3dd7409f195fd52db2d3cba5d72ca6709bf1d94121bf3748801b40f6f5c
nonce=$(n curl -s -H 'Accept: application/json' localhost:8000/v1/accounts/$from | grep -oE '"nonce":[0-9]+' | cut -d: -f2)
to=$(n pouw keygen | grep -oE 'addr:[0-9a-f]{64}' | head -n 1)
n pouw transfer --seed "$(printf '01%.0s' $(seq 32))" --chain-id 1 --nonce "$nonce" --to "$to" --amount 1000 --fee 10 |
    n curl -s -X POST -H 'Content-Type: application/json' --data-binary @- localhost:8000/v1/txs
```

## Deň pred demom

- [ ] Večerný build `main` prešiel a nasadil sa (Jenkins, `deploy-history.log`).
- [ ] Actions → Deploy → `status`: všetky služby `(healthy)`, verzie z posledného buildu.
- [ ] Uzol 1 ťaží, výška rastie, indexer je za ním menej ako 5 blokov (Grafana → PoUW · Services).
- [ ] Explorer ukazuje skutočné bloky z devnetu, nie ukážkové dáta.
- [ ] Grafana: dashboardy PoUW · Network, Services, Server a Load test majú dáta, verejné panely v exploreri sa načítajú.
- [ ] Žiadny alert nie je aktívny (Alertmanager), testovací alert príde e-mailom (`make test-alert`).
- [ ] `make load SCENARIO=devnet` prejde bez chýb, report je v `reports/load`.
- [ ] Záloha z poslednej noci existuje a `make restore-test` prejde.
- [ ] Záložný notebook je pripravený (pozri nižšie).

## Hodinu pred demom

- [ ] Otvorené záložky: explorer, Grafana (Load test a Network), Jenkins, tento checklist.
- [ ] Prihlásenie do Grafany a Jenkinsu funguje, SSH na server funguje.
- [ ] Žiadne nasadenie ani reštart počas dema: nočný build sa ten deň nespúšťa ručne.
- [ ] Posledná kontrola výšky uzla a explorera.

## Záložný plán

| Problém | Čo urobiť |
| --- | --- |
| Služba nebeží alebo je `unhealthy` | Actions → Deploy → `restart`, potom `make ps` |
| Nové nasadenie pokazilo staging | Rollback: Actions → Deploy → `deploy` so `sha` predchádzajúceho riadku v `deploy-history.log` ([Prevádzka](prevadzka.md#nasadenie-a-rollback)) |
| Databáza indexera je poškodená | `./scripts/restore.sh --replace pouw-<čas>.dump` ([Prevádzka](prevadzka.md#zálohy-a-obnova)) |
| Devnet sa zasekol (uzol 1 neťaží) | `scripts/devnet-reset.sh`: nový genesis, prázdne uzly aj databáza, explorer začne od nuly |
| Server je nedostupný | Záložný notebook: sieť lokálne zo zálohy |

**Záložný notebook** sa pripraví deň vopred:

```bash
cd pouw-deploy && git pull
make secrets && make up                      # :main z GHCR, devnet s 3 uzlami
scp ubuntu@147.175.151.18:pouw/pouw-deploy/backups/<najnovší>.dump backups/
./scripts/restore.sh --replace <najnovší>.dump
make ps                                      # všetko (healthy)
```

Explorer je potom na <http://localhost:8082>, Grafana na <http://localhost:3000>.
Lokálny devnet má vlastný genesis, takže uzly ťažia novú reťaz od nuly a
obnovená databáza slúži na ukážku explorera so staršími blokmi.

## Čo povieme otvorene

- Uzly na stagingu dokazujú cez *mock* dôkazovač, aby ťažba nebola pomalá.
  Skutočný Groth16 sa zapína cez `POUW_PROVER=groth16`.
- Objednávky dôkazov v devnete zatiaľ nikto neposiela, bloky majú
  `proof_txs` prázdne a obsahujú iba prevody mincí.
- Implementovaný je zatiaľ jeden dôkazový systém (Groth16) z troch
  plánovaných ([ADR-0006](../decisions/0006-tri-dokazovacie-systemy.md)).
- Ľahký uzol je v obraze uzla, ale v devnete ešte nebeží (pouw-node#105),
  generátor preto číta z uzla 1.
- Scenáre `steady` a `spike` generátora sú v práci (pouw-workload#48).
