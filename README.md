# DNS delegation lab

A tiny private DNS tree in Docker: a root server, a `furry.` TLD, two
second-level domains, a recursive resolver, and a client to poke at it all.

| Container | IP | Software | Role |
|-----------|----|----------|------|
| `dnslab-root` | 10.53.0.2 | Knot DNS | Authoritative for `.` (`a.root-servers.net`) |
| `dnslab-tld-furry` | 10.53.0.3 | Knot DNS | Authoritative for `furry.` (`ns1.nic.furry`) |
| `dnslab-pawprint` | 10.53.0.4 | Knot DNS | Authoritative for `pawprint.furry.` |
| `dnslab-fox` | 10.53.0.5 | Knot DNS | Authoritative for `fox.furry.`, **not delegated** (exercise) |
| `dnslab-resolver` | 10.53.0.53 | Knot Resolver 6 | Recursive resolver, root hints point at 10.53.0.2 |
| `dnslab-client` | 10.53.0.100 | Alpine + `dig` | The "laptop"; `/etc/resolv.conf` → 10.53.0.53 |

Knot Resolver is recursive-only, so the authoritative servers use its sibling,
Knot DNS. Both are from CZ.NIC.

## Run it

```bash
docker compose up -d --build
docker exec dnslab-client dig +trace www.pawprint.furry
```

The resolver is also published on the host at `127.0.0.1:1053`:

```bash
dig @127.0.0.1 -p 1053 _motd.pawprint.furry TXT +short
```

Stop with `docker compose down`.

## Layout

```
docker-compose.yml
servers/<name>/knot.conf      Knot DNS config (one zone per server)
servers/<name>/*.zone         Zone files, mounted read-only into the container
resolver/config.yaml          Knot Resolver config (DNSSEC validation off)
resolver/root.hints           Lab root hints, replaces the real ones
client/                       Client image (bind-tools + knot-utils) and resolv.conf
slides/lecture.md             presenterm slides
```

## Exercise: delegate fox.furry

1. Confirm it's broken: `docker exec dnslab-client dig fox.furry TXT` → `NXDOMAIN`
2. Confirm the server works: `docker exec dnslab-client dig +short @10.53.0.5 fox.furry TXT`
3. In `servers/tld-furry/furry.zone`, bump the SOA serial and uncomment the `fox` NS and `ns1.fox` glue lines
4. Reload: `docker exec dnslab-tld-furry knotc zone-reload furry.`
5. Query again. Still `NXDOMAIN` for up to 5 minutes (negative caching). Skip the wait with `docker exec dnslab-resolver kresctl cache clear`

To reset, re-comment the two lines, bump the serial again, and reload.

Other things to try:

* Lame delegation: point `ns1.fox` glue at `10.53.0.4`, reload, clear the cache, query
* Secondary server: add a container that pulls `pawprint.furry` from 10.53.0.4 by zone transfer, then see why a forgotten serial bump means the secondary never gets your change
* Add a third level: delegate `lab.pawprint.furry` from the `pawprint` zone to a new container

## Slides

```bash
cd slides
presenterm -x lecture.md      # -x enables the live "+exec" demo blocks (Ctrl+E to run)
```

Mermaid diagrams need `mmdc`. `mise.toml` pins it with the latest Node:

```bash
mise install
# mise skips puppeteer's postinstall, so fetch its headless Chrome once:
mise exec -- npx -y @puppeteer/browsers install chrome-headless-shell@157.0.8103.78 --path ~/.cache/puppeteer
```

If a newer mermaid-cli wants a different Chrome version, `mmdc` prints the exact version it needs.
The `+exec` blocks call `docker exec dnslab-*`, so start the lab first.
