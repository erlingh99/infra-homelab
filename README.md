## Infra repo

Sets up pihole, step-ca, caddy and tailscale

create a password.txt file with the step-ca password
create an .env file at the root with HOST_IP, TS_AUTHKEY, LAN_SUBNET and WEBSERVER_PASSWORD set

at first start only start step-ca
```
docker compose up step-ca
# Wait for "Serving ACME" in the logs, then extract the root cert
docker compose exec step-ca cat /home/step/certs/root_ca.crt > root_ca.crt
docker compose down
```
Now all services can be started.
```
docker compose up -d
```
"root_ca.crt" is the certifacte for https. I must be installed on all devices that uses Caddy websites if they want to avoid warning.
To add another website, add the container name to the Caddyfile and reload Caddy
```
docker exec caddy caddy reload --config /etc/caddy/Caddyfile
```
All tailscale stuff must be approved in the tailscale admin panel on the web

