# Automação de vagas com n8n

Uma vez por dia, às 9h, o workflow busca vagas, filtra pelo perfil (dev full stack júnior/pleno; JS/TS/React/Node, C#/.NET, IA), tira duplicadas e gera uma **página HTML** com o resultado. Se você configurar um webhook do Discord, ele também **manda uma mensagem lá**: com as vagas novas ("confira no PC o quanto antes") ou, se não houver, "nenhuma vaga nova". O Discord é **opcional**: sem ele (variável vazia), tudo funciona só pelas páginas HTML. Vagas de estágio, sênior/lead e restritas a EUA/Europa são descartadas.

Dois gatilhos disparam a mesma busca:
- **Todo dia às 9h**, se o n8n estiver ligado.
- **Ao ligar o n8n** (nó "n8n Trigger", evento *Instance Started*): espera 60 s para a rede subir e roda. Se o PC estava desligado às 9h, a mensagem chega assim que você ligar, com as vagas retroativas pendentes (até 14 dias de idade; a deduplicação garante que nada repete).

Ligar o PC depois de já ter recebido a mensagem do dia gera outra mensagem. É de propósito. Para mudar o horário, edite a expressão cron no nó "Todo dia às 9h" (o Remotive aceita até 4 consultas por dia, então não passe disso). Com `--restart unless-stopped` o n8n sobe sozinho quando o Docker abrir.

**Para demonstração:** além do Discord, cada execução grava uma página HTML (`saida/vagas-AAAA-MM-DD_HHMM.html`) com a lista de vagas ou "nenhuma vaga nova". Dá para abrir no navegador na apresentação sem abrir o Discord.

Além do arquivo com data e hora, toda execução também grava (e sobrescreve) `saida/spider-man-ultimas-vagas.html` (com `TEMA=neutro`, `saida/ultimas-vagas.html`). Abra esse arquivo uma vez e salve nos favoritos do navegador: a página **recarrega sozinha a cada 10 segundos**, então ao disparar o workflow no n8n ela mostra o resultado novo sem você abrir nada.

### Como abrir a página de vagas (e salvar o favorito)
A página é um arquivo comum da pasta `saida/`: não precisa de servidor. Escolha **uma** das formas (a 1 é a mais simples):

1. **Arrastar o arquivo para o navegador.** No Explorador de Arquivos, arraste `saida/spider-man-ultimas-vagas.html` para uma aba do Chrome ou Edge e salve nos favoritos (Ctrl+D). O navegador monta o endereço sozinho.
2. **Digitar o endereço `file:///`.** É `file:///` + o caminho completo do arquivo, com barras normais (`/`) no lugar das invertidas (`\`). Para pegar o caminho: no Windows, **Shift + botão direito** no arquivo → *Copiar como caminho*; no VS Code, botão direito no arquivo → *Copy Path*. Depois tire as aspas, troque `\` por `/` e coloque `file:///` na frente. Espaços viram `%20`. Exemplos:
   - Windows: `file:///C:/Users/voce/projetos/vagas-n8n/saida/spider-man-ultimas-vagas.html`
   - Mac: `file:///Users/voce/projetos/vagas-n8n/saida/spider-man-ultimas-vagas.html`
   - Linux: `file:///home/voce/projetos/vagas-n8n/saida/spider-man-ultimas-vagas.html`
3. **Extensão Live Server do VS Code** (opcional). Abra a pasta `vagas-n8n` no VS Code e clique em **Go Live**; o endereço fica `http://127.0.0.1:5500/saida/spider-man-ultimas-vagas.html`. Só funciona **enquanto o Live Server estiver ligado**: se o navegador mostrar `ERR_CONNECTION_REFUSED` nesse endereço, é ele que está desligado (use a forma 1 ou 2, que não dependem de nada ligado).

Com `TEMA=neutro` o arquivo se chama `ultimas-vagas.html` (e `alertas.html`): refaça o favorito. Os links "Ver fila" e "Enviar fila agora" no topo da página abrem o n8n em `localhost:5678` e funcionam em qualquer uma das formas, com o n8n ligado e o workflow ativo.

Tudo aqui usa só serviços gratuitos: n8n self-hosted, APIs públicas, e-mail por IMAP (Yahoo) e Discord.

## Arquivos
- `workflows/vagas-workflow.json`: workflow principal.
- `workflows/alerta-de-falha-workflow.json`: avisa no Discord se o principal quebrar.
- `docker-compose.yml`, `.env.example`: como subir o n8n e quais variáveis preencher.

## Fluxo
```
Gatilhos (todo dia às 9h / ao ligar o n8n / manual)
 ├─ Remotive (API REST)        ─┐
 ├─ RemoteOK (API REST)         │
 ├─ WeWorkRemotely (3 RSS→XML) ─┼─ Juntar → Filtrar/pontuar/deduplicar (Code) ─┬─ Discord (webhook, opcional)
 └─ Fila de e-mails            ─┘                                              └─ Página HTML → arquivo em saida/ (sempre)

Alertas por e-mail (Yahoo, IMAP) → Parser Glassdoor / LinkedIn / Indeed → Guardar na fila   (só guarda, não envia)
Botão "Enviar fila agora" (webhook) → roda o fluxo todo agora      Botão "Ver fila" → mostra a fila sem enviar
```

## Como usar
1. **Discord (opcional):** no canal onde quer receber, abrir *Editar canal → Integrações → Webhooks → Novo Webhook* e **copiar a URL**. Ela funciona como senha: não vai em nenhum arquivo versionado. Se não usa Discord, pule este passo.
2. **Variáveis:** copiar `.env.example` para `.env` e, se usar Discord, colar a URL em `DISCORD_WEBHOOK_URL` (vazio = só HTML).
3. **Subir o n8n** (Docker Desktop aberto), dentro desta pasta:
   ```powershell
   docker compose up -d
   ```
   Ele volta sozinho quando o Docker abrir. Abrir http://localhost:5678 e criar a conta local do n8n. Os dados ficam no volume `n8n_data`; as páginas HTML geradas caem em `saida/`.
4. **Importar:** em *Workflows → Import from file*, importar os dois JSON de `workflows/`.
5. **Testar:** abrir o workflow principal e executar o gatilho **Teste manual**. Conferir a página em `saida/spider-man-ultimas-vagas.html` (ou `ultimas-vagas.html`, no tema neutro) e, se usa Discord, a mensagem no canal.
6. **Ativar:** no principal, *Settings → Error workflow* → "alerta de falha". Depois ativar os dois: no n8n 2.x o botão no canto superior direito se chama **Publish** (nas versões 1.x era **Active**). Comece pelo alerta de falha. Para desligar, **Unpublish**. "Workflow ativo" neste README significa "publicado".

**Histórico e paginação:** a página inicial lembra as últimas 200 vagas (mais novas primeiro, 10 por página, com a etiqueta "Nova" nas da última execução). O histórico fica na memória do n8n e só é gravado nas execuções automáticas, com o workflow ativo.

**Página de alertas:** quando o workflow principal falha, o workflow de alerta avisa no Discord e registra a falha, com data e hora, em `saida/spider-man-alertas.html` (ou `alertas.html`, no tema neutro) (últimas 50). As duas páginas têm link uma para a outra. Para gerar a página de alertas sem ter uma falha, execute o gatilho **Gerar página de alertas** do workflow de alerta.

## Conta local do n8n e senha esquecida
A conta criada no primeiro acesso é **local**: o e-mail não é verificado (pode ser fictício) e a senha fica só no seu PC. Não existe recuperação por e-mail, então anote a senha.

Se esquecer, dentro desta pasta:
```powershell
docker exec -u node -it n8n n8n user-management:reset
docker compose restart
```
O n8n pede para criar o dono de novo. Confira a mensagem do comando para ver o que ele mantém. Para zerar tudo (apaga também os workflows importados), use `docker compose down -v`.

## Alertas por e-mail (LinkedIn, Indeed, Glassdoor) e a fila
Esta é a fonte que mais importa para vagas no Brasil e para vagas júnior.

**Como funciona:** o nó **E-mail de alerta (IMAP)** fica vigiando a caixa de entrada. Quando chega alerta do LinkedIn, Indeed ou Glassdoor, o parser do site (um nó Code por site, só regex) extrai as vagas e o nó **Guardar na fila** as coloca numa **fila** (memória do workflow). Nada é enviado nessa hora. A fila é lida e esvaziada quando o fluxo roda: às 9h, ao ligar o n8n, ou quando você clica num botão (abaixo). O e-mail **não é marcado como lido** — isso fica com você.

**Passo a passo**
1. Criar alertas diários nos sites (LinkedIn/Indeed com local "Pelotas, RS" e outro "remoto").
2. Gerar uma **senha de app** no Yahoo (Segurança da conta → *Gerar senha de app*). Não use a senha normal e não a compartilhe.
3. No n8n: *Credentials → Create → IMAP*. Host `imap.mail.yahoo.com`, porta `993`, SSL ligado, usuário = seu e-mail, senha = a senha de app. Escolher essa credencial no nó **E-mail de alerta (IMAP)**.
4. Ativar o workflow com **Publish** (a fila só é gravada com o workflow **ativo**, isto é, publicado).

**Botões** (links também no topo da página de vagas; só funcionam com o n8n ligado e o workflow ativo)
- http://localhost:5678/webhook/ver-fila: mostra quantas vagas esperam, sem enviar nem esvaziar.
- http://localhost:5678/webhook/enviar-fila: roda o fluxo completo agora (APIs + fila), a qualquer hora. Se a fila estiver vazia, vem o aviso de "nenhuma vaga nova".

**Cuidados**
- O gatilho lê e-mails dos remetentes dos três sites **lidos ou não**: quem evita repetição é o controle por UID do próprio nó (ele guarda o último e-mail processado e descarta os de UID menor ou igual), e não o estado de lido. Você pode ler, apagar ou organizar os e-mails no celular à vontade; um e-mail que chegou com o PC desligado entra na fila quando o n8n voltar, mesmo já lido.
- Na primeira ativação o n8n pega só e-mails **com a data de hoje em diante** (o nó usa `SINCE` a data da ativação); depois acompanha só os novos, por UID. E-mails de dias anteriores, mesmo não lidos, não entram sozinhos, então a fila começa vazia. Os alertas do dia seguinte entram normalmente.
- **Para puxar e-mails antigos uma vez (teste ou recuperar atrasados):** abra o nó **E-mail de alerta (IMAP)** → *Add option* → **Track Last Message Id** e desligue; em **Custom Email Rules**, acrescente um critério de data logo antes do último `]`, por exemplo `,["SINCE","01-Nov-2026"]` (formato dia-mês em inglês-ano; sem ele o nó buscaria todos os e-mails dos remetentes desde sempre) → **Publish** de novo. A fila não duplica vagas, porque ignora as já guardadas, e vagas mais velhas que `idade_maxima_dias` são descartadas pelo filtro. Confira em **Ver fila** e, depois, **ligue a opção de volta**, tire o critério `SINCE` e publique de novo: deixada desligada, o nó relê todos os e-mails do período a cada e-mail novo, e vagas já enviadas voltariam a aparecer na fila até o próximo envio.
- Se o e-mail tiver mais vagas do que o parser achou, o nó **falha de propósito** (o workflow de alerta registra). É o aviso de que o site mudou o formato do e-mail: ajuste o parser daquele site.
- Vaga fica no máximo 30 dias na fila; a fila guarda até 500.
- O parser do Indeed ("match") descompacta links com `zlib`, liberado no `docker-compose.yml` por `NODE_FUNCTION_ALLOW_BUILTIN=zlib`. Sem isso ele usa um identificador alternativo (empresa+título).

## Ajustes
O perfil de busca é editado **sem código**, no nó **Configurações** (clique no nó e mude os campos). Cada lista é separada por vírgula, sem se preocupar com acento ou maiúscula:

| Campo | O que faz |
|---|---|
| `nota_minima` | Vagas com nota menor que isso não são enviadas |
| `idade_maxima_dias` | Ignora vagas publicadas há mais dias que isso |
| `tecnologias` | `palavra: peso`. Use `/` para sinônimos (`node/nodejs: 1.5`). Para aceitar Angular: acrescente `, angular: 1.5` |
| `palavras_do_cargo` | O título precisa ter alguma destas palavras |
| `palavras_bloqueadas_no_titulo` | **Outra profissão** (devops, qa, sales...): título com alguma destas palavras é descartado |
| `palavras_bloqueadas_stack` | **Stack indesejada** (java, php...): descartado, a não ser que o título tenha Júnior (passa "com ressalvas") |
| `palavras_prioritarias` | Palavras que vencem os dois bloqueios acima (vazio por padrão). Ex.: para começar a aplicar para QA, escreva `qa` aqui |
| `senioridade_excluida` | **Filtro de exclusão por nível** (estágio, sênior, lead...): título com alguma destas palavras é descartado. Para voltar a receber estágio, apague as palavras de estágio (`estagio, estagiario, intern, internship, trainee, apprentice`) daqui. Vazio = não exclui nenhum nível |
| `palavras_nivel_junior` | Dão +2 pontos e, se o título também tiver stack bloqueada, a vaga passa "com ressalvas". Para tratar estágio como nível de entrada, acrescente `estagio, estagiario` aqui |
| `anos_muito_alto` | Vaga que pede **esta experiência ou mais** perde bastante nota (padrão `6`) |
| `anos_alto` | Vaga que pede esta experiência ou mais perde um pouco de nota (padrão `4`) |
| `anos_baixo` | Vaga que pede **até** esta experiência ganha um pouco de nota (padrão `2`) |
| `locais_aceitos` | Locais aceitos para vagas remotas |
| `fontes_esperadas` | Fontes que deveriam trazer vagas. Se uma vier vazia, o resumo avisa; se todas vierem vazias, o workflow falha e o alerta registra. Vazio desliga essa conferência |

As palavras valem como palavra inteira (`java` não pega `javascript`).

### Onde cada ajuste mora (três camadas)
Nem tudo está no nó Configurações, de propósito. O critério: **o que muda porque você muda fica em Configurações; o que muda só quando o sistema muda fica no código; o que muda com a máquina fica no `.env`.**

| Camada | O que é | Exemplos | Onde editar |
|---|---|---|---|
| **Perfil** (muda com você, com o tempo) | Quem você é e o que procura | palavras, nota mínima, tecnologias, locais, **limites de anos de experiência** | nó **Configurações**, sem código |
| **Regra de negócio** (muda com o sistema) | Como o sistema decide | pesos da nota (+3, +2, −4...), tamanho da mensagem, quanto tempo lembrar vaga enviada, regra de EUA/Europa | constantes comentadas no topo do nó **Filtrar, pontuar e deduplicar** (nomes `PESO_*`, `PENAL_*`) |
| **Ambiente** (muda com a máquina) | Onde o n8n roda | fuso horário, URL do webhook do Discord, foto do bot | arquivo `.env` |

Exemplo de por que os anos foram divididos: o **limite** (6, 4, 2 anos) é perfil, porque depende da sua experiência e vai mudar com o tempo (daqui a um ano você pode aceitar vaga de 4 anos sem desconto). Já o **desconto** (−4, −2, +1) é regra de negócio: é quanto cada faixa pesa na fórmula da nota.

**Fica fora do nó Configurações por já ter lugar próprio:** horário das 9h, espera de 60 s, tentativas de cada fonte (3, com 1,5 s entre elas e limite de 15 s), endereços das APIs/RSS e remetentes dos e-mails são **parâmetros do nó** correspondente na interface do n8n (nada escondido em código). Os **parsers** de e-mail são regex do formato de cada site: quando o site muda o e-mail, o ajuste é no código do parser.

**Fuso horário:** vem de `GENERIC_TIMEZONE` no `.env`. Ele vale para o gatilho das 9h e também para as datas e horas da mensagem, das páginas e dos nomes de arquivo. Sem a variável, o código usa Brasília (`America/Sao_Paulo`).

### Tema: Homem-Aranha ou neutro
As mensagens, as páginas e os nomes de arquivo vêm com a identidade do **Homem-Aranha** ("Meu Sentido de Aranha", autor "Spider-Man" no Discord, arquivos `spider-man-*.html`). É o padrão e não precisa fazer nada. Para **neutralizar** (por exemplo, para reaproveitar o projeto sem o personagem):

1. No arquivo `.env`, escreva `TEMA=neutro`.
2. Se usa Discord, troque também a foto em `DISCORD_AVATAR_URL` (ou deixe vazio: o autor da mensagem sai sem foto).
3. Aplique: `docker compose up -d`.

| | `TEMA=aranha` (padrão) | `TEMA=neutro` |
|---|---|---|
| Autor no Discord | Spider-Man | Radar de Vagas |
| Aviso de vagas | "🕷️ Meu Sentido de Aranha disparou!! 7 vaga(s) nova(s)..." | "🔔 Radar de Vagas: 7 vaga(s) nova(s)..." |
| Sem vagas | "...sentido de aranha calmo, nenhuma vaga nova..." | "...nenhuma vaga nova por enquanto..." |
| Aviso de falha | "🕸️ Oh não! Meu atirador de teia falhou!" | "⚠️ Falha na automação." |
| Título das páginas | Meu Sentido de Aranha | Radar de Vagas |
| Arquivos | `spider-man-ultimas-vagas.html`, `spider-man-alertas.html` | `ultimas-vagas.html`, `alertas.html` |

Trocar o tema muda os nomes de arquivo: atualize o favorito do navegador e, se quiser, apague os arquivos antigos de `saida/`. Para criar um tema seu, edite os textos que começam com `const NEUTRO = $env.TEMA === 'neutro'` nos nós **Filtrar, pontuar e deduplicar**, **Montar página HTML** e **Montar página de alerta**, e os nós de Discord dos dois workflows. Sem imagem da Marvel no repositório: só texto, cores e tipografia, e a foto do bot vem do `.env`.

### Como o filtro decide
Antes de tudo, vaga velha demais (`idade_maxima_dias`) ou já enviada antes (mesmo id, ou mesma empresa+título) é ignorada. Depois, as regras abaixo rodam **nesta ordem**; a primeira que barrar descarta a vaga (o mesmo roteiro está comentado no topo do código do nó):

| # | Regra | Resultado |
|---|---|---|
| 1 | Título tem senioridade excluída (`senioridade_excluida`: estágio, sênior, lead...) | Descarta. Nem Júnior nem palavra prioritária salvam; para mudar, edite o campo |
| 2 | Título tem outra profissão (`palavras_bloqueadas_no_titulo`) | Descarta, a não ser que tenha palavra de `palavras_prioritarias` |
| 3 | Título tem stack indesejada (`palavras_bloqueadas_stack`) | Descarta, a não ser que tenha palavra prioritária (passa normal) ou Júnior/Jr (passa **com ressalvas**) |
| 4 | Vaga de API sem palavra de cargo no título | Descarta (vaga de e-mail e palavra prioritária já contam como cargo) |
| 5 | Vaga traz local que não está em `locais_aceitos` | Descarta (exceto quando o parser do site garante que é Brasil) |
| 6 | Nota abaixo de `nota_minima` | Descarta |

**Vaga "com ressalvas"** é a que só passou porque o título tem uma palavra de nível júnior (as de `palavras_nivel_junior`: por padrão Júnior, Jr, entry level), apesar de uma stack bloqueada (ex.: "FullStack (NodeJS / Java) - Junior"). Ela vem depois das normais no Discord, com o motivo ("⚠️ tem "java" (stack bloqueada), mas o título tem "junior" (nível júnior)") e um aviso que lista as palavras de nível júnior configuradas, para você ajustar os filtros no nó Configurações. Se você acrescentar `trainee` em `palavras_nivel_junior`, o texto já passa a citá-lo, sem mexer em código. Na página, o botão **Com ressalvas** filtra essas vagas e mostra a mesma explicação.

### Como a nota funciona
A nota (0 a 10) é calculada **por nós, com regras fixas e sem IA**, no nó **Filtrar, pontuar e deduplicar**. Nenhum site (Indeed, LinkedIn, Glassdoor...) manda nota. Ela serve para duas coisas: **cortar** (vaga abaixo de `nota_minima` não é enviada) e **ordenar** (maiores notas primeiro, dentro de cada grupo). Não mede "o quanto a vaga é boa": só soma pistas que o título e a descrição dão.

| Critério | Pontos |
|---|---|
| Vaga veio de e-mail (já chega filtrada pelo seu alerta) | +3 |
| Palavra de cargo no título (`palavras_do_cargo`) ou palavra prioritária | +3 |
| Tecnologias do título e da descrição (`tecnologias`, cada uma com seu peso) | soma dos pesos, no máximo +5 |
| Júnior no título (`palavras_nivel_junior`) ou palavra prioritária | +2 |
| Descrição pede `anos_muito_alto` ou mais (6) / `anos_alto` ou mais (4) / até `anos_baixo` (2) | −4 / −2 / +1 |
| Descrição restringe a EUA, Canadá, Reino Unido ou Europa | −3 |

O total é arredondado e limitado a 0–10. Exemplos (todos conferidos em teste, com os pesos padrão `react: 1.5`, `node: 1.5`):

| Vaga | Conta | Nota |
|---|---|---|
| E-mail: "Desenvolvedor Fullstack Node Júnior" | 3 + 3 + 1,5 (node) + 2 (júnior) = 9,5 | 10 |
| E-mail: "Desenvolvedor React Pleno" | 3 + 3 + 1,5 (react) = 7,5 | 8 |
| E-mail: "Desenvolvedor Java Júnior" | 3 + 3 + 0 + 2 = 8 (passa **com ressalvas**) | 8 |
| API: "Full Stack Developer", descrição "React and Node, 3 years" | 0 + 3 + 3 (react + node) = 6 | 6 |
| API: mesma vaga, "1 year of experience" | 6 + 1 (até 2 anos) = 7 | 7 |
| API: mesma vaga, "7 years of experience" | 6 − 4 = 2, abaixo da nota mínima 5 | fora |

Pontos de atenção:
- **A nota de e-mail é inflada.** O e-mail só traz o título, então toda vaga de e-mail com palavra de cargo já começa em 6. Para ela, o que realmente decide é o filtro (regras 1 a 5); a nota serve mais para ordenar.
- **Vaga de API parte do zero**, e só chega a 5 com cargo + tecnologia ou Júnior. Por isso o ajuste fino de `tecnologias` importa mais para as APIs.
- Os **pesos** (+3, +3, +2, teto de +5, −4/−2/+1 e −3) são regra de negócio: ficam como constantes comentadas no topo do nó (`PESO_EMAIL`, `PESO_CARGO`, `PESO_JUNIOR`, `TETO_TECNOLOGIAS`, `PENAL_ANOS_MUITO_ALTO`, `PENAL_ANOS_ALTO`, `BONUS_ANOS_BAIXO`, `PENAL_EUA_EUROPA`). As palavras, os pesos de cada tecnologia e os **limites de anos** ficam no nó Configurações.
- O critério de anos usa o **maior número** que aparece no texto seguido de "years" ou "anos". "5 anos de mercado" na descrição da empresa também conta, então é uma heurística. Nos e-mails quase não há descrição, então ele pesa mais nas vagas das APIs.
- É um ajuste de nota, não um corte: a vaga só cai se a nota final ficar abaixo de `nota_minima`.

### Pesos da nota: o que não mexer sem entender
Os pesos (as constantes `PESO_*`, `PENAL_*`, `BONUS_*` e `TETO_*` no topo do nó **Filtrar, pontuar e deduplicar**) são **regra de negócio**: um depende do outro e da `nota_minima`. Mudar um sozinho, sem entender, pode fazer o filtro parar de cortar ou cortar tudo. Valores padrão e a regra que cada um precisa respeitar (com `nota_minima` = 5):

| Peso (padrão) | Regra que não pode quebrar | Se quebrar |
|---|---|---|
| `PESO_CARGO` (3) + `PESO_JUNIOR` (2) | A soma precisa ser **maior ou igual a `nota_minima`** | Vaga de API com palavra prioritária (ex.: `qa`) nunca chega ao corte e some. Foi o caso do "Analista de QA", que ficou com nota 3. |
| `PESO_EMAIL` (3) | Precisa ficar **abaixo de `nota_minima`** | Toda vaga de e-mail que passa nas regras 1 a 5 é enviada, mesmo sem palavra de cargo: a nota deixa de cortar e só ordena. |
| `PENAL_ANOS_MUITO_ALTO` (4) | Precisa ser grande o bastante para **cruzar o corte** | Uma vaga Pleno de e-mail parte de 6; com −4 vai a 2 e é cortada, com −1 fica em 5 e passa: a penalidade deixa de funcionar. |
| `TETO_TECNOLOGIAS` (5) | Não suba muito, não desça de ~2 | Alto: descrição cheia de siglas infla a nota de vaga errada. Baixo: vaga de API não consegue chegar a 5 (cargo vale 3 e precisa de pelo menos 2 das tecnologias). |
| `BONUS_ANOS_BAIXO` (1) | Mantenha pequeno | Com 3 ou mais, vaga de API que cita "1 year" passa sem nenhuma tecnologia. |

Pode mexer com segurança em `PENAL_EUA_EUROPA` (0 desliga a penalidade) e nos limites de anos (campos em Configurações, desde que `anos_muito_alto` > `anos_alto` > `anos_baixo`).

**Para receber mais ou menos vagas, mude a `nota_minima` (em Configurações), não os pesos.** Atenção: com `nota_minima` acima de 5, a vaga de API passa a precisar também de tecnologia; acima de 10, nada passa.

**Verificação automática:** se um campo numérico das Configurações estiver vazio, com texto, `nota_minima` fora de 0 a 10 ou os anos fora de ordem, o nó de filtro **falha de propósito** com uma mensagem como `Configurações: o campo "anos_muito_alto" precisa ser um número`. O workflow de alerta registra a falha (Discord e página de alertas). Isso evita uma nota distorcida sem você perceber (um campo vazio, por exemplo, viraria 0 e daria −4 em toda vaga que cita anos).

### Receitas de perfil
| Quero... | Faça |
|---|---|
| Voltar a receber estágio (pós-graduação, quem busca o primeiro estágio) | Apague as palavras de estágio de `senioridade_excluida`. Para estágio valer como nível de entrada (+2 e liberar stack com ressalva), acrescente `estagio, estagiario` em `palavras_nivel_junior` |
| Começar a aplicar para QA | Escreva `qa` em `palavras_prioritarias`. Acrescente `cypress: 1.5, selenium: 1.5, playwright: 1.5` em `tecnologias`, para a nota refletir QA |
| Parar de receber sênior de vez | Já é o padrão: palavras de sênior em `senioridade_excluida` |
| Aceitar uma stack que hoje é bloqueada (ex.: Java) | Tire a palavra de `palavras_bloqueadas_stack`. Se só quer ser avisado, deixe lá: com Júnior no título ela passa com ressalvas |
| Aceitar Angular | Acrescente `, angular: 1.5` em `tecnologias` |

## Limitações
- A nota é por palavra-chave e anos de experiência citados, não por IA. Dá para trocar por um nó de LLM depois.
- A memória de vagas já enviadas (deduplicação) só persiste com o workflow **ativo**. Em execuções manuais ela não é salva, então testes repetidos vão repetir vagas.
- As fontes internacionais têm poucas vagas júnior; em testes com dados reais de 23/10/2026 nenhuma passou no filtro. O volume deve vir dos e-mails.
- Remotive e RemoteOK exigem citar a fonte e linkar a vaga original (o resumo já faz isso). Não republique as vagas em outro site.
- O n8n local só roda com o PC ligado e o Docker aberto. O horário das 9h perdido não é recuperado pelo agendador, mas o gatilho "ao ligar o n8n" cobre isso quando o PC ligar. Os e-mails de alerta ficam esperando no Yahoo (lidos ou não): o gatilho do IMAP deve pegá-los quando o n8n voltar (a confirmar com a caixa real).
- O n8n sem fuso configurado usa UTC, então o cron das 9h pode disparar em outro horário e as horas das páginas saem trocadas. O `GENERIC_TIMEZONE` do `.env` resolve os dois.
