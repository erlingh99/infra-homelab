## Infra repo

Sets up pihole, caddy, tailscale and copyparty

create an .env file at the root with HOST_IP, TS_AUTHKEY, LAN_SUBNET, WEBSERVER_PASSWORD and COPYPARTY_ROOT set

Caddy issues the https certificates itself, from its own CA (the "local_certs"
option). Start everything:
```
docker compose up -d
```
Then grab the root certificate that Caddy generated on first start:
```
docker compose exec caddy cat /data/caddy/pki/authorities/local/root.crt > root_ca.crt
```
"root_ca.crt" is the certificate for https. It must be installed on all devices that
uses Caddy websites if they want to avoid warnings. If the caddy-data volume is ever
lost, Caddy makes a new CA and every device has to install the new root_ca.crt.
To add another website, add the container name to the Caddyfile and reload Caddy
```
docker exec caddy caddy reload --config /etc/caddy/Caddyfile
```
All tailscale stuff must be approved in the tailscale admin panel on the web


### copyparty

File sharing at https://files.lan (or port 3923 directly), no login: everyone gets
full read/write/move/delete on
`COPYPARTY_ROOT` (the host folder to share, e.g. `COPYPARTY_ROOT=/srv/files`).
Since it is wide open to anyone who can reach it, keep it off the internet.
Settings live in `copyparty.conf`, mounted read-only into the container's `/cfg`;
restart the container to apply changes to it.

The container runs as `COPYPARTY_UID:COPYPARTY_GID` (default `1000:1000`), which
must be able to write `COPYPARTY_ROOT`. Uploads are created 666 and new folders
777 so other users and services on the host can keep editing and adding files in
there. Existing files that copyparty should be able to modify need to be writable
by that uid/gid as well:
```
sudo chown -R 1000:1000 "$COPYPARTY_ROOT"
sudo chmod -R a+rwX "$COPYPARTY_ROOT"
```

Deduplication is on, as `dedup` plus `reflink`. Never set `dedup` on its own
here: alone it deduplicates with symlinks, and editing such a file from any
other program would change every copy of it. `reflink` makes it a filesystem
copy-on-write clone instead, which is a real independent file. That needs a CoW
filesystem (btrfs, xfs with reflink=1, zfs 2.2+); check with
`cp --reflink=always a b` in `COPYPARTY_ROOT`. On ext4 the clone just fails and
copyparty writes a full copy, so it stays safe, only without the space savings.

Share links are enabled and served under https://files.lan/share. Since there
are no accounts, `shr-who: a` in the config is what lets anonymous visitors
create them (the copyparty default only allows logged-in users, so shares would
be impossible here). The index db, thumbnails and the share db all live in the
`copyparty-cfg` docker volume, not in `COPYPARTY_ROOT`, and the index is
rescanned hourly so files dropped in by other processes turn up in search.
