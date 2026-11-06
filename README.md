# laboratorio-http-aws

## Checklist do professor — executar antes da aula

Manual de uma aula presencial de **90 minutos**, em português, com duas EC2 Ubuntu
Server 24.04. Faça toda a preparação abaixo antes de iniciar o cronômetro.
O aluno usa o terminal da EC2 Cliente, dentro do code-server já existente.

- [ ] No AWS Academy, conferir créditos, orçamento, região, quotas, permissões IAM,
  disponibilidade de uma segunda EC2 e acesso de saída aos registros de imagens.
  Se a segunda EC2 não for autorizada, registrar o impedimento antes da aula.
- [ ] Criar **manualmente** a EC2 Servidor Ubuntu 24.04, preferencialmente `amd64`
  (`uname -m`: `x86_64`), na mesma VPC da EC2 Cliente. Escolher capacidade dentro
  das quotas; anotar IP privado, sub-redes e rotas locais. Não é necessário domínio
  nem IPv4 público para os testes. Para instalar pacotes, prever saída autorizada
  à Internet ou imagens/pacotes previamente disponibilizados pelo professor.
- [ ] No SG do servidor, permitir **TCP/8441, TCP/8442, TCP/8443 e UDP/8443**,
  tendo como origem o SG da EC2 Cliente, quando suportado, ou seu IP privado `/32`.
  Restringir SSH à administração. Conferir também saída da Cliente, rotas e NACLs
  (estas são sem estado e precisam permitir o retorno às portas efêmeras).
- [ ] Conferir todos os SGs associados ao servidor: suas permissões se somam.
  SG é uma lista de permissões **allow**, sem regra explícita de deny. Identificar
  a regra UDP que será removida e anotar como restaurá-la, incluindo origem.
- [ ] Não reinstalar nem reconfigurar code-server. Não alterar SG da Cliente,
  sua interface, autenticação ou painel. Não usar `tc netem`: perda artificial
  fica fora desta aula. Navegador e terminal representam caminhos diferentes.
- [ ] Instalar ferramentas e copiar este repositório às duas EC2, como abaixo.
- [ ] Preencher `.env` em cada EC2, conferir o servidor e obter **1.1, 2 e 3** no
  pré-voo da Cliente. Reservar tempo para resolver falhas antes dos 90 minutos.
- [ ] Ensaiar remoção/restauração de UDP/8443. Se faltar IAM, escolher previamente
  o plano B do experimento 4. Sem HTTP/3 real no pré-voo, a aula ainda não está homologada.
- [ ] Se optar por DevTools, homologar DNS público, certificado confiável, rota e
  acesso direto previamente. Sem isso, usar a alternativa terminal completa.
- [ ] Planejar parada/terminação manual da EC2 adicional, volumes EBS e eventuais
  endereços públicos; conferir cobranças, inclusive de IPv4 público.

### 1. Instalar ferramentas, sem tocar no code-server

Em **ambas as EC2 Ubuntu 24.04**, antes da aula (Docker via pacotes Ubuntu;
se já houver Docker/Compose funcionando, preservar essa instalação):

```bash
sudo apt-get update
sudo apt-get install -y docker.io docker-compose-v2 curl git
sudo systemctl enable --now docker
sudo docker version
sudo docker compose version
```

Somente na **EC2 Servidor**:

```bash
sudo apt-get install -y tcpdump
```

Os comandos seguintes usam Docker sem `sudo`. Se necessário, o professor autoriza
explicitamente o usuário atual a usar o Docker e abre uma **nova sessão de terminal**:

```bash
sudo usermod -aG docker "$USER"
```

Esse grupo concede poderes equivalentes a root; use apenas contas autorizadas.
Não reinicie o code-server nem desabilite autenticação. Uma nova conexão SSH pode
ser usada para preparar o ambiente sem afetar a sessão existente.

Em cada EC2:

```bash
git clone https://github.com/tarcisiodarocha/laboratorio-http-aws.git
cd laboratorio-http-aws
cp .env.example .env
nano .env
set -a; source .env; set +a
```

Preencher `SERVER_PRIVATE_IP` com o IPv4 privado **real** do servidor; manter
`LAB_HOST=lab.test`. `.env` é configuração local confiável, lida como shell;
use apenas atribuições simples. Não inserir credenciais AWS, chaves ou endereços
pessoais. Não enviar `.env`, certificados ou capturas ao Git.

### 2. Iniciar e verificar a EC2 Servidor

Na raiz do repositório, **EC2 Servidor**:

```bash
docker compose config --quiet
docker compose pull
docker compose up -d
./scripts/verificar.sh servidor
docker compose logs --tail=40 caddy
```

O script executa `caddy adapt` e `caddy validate` na mesma imagem do serviço,
confere o contêiner ativo e suas quatro publicações. Na primeira inicialização,
aguarde a emissão do certificado local antes do teste da Cliente.

```bash
sudo ss -lntu | grep -E ':(8441|8442|8443)\b'
docker compose ps
docker compose port caddy 8443 --protocol udp
```

`ss` pode mostrar o proxy Docker; com NAT sem proxy, a publicação pode não aparecer
como processo ouvindo no host. A conexão remota e a captura são a comprovação final.
Se houver firewall local, conferir suas regras e o encaminhamento Docker sem
abrir globalmente o host nem desativar o firewall.

| Porta | Transporte publicado | Protocolos oferecidos |
|---|---|---|
| 8441 | TCP | h1 |
| 8442 | TCP | h1 h2 |
| 8443 | TCP e UDP | h1 h2 h3 |

O `Caddyfile` explicita `servers :8441`, `:8442`, `:8443` e `protocols`.
Em Caddy, habilitar h2 implica h1; **8442 não é HTTP/2 exclusivo**.
Caddy 2.11.9 foi selecionado para manter configuração e execução coerentes.
O mesmo diretório estático é montado como somente leitura em todas as portas.

### 3. Selecionar e homologar o cliente HTTP/3

**EC2 Cliente:** o curl padrão pode ter HTTP2 sem HTTP3. Não basta a opção
aparecer na ajuda. `Features:` precisa conter os dois recursos, e a requisição
`--http3-only` deve negociar efetivamente `3`.

A configuração padrão usa uma imagem comunitária de
[macbre/curl-http3](https://github.com/macbre/curl-http3), com manifesto fixado por
digest no `.env.example`. O manifesto consultado é **linux/amd64**: preparar EC2
Cliente x86_64, sem depender de emulação. O projeto de origem publica `curl -V`
com HTTP2 e HTTP3; isso é evidência do fornecedor, **não execução desta imagem aqui**.
Disponibilidade do manifesto e arquitetura foram verificadas no GHCR. Este ambiente
não possui Docker: a execução da imagem e HTTP/3 na EC2 precisam ser homologados.

Antes da aula, **EC2 Cliente**:

```bash
uname -m
curl -q -V
docker pull "$CURL_IMAGE"
./scripts/cliente.sh -V
./scripts/verificar.sh cliente
```

Se houver curl local comprovado com ambas as features, definir `CURL_MODE=native`
no `.env` e exportar novamente. Caso o digest não esteja disponível no futuro,
o professor deve selecionar outra imagem da mesma origem, executar `-V`, testar
as três versões e fixar o novo digest antes da aula. Não compilar durante os 90 minutos.
O wrapper usa `--network host` nas EC2 Linux, ignora `.curlrc` com `-q` e desativa
proxy com `--noproxy '*'`. Não usa o preview/proxy do code-server.

O teste exige resposta 200 e versão exata; retorna erro se faltar HTTP2/HTTP3,
rede, configuração ou negociação. Diagnosticar no servidor com logs e captura:
`docker compose logs --tail=40 caddy`. Falha de TCP: conferir IP, rota, SG e
publicações; falha só de HTTP/3: conferir cliente, UDP, NACL, firewall e publicação UDP.

### TLS privado, SNI e Host

`lab.test` é um nome reservado para testes, não um DNS provisionado. O `--resolve`
associa **cada porta** ao IP privado sem perder SNI/Host do hostname. Não trocar a
URL por `https://IP` e apenas adicionar `Host`: isso não preserva o mesmo SNI.
O Caddy usa `tls internal`; a CA fica no volume próprio do laboratório. Os comandos
usam `-k` somente nessa CA didática: isso desabilita verificação TLS e **não é
recomendado em produção**. O Caddy não instala a CA no sistema do usuário.

Opcional, para curl nativo com verificação TLS: no servidor, exportar apenas a CA
pública com `docker compose cp caddy:/data/caddy/pki/authorities/local/root.crt ./lab-ca.crt`,
copiá-la por canal de administração à Cliente e substituir `-k` por
`--cacert ./lab-ca.crt`. Nunca copiar a chave privada da CA. O wrapper Docker não
monta arquivos locais; esse comando de CA destina-se ao modo nativo.

## Aula — exatamente 90 minutos

| Intervalo | Atividade | Duração |
|---|---|---|
| 0–15 | Versões realmente negociadas | 15 min |
| 15–35 | 30 recursos e conexões | 20 min |
| 35–55 | Captura TCP e UDP | 20 min |
| 55–75 | Remoção de allow UDP e restauração | 20 min |
| 75–90 | Quatro respostas e discussão | 15 min |
| Total | | **90 min** |

Preparação já concluída. Nas instruções, **Cliente** significa o terminal da EC2
Cliente; **Servidor** significa sessão de administração da segunda EC2.
Não executar testes com `localhost`: ele apontaria para a máquina do terminal.
Anotar evidências em `evidencias/`, pasta local ignorada pelo Git.

### 0–15 min — Experimento 1: versão solicitada e versão observada

**Objetivo:** verificar a versão HTTP efetiva em cada porta.

**Executar na Cliente** (na raiz do repositório, em Bash):

```bash
set -a; source .env; set +a
mkdir -p evidencias
for item in '8441 --http1.1' '8442 --http2' '8443 --http3-only'; do
  read -r port flag <<< "$item"
  ./scripts/cliente.sh -kfsS --connect-timeout 5 --max-time 12 "$flag" \
    --resolve "$LAB_HOST:$port:$SERVER_PRIVATE_IP" -o /dev/null \
    -w "solicitado=$flag porta=$port efetivo=%{http_version} status=%{http_code}\n" \
    "https://$LAB_HOST:$port/"
done | tee evidencias/versoes.txt
./scripts/cliente.sh -kfsS --http2 --connect-timeout 5 --max-time 12 \
  --resolve "$LAB_HOST:8441:$SERVER_PRIVATE_IP" -o /dev/null \
  -w 'http2 solicitado em 8441: efetivo=%{http_version}\n' "https://$LAB_HOST:8441/"
```

**Observar / esperado:** primeira sequência: `1.1`, `2`, `3`, status `200`.
O último comando deve obter `1.1`: solicitar HTTP/2 não obriga o servidor a oferecê-lo.
A opção sozinha não é evidência; leia `efetivo` e eventuais erros do curl.

**Entender:** h1/h2 negociam sobre TLS/TCP (ALPN indica a seleção quando usado).
HTTP/3 usa QUIC/UDP com TLS 1.3 integrado. `--http3-only` não permite fallback;
`--http3` permite tentar TCP também. Transporte e versão HTTP são observações distintas.

**Exceções comuns:** falta de HTTP3 em `-V` é falha de cliente; timeout pode ser
rede. `000` ou saída de erro não comprovam sucesso. Não avançar afirmando HTTP/3
se o pré-voo falhou; registrar o impedimento.

**Pergunta:** que transporte sustenta cada uma das três versões?

### 15–35 min — Experimento 2: 30 recursos e multiplexação

**Objetivo:** observar concorrência e reutilização de conexões sem declarar um vencedor por tempo.

**Executar na Cliente**, alternativa terminal que não depende de domínio:

```bash
for item in '8441 --http1.1' '8442 --http2' '8443 --http3-only'; do
  read -r port flag <<< "$item"
  ./scripts/cliente.sh -kfsSv --parallel --parallel-max 6 "$flag" \
    --connect-timeout 5 --max-time 15 \
    --resolve "$LAB_HOST:$port:$SERVER_PRIVATE_IP" -o /dev/null \
    -w 'url=%{url_effective} http=%{http_version} status=%{http_code} local=%{local_port} novas=%{num_connects} total=%{time_total}\n' \
    "https://$LAB_HOST:$port/assets/recurso-[01-30].svg" \
    >"evidencias/recursos-$port.txt" 2>"evidencias/conexoes-$port.txt"
done
for port in 8441 8442 8443; do
  echo "Porta $port: $(wc -l < "evidencias/recursos-$port.txt") transferências"
  cat "evidencias/recursos-$port.txt"
  grep -Ei 'multiplex|re.?us|stream|connection' "evidencias/conexoes-$port.txt" || true
done
```

Uma única invocação curl por porta permite partilhar conexões. O glob gera
**30 URLs distintas**, com até seis transferências ativas. Não usar 30 processos
curl independentes como prova de multiplexação. Repetir o primeiro comando mais
duas vezes; guardar os registros anteriores se quiser compará-los.

**Observar / esperado:** 30 respostas `200` por porta, versões `1.1`, `2`, `3`.
`novas` é o número de novas conexões **naquela transferência**, não um ID global;
some os valores por invocação e confronte com `local` e o log detalhado. Uma porta
local pode reaparecer por reutilização; em QUIC, o conceito de conexão não se
resume para sempre à tupla IP/porta. Logs de streams/multiplexação, quando
presentes, complementam a evidência. Não exigir uma conexão exata: tamanhos pequenos,
negociação e escalonamento podem mudar o conjunto observado.

**Entender:** HTTP/2 e HTTP/3 permitem vários streams na mesma conexão.
Em HTTP/1.1, este cliente pode abrir várias conexões para concorrência e reutilizá-las
sequencialmente. HTTP/2 ainda sofre com entrega ordenada do TCP sob perda;
QUIC permite independência de entrega entre streams, com limites compartilhados
como controle de congestionamento. Esta aula não injeta perda nem mede esse efeito.

**Opção DevTools, somente se o professor homologou o modo público:** abrir
`https://DOMINIO:8441/`, depois `:8442/` e `:8443/` diretamente no Chrome/Chromium.
Em Network, habilitar **Protocol**, **Connection ID**, **Disable cache**; manter
DevTools aberto, limpar a lista e recarregar. Filtrar `recurso-` para contar os
30 assets (há também HTML e talvez favicon). Registrar waterfall, protocolos e IDs.
Repetir três vezes por porta. O servidor também envia `Cache-Control: no-store`.

**Exceções comuns:** um primeiro acesso em 8443 pode ser h2, descobrir h3 por
`Alt-Svc` e usar h3 depois. `Alt-Svc` é anúncio de alternativa, não prova de uso.
Sem TLS público confiável/UDP no caminho do navegador, continuar pelo terminal.
O tempo total de um carregamento não demonstra superioridade causal.

**Pergunta:** que evidência mostra várias requisições compartilhando uma conexão?

### 35–55 min — Experimento 3: enxergar transporte no servidor

**Objetivo:** distinguir pacotes TCP e UDP que chegam ao servidor.

**Executar no Servidor**, professor, em um terminal dedicado:

```bash
sudo tcpdump -ni any -nn -c 120 '(tcp port 8441 or tcp port 8442 or tcp port 8443 or udp port 8443)'
```

**Executar na Cliente**, enquanto a captura está ativa:

```bash
for item in '8441 --http1.1' '8442 --http2' '8443 --http3-only'; do
  read -r port flag <<< "$item"
  ./scripts/cliente.sh -kfsS "$flag" --connect-timeout 5 --max-time 12 \
    --resolve "$LAB_HOST:$port:$SERVER_PRIVATE_IP" -o /dev/null \
    -w "porta=$port http=%{http_version}\n" "https://$LAB_HOST:$port/"
done
```

**Observar / esperado:** pacotes TCP para 8441/8442; pacotes UDP/8443 durante o
teste que retorna HTTP `3`. Anotar endereços e portas. Professor encerra com
`Ctrl+C` se o limite de 120 pacotes não for alcançado.

**Entender:** tcpdump mostra endereços, portas e transporte. UDP na porta do
laboratório, correlacionado ao teste `3`, sustenta a identificação de QUIC;
qualquer UDP isolado não prova HTTP/3. TLS protege o conteúdo: a captura não lê
corpos HTTP, nem comprova sozinha multiplexação ou versão h1 versus h2.

**Exceções comuns:** `any` inclui bridge Docker e pode mostrar o mesmo fluxo
mais de uma vez; não contar essas linhas como novas conexões. Sem tráfego,
conferir interface, comando executado na Cliente e SG. Não capturar SSH,
code-server ou tráfego alheio ao exercício.

**Pergunta:** quais informações a captura criptografada permite concluir?

### 55–75 min — Experimento 4: interromper UDP e conferir fallback

**Objetivo:** testar novas conexões QUIC sem a permissão UDP e verificar a restauração.

1. **Cliente:** antes da mudança, executar o teste abaixo e exigir `http=3`.
2. **Professor, console AWS, SG do Servidor:** remover temporariamente apenas a
   permissão de entrada UDP/8443 da origem da Cliente. Conferir que nenhum outro
   SG/regra permite o mesmo tráfego. Não mudar TCP/8443, SSH ou SG da Cliente.
3. **Cliente:** executar o bloco de comparação abaixo, em novos processos.
4. **Professor:** restaurar a mesma permissão UDP/8443 e origem, mesmo se a
   demonstração falhar. **Cliente:** repetir o teste inicial e exigir `3`.

**Teste inicial e após restauração, Cliente:**

```bash
./scripts/cliente.sh -kfsS --http3-only --connect-timeout 5 --max-time 12 \
  --resolve "$LAB_HOST:8443:$SERVER_PRIVATE_IP" -o /dev/null \
  -w 'http=%{http_version}\n' "https://$LAB_HOST:8443/"
```

**Durante a remoção, Cliente:**

```bash
for flag in --http3-only --http3 --http2; do
  ./scripts/cliente.sh -kfsS "$flag" --connect-timeout 5 --max-time 12 \
    --resolve "$LAB_HOST:8443:$SERVER_PRIVATE_IP" -o /dev/null \
    -w "pedido=$flag efetivo=%{http_version}\n" "https://$LAB_HOST:8443/"
  rc=$?; echo "pedido=$flag código_de_saída=$rc"
done
```

**Registrar restauração, Cliente**, após sucesso do teste inicial:

```bash
./scripts/verificar.sh cliente && printf 'UDP restaurado e HTTP/3 confirmado em %s\n' "$(date -Is)" | tee evidencias/restauracao.txt
```

**Observar / esperado:** com UDP realmente inacessível, `--http3-only` falha
(código diferente de zero); `--http3` **pode** cair para `2` ou `1.1`; `--http2`
deve continuar em `2`. Após restaurar, `--http3-only` volta a `3`.
Falha do comando e versão `0` não contam como HTTP negociado.

**Entender:** servidor com h3 habilitado não garante UDP no caminho completo.
Cada invocação aqui abre uma nova conexão e não usa cache persistente de Alt-Svc.
SG tem rastreamento de conexões: uma sessão já estabelecida pode sobreviver à
remoção. Use processos novos, não uma aba com conexão aberta; se ainda houver
sucesso, conferir regras sobrepostas, novas portas de origem e propagação/estado.
Não enfraquecer regras para forçar o resultado.

**Exceções / plano B sem IAM:** manter a rede intacta e executar os mesmos dois
blocos com UDP permitido, comparando as opções e a versão real. Esperar `3` com
`--http3-only`, possibilidade de `3` ou TCP com `--http3` e `2` com `--http2`.
Discutir o resultado previsto da remoção e anotar **“bloqueio não executado por
falta de permissão IAM”**; não apresentar uma falha artificial como observada.
Esse plano ocupa o mesmo intervalo com análise dos registros e da captura.
Nenhum script deste repositório modifica AWS ou a rede da Cliente.

**Pergunta:** um servidor que oferece HTTP/3 garante que o cliente consiga usá-lo?

### 75–90 min — Síntese e quatro respostas

**Objetivo:** explicar as diferenças a partir das evidências da turma.

**Executar na Cliente:**

```bash
cat evidencias/versoes.txt
printf '1. Principal mudança de HTTP/1.1 para HTTP/2?\n2. O que muda com HTTP/3?\n3. Por que bloquear UDP/8443 afeta HTTP/3?\n4. HTTP/3 é sempre mais rápido?\n' > evidencias/respostas.txt
```

**Observar / esperado:** preencher quatro respostas no arquivo ou em papel,
comparar em duplas e discutir com o professor, citando versão, conexões e transporte.
Dedicar 5 minutos às respostas, 5 à comparação e 5 ao gabarito.

**Exceções comuns:** substituir “é mais rápido” por explicação sustentada pelas
medições; anotar limitações ou experimentos não executados. Não deduzir versão
HTTP apenas da porta ou do tempo.

**Quatro perguntas e gabarito breve do professor:**

1. **Principal mudança de HTTP/1.1 para HTTP/2?** Multiplexação de streams na mesma
   conexão TCP, enquadramento binário e compressão de cabeçalhos; a entrega ordenada
   do TCP continua relevante sob perda.
2. **O que muda com HTTP/3?** HTTP opera sobre QUIC/UDP, com TLS 1.3 integrado;
   perdas não bloqueiam necessariamente a entrega de streams independentes como
   no TCP, embora recursos e congestionamento continuem compartilhados.
3. **Por que bloquear UDP/8443 afeta HTTP/3?** QUIC exige UDP funcional;
   `--http3-only` falha sem esse caminho. TCP na mesma porta pode continuar e
   `--http3` pode fazer fallback, cuja versão precisa ser observada.
4. **HTTP/3 é sempre mais rápido?** Não. Rede, latência, perda, cache, cliente,
   configuração e aquecimento mudam os resultados; esta aula não estabelece ranking.

Gabarito das perguntas dos experimentos: (1) TCP/TCP/QUIC sobre UDP; (2) streams
simultâneos no mesmo ID de conexão, apoiados por logs e portas, sem confundir
processos independentes; (3) endereços, portas e TCP/UDP, sem conteúdo TLS ou
versão h1/h2 determinada só pelos pacotes; (4) não, cliente e caminho também importam.

## Modo público opcional — preparar antes, nunca exigir para a aula

É necessário controlar um **domínio próprio com DNS apontando ao IPv4 público do
servidor**, ter rota via Internet Gateway e orçamento para esse endereço. O DNS
público automático da EC2 não substitui o controle de domínio exigido aqui.
O navegador acessa diretamente o domínio e as portas; o proxy/preview code-server
não transporta o experimento. Um certificado interno com aviso ignorado não
basta para habilitar QUIC no navegador.

No SG do servidor, liberar TCP/8441–8443 e UDP/8443 às redes autorizadas da turma
quando possível. Para a configuração abaixo, a validação ACME HTTP-01 exige
**TCP/80 acessível pela autoridade certificadora** e saída HTTPS/DNS do servidor.
Se as políticas do Academy impedirem isso, manter o modo privado. Não publicar
SSH/code-server junto. DNS-01 seria outra estratégia, fora deste manual.

No **Servidor**, depois de configurar DNS e SG manualmente, editar `.env`:
`LAB_HOST=seu.dominio` e `CADDYFILE=Caddyfile.public`. Criar os arquivos locais
abaixo. A substituição troca a CA local por ACME público e desativa TLS-ALPN-01,
para usar explicitamente HTTP-01 em TCP/80. Não é preciso publicar 443 nessa opção.

```bash
python3 - <<'PY'
from pathlib import Path
text = Path('Caddyfile').read_text()
text = text.replace('tls internal', 'tls {\n\t\tissuer acme {\n\t\t\tdisable_tlsalpn_challenge\n\t\t}\n\t}')
Path('Caddyfile.public').write_text(text)
PY
cat > compose.public.yaml <<'YAML'
services:
  caddy:
    ports:
      - "80:80/tcp"
YAML
set -a; source .env; set +a
docker compose -f compose.yaml -f compose.public.yaml run --rm --no-deps caddy caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
docker compose -f compose.yaml -f compose.public.yaml up -d
```

Conferir emissão nos logs, DNS e ausência de avisos de certificado nas três URLs;
confirmar coluna Protocol e captura UDP no **caminho do navegador**. TCP/80 deve
permanecer disponível para renovações HTTP-01. Se houver AAAA publicado, ele também
precisa alcançar o servidor; não publicar IPv6 incorreto. Sem emissão confiável,
retornar a `.env` privado, `CADDYFILE=Caddyfile`, `docker compose up -d` e usar CLI.

A configuração pública é alternativa à privada nesta instância, não duas
implantações simultâneas. No modo público, usar os dois `-f` em comandos Compose
(inclusive diagnóstico e `down`). O teste privado com `-k` não homologa TLS público;
para isso, usar curl **sem `-k`** e o navegador. O modo público não foi executado neste ambiente.

## Encerramento e limpeza

**Servidor, modo privado:**

```bash
./scripts/limpar.sh
```

Para reiniciar: `docker compose up -d`. Para apagar também os volumes da CA e
configuração **somente deste projeto**: `./scripts/limpar.sh --apagar-dados`.
Isso invalida a confiança de uma CA que tenha sido copiada; uma nova será gerada.
O script não usa `docker system prune`, não remove imagens globais e não toca AWS.
No modo público: `docker compose -f compose.yaml -f compose.public.yaml down`
(acrescentar `--volumes` somente se desejar descartar os dados desse projeto).

Professor restaura UDP se necessário, encerra tcpdump, para/termina manualmente
**a EC2 Servidor adicional** conforme a política da disciplina e verifica EBS,
endereços IPv4 e recursos remanescentes no console. Não encerrar a EC2 Cliente
que hospeda code-server por meio deste laboratório.

## Arquivos e validação da entrega

Poucos arquivos operacionais: `README.md` (único manual), `compose.yaml`,
`Caddyfile`, `.env.example`, `.gitignore`, três scripts (`cliente`, `verificar`,
`limpar`), `site/index.html` e 30 SVG determinísticos. O wrapper adicional mantém
idênticos os comandos quando se alterna entre curl local e o cliente Docker.

**Validação local desta entrega (2026-11-06):** Caddy nativo macOS arm64 2.11.9,
com arquivo de distribuição conferido por SHA-512 publicado; curl nativo 8.7.1
com HTTP2 e sem HTTP3. Rede AWS e Docker indisponíveis neste ambiente.

| Critério | Estado e evidência |
|---|---|
| 1. Manual em português e 90 minutos | **OK**: 15 + 20 + 20 + 20 + 15; comandos, observações, exceções e gabarito |
| 2. Quatro publicações Compose | **OK por inspeção**: 8441/tcp, 8442/tcp, 8443/tcp e 8443/udp; Docker não executado |
| 3. Configuração Caddy | **OK**: `caddy adapt` e `caddy validate` 2.11.9, `Valid configuration`; runtime confirmou protocolos por porta |
| 4. Conteúdo idêntico, 30 assets | **OK local**: 93 respostas 200 (HTML + 30 SVG × 3 portas), SHA-256 idêntico por caminho |
| 5. HTTP/1.1 em 8441 | **OK local**: curl `--http1.1` retornou `1.1 200` |
| 6. HTTP/2 em 8442 | **OK local**: curl `--http2` retornou `2 200`; fallback TCP em 8443 também retornou `2 200` |
| 7. HTTP/3 real em 8443 | **NÃO EXECUTADO**: curl local sem HTTP3 e sem Docker; manifesto do cliente disponível linux/amd64, execução a homologar |
| 8. Captura de TCP/UDP | **NÃO EXECUTADO**: instrução de tcpdump verificável no experimento 3; listener h3 do Caddy não equivale a tráfego h3 comprovado |
| 9. Bloqueio e restauração | **OK no manual / NÃO EXECUTADO em AWS**: procedimento manual e plano B sem IAM no experimento 4 |
| 10. Diagnóstico e limpeza | **OK local**: sintaxe Bash, erro explícito sem HTTP3 e escopo dos comandos de limpeza com Docker simulado; operações Docker reais não executadas |

O teste terminal de concorrência também foi executado com 30 recursos: seis
conexões novas em HTTP/1.1 (8441), uma em HTTP/2 (8442) e uma em HTTP/2 de fallback
(8443). São observações desta execução local, não números garantidos na VPC.
A configuração pública gerada no exemplo passou em `caddy validate`, sem emissão
ACME real. O diagnóstico retornou erro explícito “sem HTTP3”; a limpeza foi
verificada com um executável Docker simulado, aceitando somente `down` ou
`down --volumes` com arquivo e nome deste projeto explícitos. Os scripts passaram
em `bash -n`. Manifestos das imagens Caddy e cliente estavam disponíveis nos
registros; isso não substitui sua execução pelo Docker nas EC2.

Conferir o pré-voo nas EC2 antes de declarar a implantação homologada. Os testes
locais não certificam permissões Academy, SG, UDP da VPC, imagem executada ou
TLS público. Preencher região, disponibilidade/permissões, IP privado real e
opção de cliente; domínio/TLS público continuam opcionais.

## Referências consultadas

- [Caddy: opções globais, listeners e protocols](https://caddyserver.com/docs/caddyfile/options)
- [Caddy: HTTPS automático e CA local](https://caddyserver.com/docs/automatic-https)
- [Caddy: diretiva tls e emissores](https://caddyserver.com/docs/caddyfile/directives/tls)
- [curl: opções, paralelismo e write-out](https://curl.se/docs/manpage.html)
- [curl: suporte HTTP/3](https://curl.se/docs/http3.html)
- [Imagem do cliente: fonte e saída de versão publicada](https://github.com/macbre/curl-http3)
- [Docker no Ubuntu: documentação de referência](https://docs.docker.com/engine/install/ubuntu/)
