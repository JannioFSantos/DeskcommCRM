# Coolify — operar o DeskcommCRM em uma VPS compartilhada

Este guia cobre uma VPS administrada pelo **Coolify**, com o Traefik dele nas
portas 80 e 443. Hoje, o caminho suportado é deixar o Coolify cuidar do proxy,
DNS e HTTPS, e deixar o **kit de instalação do DeskcommCRM** cuidar da pilha do
produto, das migrações, backups e atualizações.

> O deploy por repositório/Git do Coolify **ainda não substitui** o kit neste
> projeto. Não o trate como equivalente até existir uma configuração do Coolify
> versionada e testada no repositório.

## Antes de instalar: escolha o modelo de clientes

| Modelo | Quando faz sentido | Consequência operacional |
|---|---|---|
| **Uma instalação multiempresa** | Você opera uma plataforma e atende todas as empresas dentro dela. | Um domínio público e uma instalação; separe os clientes por organizações e permissões do CRM. |
| **Uma instalação por cliente** | Cada cliente precisa de domínio, credenciais, banco ou operação próprios. | Repita a instalação em diretórios distintos, com domínio, segredos, dados e backup próprios. |

Uma VPS pode hospedar mais de uma instalação, mas isso não transforma 4 GB em
capacidade para todos os clientes: a referência de memória do projeto é para
uma pilha completa. Dimensione a máquina depois de medir o uso de cada CRM e de
cada sessão de WhatsApp. Não compartilhe volumes, chaves ou projeto Supabase
entre clientes sem uma política explícita de isolamento.

## Caminho suportado na VPS com Coolify

1. Deixe o Coolify e o Traefik dele em execução. Não pare o proxy para liberar
   as portas 80/443: isso interrompe os demais aplicativos da VPS.
2. Para cada cliente, crie um registro DNS `A` do domínio escolhido apontando
   para o IP público da VPS. Aguarde a propagação antes de validar o HTTPS.
3. Acesse a VPS por SSH e instale cada cliente em uma pasta diferente. Exemplo:

   ```bash
   git clone --depth 1 https://github.com/JannioFSantos/DeskcommCRM.git deskcomm-acme
   cd deskcomm-acme
   bash hostgator-setup-kit/install.sh
   ```

4. Responda o domínio, as credenciais do Supabase e as demais perguntas do
   instalador. Ele detecta o Traefik do Coolify, grava
   `REVERSE_PROXY=traefik` e usa o override que desativa o Caddy do produto.
5. Abra `https://seu-dominio-do-cliente` e conclua o primeiro acesso. Se usar
   autenticação por e-mail, confirme também a URL do site e as URLs de
   redirecionamento no Supabase. O token opcional do Supabase permite que o
   instalador faça essa configuração durante o processo.

O instalador pode pedir confirmação quando não consegue provar qual processo
está atendendo as portas públicas. Leia a informação exibida e não force a
decisão se houver outro proxy na VPS. Em uma execução não interativa, declare
`REVERSE_PROXY=traefik` no `.env` somente depois de confirmar que o Traefik do
Coolify é o proxy correto.

## Regras para repetir a instalação por cliente

- Use uma pasta exclusiva por cliente (`deskcomm-acme`, `deskcomm-beta` etc.).
  O nome da pasta participa do nome do projeto Docker; reutilizá-lo pode fazer
  uma operação alcançar a pilha errada.
- Use um domínio exclusivo por instalação e mantenha um inventário com cliente,
  domínio, pasta, projeto Supabase e responsável pelo backup.
- Prefira um projeto Supabase por cliente quando houver exigência de isolamento
  de dados. Nunca reutilize as credenciais de produção de um cliente em outro.
- Acompanhe CPU, RAM, disco, filas e sessões do WhatsApp por instalação.
  Reserve capacidade para atualizações e para a recuperação de backups.
- Faça backup e teste de restauração por cliente. Não execute
  `docker compose down -v`: esse comando remove volumes e pode apagar dados.

## Atualizações e operação contínua

Para uma instalação criada pelo kit, atualize pelo fluxo do próprio produto ou
com `bash hostgator-setup-kit/update.sh` dentro da pasta daquele cliente. O
script preserva o modo de proxy externo e aplica os arquivos de Compose
necessários.

Não habilite o auto-deploy Git do Coolify para assumir essa mesma pilha e não
rode `docker compose up` manualmente com apenas o compose base. Em VPS com
Traefik externo, o override de Traefik é necessário; sem ele o contêiner pode
ficar saudável internamente, mas o domínio responder 404 no proxy.

## Por que o deploy Git nativo do Coolify ainda não é o caminho documentado

O Coolify consegue construir aplicações e arquivos Docker Compose a partir de
um repositório, mas o DeskcommCRM também precisa provisionar variáveis,
migrações, administração, backup e um fluxo de atualização compatível. O
compose padrão do projeto ainda inclui Caddy; em uma VPS com Coolify ele precisa
do override específico do Traefik. Portanto, um adaptador nativo do Coolify só
deve ser anunciado depois de ser versionado, testado em instalação nova e
validado em atualização e restauração.

Até lá, use o Coolify como proxy e painel da VPS, e o kit do DeskcommCRM como
orquestrador da instalação do CRM. Consulte também o
[guia do kit](../hostgator-setup-kit/README.md#vps-que-já-vem-com-proxy-próprio-hostinger-coolify-dokploy)
e o [runbook de deploy](runbooks/deploy.md).
