# DNS Search Domains in Docker Compose

Docker containers normally inherit DNS configuration from the Docker host. When an application needs to resolve short hostnames on a local network, a DNS search domain can be configured explicitly in Docker Compose.

## Configuration

Add the following to the service:

```yaml
dns_search:
  - <LOCAL_DNS_DOMAIN>
dns_opt:
  - ndots:5
```

For example:

```yaml
services:
  application:
    image: <IMAGE>
    dns_search:
      - <LOCAL_DNS_DOMAIN>
    dns_opt:
      - ndots:5
```

## What `dns_search` does

The search domain tells the resolver which domain to append when resolving an unqualified hostname.

For example, with:

```yaml
dns_search:
  - lan.example
```

an application resolving:

```text
mydevice
```

can resolve:

```text
mydevice.lan.example
```

without requiring the application to use the fully qualified hostname.

## What `ndots:5` does

`ndots` controls when a DNS name is considered sufficiently qualified to be queried as an absolute name.

With:

```yaml
dns_opt:
  - ndots:5
```

a hostname containing fewer than five dots is eligible for the configured search domain.

Therefore a short name such as:

```text
mydevice
```

is tried with the search domain:

```text
mydevice.<LOCAL_DNS_DOMAIN>
```

This is useful when applications or command-line tools use short hostnames.

## Verify from inside the container

Enter the running container:

```bash
docker exec -it <CONTAINER> sh
```

Then test the hostname:

```bash
getent hosts <HOSTNAME>
```

or:

```bash
nslookup <HOSTNAME>
```

If the configuration is working, the short hostname should resolve to the corresponding address in the configured DNS domain.

## DNS versus mDNS

This configuration uses **normal DNS**.

It is therefore different from the `.local` mechanism used by mDNS:

```text
DNS
  mydevice
    ↓ search domain
  mydevice.<LOCAL_DNS_DOMAIN>

mDNS
  mydevice.local
```

Do not add `.local` merely because a hostname does not resolve from a container. First determine whether the device is actually registered in conventional DNS or is using mDNS.

## Troubleshooting

If the short hostname does not resolve:

1. Check the Compose configuration.
2. Recreate the container after changing the DNS settings.
3. Inspect the resolver configuration inside the container:

```bash
cat /etc/resolv.conf
```

Look for entries corresponding to the configured search domain and resolver options.

4. Test the fully qualified hostname:

```bash
getent hosts <HOSTNAME>.<LOCAL_DNS_DOMAIN>
```

5. Test the short hostname:

```bash
getent hosts <HOSTNAME>
```

If the fully qualified name works but the short name does not, the problem is likely the search-domain or `ndots` configuration rather than DNS connectivity itself.

## Recreating the container

Changes to `dns_search` or `dns_opt` apply when the container is created. Recreate the container after changing them:

```bash
docker compose up -d --force-recreate
```

## Security and portability

The search domain is environment-specific. Public Compose examples should use a placeholder rather than exposing an actual private DNS domain:

```yaml
dns_search:
  - <LOCAL_DNS_DOMAIN>
```

The same applies to private DNS server addresses, internal hostnames, and other network-specific information.
