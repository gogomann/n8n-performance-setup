# ⚡ n8n Performance Setup - Queue-Mode mit Worker-Skalierung

> **High-Performance n8n Deployment** mit PostgreSQL, Redis, Queue-Mode und skalierbaren Workern für produktive Workflow-Automatisierung

---

## 📋 Inhaltsverzeichnis

- [Überblick](#-überblick)
- [Features](#-features)
- [Architektur](#-architektur)
- [Voraussetzungen](#-voraussetzungen)
- [Installation](#-installation)
- [Konfiguration](#-konfiguration)
- [Performance-Optimierungen](#-performance-optimierungen)
- [Skalierung](#-skalierung)
- [Monitoring](#-monitoring)
- [Troubleshooting](#-troubleshooting)
- [Sicherheit](#-sicherheit)

---

## 🎯 Überblick

Dieses Setup zeigt eine **produktionsreife n8n-Installation** mit:

- **Queue-Mode**: Ausführungen laufen asynchron über Redis-Queue
- **Worker-Skalierung**: Beliebig viele Worker-Instanzen parallel
- **PostgreSQL**: Robuste, performante Datenbank
- **Redis**: Schnelle Queue-Verwaltung mit Persistierung
- **Health Checks**: Automatische Service-Überwachung
- **Horizontal Scalability**: Einfach weitere Worker hinzufügen

---

## ✨ Features

### Performance-Vorteile

✅ **Bis zu 10x schnellere Ausführung** bei parallelen Workflows  
✅ **Keine Blockierung** der Haupt-Instanz durch lange Workflows  
✅ **Automatisches Load-Balancing** über Redis-Queue  
✅ **Horizontal skalierbar** - einfach mehr Worker starten  
✅ **Fehlertoleranz** durch Queue-Persistierung  
✅ **Webhook-Verfügbarkeit** bleibt auch bei hoher Last erhalten  

### Technische Features

- **PostgreSQL 15** mit optimierten Health-Checks
- **Redis 7** mit AOF-Persistierung und LRU-Eviction
- **n8n Main-Instance**: UI + Webhook-Empfang
- **n8n Worker**: Reine Ausführungs-Instanzen
- **Concurrency Control**: Konfigurierbare parallele Ausführungen pro Worker
- **Shared Volumes**: Gemeinsamer Zugriff auf Credentials und Workflows

---

## 🏗️ Architektur

```
┌────────────────────────────────────────────────────────────┐
│                    Reverse Proxy                        │
│                  (Traefik/Nginx/etc.)                   │
└────────────────────────┬───────────────────────────────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │   n8n Main Instance  │
              │   (UI + Webhooks)    │
              └──────────┬───────────┘
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
   ┌──────────┐    ┌─────────┐    ┌──────────┐
   │PostgreSQL│    │  Redis  │    │  Shared  │
   │    DB    │    │  Queue  │    │  Volume  │
   └──────────┘    └────┬────┘    └──────────┘
                        │
              ┌─────────┼─────────┐
              ▼         ▼         ▼
        ┌─────────┬─────────┬─────────┐
        │ Worker-1│ Worker-2│ Worker-N│
        │  (5x)   │  (5x)   │  (5x)  │
        └─────────┴─────────┴─────────┘
            Concurrency: 5 Jobs parallel
```

### Workflow-Ablauf

1. **Trigger**: Workflow wird gestartet (manuell/Webhook/Cron)
2. **Queue**: Main-Instance schreibt Job in Redis-Queue
3. **Processing**: Erster freier Worker holt Job und führt ihn aus
4. **Result**: Ergebnis wird in PostgreSQL gespeichert
5. **UI**: Main-Instance zeigt Status/Ergebnis an

---

## 📦 Voraussetzungen

- **Docker** >= 20.10
- **Docker Compose** >= 2.0
- **Mind. 2 GB RAM** (4 GB empfohlen)
- **2 CPU Cores** (4+ empfohlen)
- **Disk Space**: ~5 GB für Images + Daten

### Empfohlene Ressourcen pro Worker

| Workers | RAM   | CPU Cores | Gleichzeitige Workflows |
|---------|-------|-----------|------------------------|
| 1       | 2 GB  | 1-2       | ~5                     |
| 2       | 4 GB  | 2-4       | ~10                    |
| 4       | 8 GB  | 4-8       | ~20                    |
| 8       | 16 GB | 8-16      | ~40                    |

---

## 🚀 Installation

### 1. Repository klonen

```bash
git clone https://github.com/gogomann/n8n-performance-setup.git
cd n8n-performance-setup
```

### 2. Umgebungsvariablen konfigurieren

```bash
cp .env.example .env
nano .env  # oder vi, vim, code, etc.
```

**Wichtig:** Ändere mindestens:
- `POSTGRES_PASSWORD`
- `N8N_ENCRYPTION_KEY`
- `N8N_HOST`
- `WEBHOOK_URL`
- Pfade zu deinen Data-Directories

### 3. Data-Directories erstellen

```bash
mkdir -p /path/to/data/postgres
mkdir -p /path/to/data/redis
mkdir -p /path/to/data/n8n
```

### 4. Berechtigungen setzen

```bash
# User ID 1000 ist Standard für n8n Container
sudo chown -R 1000:1000 /path/to/data/n8n
```

### 5. Container starten

```bash
docker-compose up -d
```

### 6. Logs prüfen

```bash
# Alle Services
docker-compose logs -f

# Nur n8n Main
docker-compose logs -f n8n-main

# Nur Worker
docker-compose logs -f n8n-worker-1 n8n-worker-2
```

### 7. n8n öffnen

```
http://localhost:5678
```

Oder über deine konfigurierte Domain.

---

## ⚙️ Konfiguration

Alle wichtigen Einstellungen werden über die `.env` Datei verwaltet. Siehe `.env.example` für vollständige Dokumentation.

### Wichtigste Parameter

```env
# Domain und Protokoll
N8N_HOST=your-domain.com
N8N_PROTOCOL=https
WEBHOOK_URL=https://your-domain.com/

# Verschlüsselung (KRITISCH!)
N8N_ENCRYPTION_KEY=your-super-secret-key-min-32-chars

# Worker-Leistung
WORKER_CONCURRENCY=5  # Parallele Workflows pro Worker

# Datenbank
POSTGRES_PASSWORD=your-secure-password
```

---

## 🔥 Performance-Optimierungen

### 1. Redis-Tuning

Passe `REDIS_MAX_MEMORY` in `.env` an:
- Kleine Setups (< 100 Workflows/Tag): `256mb`
- Mittlere Setups (100-1000): `512mb`
- Große Setups (> 1000): `1gb+`

### 2. Worker-Skalierung

**Horizontal:** Mehr Worker hinzufügen

```bash
# In docker-compose.yml: Worker-3, Worker-4, etc. hinzufügen
docker-compose up -d
```

**Vertikal:** Mehr Concurrency pro Worker

```env
# In .env
WORKER_CONCURRENCY=10  # Erhöhen auf 10
```

**Faustregel für Concurrency:**
- **CPU-bound Workflows**: `Concurrency = CPU Cores`
- **I/O-bound Workflows**: `Concurrency = CPU Cores * 2-3`
- **Mixed**: `Concurrency = CPU Cores * 1.5`

### 3. PostgreSQL-Optimierungen

```sql
-- Ausführen in PostgreSQL
ALTER SYSTEM SET shared_buffers = '256MB';
ALTER SYSTEM SET effective_cache_size = '1GB';
ALTER SYSTEM SET work_mem = '4MB';
SELECT pg_reload_conf();
```

### 4. n8n-Einstellungen

```env
# Logging reduzieren in Produktion
N8N_LOG_LEVEL=warn

# Alte Ausführungen automatisch löschen
EXECUTIONS_DATA_PRUNE=true
EXECUTIONS_DATA_MAX_AGE=336  # 14 Tage
```

---

## 📈 Monitoring

### Queue-Status prüfen

```bash
# Wartende Jobs
docker-compose exec redis redis-cli LLEN bull:n8n:jobs:wait

# Aktive Jobs
docker-compose exec redis redis-cli LLEN bull:n8n:jobs:active

# Fehlgeschlagene Jobs
docker-compose exec redis redis-cli LLEN bull:n8n:jobs:failed
```

### Health Checks

```bash
# PostgreSQL
docker-compose exec postgres pg_isready -U n8n

# Redis
docker-compose exec redis redis-cli ping

# n8n (wenn N8N_METRICS=true)
curl http://localhost:5678/metrics
```

---

## 🔧 Troubleshooting

### Problem: Worker starten nicht

```bash
# Logs prüfen
docker-compose logs n8n-worker-1

# Häufige Ursachen:
# 1. Falscher N8N_ENCRYPTION_KEY
# 2. Keine Verbindung zu PostgreSQL/Redis
# 3. Fehlende Berechtigungen

# Berechtigungen korrigieren
sudo chown -R 1000:1000 /path/to/data/n8n
```

### Problem: Workflows bleiben in Queue hängen

```bash
# Worker-Status prüfen
docker-compose ps

# Failed Jobs inspizieren
docker-compose exec redis redis-cli LRANGE bull:n8n:jobs:failed 0 10

# Queue leeren (VORSICHT: Löscht alle wartenden Jobs!)
docker-compose exec redis redis-cli DEL bull:n8n:jobs:wait
```

### Problem: "Encryption key changed" Error

Stelle sicher, dass `N8N_ENCRYPTION_KEY` in der `.env` für **alle** Services identisch ist und **nie geändert** wurde nach dem ersten Start.

---

## 🔒 Sicherheit

### Wichtige Sicherheitsmaßnahmen

1. **Starke Passwörter generieren**
```bash
# Für POSTGRES_PASSWORD und N8N_ENCRYPTION_KEY
openssl rand -base64 32
```

2. **Umgebungsvariablen schützen**
```bash
# .env nie committen!
echo ".env" >> .gitignore
chmod 600 .env
```

3. **HTTPS erzwingen**
```env
N8N_PROTOCOL=https
N8N_SECURE_COOKIE=true
```

4. **Regelmäßige Backups**
```bash
# PostgreSQL Backup
docker-compose exec postgres pg_dump -U n8n n8n > backup.sql

# n8n Files Backup
tar -czf n8n_backup.tar.gz /path/to/data/n8n/
```

5. **Container-Updates**
```bash
docker-compose pull
docker-compose up -d
```

---

## 📚 Weiterführende Ressourcen

- **n8n Dokumentation**: https://docs.n8n.io
- **n8n Forum**: https://community.n8n.io
- **Docker Compose Docs**: https://docs.docker.com/compose
- **Redis Best Practices**: https://redis.io/docs/management/optimization

---

## 🤝 Beitragen

Beiträge sind willkommen! 

1. Fork das Repository
2. Erstelle einen Feature-Branch
3. Commit deine Änderungen
4. Öffne einen Pull Request

---

## 📄 Lizenz

Dieses Projekt ist unter der MIT-Lizenz lizenziert.

---

## ⚠️ Disclaimer

Dieses Setup dient als **Referenz und Ausgangspunkt**. Passe es an deine spezifischen Anforderungen und Infrastruktur an. Teste ausführlich vor dem Produktionseinsatz!

---

**Viel Erfolg mit deinem High-Performance n8n Setup! 🚀**