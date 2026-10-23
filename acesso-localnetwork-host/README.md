# Fazer o próprio servidor acessar os serviços .localnetwork

Esta pasta guarda a configuração que o host, o computador que roda esse
Docker Swarm, precisa para usar o AdGuard como DNS dos nomes `.localnetwork`
e acessar os serviços pelo Traefik. Isso permite que programas rodando
diretamente no servidor, como o OpenCode, usem `http://9router.localnetwork/v1`.

## Por que precisa disso?

O computador usa o IP `192.168.10.90` na interface `bond0`, que junta o cabo
e o Wi-Fi. O Traefik roda em um contêiner com o IP `192.168.10.91`, na rede
IPvlan `traefik_local_lan`.

Os dois estão na mesma rede local, mas o IPvlan isola a comunicação entre
a interface principal do host e a interface do contêiner ligada a ela.
Por isso, o próprio servidor não conseguia acessar `.91`.

Esses arquivos criam uma interface virtual chamada `local-lan`, ligada ao
`bond0`. Ela funciona como uma interface irmã da usada pelo Traefik e permite
que o host chegue ao `.91`. Ela reutiliza o IP `.90` do computador, com prefixo
`/32`, e recebe uma rota específica para `.91`.

O DNS do Linux também passa a consultar `.91` para os nomes `.localnetwork`.
O Traefik encaminha essas consultas ao AdGuard, que tem a regra
`*.localnetwork → 192.168.10.91`. Depois de descobrir o IP, o programa acessa
o Traefik, que encaminha o pedido ao serviço correspondente.

```text
Consulta DNS: host → local-lan → Traefik .91 → AdGuard
Acesso ao serviço: host → local-lan → Traefik .91 → contêiner do serviço
```

Os outros domínios continuam usando o DNS que já estava configurado no
`bond0`. Outro computador da LAN, como `192.168.10.10`, acessa `.91` normalmente
e só precisa usar `.91` como DNS para resolver esses nomes. Ele não precisa
instalar os arquivos desta pasta.

## O que cada arquivo faz

| Arquivo | Função |
| --- | --- |
| `20-local-lan.netdev` | Define a interface IPvlan `local-lan`. |
| `10-netplan-bond0.network.d/50-local-lan.conf` | Liga essa interface ao `bond0`. |
| `20-local-lan.network` | Configura o IP, a rota até `.91` e o DNS de `.localnetwork`. |
| `05-local-lan.link` | Evita que uma regra de nomes deste computador tente renomear `local-lan` para o nome da placa física. |

Os quatro arquivos são usados nesta configuração. O `README.md` é só a
explicação de como instalar e conferir.

Esta é uma configuração de rede do host. As cópias ficam nesta pasta para
guardar e reinstalar; o Linux usa os arquivos instalados em
`/etc/systemd/network/`.

## Instalação

No servidor atual, os arquivos já estão instalados. Para reinstalar,
entre nesta pasta e execute os comandos abaixo como root:

```bash
install -m 644 05-local-lan.link 20-local-lan.netdev 20-local-lan.network /etc/systemd/network/
install -d /etc/systemd/network/10-netplan-bond0.network.d
install -m 644 10-netplan-bond0.network.d/50-local-lan.conf /etc/systemd/network/10-netplan-bond0.network.d/
udevadm control --reload
networkctl reload
networkctl reconfigure bond0
```

O arquivo `50-local-lan.conf` complementa a configuração de `bond0` gerada
pelo Netplan. Assim, a interface principal continua sendo configurada pelo
Netplan e passa a ter a interface `local-lan` ligada a ela.

## Como conferir se está funcionando

```bash
networkctl status local-lan
resolvectl status local-lan
getent ahostsv4 9router.localnetwork
curl -i http://9router.localnetwork/v1/models
```

O primeiro comando deve mostrar a interface `local-lan` configurada. O segundo
deve mostrar `192.168.10.91` como DNS e `~localnetwork` como domínio atendido.
O `getent` deve retornar `192.168.10.91` para `9router.localnetwork`.

O `curl` sem chave retorna HTTP 401, com a mensagem de que a API key é
necessária. Isso confirma que o pedido chegou à API do 9router. Com uma chave
válida, `/v1/models` retorna HTTP 200 e a lista de modelos.

## Depois de reiniciar

O Linux carrega os arquivos de `/etc/systemd/network/` automaticamente.
Reiniciar o computador ou o Docker não apaga essa configuração.

Se mudar o IP do host `.90`, o IP do Traefik `.91` ou o nome `bond0`, ajuste
os arquivos e instale novamente. O Traefik e o AdGuard precisam estar rodando
para atender as consultas e os acessos.
