# CLAUDE.md

App de apoio à função de delegado da FPF em jogos de futebol, futsal e futebol de praia.
Antes de escrever código, lê `docs/PLANO.md`. Para a fase atual, lê `docs/ARRANQUE.md`.

## Regras invioláveis

1. **Tudo no dispositivo.** Não há cloud, contas nem servidor. IndexedDB é a única fonte de verdade. Nunca assumir rede. A transferência entre dispositivos é por ficheiro `.dfpf` e partilha nativa do Android.
2. **Persistência imediata.** Todo o registo escreve em disco no momento. Não existe estado de jogo apenas em memória nem ação de "guardar" no fim.
3. **O relógio é derivado, nunca acumulado.** Guardar timestamps e calcular a partir da hora do sistema. Nada de contadores de ticks.
4. **Nada de lógica de desporto ou competição em código.** Tudo vem de configuração (`seed/`). Se aparecer um `if (desporto === ...)`, está errado.
5. **Velocidade acima da estética.** Qualquer registo em três toques ou menos, nunca mais de um ecrã de profundidade. Sem diálogos encadeados e sem confirmações — desfazer em vez de confirmar.
6. **TypeScript strict.** Sem `any`. Domínio puro e testável, separado de adaptadores e UI.
7. **Nunca apagar dados em silêncio.** Desligar uma condição oculta, não elimina. Apagar exige confirmação explícita.
8. **Português europeu** em toda a interface, comentários e documentação.

## Stack

React + TypeScript + Vite + Tailwind + shadcn/ui, empacotado com Capacitor (Android).
Dexie (IndexedDB), pdf.js, Vitest. Distribuição por APK.
Custo zero: sem servidor, sem serviços pagos, sem loja de aplicações.

## Estrutura

```
src/
├─ domain/      puro, sem I/O, testável
├─ adapters/
│   ├─ storage/    Dexie
│   ├─ ficheiros/  export, import, partilha nativa
│   └─ nativo/     foreground service, câmara
└─ ui/
```

Regra de dependência: `ui` → `domain`, `adapters` → `domain`. O `domain` não importa de ninguém.

## Método

- Uma fase de cada vez. Não avançar sem a atual a funcionar.
- Tipos e esquemas de configuração primeiro; UI depois, gerada a partir deles.
- Testes onde interessa: motor de frases-modelo, regras de tempo por desporto, condições e pendentes, import/export.
- `versaoEsquema` em cada jogo desde o início — um ficheiro exportado de uma versão mais recente não pode ser lido em silêncio por uma versão antiga.
