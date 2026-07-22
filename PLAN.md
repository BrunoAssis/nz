# Neutral Zone — PLAN v0

Plano de implementação da v0, jogável no browser. Decisões fechadas com base no [GDD](GDD.md) e nas respostas de escopo. Valores numéricos marcados com ⚖️ são propostas para revisão/playtesting e viverão todos em `src/sim/config.ts`.

## Decisões de escopo (fechadas)

| Tema | Decisão |
|---|---|
| Tempo | Tempo real com **ticks econômicos** (1 tick = 5s ⚖️). Movimento, exploração e combate correm em tempo real contínuo. |
| Jogadores | **1 humano vs 1 IA simples**. Sem rede. |
| Visual | **3D low-poly com Three.js** desde a v0. |
| Território | **Sistema de influência completo** (GDD §8). |
| Combate | Fórmula do GDD + **suporte 1.1x** de aliados adjacentes. Engaja **só ao tentar entrar em hex ocupado**. |
| Vitória | **Territorial (50%+1)** e **Militar** (eliminar unidades e planetas inimigos). Econômica fica pra v1. |
| Fora da v0 | Unidades neutras/eventos, Stealth Ship, vitória econômica, multiplayer, objetivos secundários, recuperação de Health/Shield (ver Simplificações). |

## Stack técnico

- **Vite + TypeScript**, sem framework de UI (HUD em HTML/CSS overlay sobre o canvas).
- **Three.js** para renderização; raycasting para picking de hex/unidade.
- **Vitest** para testes da simulação.
- Arquitetura em duas camadas rigorosamente separadas:
  - `src/sim/` — simulação pura e determinística. Nenhum import de Three.js ou DOM. Avança por `tick(state, dtMs)`. Toda aleatoriedade via RNG com seed.
  - `src/render/` + `src/ui/` — leem o estado e desenham; enviam comandos (`MoveUnit`, `Colonize`, `Build`...) para a simulação. A IA usa a mesma interface de comandos do jogador.

```
src/
  sim/        config.ts, hex.ts, state.ts, tick.ts, movement.ts, combat.ts,
              exploration.ts, colonization.ts, influence.ts, economy.ts,
              production.ts, victory.ts, commands.ts, rng.ts
  ai/         ai.ts (emite comandos como um jogador)
  render/     scene.ts, hexGrid.ts, units.ts, planets.ts, fog.ts, camera.ts, picking.ts
  ui/         hud.ts, panels.ts (planeta/unidade), toasts.ts, gameOver.ts
  main.ts     loop: input → comandos → sim.tick → render
```

## Regras da simulação

### Grid e mapa

- Hex axial `(q, r)`, orientação pointy-top.
- Mapa: hexágono de **raio 5** (91 hexes) para 2 jogadores ⚖️.
- ~**12 planetas** ⚖️ posicionados aleatoriamente (com distância mínima de 2 hexes entre si ⚖️), mais 2 planetas-casa.
- Homes em lados opostos do mapa, sorteados entre ~10 hexes candidatos por lado (GDD).
- Estado inicial por jogador: planeta-casa, 1 Scout no hex da casa, **$10**, visibilidade raio 2 da casa.

### Tempo

- Simulação avança por `dtMs` a cada frame (determinística: mesmos comandos + mesma seed = mesmo resultado).
- **Tick econômico a cada 5s** ⚖️: cada planeta do jogador gera $1 (×1.2 se tiver Market).

### Unidades e movimento

- Stats do GDD (Scout 3/2/1/10/5, Fighter 2/3/2/10/10, Battlecruiser 1/5/3/20/10).
- **1 unidade por hex**.
- Click-to-move com pathfinding A* sobre hexes explorados; hexes ocupados por aliados são obstáculos, hex final ocupado por inimigo = ordem de ataque.
- Tempo por hex = `6s / Speed` ⚖️ (Scout 2s, Fighter 3s, Battlecruiser 6s), com interpolação visual do movimento.
- Destino inexplorado adjacente a caminho explorado: a unidade anda até ele e inicia exploração ao entrar.
- **Sem recuperação de Health/Shield na v0** (ver Simplificações).

### Exploração e fog

- Estados de hex: `unexplored` (fog cinza; pitch-black se nenhum vizinho explorado) e `explored`.
- Explorar = mover unidade para hex inexplorado → **10s** com barra de progresso no perímetro do hex → revela só aquele hex. Sair antes cancela tudo (sem progresso parcial).
- Unidades **não** revelam fog por proximidade.
- Visibilidade de unidades inimigas: inimigos são visíveis quando estão em hexes explorados pelo jogador (simplificação v0; "hidden enemies" do GDD §Design Note fica pra v1, já que não há stealth).

### Combate

- Engaja quando uma unidade recebe ordem de entrar num hex ocupado por inimigo: a atacante para no hex adjacente, **travada em combate** (não se move até o fim, salvo ordem explícita de recuo, que cancela o combate).
- **Rounds a cada 1s** ⚖️: ambos aplicam simultaneamente `max(1, Atk_efetivo − Def_efetivo)`; dano consome Shield antes de Health; Health 0 = destruição.
- **Suporte**: cada aliado em hex adjacente à unidade e que não esteja em combate dá ×1.1 no Attack e na Defense dela (2 aliados = ×1.21; arredonda no cálculo final, não por fator).
- Atacante vence → completa o movimento para o hex. Defensor vence → permanece.

### Planetas: statblock, defesa e conquista

- Todo planeta tem statblock próprio (Attack/Defense/Health/Shield, sem Speed), mantido independente do dono. Arquétipos ⚖️:

| Arquétipo | Atk | Def | HP | Shield | Nota |
|---|---|---|---|---|---|
| Pacific | 0 | 0 | 10 | 0 | Neutro não resiste: colonização direta |
| Standard | 2 | 2 | 15 | 5 | Revida |
| Fortress | 4 | 3 | 25 | 10 | Raro, defesa pesada |
| Home | 3 | 2 | 20 | 10 | Planeta inicial dos jogadores |

- Distribuição dos neutros na geração: ~50% Pacific, ~35% Standard, ~15% Fortress ⚖️.
- Planetas nunca iniciam combate. Planetas de jogador sempre se defendem (mesmo Pacific); neutros só Standard/Fortress resistem.
- **Atacar planeta**: mesma mecânica do combate de unidades (atacante trava no hex adjacente, rounds de 1s, mesma fórmula). Se há unidade inimiga no hex do planeta, luta-se contra ela primeiro (planeta com Atk > 0 conta como 1 suporte pra ela); destruída, renovar o ataque engaja o planeta.
- **Suporte ao planeta**: todas as unidades aliadas adjacentes + a unidade no próprio hex dão ×1.1 (Atk e Def) ao planeta, desde que não estejam travadas em combate próprio.
- **Health 0**: planeta não se defende mais; 1 improvement aleatório destruído; construção em progresso destruída sem reembolso e resto da fila reembolsado 100% ao dono.
  - Planeta **neutro** a 0 (ou Pacific neutro): atacante entra no hex e a **colonização inicia automaticamente** (custo normal; sem $, inicia assim que der ⚖️).
  - Planeta **de outro jogador** a 0: atacante entra e inicia **de-colonização** automática — duração = colonização/2 (**15s** ⚖️), grátis, com barra. Completa → planeta vira neutro; se o ocupante ficar, colonização inicia automaticamente em seguida.
  - Ocupante sai/morre durante a de-colonização → cancela tudo; planeta segue do dono original, indefeso a health 0 (novo invasor recomeça a de-colonização do zero).
  - Planeta a health 0 continua produzindo $ pro dono.
- Sem regen de planeta na v0: Health/Shield restauram full apenas ao completar uma colonização (novo dono recebe o planeta inteiro).

### Colonização

- Requer: hex explorado, contém planeta neutro, unidade do jogador no hex. Neutro Standard/Fortress precisa antes ser reduzido a health 0 em combate; Pacific coloniza direto.
- Custo: $1 × 2^(planetas já colonizados pelo jogador) — $1, $2, $4, $8... (casa não conta ⚖️). Cobrado ao iniciar.
- Duração: **30s** ⚖️ com barra de progresso. Cancelar (ou unidade sair/morrer) reembolsa proporcional ao tempo restante.
- Completa → planeta passa ao jogador e injeta influência no território.

### Território (influência — GDD §8)

- Planeta colonizado: hex vira território do dono imediatamente.
- Cada planeta tem `influence = 5` ⚖️ e emana influência por BFS através do território contíguo do dono; ao atingir um hex não-possuído, aplica `influence − distância` (mínimo 0) nesse hex.
- Hex não-possuído acumula influência por jogador ao longo do tempo (+valor por tick econômico ⚖️); há também influência "neutra" fixa (2 ⚖️) representando o vazio. Quando a influência de um jogador supera todas as outras, o hex **vira** território dele e os acumuladores zeram.
- Território que perde contiguidade com um planeta do dono perde a influência daquele planeta; hex órfão (sem planeta alcançável) reverte a neutro.
- Recalculado a cada tick econômico (91 hexes — barato).
- Vantagem de combate em território próprio (GDD, TBD mecânico): **+10% Defense** para unidades no próprio território ⚖️.

### Planetas e melhorias

- Produção base $1/tick; **Market** $20 (produção ×1.2); **Shipyard** $30 (construção de naves ×1.2 de velocidade). Máx. 1 de cada por planeta.
- **Fila de construção por planeta**:
  - Enfileirar cobra o custo **na hora**. Cancelar (item na fila ou em progresso) reembolsa `custo × (1 − progresso)`; item ainda não iniciado reembolsa 100%.
  - A construção só **inicia** com o hex do planeta desocupado; se há nave estacionada, a fila espera e o planeta exibe **ícone de "fila bloqueada"**. Quando a nave sai, a construção começa automaticamente.
  - Nave em construção **ocupa o hex** do planeta (nenhuma outra nave entra até concluir). A nave pronta surge no hex.
- Custos/tempos ⚖️ (para revisão):

| Nave | Custo | Tempo |
|---|---|---|
| Scout | $5 | 10s |
| Fighter | $10 | 15s |
| Battlecruiser | $20 | 25s |

### Vitória / derrota

- **Territorial**: controlar 46+ dos 91 hexes.
- **Militar**: oponente sem unidades **e** sem planetas.
- Checada a cada tick econômico; tela de fim de jogo com botão de novo jogo.

## IA (v0 — heurística simples)

Roda a cada tick econômico, emite comandos normais:

1. Sem Scout vivo e $ suficiente → constrói Scout.
2. Scout ocioso → explora o hex inexplorado mais próximo da casa.
3. Planeta neutro conhecido + $ pra colonizar → envia unidade mais próxima e coloniza.
4. Economia: se $ ≥ custo, constrói Market no planeta mais produtivo sem Market.
5. Militar: mantém ~1 Fighter por 2 planetas; com sobra, Battlecruiser. Ataca unidade/planeta inimigo visível se tiver vantagem local de stats.

Sem cheats: a IA respeita o próprio fog.

## Renderização e UI

- **Cena**: fundo espacial escuro (skybox/estrelas com shader simples), grid de hexes translúcidos com fill da cor do dono, fog cinza-nublado (inexplorado) e preto (inalcançável).
- **Modelos low-poly geométricos**: planetas = esferas (variações: anéis/estação), naves = formas simples; cor predominante do jogador (azul vs vermelho) + detalhe fixo amarelo.
- **Câmera**: perspectiva inclinada, pan (arrasto/bordas), zoom (scroll/pinch).
- **Interação**: clique/tap seleciona; clique em destino move; feedback visual de seleção, caminho e alvos válidos. Mouse-only e touch funcionais.
- **HUD**: barra superior ($, nº de planetas), painel contextual inferior (planeta: fila de construção com progresso e botão de cancelar, melhorias, status de produção; unidade: stats, HP/Shield, botão Colonizar quando aplicável), ícone de "fila bloqueada" sobre o planeta, barras de progresso (exploração no perímetro do hex; colonização/construção), tela de vitória/derrota.

## Simplificações da v0 (explícitas)

- Sem unidades neutras nem eventos aleatórios (GDD §10).
- Sem regeneração de Health/Shield (unidades e planetas) — o GDD prevê recuperação lenta; entra na v1 com os valores de taxa. Planeta só restaura ao ser colonizado.
- Sem "hidden enemies"/surpresa (depende de stealth, que é v1+).
- Sem vitória econômica (threshold TBD no GDD).
- Sem stacks de unidades, sem múltiplos jogadores/rede.

## Etapas de implementação

1. **Fundação sim**: projeto Vite+TS, `hex.ts` (axial, vizinhos, distância, A*), `state.ts`, `config.ts`, geração de mapa com seed. Testes.
2. **Loop e movimento**: `tick`, comandos, movimento temporizado, exploração com cancelamento. Testes.
3. **Render mínimo**: cena Three.js, grid, fog, planetas/naves primitivos, câmera, picking, click-to-move jogável.
4. **Economia e produção**: ticks de $, colonização (custo dobrado/reembolso), melhorias, fila de construção de naves (cobrança imediata, cancelamento com reembolso, bloqueio por hex ocupado). Painéis de planeta/unidade no HUD.
5. **Combate**: engajamento, rounds, suporte, combate contra planetas (statblocks, suporte ao planeta), health 0 → de-colonização → colonização automática. Feedback visual (barras de HP/Shield de unidades e planetas, efeito de dano).
6. **Influência**: propagação, conversão de hexes, contiguidade, bônus defensivo territorial. Fill colorido no render.
7. **IA + vitória**: heurística acima, condições de vitória, tela de fim de jogo.
8. **Polish v0**: modelos low-poly finais, barra de progresso perimetral, touch, balanceamento inicial via `config.ts`.

Cada etapa termina com o jogo rodando (`npm run dev`) e testes verdes (`npm test`).
