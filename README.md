# Traefik (Docker Swarm) — Infraestrutura

Este repositório contém a configuração para executar o Traefik (edge reverse proxy / ingress) em um cluster Docker Swarm. A configuração usa:
- Traefik v (traefik:latest)
- Resolvedor ACME via Cloudflare DNS (DNS challenge)
- Deployment em modo `global` (uma instância por nó)
- Dashboard disponível no manager via `http://localhost:8080/dashboard/`
- Cloudflare API Token fornecido como Docker Secret (cloudflare_api_token)

Estrutura do projeto
```txt
infra/
├── traefik/
│   ├── traefik.yaml       # configuração estática do Traefik
│   ├── dynamic/           # (opcional) configs dinâmicas (middlewares, routers, tls)
│   └── acme/acme.json     # arquivo ACME (deve ser criado antes do deploy)
docker-compose.yml         # stack do Swarm (na raiz do projeto)
.env                       # variáveis de ambiente(junto com o compose)
```

Como o projeto funciona (resumo técnico)
- Traefik lê a configuração estática (traefik.yaml) e o provider swarm.
- Para emitir certificados via Let's Encrypt usa DNS challenge com Cloudflare; o token é lido via Docker Secret dentro do container.
- O serviço é executado em modo global: cada nó executa uma instância do Traefik. As portas 80/443/8080 estão publicadas em modo host.
- O dashboard é servido no entryPoint `traefik` (porta 8080) e está publicado em host para permitir acesso via `localhost` no manager.

Pré‑requisitos
- Docker Engine instalado e Swarm inicializado (no manager): docker swarm init
- Ter criado o Docker Secret com o token Cloudflare no manager:
   ```bash
   read -sp "Digite seu Cloudflare API Token: " CF_TOKEN
   echo "$CF_TOKEN" | docker secret create cloudflare_api_token -
   unset CF_TOKEN
   ```

- Ter criado o arquivo acme.json com permissões seguras:
  ```bash
  mkdir -p infra/traefik/acme
  touch infra/traefik/acme/acme.json
  chmod 600 infra/traefik/acme/acme.json
  ```
- Verificar que os caminhos relativos do docker-compose.yml apontam corretamente para ./infra/traefik/...

Passos para colocar em funcionamento
1) Prepare acme.json e secret (se ainda não feitos)
   ```bash
   mkdir -p infra/traefik/acme
   touch infra/traefik/acme/acme.json
   chmod 600 infra/traefik/acme/acme.json
   ```

2) Ajuste traefik.yaml
   - Substitua o campo `email` do resolver ACME pelo e‑mail de contato 

3) Deploy do stack (executar no diretório raiz onde está docker-compose.yml)
   `docker stack deploy -c docker-compose.yml traefik`

4) Verificar se o serviço iniciou corretamente
   ```bash
   docker stack services traefik
   docker service ps traefik_traefik
   docker logs <TRAEFIK_CONTAINER_ID>   # ver logs para erros ACME/Cloudflare
   ```

Acesso ao dashboard
- No manager (host onde está o Swarm manager) abra no navegador:
  http://localhost:8080/dashboard/
- Observação: a porta 8080 foi publicada em mode: host. Garanta regras de firewall/ACLs para limitar acesso.

Serviços locais
- O host local `fred.localnetwork` está exposto em `http://fred.localnetwork:8181` via Traefik, sem Cloudflare Tunnel.
- O backend desse host aponta para a API do Fred via a rede `traefik-local`, usando o alias `fred-api`.
- O host local `waha.localnetwork` expõe o dashboard do WAHA em `http://waha.localnetwork:8181` pela mesma rede `traefik-local`, sem porta publicada no host.
- Para acessar no Windows/WSL, adicione `fred.localnetwork` e `waha.localnetwork` no `hosts` local apontando para o IP da máquina que publica o Traefik.

Notas de segurança e recomendações
- Tokens (Cloudflare API Token) devem ser gerenciados via Docker Secrets e nunca commitados.
- Use um e‑mail de função (ops@, infra@) para ACME em ambientes de produção; e‑mails pessoais funcionam, mas perdem rastreabilidade.
- acme.json deve ter permissão 600 e ser persistido (volume). Evite múltiplas instâncias concorrendo sobre um arquivo local não compartilhado.
- Publicar 8080 em todos os nós (mode: host + deploy: global) expõe o dashboard em cada nó. Se quiser que apenas o manager exponha o dashboard, ajuste placement/constraints no compose para limitar execução.

Troubleshooting rápido
- ACME falha: verifique token Cloudflare e permissões, zona correta no token.
- Sem certificados: verifique logs do Traefik e checar se acme.json está gravando alterações.
- Service não cria: verifique constraints e se o nó tem recursos.

Comandos úteis
- Listar secrets: docker secret ls
- Ver serviços do stack: docker stack services traefik
- Ver containers do serviço: docker service ps traefik_traefik
- Acessar shell de um container Traefik: docker exec -it <CONTAINER_ID> sh

Contato / manutenção
- Mantenha o token rotacionado periodicamente.
- Atualize a imagem Traefik quando estabilidade for validada em ambiente de teste.
