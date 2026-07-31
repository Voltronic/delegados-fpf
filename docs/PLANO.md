# Delegado FPF — Plano de Desenvolvimento

Aplicação de apoio à função de delegado da FPF em jogos de futebol, futsal e futebol de praia.

---

## 1. Princípios que mandam em tudo

1. **Tudo no dispositivo.** Não há cloud, contas nem servidor. O armazenamento local é a única fonte de verdade. A app funciona sempre, em qualquer sítio, sem rede.
2. **Custo zero.** Sem serviços pagos, sem loja de aplicações, sem infraestrutura a manter.
3. **Velocidade de introdução acima da estética.** No jogo há segundos, não minutos. Qualquer registo em três toques ou menos e nunca mais de um ecrã de profundidade — critério de aceitação, não intenção.
4. **Tudo configurável dentro da app.** Desportos, competições, regulamentos, checklists, guiões e frases-modelo são dados editáveis por ecrã — nunca é preciso mexer em ficheiros nem recompilar.
5. **Nunca apagar dados em silêncio.** Desligar uma condição oculta, não elimina. Apagar exige confirmação explícita.

---

## 2. Stack

| Camada | Escolha |
|---|---|
| Linguagem | TypeScript (strict) |
| UI | React + Vite + Tailwind + shadcn/ui |
| Empacotamento | Capacitor (Android; iOS possível sem reescrever) |
| Armazenamento | Dexie (IndexedDB) — única fonte de verdade |
| Transferência | Ficheiro `.dfpf` + partilha nativa do Android |
| Regulamentos | pdf.js |
| Testes | Vitest (domínio puro) |
| Distribuição | APK direto |

**Porquê Capacitor e não PWA pura:** armazenamento nativo que o sistema não despeja sob pressão de espaço, foreground service para funcionamento em segundo plano, acesso ao sistema de ficheiros e ao menu de partilha. Durante o desenvolvimento continua a correr no browser com hot reload.

**Porquê não Android nativo:** um só código, ciclo de iteração muito mais rápido com agente, e porta aberta para iOS.

---

## 3. Armazenamento e transferência entre dispositivos

### Sem cloud

Todos os dados vivem em IndexedDB no dispositivo: jogos, fotos, anexos, regulamentos e configuração. Não há autorizações, não há sincronização em segundo plano, não há conflitos a resolver.

### Transferência por ficheiro

Um jogo transfere-se para outro dispositivo através de um ficheiro `.dfpf` — um contentor com o JSON do jogo mais as suas fotos e anexos.

**Exportar:** a app gera o ficheiro e abre o **menu de partilha nativo do Android**. A partir daí vai por Quick Share (dispositivo a dispositivo, sem contas nem internet), Bluetooth, WhatsApp, email ou cabo — o que for mais prático no momento.

**Importar:** tocar no ficheiro recebido abre a app, que mostra o que vem lá dentro e pede confirmação antes de importar. A app regista um *intent filter* para a extensão `.dfpf`.

**Se o jogo já existir**, a app mostra as diferenças e pergunta: manter o local, substituir pelo importado, ou guardar como cópia. Nunca decide sozinha.

### Ficheiros de configuração

A configuração completa (guiões, competições, campos, frases-modelo, catálogos) exporta e importa da mesma forma, num `.dfpfconf`. É assim que se propaga uma alteração de guião para o segundo dispositivo, e é também a forma de repor a app num telemóvel novo.

### Cópia de segurança

Export de todo o arquivo para a pasta Downloads, manual ou com aviso periódico. É a única salvaguarda contra perda do dispositivo — convém ser hábito antes de cada jogo.

---

## 4. Modelo de dados

### Configuração (partilhada entre jogos)

```
Desporto → { id, nome, cronometro: "app" | "marcador",
             estruturaTempo, tiposEvento[], catalogoVerificacoes[] }

Competicao → { id, nome, epoca, desportoId, regulamentoId?,
               guiaoSugerido?, camposObrigatorios[] }

Clube  → { id, nome }
Equipa → { id, clubeId, competicaoId }          único por (clube, competição)

Campo  → { id, etiqueta, tipo, obrigatorio,
           ambito: "jogo" | "por-equipa",
           momento: "pre" | "jogo" | "pos",
           confirmacaoPos?, condicao?, herdaDeVerificacao? }

  tipos: texto | numero | sim-nao | sim-nao-naoutilizado | selecao |
         presenca | escala-qualitativa | lista-repetivel
  presenca: presente/ausente + nome

Condicao → { id, nome }
  referência: { id, quando: "sim" | "nao" }

Guiao → { id, nome, desporto, condicoes[], slides[] }
  └─ Slide  → { id, ordem, tema, tipo?, pessoa?, condicao?, ordemAlternativa?,
                bullets[], campos[] }
       ├─ Bullet → { texto, descricao?, condicao?, slideAlternativo?, seccao? }
       └─ Campo  → referência à biblioteca de campos

FraseModelo → { id, seccao, etiqueta, texto, subFrase? }
Fragmento   → { id, texto }
```

### Jogo (uma instância por jogo)

```
Jogo → { id, versaoEsquema, competicaoId, jornada,
         equipaVisitada, equipaVisitante, dataHora, estadio,
         guiaoCongelado, fase: "pre" | "reuniao" | "jogo" | "pos" }

  ├─ Condicoes[]       { condicaoId, estado: "sim" | "nao" | "indeterminado" }
  ├─ ValoresCampos[]   { campoId, equipa?, valor,
                         estado: "preenchido" | "pendente"
                               | "a-definir-na-reuniao" | "nao-aplicavel" }
  ├─ PontosValidar[]   { descricao, origem, desfecho: "resolvido"
                                          | "mantem-se" | "agravou-se" }
  ├─ Verificacoes[]    { item, estado: "ok" | "anomalia" | "na", nota, fotos[] }
  ├─ RegistosBoletim[] { descricao, anomalia: bool, momento, fotos[] }
  ├─ Anexos[]          { etiqueta, ficheiro }
  ├─ Eventos[]         { tempo, periodo, tipo, equipa, numeroAtleta, detalhe }
  ├─ Ocorrencias[]     { tempo, periodo, seccaoRelatorio, equipa,
                         descricao, fraseModeloId?, fotos[] }
  └─ Relatorios[]      { tipo, textoEditado, geradoEm, atualizadoEm }
```

Os dados do jogo (arbitragem, delegados, segurança, emergência médica) **não são campos fixos** — vêm dos campos do guião escolhido. Só o essencial para identificar o jogo está no cabeçalho.

`versaoEsquema` acompanha cada jogo desde o início: sem ele, um ficheiro exportado de uma versão mais recente da app é lido em silêncio e mal por uma versão antiga.

### Diferenças por desporto (derivadas de configuração, nunca de `if`)

| | Futebol | Futsal | Futebol de Praia |
|---|---|---|---|
| Estrutura | 2×45 | 2×20 | 3×12 |
| **Cronometragem** | **relógio na app** | **lida do marcador** | **lida do marcador** |
| Eventos próprios | — | faltas acumuladas, tempos mortos, 2 min de inferioridade | 3 períodos, sem empate |
| Verificações | relvado, balizas | pavimento, pavilhão | areia, delimitações |

**Cronometragem.** Só o futebol tem relógio a correr na app, e mesmo aí o minuto é **pré-preenchido e sempre editável**: entre o lance e o registo passa tempo, e há eventos registados em diferido. No futsal e no futebol de praia há sempre marcador no recinto, e o minuto é **introduzido manualmente**, com teclado numérico próprio e seletor de período. O ecrã de registo é o mesmo; muda apenas a origem do valor por omissão.

### Funcionamento em segundo plano

A app tem de se comportar como se estivesse sempre em primeiro plano, mesmo minimizada, com o ecrã desligado ou depois de ter sido morta pelo sistema.

1. **O relógio é derivado, nunca acumulado.** Guarda-se o timestamp do apito inicial e os intervalos de pausa; o minuto é recalculado a partir da hora do sistema sempre que é mostrado. Um contador de ticks derrapa assim que o WebView é suspenso.
2. **Persistência imediata.** Cada registo escreve em IndexedDB no momento. Não existe estado de jogo apenas em memória, nem ação de "guardar" no fim.
3. **Foreground service com notificação persistente**, mostrando o minuto em curso.

**Nota de realidade:** fabricantes como Xiaomi, Huawei e Samsung terminam apps em segundo plano de forma agressiva, mesmo com foreground service, salvo se a otimização de bateria for desativada para a app. A arquitetura acima sobrevive a isso — a app reconstrói o estado ao reabrir — mas deve ser testada no dispositivo real durante os espigões.

---

## 5. Os quatro modos

A app organiza-se por **modo**, não por menu de ecrãs. Cada jogo carrega a sua fase e abre sempre no modo em que ficou.

| | Pré-jogo | Reunião | Jogo | Pós-jogo |
|---|---|---|---|---|
| Verificações / anomalias | ✅ foco | consulta | ✅ | consulta |
| Recolha para a reunião | ✅ foco | ✅ | — | consulta |
| Ocorrências | ✅ | ✅ | ✅ | ✅ |
| Registos do boletim | ✅ | ✅ | ✅ | ✅ |
| Apresentação da reunião | — | ✅ foco | — | consulta |
| Eventos (golos, cartões) | — | — | ✅ foco | consulta |
| Relatórios e ficha | rascunho | — | rascunho | ✅ foco |

**Pré-jogo.** Recolha da informação para a reunião, checklist de verificações com anomalias a alimentar o boletim, preenchimento antecipado do que já se sabe, e registo de ocorrências.

**Reunião.** Apresentação em fullscreen, tipografia grande, navegação por swipe, alimentada pelo que foi recolhido no pré-jogo.

**Jogo.** Tudo o que existe no pré-jogo exceto a recolha para a reunião, mais os eventos do jogo.

**Pós-jogo.** Visão consolidada e extração para os dois relatórios e para a ficha de jogo.

### Três regras que o desenho tem de respeitar

1. **O modo muda a ênfase, não as permissões.** As coleções são as mesmas em todos os modos. Uma anomalia detetada ao intervalo entra no boletim sem obrigar a mudar de modo.
2. **O modo Reunião é editável.** Os campos que ficaram por obter preenchem-se durante a reunião, ali mesmo, sem sair do fullscreen.
3. **O pós-jogo não é só de leitura.** Incidentes à saída dos balneários ou no túnel ocorrem depois do apito final e são material do relatório de Ocorrências.

### Prontidão para a reunião

Cada campo pode ser marcado como **obrigatório**. O modo pré-jogo mostra permanentemente a lista do que falta obter, com contador.

O que não for obtido até à reunião não desaparece: aparece na apresentação **a vermelho**, identificado como "a obter durante a reunião". Preenchido no momento, passa a verde à frente dos presentes.

### Contagem decrescente para o início

Ao criar o jogo regista-se a hora de início. A app mostra no topo, em todos os modos, o tempo que falta — derivado do timestamp, e por isso imune a suspensão ou reinício da app.

Ao lado da contagem fica o número de pendentes. As duas informações juntas — "faltam 47 min", "3 por obter" — são o que decide a ação seguinte. Tempos de aviso configuráveis.

### Painel de pendentes

Agrupados por categoria, cada uma com contador próprio:

| Categoria | Onde se resolve | Bloqueia |
|---|---|---|
| **Campos do guião** | pré-jogo ou durante a reunião | fecho da reunião |
| **Pontos a validar** | no terreno, com desfecho de três valores | fecho do boletim |
| **Campos obrigatórios da competição** | conforme o momento do campo | fecho do boletim |
| **Dependentes de confirmação** | quando a condição deixar de ser indeterminada | não bloqueia |

### Velocidade de introdução

- **Registar primeiro, detalhar depois.** Um toque grava o evento com tempo e equipa; o detalhe fica editável a seguir, sem bloquear.
- **Sem diálogos encadeados e sem confirmações.** Desfazer em vez de confirmar.
- **Tocar em vez de escrever.** Equipas, tipos de evento e categorias pré-carregados como botões.
- **Números de atleta introduzidos à mão** — não há plantéis na app. Teclado numérico próprio no ecrã, nunca o do sistema, com botões grandes e posição fixa.
- **Números já usados ficam como atalho por equipa.** Segundo amarelo, bis ou substituição do mesmo atleta deixam de exigir digitação.
- **Alvos grandes**, para uso de pé, com uma mão, possivelmente com chuva.
- Ditado por voz como atalho para descrições longas.

---

## 6. Registo do jogo e ficha consultável

Golos, cartões e substituições **não alimentam o corpo de nenhum dos relatórios** — servem para preencher o documento enviado por email. O resultado final é a única coisa que passa para o cabeçalho dos dois relatórios, e é calculado a partir dos golos registados.

- Ecrã de **ficha de jogo** consultável a qualquer momento, organizado por equipa e período
- **Todos os eventos editáveis e apagáveis depois do registo.** Como os números são escritos à mão e não há plantel para validar, um erro só é apanhado a frio — a correção tem de ser trivial
- Continua disponível no histórico depois do jogo terminado
- Versão em texto copiável, pelo mesmo motor dos relatórios, para evitar transcrição manual

### Clubes, equipas e competições

O escalão **é** a competição — Sub-23 é uma competição, não um atributo à parte. O desporto vem da competição.

```
Desporto → Competição → Equipa (uma por clube)
```

Uma equipa é o par **clube + competição**, com unicidade. Um clube pode ter equipas em muitas competições, e cada uma é um registo próprio.

Clube, competição e equipa são entidades com identificador, nunca texto livre. Devem poder ser criadas **no momento de criar o jogo**, não só no ecrã de configuração — senão, à primeira vez que se vai a um clube novo, é preciso interromper o que se está a fazer.

Escolhidos o desporto e a competição, a app oferece apenas as equipas compatíveis.

### Pontos a validar

Nada transita automaticamente entre jogos. Os pontos a verificar são sempre escolha explícita, com duas origens:

**1. Cloud da FPF** — acrescentados à mão, numa secção própria da criação do jogo.

**2. Sugestão dos dois últimos jogos** — a app lista as anomalias detetadas nos **dois jogos anteriores** dessa equipa, **nenhuma pré-selecionada**, cada uma com data e jogo de origem.

Não requer entidade própria: as anomalias são as verificações com estado *anomalia* em jogos anteriores da mesma equipa.

**Comum às duas origens:**

- Entram na checklist do pré-jogo **destacados** dos itens normais do catálogo
- Desfecho de cada um: **resolvido**, **mantém-se** ou **agravou-se** — é o que alimenta o boletim

---

## 7. Relatórios

Saída em **texto simples editável**, não PDF — o destino é copiar para o SCORE. Botão de copiar por secção e para o documento completo. O texto editado fica guardado no jogo, para retomar e reeditar.

Ambos os relatórios são **iguais em todas as competições**; só muda o nome no cabeçalho e os campos obrigatórios que cada competição acrescenta. Não há herança de templates por desporto.

### Relatório de Ocorrências

Configuração em `seed/relatorio-ocorrencias.json`.

Cada ocorrência é classificada com uma **secção do relatório** e uma **equipa** (visitada ou visitante). A partir daí o relatório monta-se:

- Secção sem ocorrências fica a "Não", com caixa vazia
- Secção com ocorrências remete para "Ver Outros comentários"
- O detalhe sai numerado no bloco final, por ordem cronológica, com o fecho padrão acrescentado automaticamente

**Registo ao vivo mínimo, redação diferida.** No estádio regista-se apenas **tipo de reporte, tempo, equipa e texto corrido**. Nenhum modelo é escolhido com o jogo a decorrer.

Na geração, cada ocorrência aparece com a nota original visível como referência e o parágrafo já montado a partir de um modelo do seu tipo, com **blocos editáveis individualmente** — toca-se no bloco da bancada, da equipa ou da citação e altera-se só esse pedaço. O modelo proposto pode ser trocado por outro do mesmo tipo.

**Biblioteca de frases-modelo.** As descrições são altamente formulaicas. Trinta modelos de partida, com espaços a preencher. Adicionáveis, editáveis, removíveis e duplicáveis na app.

Muitas frases partilham remates recorrentes — o aval das forças de segurança comunicado pelo Comandante, o "não foram presenciados pelos Delegados da FPF", o "sem atingir qualquer elemento". Ficam como **fragmentos** anexáveis a qualquer frase.

**Formato do tempo livre:** aceita `11:08`, `29 min.` e `45+3 min.`. O período é campo separado. Sem validação rígida.

### Boletim de Organização de Jogo

Configuração em `seed/boletim-organizacao-jogo.json`. Montado a partir de quatro origens:

1. **Pontos a validar**, com o desfecho de cada um
2. **Anomalias** detetadas neste jogo
3. **Informações** — observações neutras que vale a pena reportar sem serem anomalia
4. **Campos obrigatórios da competição**

**Anomalia e informação são o mesmo registo, distinguido por checkbox.** Muito do que há a reportar não corresponde a item algum do catálogo. Lista livre, adicionável em qualquer modo, com uma caixa "é anomalia" — desmarcada, é informação. Reclassificável a qualquer momento.

**O boletim monta-se quase sozinho.** O bloco "Informações adicionais" é um recapitulativo de estrutura estável — elementos de organização de cada equipa, transmissão, MVP, Ranking Puro Futebol, força policial e ARDs, emergência médica, debriefing. Quase tudo já foi recolhido no guião e no pré-jogo. O delegado só acrescenta o debriefing e os desvios.

Preparação do jogo e Organização da imprensa são avaliadas **apenas para a equipa visitada**.

---

## 8. Reunião preparatória

### Biblioteca de guiões

Não há um guião — há uma **biblioteca de guiões**, cada um uma entidade com nome próprio.

**A app é distribuída com três guiões já configurados**: futebol, futsal e futebol de praia.

- **Escolhido ao criar o jogo.** O desporto sugere o mais provável, mas a escolha é livre.
- **Criados por duplicação.** Quase todos os guiões são variações de outro; duplicar e ajustar é a operação principal.
- **Congelados no jogo.** Ao ser escolhido, o conteúdo é copiado para dentro do jogo, não referenciado. Editar um guião mais tarde nunca altera jogos passados.

### Estrutura de um guião

Um guião é uma sequência de **slides**, um por tema. Cada slide contém:

- **Bullets** com texto e descrição — o que há a dizer
- **Campos a recolher**, referenciados da biblioteca comum

Os campos vivem dentro do slide a que pertencem. A lista de pendentes é a varredura dos campos obrigatórios de todos os slides, e um pendente a vermelho aparece exatamente no slide onde vai ser necessário.

Um slide pode ser **apenas de dados**, sem bullets — é o caso do primeiro, com os dados do jogo.

### Condições

O que é condicional são os **bullets**, os **campos** e, nalguns casos, os próprios **slides**.

O estado de cada condição é próprio de cada jogo e **pode mudar a qualquer momento** — na criação, no pré-jogo, na reunião ou a meio do jogo.

**Três estados, não dois.** O estado inicial é sempre *indeterminado*, nunca *não* — caso contrário um ponto desaparece em silêncio e a falta só se nota quando já não há como a corrigir.

| Estado | Item na apresentação | Campos dependentes |
|---|---|---|
| **Sim** | visível | entram nos pendentes |
| **Não** | oculto | não existem (N/A automático) |
| **Indeterminado** | visível, **marcado como condicional** | listados à parte como "dependentes de confirmação", sem bloquear o fecho |

O estado indeterminado reproduz o que se faz na prática: mencionar o ponto em condicional, para o caso de vir a aplicar-se.

**Condições com nome** são partilhadas por vários itens em sítios diferentes — "flash interview" governa um bullet da reunião, uma verificação de pré-jogo (backdrop) e campos de pós-jogo. Marca-se uma vez, aplica-se a todos.

**Negação.** Uma condição pode ser referenciada por `{ id, quando: "sim" | "nao" }`, permitindo itens que só existem na *ausência* da condição.

### Reorganização condicional de slides

Dois mecanismos, ambos usados no caso do policiamento:

**Posição alternativa de slide.** Um slide pode assumir outra posição na sequência quando uma condição não se verifica. Sem policiamento, o slide da força policial não existe e o do Gestor de Segurança sobe para a sua posição.

**Migração de bullets entre slides.** Um bullet pode mudar de slide conforme a condição — escrito uma vez, dirigido a interlocutor diferente. No caso do policiamento, quatro das cinco perguntas passam do comandante para o gestor de segurança.

### Vista de script

Fora do modo apresentação existe a **vista de script**: a reunião em formato de lista, percorrível slide a slide, onde se decide para *aquele jogo* que bullets e campos entram e quais ficam de fora.

Não é edição do guião — é marcação do jogo. O guião congelado permanece intacto.

**Regras:**

- **Alternar está sempre à mão**, incluindo no modo Jogo. Confirmada a flash interview ao intervalo, é um toque, e os campos pós-jogo passam nesse momento a contar como pendentes.
- **Desligar uma condição nunca apaga dados.** Informação já preenchida fica guardada e oculta, com aviso de que existe.
- **Aviso de confirmação tardia.** Se uma condição que governa campos de pré-jogo for marcada como "sim" já com o jogo a decorrer, a app avisa que ficaram itens por verificar.

### Campos: biblioteca única

Os campos do guião e os campos obrigatórios de competição **não são coisas distintas** — uns e outros são referências à mesma biblioteca. A lista de ações, por exemplo, é mostrada no slide da reunião **e** consumida pelo boletim: preenche-se uma só vez.

- **Biblioteca reutilizável.** A 1.ª linha de publicidade, o diretor de comunicação e as ações de promoção repetem-se entre Liga 3 e Liga BPI. Corrigir a redação num sítio propaga a todas.
- **Âmbito: jogo ou por equipa.** No segundo caso a app instancia o campo duas vezes, casa e visitante, sem duplicação na configuração.
- **Ligação a verificações.** Um campo de competição pode apontar para um item da checklist e **herdar o que lá está**, ficando editável — é o caso do relvado na Liga 3, que existe como verificação e como campo do boletim mas se preenche uma vez.
- **Declarados no pré-jogo, confirmados no pós-jogo.** As ações de responsabilidade social e de promoção declaram-se antes (quais são, origem, se vão ocorrer) e confirmam-se no fim (se ocorreram e como correram). A confirmação é sempre no fim, em todas as competições.
- **Preenchíveis na criação do jogo**, se a informação já for conhecida; nesse caso nunca chegam a aparecer como pendentes.

### Competições configuradas

Cinco competições com campos próprios (ver `seed/competicoes.json`): Liga 3 Placard, Liga BPI, Liga Next Gen, Liga Placard de Futsal e Liga Feminina Placard.

**Todas as outras não têm campos específicos** — apenas o fluxo normal. A configuração de competição é opcional, não obrigatória: um conjunto vazio é o caso normal, não configuração incompleta. Criar uma competição pontual deve exigir apenas nome e desporto.

---

## 9. Regulamentos

- Um PDF por competição, importado do armazenamento do dispositivo, com metadados de competição, época e versão
- Guardado localmente e sempre disponível offline
- Visualizador **pdf.js** — dá pesquisa de texto, que é o que interessa quando há um delegado à espera de resposta
- Acesso rápido durante o jogo: botão sempre visível, abre por cima sem perder o registo em curso

---

## 10. Fotos e anexos

### Anexos do jogo

Imagens de referência adicionadas na preparação e consultadas no dia: maquetes de equipamento, cartaz de segurança, planta do recinto, comunicações da FPF.

- Várias por jogo, com etiqueta livre (equipa A, equipa B, arbitragem, ...)
- Comprimidas ao importar — uma captura de maquete traz vários megabytes desnecessários
- Consultáveis em qualquer modo, com acesso rápido a partir do slide dos Delegados
- Visualizador com zoom, para comparar detalhes de cor
- Incluídas no ficheiro `.dfpf` ao exportar o jogo

### Fotos de registo

**Nomenclatura fixa:** `foto_1`, `foto_2`, ... numeradas sequencialmente por jogo. É por este nome que são referenciadas no texto dos relatórios — "conforme fotografias foto_1 a foto_5".

**A numeração nunca se reajusta.** Apagada a `foto_2`, as restantes mantêm os números e fica um espaço vazio. Renumerar quebraria referências já escritas.

**Inserção de referência no texto com um toque**, gerando a formulação completa.

**Download** individual ou em lote, com os nomes preservados, para anexar onde for necessário.

Guardadas comprimidas em IndexedDB.

---

## 11. Fases

As fases seguem os modos — cada uma entrega um modo utilizável de ponta a ponta.

**Espigões técnicos (antes de tudo)**
Dois pontos onde a documentação costuma mentir:
- pdf.js a abrir um PDF local dentro da app empacotada, com pesquisa de texto
- Foreground service com notificação persistente **no dispositivo real**, com a otimização de bateria do fabricante ativa

**F0 — Fundação e esqueleto de modos**
Projeto, Capacitor, tipos do domínio, Dexie, importação da configuração de `seed/`, criar e listar jogos, navegação por modos com a fase persistida.

**F1 — Modo Pré-jogo**
Entidades clube, competição e equipa (criáveis no momento), anexos do jogo, pontos a validar, checklist derivada do desporto, registos do boletim com checkbox, recolha para a reunião com pendentes, contagem decrescente, ocorrências, fotos numeradas.

**F1.5 — Configuração (fase mais pesada de UI)**
Biblioteca de campos reutilizáveis, condições com três estados e negação, biblioteca de guiões com duplicação e congelamento, gestão de desportos, competições e regulamentos, editor de frases-modelo e fragmentos. Tudo o resto consome esta configuração.

**F2 — Modo Reunião**
Vista de script, apresentação fullscreen, pendentes a vermelho com preenchimento inline, posição alternativa de slides e migração de bullets.

**F3 — Modo Jogo**
Ecrã único com botões grandes, relógio derivado apenas no futebol com minuto editável, introdução manual nos restantes, teclado numérico próprio, atalhos de números usados, persistência imediata, foreground service, ficha de jogo consultável e editável.

**F4 — Modo Pós-jogo e relatórios**
Campos de recolha pós-jogo com pendentes próprios, motor de frases-modelo com blocos editáveis, referências a fotos, geração dos dois relatórios, copiar por secção e completo.

**F5 — Histórico e transferência**
Lista, pesquisa, filtros. Export e import de `.dfpf` e `.dfpfconf` com partilha nativa, resolução de duplicados por escolha explícita, cópia de segurança do arquivo completo.

**Eliminação de um jogo** apaga tudo o que lhe está associado: registos, fotos, anexos e relatórios. Exige confirmação, por ser irreversível. Sem cloud, não há propagação a resolver — o que existir noutro dispositivo apaga-se lá.

---

## 12. Notas para o Claude Code

**`CLAUDE.md` na raiz** com as regras invioláveis, lido em todas as mensagens.

**Fronteiras explícitas de pastas:**

```
src/
├─ domain/      puro, sem I/O, testável com Vitest
├─ adapters/
│   ├─ storage/    Dexie
│   ├─ ficheiros/  export, import, partilha nativa
│   └─ nativo/     foreground service, câmara
└─ ui/
```

Regra de dependência: `ui` → `domain`, `adapters` → `domain`. O `domain` não importa de ninguém.

**Ordem de construção:** tipos e esquemas de configuração **primeiro**; só depois a UI, gerada a partir deles.

**Testes onde interessa:** motor de frases-modelo, regras de tempo por desporto, resolução de pendentes e condições, import/export. O resto é UI e verifica-se a olho.

**Uma fase por sessão.** Dar o documento todo e pedir "constrói" produz trabalho a meio em todo o lado.

---

## 13. Configuração inicial

Todo o material da FPF foi recolhido e transposto para `seed/`:

- `guiao-futebol.json` — 10 slides
- `guiao-futsal.json` — 8 slides
- `guiao-futebol-praia.json` — 8 slides
- `relatorio-ocorrencias.json` — secções, regras de composição, 30 frases-modelo e fragmentos
- `boletim-organizacao-jogo.json` — secções e bloco de informações adicionais
- `competicoes.json` — as cinco competições regulares e os seus campos obrigatórios
