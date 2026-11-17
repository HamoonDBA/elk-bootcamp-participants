# ELK workshop - Hamrah-e Aval Data Engineering Bootcamp 2

Class material for the ELK workshop (Elasticsearch, Logstash and Kibana 8.19.5).

Before class: install Docker and pull the three `8.19.5` images:

```bash
docker pull docker.elastic.co/elasticsearch/elasticsearch:8.19.5
docker pull docker.elastic.co/kibana/kibana:8.19.5
docker pull docker.elastic.co/logstash/logstash:8.19.5
```

During class, new files are added to this repository after some blocks.
Get them with:

```bash
git pull
```

## Layout

| Path | What |
|---|---|
| `data/search_logs.jsonl` | request log of the search service (JSON lines) |
| `data/nginx_access.log` | Nginx access log of the gateway in front of the service |
| `docs/elk-workshop.html` | workshop slides (open in a browser) |
| `docker-compose.yml`, `.env` | local stack: 3 Elasticsearch nodes, Kibana, Logstash |
| `docker-compose.logstash-shell.yml` | interactive Logstash shell for running your own pipelines |

Files added during class go in new folders (`logstash/`, `mapping/`, `devtools/` and so on).

## Run the stack

The stack needs about 5.5 GB of memory in Docker.

```bash
docker compose up -d          # es01 :9200, es02 :9201, es03 :9202, Kibana :5601 (user elastic / changeme)
```

Open Kibana at http://localhost:5601. `GET _cat/nodes?v` must list three nodes.

The first start runs a `certs` job that creates TLS certificates between the
nodes. HTTP stays plain, so Kibana, Logstash and curl need no certificate.

## Run Logstash

Put your pipelines in `logstash/` (create the folder if it does not exist).
Create the index in Dev Tools first.

Interactive shell (start the main stack first):

```bash
docker compose -f docker-compose.yml -f docker-compose.logstash-shell.yml run --rm logstash-shell
```

`./logstash` is mounted at `/usr/share/logstash/host-config` inside the shell.
Ports `3000/udp`, `5000`, `8080` and `9600` (monitoring API) are published.

One-shot service (reads `logstash/pipeline/`):

```bash
docker compose --profile ingest up logstash
```

Logstash does not exit when the file is done. When the document count stops
growing, stop it with Ctrl+C.

Logstash joins every `.conf` file in one directory into a single pipeline.
Keep pipelines for different sources in different directories.

## Stop and clean up

```bash
docker compose down -v        # then delete the volumes/ folder
```

The password `changeme` is for this local class stack only.

## Things that trip people up

- **Logstash reads nothing on the second run:** the `file` input remembers
  its position. Set `sincedb_path => "/dev/null"` (`NUL` on Windows).
- **The data file disappears:** in `mode => "read"` the default
  `file_completed_action` is `delete`. Set it to `log`.
- **Kibana shows no data:** the data covers 2026-11-03. Set the time picker to an absolute range.
- **`localhost` inside Docker:** it is the container itself. Use the service
  names (`http://es01:9200`).
- **A node exits at start with `max virtual memory areas vm.max_map_count [65530] is too low`:**
  set `vm.max_map_count=262144` on the host (Linux, and the Docker VM on some setups).
- **A node exits with `cannot read configured PEM certificate_authorities`:** the `certs`
  job did not finish. `docker compose logs certs`, then `docker compose down -v` and start again.
- **Linux:** run `sudo chown -R 1000:0 volumes` once; the containers run as uid 1000.
