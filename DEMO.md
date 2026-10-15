# Part 6 Demo - Maximum Five Minutes

Target 4 minutes 30 seconds. Rehearse after building the images so downloads do not consume recording time. Show the remote VS Code terminal and a browser at `http://<VM_PUBLIC_IP>/`. Keep the terminal readable and avoid showing private keys or account credentials.

## Before recording

Run on the VM:

```bash
cd ~/CST8915-lab-04-dy/docker-compose-lab
docker compose up -d --build --scale web=1
curl --retry 10 --retry-connrefused --retry-delay 1 http://localhost/
```

Keep `app.py`, `Dockerfile`, and both Compose files open. Verify browser access. The counter need not start at 1; demonstrate that it increases.

## Recording outline

| Time | Show | Explain |
| --- | --- | --- |
| 0:00-0:20 | `hostname`, `pwd` in the remote terminal | Introduce the lab and identify the Azure VM. |
| 0:20-1:05 | `app.py`, Dockerfile, `docker-compose.yml` | Flask increments Redis's `hits`; `redis` is the internal service hostname; identify the port, network, named volume, and readiness check. |
| 1:05-1:35 | `docker compose up -d`, `docker compose ps` | Both services run together; Redis is healthy. |
| 1:35-2:00 | Browser and three refreshes | Each request increases the shared visit count. |
| 2:00-2:20 | `docker compose logs --tail=10` | Show logs from the web and Redis services. |
| 2:20-3:05 | GET, SAVE, down, up, GET, curl below | Containers are recreated while saved data remains in the named volume. |
| 3:05-4:10 | Scaling configuration, scale command, service list, assigned ports, curls | Three web instances share one Redis instance. |
| 4:10-4:30 | Optional shell inspection and restoration | Inspect one web container; restore the public port 80 configuration. |

### Persistence commands

```bash
docker compose exec redis redis-cli GET hits
docker compose exec redis redis-cli SAVE
docker compose down
docker compose up -d
docker compose exec redis redis-cli GET hits
curl --retry 10 --retry-connrefused --retry-delay 1 http://localhost/
```

### Scaling commands

Use the scaling file by itself:

```bash
docker compose -f docker-compose.scale.yml up -d --scale web=3
docker compose -f docker-compose.scale.yml ps
docker compose -f docker-compose.scale.yml port --index 1 web 5000
docker compose -f docker-compose.scale.yml port --index 2 web 5000
docker compose -f docker-compose.scale.yml port --index 3 web 5000
```

Curl each actual assigned port in the VM terminal: `curl http://localhost:<PORT>/`. Show the increasing shared count. You do not need to open those assigned ports publicly in Azure.

Optional inspection:

```bash
docker compose -f docker-compose.scale.yml exec --index 1 web sh
```

Run `ls /app` and `exit`. Restore:

```bash
docker compose up -d --scale web=1
```

## After recording

1. Check the recording is no longer than five minutes and the text is readable.
2. Upload to YouTube (Unlisted is suitable when the instructor can access the link).
3. Verify playback in a private browser window.
4. Put the working link in README.md and review its draft answers and TODOs.
5. Push to the public submission repository and submit its URL in Brightspace.
