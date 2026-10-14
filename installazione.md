# Guida: installare PowerMaster+ su un altro PC con Docker

Questa guida spiega come scaricare l'immagine `rpi-powerwalker-plus` da GitHub Container Registry (GHCR), trasferire il volume Docker `powermaster_data` e avviare PowerMaster+ su un altro PC Linux.

> **Prima di iniziare:** i comandi presumono che l'immagine sia pubblicata come `ghcr.io/hextor94/rpi-powerwalker-plus:latest`. Se il nome del pacchetto è diverso, aggiornalo nei comandi. Se il pacchetto è privato, accedi a GHCR sul PC di destinazione prima di scaricarlo.

## 1. Requisiti sul nuovo PC

- Docker installato e funzionante.
- UPS collegato via USB al nuovo PC.
- Accesso alla rete e permessi `sudo`.

Verifica Docker:

```bash
sudo docker version
sudo docker info
```

Se Docker non è installato, installalo seguendo le istruzioni della tua distribuzione Linux.

## 2. Scaricare l'immagine dal registry

Sul nuovo PC:

```bash
sudo docker pull ghcr.io/hextor94/rpi-powerwalker-plus:latest
```

Se il pacchetto è privato, effettua prima il login con un Personal Access Token GitHub con permesso `read:packages`:

```bash
echo "IL_TUO_TOKEN" | sudo docker login ghcr.io -u Hextor94 --password-stdin
```

Non condividere il token e non inserirlo in questo file.

## 3. Trasferire il volume `powermaster_data`

L'immagine Docker non trasferisce automaticamente le configurazioni e i dati persistenti contenuti nel volume. Devi creare un archivio sul vecchio PC e copiarlo sul nuovo.

### 3.05 Nuova installazione

Comando per la creazione del volume

```bash
sudo docker volume create powermaster_data
```

### 3.1 Vecchio PC: creare il backup

Per evitare modifiche ai dati durante la copia, arresta temporaneamente il container:

```bash
sudo docker stop powermaster
```

Crea un archivio del volume nella directory corrente:

```bash
sudo docker run --rm \
  -v powermaster_data:/data:ro \
  -v "$PWD":/backup \
  debian:bookworm-slim \
  tar -czf /backup/powermaster_data.tar.gz -C /data .
```

Il backup creato è `powermaster_data.tar.gz`. Copialo sul nuovo PC tramite chiavetta USB, disco esterno o `scp`.

Esempio con `scp` (sostituisci `utente` e `IP_NUOVO_PC`):

```bash
scp powermaster_data.tar.gz utente@IP_NUOVO_PC:~/
```

Riavvia il container sul vecchio PC:

```bash
sudo docker start powermaster
```

### 3.2 Nuovo PC: ripristinare il volume

Spostati nella directory in cui hai copiato `powermaster_data.tar.gz`, quindi crea il volume:

```bash
sudo docker volume create powermaster_data
```

Estrai l'archivio dentro il volume:

```bash
sudo docker run --rm \
  -v powermaster_data:/data \
  -v "$PWD":/backup:ro \
  debian:bookworm-slim \
  tar -xzf /backup/powermaster_data.tar.gz -C /data
```

Non eliminare il volume e non eseguire `docker volume prune` prima di aver verificato che il ripristino sia riuscito.

## 4. Avviare PowerMaster+

Sul nuovo PC esegui:

```bash
sudo docker run -d \
  --name powermaster \
  --restart unless-stopped \
  --privileged \
  --security-opt label=disable \
  -p 3052:3052/tcp \
  -p 3052:3052/udp \
  -p 53568:53568/tcp \
  -p 162:162/udp \
  -p 53566:53566/udp \
  -v powermaster_data:/opt/pmasterp/data \
  ghcr.io/hextor94/rpi-powerwalker-plus:latest
```

`--privileged` concede al container ampi privilegi sul sistema host. È incluso nella configurazione usata per l'accesso USB all'UPS, ma riduce l'isolamento del container.

## 5. Verificare il funzionamento

Controlla che il container sia in esecuzione:

```bash
sudo docker ps
```

Controlla i log:

```bash
sudo docker logs --tail 100 powermaster
```

Apri l'interfaccia sul nuovo PC:

- http://127.0.0.1:3052/local/

Per accedervi da un altro dispositivo della rete locale, usa `http://IP_DEL_NUOVO_PC:3052/local/`, se firewall e rete lo consentono.

Verifica che il sistema rilevi l'UPS USB:

```bash
lsusb
```

Se il container non parte, controlla i log e verifica che le porte necessarie non siano già occupate.

## 6. Comandi di gestione

Arrestare il container:

```bash
sudo docker stop powermaster
```

Avviarlo:

```bash
sudo docker start powermaster
```

Riavviarlo:

```bash
sudo docker restart powermaster
```

Seguire i log in tempo reale:

```bash
sudo docker logs -f powermaster
```

Verificare il volume:

```bash
sudo docker volume inspect powermaster_data
```

## 7. Aggiornare l'immagine in futuro

Prima di aggiornare, è consigliabile creare un nuovo backup del volume seguendo la sezione 3.1. Poi scarica l'immagine aggiornata e ricrea il container:

```bash
sudo docker pull ghcr.io/hextor94/rpi-powerwalker-plus:latest
sudo docker stop powermaster
sudo docker rm powermaster
```

Esegui nuovamente il comando `docker run` della sezione 4.

Il comando `docker rm` rimuove il container, non il volume: `powermaster_data` resta disponibile e i dati persistono. **Non usare `docker volume rm powermaster_data`**, a meno che tu non voglia eliminare definitivamente quei dati.

## Note importanti

- Conserva `powermaster_data.tar.gz` in un posto sicuro: contiene i dati persistenti dell'applicazione.
- La procedura trasferisce il volume montato in `/opt/pmasterp/data`, ma non eventuali dati salvati fuori da quella directory.
- Se il package GHCR è privato, il nuovo PC deve autenticarsi prima del `docker pull`.
- Dopo il trasferimento, verifica nell'interfaccia che la configurazione sia stata ripristinata correttamente.
