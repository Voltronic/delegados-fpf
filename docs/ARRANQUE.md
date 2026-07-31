# Roteiro de arranque

Ordem de trabalho para o Claude Code. **Uma fase por sessão.** Dar o documento todo e pedir "constrói" produz trabalho a meio em todo o lado.

Antes de cada fase, o agente lê `docs/PLANO.md` e a secção correspondente aqui.

---

## Espigões técnicos (fazer primeiro, ~1 dia)

Pontos onde a documentação costuma mentir. Validar isoladamente, sem construir app à volta.

1. **pdf.js dentro da app empacotada** — abrir um PDF guardado localmente, com pesquisa de texto a funcionar.
2. **Foreground service com notificação persistente** — no dispositivo real, com a otimização de bateria do fabricante ativa. Confirmar que a app sobrevive a 30 minutos em segundo plano com o ecrã desligado.
3. **Partilha e receção de ficheiros** — exportar um ficheiro `.dfpf`, enviá-lo por Quick Share para outro dispositivo, e confirmar que tocar nele abre a app (intent filter registado).

Se algum falhar, o plano ajusta-se **antes** de existir código a depender dele.

---

## F0 — Fundação e esqueleto de modos

- Projeto Vite + React + TS + Tailwind + shadcn/ui, Capacitor configurado para Android
- `domain/` com os tipos do modelo de dados (ver `docs/PLANO.md`, secção 4)
- Dexie com os esquemas correspondentes
- Importação da configuração de `seed/` no primeiro arranque
- `versaoEsquema` em cada jogo
- Criar e listar jogos
- Navegação pelos quatro modos, com a fase persistida em cada jogo

**Feito quando:** criar um jogo, fechar a app, reabrir e encontrá-lo no modo onde ficou.

---

## F1 — Modo Pré-jogo

- Entidades clube, competição e equipa (equipa = clube × competição, única)
- Catálogo de verificações derivado do desporto
- Estados ok / anomalia / n/a, com nota e fotos numeradas `foto_1`, `foto_2`...
- Registos do boletim: lista livre com checkbox "é anomalia"
- Pontos a validar: adicionados à mão, mais sugestão das anomalias dos **dois últimos jogos** da equipa, nenhuma pré-selecionada
- Anexos do jogo (maquetes, plantas), importados do dispositivo e comprimidos
- Contagem decrescente para o início do jogo, derivada de timestamp
- Painel de pendentes por categoria

**Feito quando:** preparar um jogo do início ao fim em modo avião.

---

## F1.5 — Configuração (fase mais pesada de UI)

O motor de campos configuráveis é o coração do projeto. Construir bem aqui poupa tudo o resto.

- Biblioteca de campos reutilizáveis entre competições
  - Tipos: texto, número, sim/não, sim/não/não-utilizado, seleção, presença, escala qualitativa, lista repetível
  - Âmbito: jogo ou por equipa
  - Momento: pré, jogo, pós; com confirmação diferida para pós
  - `herdaDeVerificacao` para não preencher duas vezes
  - Estados do valor: preenchido, pendente, a definir na reunião, não aplicável
- Condições com três estados (sim / não / indeterminado) e negação (`quando: "sim" | "nao"`)
- Biblioteca de guiões, criados por duplicação, congelados no jogo ao serem escolhidos
- Gestão de desportos, competições e regulamentos (importados do dispositivo)
- Posição alternativa de slides e migração de bullets entre slides
- Editor de frases-modelo e fragmentos
- Importar `seed/` como valores por omissão

**Feito quando:** as cinco competições e os três guiões de `seed/` carregam e são editáveis na app, sem tocar em ficheiros.

---

## F2 — Modo Reunião

- Vista de script: percorrer slides e marcar, para aquele jogo, que bullets e campos entram
- Modo apresentação: fullscreen, tipografia grande, navegação por swipe
- Pendentes a vermelho, marcados como "a obter durante a reunião", preenchíveis inline sem sair do fullscreen
- Slides com posição alternativa (Gestor de Segurança sobe quando não há policiamento)
- Bullets que migram de slide conforme a condição

**Feito quando:** conduzir uma reunião completa a partir da app, resolvendo pendentes à frente dos presentes.

---

## F3 — Modo Jogo

- Ecrã único, botões grandes, uso com uma mão
- Relógio derivado de timestamp **apenas no futebol**, minuto pré-preenchido e sempre editável
- Futsal e futebol de praia: introdução manual, teclado numérico próprio no ecrã, seletor de período
- Números de atleta escritos à mão; números já usados ficam como atalho por equipa
- Registo de ocorrência sempre visível: tipo, tempo, equipa, texto corrido — nada mais
- Campos com momento "jogo" (contagem de adeptos, espectadores) com lembrete ao intervalo
- Ficha de jogo consultável, com todos os eventos editáveis e apagáveis
- Foreground service com o minuto na notificação

**Feito quando:** registar um jogo inteiro com o telemóvel no bolso entre registos, sem perder nada.

---

## F4 — Modo Pós-jogo e relatórios

- Campos de recolha pós-jogo com pendentes próprios
- **Relatório de Ocorrências** (`seed/relatorio-ocorrencias.json`): secções classificadas automaticamente, "Ver Outros comentários", numeração sequencial, fecho padrão
- Cada ocorrência montada a partir de uma frase-modelo do seu tipo, com a nota original visível e **blocos editáveis individualmente**
- Inserção de referências a fotos (`foto_1`, `foto_2`) no texto
- **Boletim de Organização de Jogo** (`seed/boletim-organizacao-jogo.json`): secções, e bloco de informações adicionais composto a partir de dados já recolhidos
- Copiar por secção e copiar tudo

**Feito quando:** gerar os dois relatórios de um jogo real e colá-los no SCORE sem reescrever nada de raiz.

---

## F5 — Histórico e transferência

- Lista, pesquisa, filtros por competição e data
- Apagar com eliminação em cascata (registos, fotos, anexos, relatórios), com confirmação
- **Export/import `.dfpf`** (um jogo com fotos e anexos) através da partilha nativa do Android
- **Export/import `.dfpfconf`** (configuração completa) para propagar guiões ao segundo dispositivo
- Resolução de duplicados na importação: manter o local, substituir, ou guardar como cópia — nunca automático
- Cópia de segurança do arquivo completo para a pasta Downloads
- Descarregar fotos, individualmente ou todas, com os nomes preservados

---

## Notas de trabalho

- O Claude Code não vê a interface. Tirar captura de ecrã e colar no chat é a forma mais rápida de corrigir layout.
- `npm run dev` para o dia a dia; `npm run dev -- --host` para testar no tablet e telemóvel na mesma Wi-Fi; `npx cap run android --live-reload` quando forem precisos plugins nativos.
- O critério dos três toques só se avalia com o dedo, não com o rato. Testar em dispositivo cedo e com frequência.
