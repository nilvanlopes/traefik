# Traefik no Docker Swarm

A stack `traefik` executa um único Traefik no nó `pyuserver`. O mesmo processo
atende os serviços públicos pelo Cloudflare Tunnel e os serviços locais pela
entrada `traefik_local`, no endereço `192.168.10.91`.

## Entradas e redes

- `websecure`: HTTPS na porta 443, com middleware `tunnel-only` permitindo
  conexões da rede Docker `traefik-public` (`10.0.1.0/24`). O Tunnel acessa
  `https://traefik.${TRAEFIK_DOMAIN}:443` por essa rede, inclusive quando o
  navegador entrou na Cloudflare por HTTP. A descoberta Swarm usa o socket proxy.
- `traefik_local`: HTTP em `192.168.10.91:80`, para `adguard.localnetwork`,
  `fred.localnetwork` e `waha.localnetwork`. A raiz do WAHA encaminha a `/dashboard/`.
- `dns_tcp` e `dns_udp`: DNS em `192.168.10.91:53`, encaminhado ao AdGuard.

Não há publicação de portas no host `.90`, porta 8181 nem segunda stack de
proxy local. Os routers locais só estão associados à entrada `traefik_local`;
eles mantêm os logins nativos e aplicam a allowlist LAN e CrowdSec.

O Traefik carrega diretamente `infra/traefik/traefik.yml`, sem template ou
descoberta de IP. A porta 80 atende exclusivamente a entrada local `.91:80`;
não há entrypoint público `web`. Se mudar a sub-rede `traefik-public`, atualize
a allowlist `tunnel-only`. Ela verifica a origem da conexão, sem usar
`X-Forwarded-For` para autorizar o acesso.

O serviço permanece global, mas a restrição `node.hostname == pyuserver`
permite exatamente uma tarefa. DNSRR evita alocar outro IP LAN para uma VIP;
a atualização stop-first libera `.91` antes de iniciar a tarefa substituta.

## Deploy e operação

Na raiz do repositório:

```bash
make deploy-adguard
make deploy-traefik
make logs-traefik
docker service ps traefik_traefik
```

O deploy prepara as redes `traefik-public`, `traefik-local` e
`traefik_local_lan`. Preencha `traefik/.env` e mantenha o secret
`cloudflare_api_token` existente. ACME e logs continuam nos volumes persistentes.
O dashboard público permanece em `https://traefik.${TRAEFIK_DOMAIN}/dashboard/`,
protegido pelo Authentik e CrowdSec.

## Windows e IPv6

Configure `192.168.10.91` como DNS e mantenha IPv6 habilitado. As instruções
NRPT para `.localnetwork` estão no README da raiz.

O Makefile prepara a rede IPvlan `traefik_local_lan`, ligada a `bond0`, com
alocação exclusiva de `.91`. Reserve esse endereço fora do DHCP do roteador.
O Linux isola esse IP da interface principal do host `.90`. O host acessa
`.91` pela interface IPvlan `local-lan`, configurada de forma persistente em
`/etc/systemd/network/`. O domínio `.localnetwork` usa o AdGuard via `.91`.
Os arquivos e instruções estão em [acesso-localnetwork-host](acesso-localnetwork-host/README.md).
Os logs DNS do AdGuard mostram o endereço overlay do Traefik; os logs HTTP
mantêm o IP do cliente LAN.
