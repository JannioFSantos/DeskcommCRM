# Coolify — uma instalação DeskcommCRM por cliente

Este é o caminho para criar **um recurso Coolify por cliente**. Cada recurso
tem seu domínio, variáveis e volumes próprios; ele não mistura clientes na
mesma instalação do CRM.

O arquivo [`docker-compose.coolify.yml`](../docker-compose.coolify.yml) sobe a
pilha essencial: app, worker, scheduler, WAHA, Redis e o adaptador HTTP do
Redis. O Coolify fica responsável pelo clone do repositório, rede do recurso,
proxy, certificado HTTPS e ciclo de deploy.

> Este caminho usa imagens publicadas e versionadas. Não configure o Coolify
> para compilar uma cópia de desenvolvimento do CRM na VPS de um cliente.

## O que preparar antes do primeiro deploy

- Um domínio ou subdomínio exclusivo, como `crm.cliente.com.br`, com registro
  DNS `A` apontando para o IP da VPS.
- Um projeto Supabase exclusivo para o cliente, com o schema/base do
  DeskcommCRM aplicado e as URLs de autenticação configuradas para o domínio.
- As imagens `app`, `worker` e `scheduler` da **mesma release** publicadas em
  um registry que a VPS pode acessar.
- As chaves do Supabase, IA e WhatsApp guardadas nas variáveis do recurso no
  Coolify — nunca no repositório.

O Compose não cria schema nem conta administrativa no Supabase. Para esse
provisionamento inicial, use o fluxo do
[kit de instalação](../hostgator-setup-kit/README.md) ou aplique o baseline do
projeto de forma controlada no projeto Supabase do cliente. Só depois suba o
recurso no Coolify.

## Criar o recurso no Coolify

1. Crie uma **Application** a partir do repositório Git e escolha o Build Pack
   **Docker Compose**.
2. Use `docker-compose.coolify.yml` como Docker Compose Location. Não selecione
   `docker-compose.prod.yml`, pois ele contém o Caddy pensado para uma VPS sem
   Coolify.
3. Em Environment Variables, informe ao menos as imagens versionadas e as
   variáveis já preparadas para o cliente:

   ```dotenv
   APP_IMAGE=ghcr.io/seu-registro/deskcommcrm:X.Y.Z
   WORKER_IMAGE=ghcr.io/seu-registro/deskcomm-worker:X.Y.Z
   SCHEDULER_IMAGE=ghcr.io/seu-registro/deskcomm-scheduler:X.Y.Z
   APP_PULL_POLICY=missing
   WORKER_PULL_POLICY=missing
   SCHEDULER_PULL_POLICY=missing

   NEXT_PUBLIC_APP_URL=https://crm.cliente.com.br
   NEXT_PUBLIC_ADMIN_URL=https://crm.cliente.com.br
   WAHA_API_BASE_URL=http://waha:3000
   WAHA_WEBHOOK_BASE_URL=http://app:3000
   UPSTASH_REDIS_REST_URL=http://srh:80
   UPSTASH_REDIS_REST_TOKEN=<o mesmo valor de SRH_TOKEN>
   ```

   Inclua também os segredos obrigatórios do CRM, como as chaves do Supabase,
   `INTERNAL_SECRET`, `SRH_TOKEN`, `WAHA_API_KEY`, `WAHA_API_KEY_SHA512` e
   `WAHA_HMAC_SECRET`. Use o [template de ambiente](../.env.example) e o kit
   como referência dos valores; não copie segredos de outro cliente.

4. Em **Domains** do serviço `app`, informe
   `https://crm.cliente.com.br:3000`. O sufixo `:3000` escolhe a porta interna
   do contêiner; quem acessa o CRM continua usando apenas
   `https://crm.cliente.com.br`.
5. Salve, faça o deploy e espere o health check do `app` ficar saudável antes
   de abrir o domínio.

O Coolify grava as variáveis no `.env` do recurso; o Compose as entrega aos
serviços necessários. Os volumes `waha-data` e `waha-media` são persistentes e
pertencem somente a esse recurso. Não os apague: eles guardam as sessões e as
mídias do WhatsApp.

## O que não deve ser configurado no painel

Não publique manualmente `80`, `443` ou `3000` no host. Não adicione Caddy,
labels `traefik.*` nem uma rede externa do proxy ao Compose. O Coolify gera a
rota, conecta o proxy à aplicação e emite o certificado a partir do domínio do
serviço `app`.

O adaptador também não inclui, por enquanto, a chamada de voz WaCalls nem a
telefonia SIP. Esses módulos exigem portas UDP diretas e devem ganhar um fluxo
isolado antes de serem habilitados no Coolify.

## Atualizar uma instalação

1. Espere uma release publicada com as três imagens no mesmo `X.Y.Z`.
2. Troque `APP_IMAGE`, `WORKER_IMAGE` e `SCHEDULER_IMAGE` para essa release no
   recurso daquele cliente.
3. Faça o deploy pelo Coolify e verifique o domínio e o health check.

Não acompanhe `main`, `latest` ou `stable` em uma instalação de cliente e não
ative auto-deploy para mudar código de produção sem uma release. Assim, cada
cliente fica em uma versão identificável e a atualização é uma decisão
reversível.

## Limites deste primeiro adaptador

O deploy Git do Coolify agora é o dono do runtime. Por isso não execute
`hostgator-setup-kit/update.sh` sobre a mesma instalação: o script e o Coolify
tentariam gerenciar os mesmos contêineres. Continue usando o kit para o
provisionamento inicial do Supabase até existir uma etapa de bootstrap segura e
idempotente no próprio Coolify.
