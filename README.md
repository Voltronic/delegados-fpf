# Delegado FPF

Aplicação de apoio à função de delegado da Federação Portuguesa de Futebol em jogos de futebol, futsal e futebol de praia.

Cobre o ciclo completo de um jogo: verificações pré-jogo, reunião preparatória com modo de apresentação, registo ao vivo de eventos e ocorrências, e geração do Relatório de Ocorrências e do Boletim de Organização de Jogo em texto, prontos a copiar para o SCORE.

## Princípios

- **Tudo no dispositivo** — sem cloud, sem contas, sem servidor; funciona 100% offline
- **Custo zero** — sem serviços pagos, sem loja de aplicações, sem infraestrutura
- **Transferência por ficheiro** — um jogo passa para outro dispositivo por Quick Share, Bluetooth ou email
- **Tudo configurável na app** — guiões, competições, campos, regulamentos e frases-modelo são dados, não código

## Stack

React + TypeScript + Vite + Tailwind + shadcn/ui, empacotado com Capacitor para Android.
Dexie (IndexedDB) como única fonte de verdade, pdf.js para regulamentos, Vitest para testes.

## Documentação

| Ficheiro | Conteúdo |
|---|---|
| `CLAUDE.md` | Regras invioláveis, lidas pelo agente em todas as mensagens |
| `docs/PLANO.md` | Especificação completa: princípios, modelo de dados, modos, relatórios |
| `docs/ARRANQUE.md` | Roteiro por fases, uma por sessão de trabalho |

## Configuração inicial (`seed/`)

Valores por omissão da app, importados na primeira execução e editáveis a partir daí.

| Ficheiro | Conteúdo |
|---|---|
| `guiao-futebol.json` | Guião da reunião preparatória de futebol (10 slides) |
| `guiao-futsal.json` | Guião de futsal (8 slides) |
| `guiao-futebol-praia.json` | Guião de futebol de praia (8 slides) |
| `competicoes.json` | Cinco competições regulares e os seus campos obrigatórios |
| `relatorio-ocorrencias.json` | Secções, regras de composição e 30 frases-modelo |
| `boletim-organizacao-jogo.json` | Secções e bloco de informações adicionais |

### Formatos de ficheiro

| Extensão | Conteúdo |
|---|---|
| `.dfpf` | Um jogo completo: dados, fotos e anexos |
| `.dfpfconf` | Configuração completa: guiões, competições, campos, frases-modelo |

Ambos se partilham pelo menu nativo do Android e importam-se tocando no ficheiro recebido.

## Arranque

```bash
npm install
npm run dev              # browser, hot reload
npm run dev -- --host    # testar em tablet/telemóvel na mesma Wi-Fi
npx cap run android --live-reload   # app nativa com hot reload
```
