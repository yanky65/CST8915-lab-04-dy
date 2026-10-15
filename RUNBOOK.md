# Lab 4 Procedure

The optional local Docker prelab is omitted. Windows commands below transfer the prepared files. All later Bash commands run on the Azure Ubuntu VM. Do not paste Bash commands into Windows PowerShell.

## 1. Copy the prepared files to Azure

On Windows PowerShell, replace the public IP and private-key path:

```powershell
Set-Location "C:\Users\soonr\OneDrive\Documents\01_coding\81_algonquinlive\12_cst8915\lab04\CST8915-lab-04-dy"
$vmIp = "REPLACE_WITH_VM_PUBLIC_IP"
$keyPath = "C:\REPLACE_WITH_YOUR_PRIVATE_KEY_PATH"
ssh -i $keyPath "azureuser@${vmIp}" "mkdir -p ~/CST8915-lab-04-dy"
scp -i $keyPath -r docker-lab docker-compose-lab README.md RUNBOOK.md DEMO.md "azureuser@${vmIp}:~/CST8915-lab-04-dy/"
```

The key is the private key matching the custom public key selected during VM creation. A newly generated unrelated key will not authenticate.

Connect in VS Code using the Remote - SSH extension. Add/select the host using `ssh -i <PRIVATE_KEY_PATH> azureuser@<VM_PUBLIC_IP>`. Open the remote folder `/home/azureuser/CST8915-lab-04-dy`, then open its terminal.

```bash
hostname
whoami
cat /etc/os-release
```

Check the VM's Azure network security rules: allow SSH 22 for your connection and HTTP 80 for browser access. The provided screenshot records the Basics settings, not the networking rules.

## 2. Install Docker Engine using the APT repository

These commands target a fresh Ubuntu VM. If Docker Engine is already installed from the official repository, verify it before deciding whether installation is needed.

```bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo docker --version
sudo docker compose version
sudo docker run hello-world
sudo usermod -aG docker "$USER"
newgrp docker
docker ps
```

If Docker access still requires sudo, reconnect to the VM. Use Docker Engine on the VM; Docker Desktop on Windows does not provide the VM's engine.

## 3. Basic Docker commands (Part 2)

```bash
docker info
docker ps
docker ps -a
docker images
docker pull nginx:latest
docker run -d -p 80:80 --name my-nginx nginx:latest
curl --retry 10 --retry-connrefused --retry-delay 1 http://localhost/
docker logs my-nginx
docker exec -it my-nginx bash
```

Inside the container, run `ls /usr/share/nginx/html`, then `exit`. In the VM terminal:

```bash
docker stop my-nginx
docker start my-nginx
docker stop my-nginx
docker rm my-nginx
docker rmi nginx:latest
```

Removing/stopping Nginx frees host port 80 for the Flask application.

## 4. Build the Flask image (Part 3)

```bash
cd ~/CST8915-lab-04-dy/docker-lab
docker build -t my-python-app:v1 .
docker images
docker run -d -p 80:5000 --name python-app my-python-app:v1
curl --retry 10 --retry-connrefused --retry-delay 1 http://localhost/
curl http://localhost/health
docker inspect python-app
```

Expected: the greeting includes the container hostname, and `/health` returns `OK`. Browse to `http://<VM_PUBLIC_IP>/` on your computer.

## 5. Image history and caching (Part 4)

```bash
docker history my-python-app:v1
docker image inspect my-python-app:v1 --format '{{json .RootFS.Layers}}'
docker build --progress=plain -t my-python-app:v2 .
sed -i 's/Hello from Docker!/Hello from Docker v2!/' app.py
docker build --progress=plain -t my-python-app:v3 .
```

Observe the cached dependency-copy and installation steps. The application-copy step changes after editing `app.py`. Metadata instructions do not all create filesystem layers. The existing `python-app` still runs the v1 image until recreated.

## 6. Writable layers and copy-on-write (Part 5)

```bash
docker run -d -p 5001:5000 --name app1 my-python-app:v3
docker run -d -p 5002:5000 --name app2 my-python-app:v3
docker run -d -p 5003:5000 --name app3 my-python-app:v3
docker exec app1 sh -c 'echo "Container 1 data" > /app/data1.txt'
docker exec app2 sh -c 'echo "Container 2 data" > /app/data2.txt'
docker exec app3 sh -c 'echo "Container 3 data" > /app/data3.txt'
docker exec app1 ls /app
docker exec app2 ls /app
docker exec app3 ls /app
docker ps -s
docker exec app1 sh -c 'echo "# Modified only in app1" >> /app/app.py'
docker exec app1 tail -n 3 /app/app.py
docker exec app2 tail -n 3 /app/app.py
docker diff app1
docker stop python-app
```

Expected: each data file exists only in its own container; the added comment appears only in app1. The final stop frees port 80 for Compose.

## 7. Compose application (Part 6)

```bash
cd ~/CST8915-lab-04-dy/docker-compose-lab
docker compose config
docker compose up -d --build
docker compose ps
curl --retry 10 --retry-connrefused --retry-delay 1 http://localhost/
curl http://localhost/
curl http://localhost/
docker compose logs --tail=20
docker compose logs --tail=10 web
docker compose logs --tail=10 redis
docker compose logs -f
```

Press Ctrl+C to stop following logs. Refresh `http://<VM_PUBLIC_IP>/` in your browser. Expected: the visit count increases; previous requests may mean it starts above 1.

Run these lifecycle commands separately, waiting for the web process to start before each curl:

```bash
docker compose stop
docker compose ps -a
docker compose start
curl --retry 10 --retry-connrefused --retry-delay 1 http://localhost/
docker compose restart
curl --retry 10 --retry-connrefused --retry-delay 1 http://localhost/
```

`restart` restarts existing containers. Apply changed configuration with `up -d`; rebuild changed source with `up -d --build`.

### Verify saved Redis data survives container recreation

```bash
docker compose exec redis redis-cli GET hits
docker compose exec redis redis-cli SAVE
docker compose down
docker compose up -d
docker compose exec redis redis-cli GET hits
curl --retry 10 --retry-connrefused --retry-delay 1 http://localhost/
```

Expected: the saved count is retained and the next request increments it. Do not use `down -v` here; that deletes the named volume.

### Scale the web service

```bash
docker compose -f docker-compose.scale.yml up -d --scale web=3
docker compose -f docker-compose.scale.yml ps
docker compose -f docker-compose.scale.yml port --index 1 web 5000
docker compose -f docker-compose.scale.yml port --index 2 web 5000
docker compose -f docker-compose.scale.yml port --index 3 web 5000
```

For each returned host port, run `curl http://localhost:<PORT>/` in the VM terminal. All three instances share Redis. The public browser URL on port 80 stops working during scaling because the alternative file uses assigned ports.

```bash
docker compose -f docker-compose.scale.yml exec --index 1 web sh
```

Inside the container run `ls /app`, then `exit`. Restore the single-instance configuration:

```bash
docker compose up -d --scale web=1
curl --retry 10 --retry-connrefused --retry-delay 1 http://localhost/
```

## 8. Record and submit

Follow [DEMO.md](DEMO.md). Update the local README with the YouTube link and final reflection answers. If you want to retain the VM's greeting edit locally, copy it back before committing; otherwise keep the initial source so the runbook's cache exercise remains reproducible.

In Windows PowerShell, inside the cloned repository:

```powershell
git status
git add .
git diff --cached --stat
git commit -m "Complete CST8915 Lab 4 submission"
git push origin main
```

Verify the public repository and video in a private browser window. Submit `https://github.com/yanky65/CST8915-lab-04-dy` in Brightspace and verify the submission confirmation. Check the exact deadline in Brightspace.

## 9. Cleanup (Part 7)

Only after the video and submission are verified:

```bash
cd ~/CST8915-lab-04-dy/docker-compose-lab
docker compose down -v
docker rm -f python-app app1 app2 app3
docker rmi my-python-app:v1 my-python-app:v2 my-python-app:v3
```

These commands target the named lab resources. Delete the dedicated `rg-cst8915-lab4` Azure resource group when no longer needed, after checking it contains only resources you intend to remove.

## Troubleshooting

- **Port 80 already allocated:** run `docker ps` and `sudo ss -ltnp '( sport = :80 )'`. Stop the specific conflicting container or service. Do not stop an unrelated application without checking it.
- **Container name already exists:** inspect it with `docker ps -a`. If it is a disposable previous run of this lab, remove that named container before rerunning its creation command.
- **VM curl works but browser fails:** check the VM public IP and Azure inbound TCP 80 rule; use `http://`, not `https://`.
- **Permission denied connecting to Docker:** reconnect after adding your user to the docker group, or use sudo temporarily.
- **Docker daemon unavailable:** run `sudo systemctl status docker`, then `sudo systemctl start docker` if necessary.
- **Compose fails:** run `docker compose config` and `docker compose logs --tail=50` from the Compose application folder.
