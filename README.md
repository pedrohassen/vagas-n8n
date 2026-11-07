# Automação de vagas com n8n

Uma vez por dia, às 9h, o workflow busca vagas, filtra pelo perfil (dev full stack júnior/pleno; JS/TS/React/Node, C#/.NET, IA), tira duplicadas e gera uma **página HTML** com o resultado. Se você configurar um webhook do Discord, ele também **manda uma mensagem lá**: com as vagas novas ("confira no PC o quanto antes") ou, se não houver, "nenhuma vaga nova". O Discord é **opcional**: sem ele (variável vazia), tudo funciona só pelas páginas HTML. Vagas de estágio, sênior/lead e restritas a EUA/Europa são descartadas.

Dois gatilhos disparam a mesma busca:
- **Todo dia às 9h**, se o n8n estiver ligado.
- **Ao ligar o n8n** (nó "n8n Trigger", evento *Instance Started*): espera 60 s para a rede subir e roda. Se o PC estava desligado às 9h, a mensagem chega assim que você ligar, com as vagas retroativas pendentes (até 14 dias de idade; a deduplicação garante que nada repete).

Ligar o PC depois de já ter recebido a mensagem do dia gera outra mensagem. É de propósito. Para mudar o horário, edite a expressão cron no nó "Todo dia às 9h" (o Remotive aceita até 4 consultas por dia, então não passe disso). Com `--restart unless-stopped` o n8n sobe sozinho quando o Docker abrir.

**Para demonstração:** além do Discord, cada execução grava uma página HTML (`saida/vagas-AAAA-MM-DD_HHMM.html`) com a lista de vagas ou "nenhuma vaga nova". Dá para abrir no navegador na apresentação sem abrir o Discord.

Além do arquivo com data e hora, toda execução também grava (e sobrescreve) `saida/spider-man-ultimas-vagas.html`. Abra esse arquivo uma vez e salve nos favoritos do navegador: a página **recarrega sozinha a cada 10 segundos**, então ao disparar o workflow no n8n ela mostra o resultado novo sem você abrir nada.

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
5. **Testar:** abrir o workflow principal e executar o gatilho **Teste manual**. Conferir a página em `saida/spider-man-ultimas-vagas.html` e, se usa Discord, a mensagem no canal.
6. **Ativar:** no principal, *Settings → Error workflow* → "alerta de falha". Depois ativar (**Active**) os dois.

**Histórico e paginação:** a página inicial lembra as últimas 200 vagas (mais novas primeiro, 10 por página, com a etiqueta "Nova" nas da última execução). O histórico fica na memória do n8n e só é gravado nas execuções automáticas, com o workflow ativo.

**Página de alertas:** quando o workflow principal falha, o workflow de alerta avisa no Discord e registra a falha, com data e hora, em `saida/spider-man-alertas.html` (últimas 50). As duas páginas têm link uma para a outra. Para gerar a página de alertas sem ter uma falha, execute o gatilho **Gerar página de alertas** do workflow de alerta.

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
4. Ativar o workflow (a fila só é gravada com o workflow **ativo**).

**Botões** (links também no topo da página de vagas; só funcionam com o n8n ligado e o workflow ativo)
- http://localhost:5678/webhook/ver-fila: mostra quantas vagas esperam, sem enviar nem esvaziar.
- http://localhost:5678/webhook/enviar-fila: roda o fluxo completo agora (APIs + fila), a qualquer hora. Se a fila estiver vazia, vem o aviso de "nenhuma vaga nova".

**Cuidados**
- O gatilho lê só e-mails **não lidos** dos remetentes dos três sites. E-mail que você já abriu no Yahoo é ignorado.
- Na primeira ativação o n8n pega os não lidos do dia da ativação em diante; depois acompanha só os novos.
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
| `locais_aceitos` | Locais aceitos para vagas remotas |
| `fontes_esperadas` | Fontes que deveriam trazer vagas. Se uma vier vazia, o resumo avisa; se todas vierem vazias, o workflow falha e o alerta registra. Vazio desliga essa conferência |

As palavras valem como palavra inteira (`java` não pega `javascript`). O que sobra no código do nó **Filtrar, pontuar e deduplicar** são ajustes técnicos: fuso, tamanho das mensagens e a regra de exigência de morar em certos países.

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

**Vaga "com ressalvas"** é a que só passou porque o título tem Júnior, apesar de uma stack bloqueada (ex.: "FullStack (NodeJS / Java) - Junior"). Ela vem depois das normais no Discord, com o motivo ("⚠️ tem "java" (stack bloqueada)") e um aviso para ajustar os filtros no nó Configurações. Na página, o botão **Com ressalvas** filtra essas vagas e mostra a mesma explicação.

### Como a nota funciona
A nota (0 a 10) é calculada **por nós, com regras fixas e sem IA**, no nó **Filtrar, pontuar e deduplicar**. Nenhum site (Indeed, LinkedIn, Glassdoor...) manda nota. Ela serve para duas coisas: **cortar** (vaga abaixo de `nota_minima` não é enviada) e **ordenar** (maiores notas primeiro, dentro de cada grupo). Não mede "o quanto a vaga é boa": só soma pistas que o título e a descrição dão.

| Critério | Pontos |
|---|---|
| Vaga veio de e-mail (já chega filtrada pelo seu alerta) | +3 |
| Palavra de cargo no título (`palavras_do_cargo`) ou palavra prioritária | +3 |
| Tecnologias do título e da descrição (`tecnologias`, cada uma com seu peso) | soma dos pesos, no máximo +5 |
| Júnior no título (`palavras_nivel_junior`) ou palavra prioritária | +2 |
| Descrição pede 6 anos ou mais / 4 a 5 anos / até 2 anos | −4 / −2 / +1 |
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
- Os valores +3, +3, +2, −4/−2/+1 e −3 estão no código do nó (comentados); as palavras e os pesos das tecnologias ficam no nó Configurações.

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
- O n8n local só roda com o PC ligado e o Docker aberto. O horário das 9h perdido não é recuperado pelo agendador, mas o gatilho "ao ligar o n8n" cobre isso quando o PC ligar. Os e-mails de alerta não lidos ficam esperando no Yahoo: o gatilho do IMAP deve pegá-los quando o n8n voltar (a confirmar com a caixa real).
- O n8n sem fuso configurado usa UTC, então o cron das 9h pode disparar em outro horário. O `GENERIC_TIMEZONE` do `.env` resolve.
