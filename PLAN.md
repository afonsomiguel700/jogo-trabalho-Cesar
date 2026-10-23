# GRID ZERO — Planejamento Técnico

> **Comece do zero. Termine na frente.**

Documento de planejamento do jogo de corrida 2D top-down **Grid Zero**.
Versão: v2 (correções de projeto incorporadas).

---

## 1. Identidade

| Campo | Valor |
|---|---|
| Título | **GRID ZERO** |
| Slogan | *Comece do zero. Termine na frente.* |
| Slug / pacote | `grid-zero` |
| Gênero | Corrida 2D top-down, arcade |
| Campeonato | Campeonato Grid Zero |
| Equipe do jogador | Grid Zero Racing |
| Rival | V. Ramos (IA, skill fixa) |
| Símbolo | Um `0` formado por uma pista vista de cima, cortado pela linha de chegada |
| Paleta | Preto, branco e um acento (vermelho ou ciano) |
| Tipografia | Geométrica, sem serifa; números tabulares para tempos |

**Cuidados de nome:** existe a franquia *GRID* (Codemasters). "Grid Zero" é distinto como
título, mas evitar logo que sugira continuação; usar `gridzero`/`grid-zero` como slug e
sempre acompanhar de "jogo de corrida 2D" nos metadados.

---

## 2. Visão e pilares

Corrida 2D top-down estilo F1 arcade: o jogador pilota um carro em pistas fechadas, com
aceleração, frenagem, perda de aderência em zebras e grama, voltas cronometradas e
adversários com IA. Física **arcade** — divertida de pegar, com sensação de peso e
derrapagem.

**Pilares**

1. Controle responsivo em 5 segundos de jogo.
2. Curvas recompensam trajetória limpa, não só acelerar.
3. Base de código pequena, determinística e testável, fácil de expandir.

---

## 3. Objetivos

**Objetivo geral**
Desenvolver um protótipo jogável de corrida 2D top-down com física arcade, corrida
completa (voltas, tempos, validação) e base de código preparada para evoluir.

**Objetivos específicos**

1. Implementar física de carro em passo fixo, com unidades SI e metas numéricas de *game feel*.
2. Modelar pistas como dados (spline + largura) e resolver progresso na pista sem ambiguidade.
3. Implementar corrida com setores, validação de volta e HUD.
4. Implementar IA com linha de corrida e perfil de velocidade.
5. Garantir qualidade por testes determinísticos de replay, CI e critérios de aceite por fase.

---

## 4. Escopo

### MVP (protótipo jogável)

- 1 pista fechada, 1 carro, câmera top-down
- Acelerar, frear, virar; atrito; zebra e grama com menos aderência
- Colisão com bordas (empurrão suave + perda de velocidade)
- Voltas: cronômetro, melhor volta, contador, setores
- Volta inválida ao cortar caminho; reset do carro
- HUD: velocidade, volta, tempo, melhor volta
- Menu inicial, pausa e tela de resultado

### Backlog (em ordem de valor)

1. IA adversária (2–5 carros) + posição na corrida
2. 3+ pistas em arquivo de dados
3. Campeonato de 4–5 etapas + Escola de Pilotagem + medalhas
4. Multiplayer local (2 jogadores)
5. Efeitos: fumaça, marcas de pneu, som de motor
6. Ghost do melhor tempo / recordes locais versionados
7. Editor simples de pista (waypoints)

### Não-objetivos

- Multiplayer online
- 3D / perspectiva pseudo-3D
- Mobile / controles por toque
- Dano, pit stop, desgaste de pneu
- Carros e marcas licenciadas (usar times e modelos fictícios)
- Simulação realista (suspensão, câmbio, diferencial)

---

## 5. Stack

| Item | Escolha | Motivo |
|---|---|---|
| Linguagem | TypeScript strict | Segurança de tipos na simulação |
| Build/dev | Vite | Hot reload, zero config |
| Render | Canvas 2D nativo | Top-down não exige engine; controle total |
| Loop | `requestAnimationFrame` + passo fixo 1/60 s | Física determinística |
| Testes | Vitest | Simulação headless |
| Lint/format | ESLint + Prettier | Consistência |
| CI | GitHub Actions | `typecheck` + `test` a cada push |
| Deploy | GitHub Pages (a partir da F1) | Link jogável desde cedo |
| Áudio | Web Audio API | Sem dependências no MVP |

**Alternativa descartada:** Phaser 3. Aceleraria HUD/menus, mas esconderia a física, que é
justamente o núcleo que se quer demonstrar.

---

## 6. Arquitetura

Regra de ouro: **`sim/` não conhece Canvas nem DOM.** Isso permite rodar a corrida headless
nos testes.

```
src/
  main.ts                # bootstrap, loop, passo fixo
  core/
    game.ts              # máquina de estados (menu → corrida → resultado)
    loop.ts              # fixed timestep + interpolação
    input.ts             # teclado/gamepad → ações abstratas
  sim/
    car.ts               # estado + física do carro (puro)
    physics.ts           # constantes e integração
    track.ts             # centerline, largura, progresso, on/off-track
    race.ts              # voltas, setores, tempos, posições
    ai.ts                # piloto automático
    rng.ts               # RNG com seed (determinismo)
  render/
    camera.ts            # follow, zoom por velocidade
    trackRenderer.ts     # desenha segmentos visíveis
    carRenderer.ts
    hud.ts
  data/
    tracks/*.ts          # centerline + largura + spawn
    career.ts            # campeonato, briefings, rival
  assets/
```

Convenções: constantes de tuning em um único módulo tipado; dados de jogo em `data/`;
estado de simulação imutável do ponto de vista do render (snapshots para interpolação).

---

## 7. Física

### 7.1 Escala e unidades

`1 unidade = 1 metro`. Tudo em SI; a renderização converte no final
(`px = m * pxPerMeter`).

```ts
export const PHYSICS = {
  massKg: 750,
  wheelbaseM: 3.2,
  maxSteerDeg: 28,
  topSpeedMps: 92,          // ~331 km/h
  engineAccelMps2: 10.5,    // ~0–100 km/h em ~2,9 s
  brakeAccelMps2: 28,
  dragCoeff: 0.0009,        // resistência ~ v²
  rollingResist: 0.35,
  gripAsphalt: 1.7,         // g lateral máxima
  gripKerb: 1.35,
  gripGrass: 0.45,
  dragGrass: 3.5,
  pxPerMeterBase: 6,
} as const;
```

### 7.2 Modelo de bicicleta + aderência

Estado: `pos`, `angle`, `velocity`, `steer`.

```ts
const forward = { x: Math.cos(angle), y: Math.sin(angle) };
const right   = { x: -Math.sin(angle), y: Math.cos(angle) };

let vf = dot(velocity, forward);   // longitudinal
let vl = dot(velocity, right);     // lateral

vf += (throttle * accel - brake * brakeAccel - drag * vf * Math.abs(vf) - rolling * vf) * dt;
vl *= grip;                        // asfalto 0.85/frame; grama ~0.98 (derrapa)

const omega = (vf / wheelbaseM) * Math.tan(steerRad);
const aLat  = omega * vf;          // aceleração lateral exigida
// se |aLat| > grip * 9.81 → limita omega e transfere excesso para vl

angle += omega * dt;
velocity = forward * vf + right * vl;
pos += velocity * dt;
```

- Direção só atua com velocidade: `turnRate = steer * maxTurn * clamp(|vf| / turnRef, 0, 1)`.
- Em ré, o sentido de giro é invertido.
- Zona da pista define `grip` e atrito extra (grama freia e puxa para fora).
- Metas numéricas: top speed ~92 m/s, 0–100 km/h ~2,9 s, raio mínimo ~6 m
  (`wheelbase / tan(28°)`), compatível com pistas de no mínimo 14 m de largura em hairpins.

---

## 8. Pista e progresso

### 8.1 Modelo

Centerline como spline **Catmull-Rom centrípeta** (`alpha = 0.5` — evita laço/overshoot em
curvas fechadas), reamostrada por **comprimento de arco** em tabela uniforme (passo 2 m).

```ts
type Track = {
  samples: { x: number; y: number; s: number; tx: number; ty: number }[];
  stepM: number;      // 2 m
  lengthM: number;
  halfWidthM: number;
};
```

Física, IA, câmera e render usam **apenas `s`** — nunca o `t` da spline.

### 8.2 Projeção com janela (evita salto de `s`)

Buscar o segmento mais próximo no array inteiro faz o `s` pular em retas paralelas → volta
conta errado. A busca é restrita a uma janela ao redor do último índice, com fallback global
apenas em teleporte/respawn:

```ts
function project(track: Track, p: Vec2, lastIdx: number) {
  const n = track.samples.length;
  const win = Math.max(4, Math.ceil(60 / track.stepM)); // ±60 m
  let best = { d2: Infinity, idx: lastIdx, s: 0, t: 0 };

  for (let k = -win; k <= win; k++) {
    const i = (lastIdx + k + n * 2) % n;
    const a = track.samples[i], b = track.samples[(i + 1) % n];
    const seg = { x: b.x - a.x, y: b.y - a.y };
    const len2 = seg.x ** 2 + seg.y ** 2;
    const t = clamp(((p.x - a.x) * seg.x + (p.y - a.y) * seg.y) / len2, 0, 1);
    const q = { x: a.x + seg.x * t, y: a.y + seg.y * t };
    const d2 = (p.x - q.x) ** 2 + (p.y - q.y) ** 2;
    if (d2 < best.d2) best = { d2, idx: i, s: a.s + t * track.stepM, t };
  }

  if (Math.sqrt(best.d2) > track.halfWidthM + 10) return projectGlobal(track, p);
  return best;
}
```

**Critério de aceite:** percorrer 3 voltas → `s` monotônico com exatamente 1 wrap por volta e
`lap` incrementando 3 vezes.

---

## 9. Corrida

- **Setores:** 16 setores por `s`; a volta só fecha se todos forem cruzados na ordem.
- **Volta válida:** ficar com `dist > halfWidth + 0,5 m` por mais de 0,15 s → `lapValid = false`.
  Tocar a zebra (`halfWidth` até `+1,5 m`) reduz grip, mas não invalida.
- **Sentido contrário:** `dot(velocity, tangent) < 0` por > 1 s → aviso no HUD (sem reset automático).
- **Reset:** `R` reposiciona no ponto mais próximo da centerline, alinhado à tangente,
  velocidade zero, cooldown de 3 s. Auto-reset após 5 s parado fora da pista.
- **Cronômetro:** acumular `simTime` do passo fixo — nunca `Date.now()`.
- **Posições (com IA):** ordenar por `lap * lengthM + s`.
- **Largada:** contagem 3-2-1, grid de largada.
- **Fim:** N voltas → tela de resultado com tempos e melhor volta.

---

## 10. IA

Pré-computar **offline, por pista**:

1. **Linha de corrida:** partir da centerline e suavizar (Laplacian) puxando para o interior
   das curvas, limitada a `±(halfWidth - margemCarro)`.
2. **Perfil de velocidade:**

```ts
for (let i = 0; i < n; i++) vMax[i] = Math.sqrt(aLatMax / Math.max(Math.abs(curv[i]), 1e-4));
for (let pass = 0; pass < 2; pass++)
  for (let i = n - 1; i >= 0; i--)
    vMax[i] = Math.min(vMax[i], Math.sqrt(vMax[(i + 1) % n] ** 2 + 2 * aBrake * ds));
```

3. **Direção:** pure pursuit com lookahead `L = clamp(0.35 * v, 6, 25) m` sobre a linha.
4. **Aceleração:** P-controller em `vMax[i]` (acelera se `v < alvo`, freia com margem).
5. **Desvio:** carro a menos de 15 m à frente → offset lateral para o lado com mais espaço.
6. **Determinismo:** RNG com seed; IA roda no mesmo `step(DT)`.

**Critério de aceite:** IA completa 5 voltas sem sair da pista, tempo dentro de ~8% do melhor
tempo do jogador, sem tremida de direção.

---

## 11. Renderização

- Desenhar a pista **por segmentos visíveis**: AABB pré-calculado por segmento, filtro pela
  viewport da câmera e `ctx.clip()`. Nunca criar um canvas offscreen gigante (limite ~16k px).
- **DPR correto:** `canvas.width = cssW * dpr`, `ctx.setTransform(dpr, ...)`, DPR máximo 2.
- **Câmera:** segue com `lerp`, olha à frente na direção da velocidade, zoom levemente para
  fora com a velocidade, limitada aos limites da pista; tratamento especial para ré e 180°.
- **Camadas:** pista → zebras/bordas → sombras → carros → HUD.
- **Interpolação:** render entre estado anterior e atual; ângulo pelo caminho mais curto.

```ts
const lerpAngle = (a: number, b: number, t: number) => a + wrapPi(b - a) * t;
```

---

## 12. Loop e determinismo

```ts
const DT = 1 / 60;
const MAX_FRAME = 0.25;
const MAX_STEPS = 5;

let acc = 0, last = 0, prev: Snapshot, cur: Snapshot;

function frame(now: number) {
  const frameDt = Math.min((now - last) / 1000, MAX_FRAME);
  last = now;
  acc += frameDt;

  let steps = 0;
  while (acc >= DT && steps < MAX_STEPS) {
    prev = snapshot(world);
    step(world, DT);
    cur = snapshot(world);
    acc -= DT;
    steps++;
  }
  if (steps === MAX_STEPS) acc = 0;
  render(prev, cur, acc / DT);
}
```

- Input lido **uma vez por passo fixo** e guardado no `World`.
- Pausar em `blur` / `visibilitychange`.
- **Critério de aceite:** mesmo input a 30, 60 e 144 Hz de frame → mesmo estado final
  (tolerância 1e-6).

---

## 13. Input

- Teclado: `←`/`A`, `→`/`D`, `↑`/`W`, `↓`/`S`, `R` (reset), `Esc` (pausa).
- `preventDefault()` em setas/espaço (evitar rolagem da página).
- Gamepad: eixo X analógico, zona morta ~0,15 com histerese.
- Mapa de teclas como dados (facilita rebind e futuro 2P).

---

## 14. Progressão (estilo Gran Turismo)

- **Campeonato Grid Zero:** 4–5 etapas, briefing de uma frase por prova.
- **Escola de Pilotagem:** 3 desafios (curva, frenagem, volta limpa) como tutorial jogável.
- **Medalhas** ouro/prata/bronze por tempo em cada etapa.
- **Rival:** V. Ramos com skill fixa, presente em todas as etapas.
- **Times e carros fictícios** (sem licenciamento).
- **Recordes e ghost** salvos em `localStorage` com campo `version`.

```ts
export const CAREER = {
  team: "Grid Zero Racing",
  rival: { name: "V. Ramos", aiSkill: 0.92, color: "#e10600" },
  events: [
    { trackId: "oval",    laps: 3, brief: "Etapa 1 — aprenda a pista." },
    { trackId: "monza",   laps: 3, brief: "Etapa 2 — o rival acelera forte." },
    { trackId: "hairpin", laps: 5, brief: "Final — tudo ou nada." },
  ],
  epilogue: "Temporada encerrada. Seu melhor tempo ficou na história.",
} as const;
```

---

## 15. Qualidade

- **Testes de replay headless:** `Input[] = [{ tick, steer, throttle, brake }]` → rodar N ticks
  → comparar `lap`, `s`, `bestLap`, `pos`.
- Cenários: volta limpa, corte de pista, batida na borda, respawn.
- **Hash de estado** a cada 60 ticks → detecta divergência determinística.
- **Property test:** 10.000 ticks de input aleatório → sem `NaN`, `|pos|` finito, `s` em
  `[0, lengthM)`.
- **Teste de escala:** mesmo input em 30/60/144 Hz de frame → mesmo resultado.
- **CI:** `typecheck` + `test` a cada push.
- **Deploy:** GitHub Pages a partir da F1.
- **Overlay de debug (`F3`):** FPS, `s`, `lap`, setor, `vf`/`vl`, grip, índice do segmento,
  tempo fora da pista.

---

## 16. Roadmap

| Fase | Entrega | Critério de pronto |
|---|---|---|
| **F0** | Vite + TS strict, ESLint, Vitest, CI, deploy vazio, loop com passo fixo | `npm run dev` e `npm test` funcionando |
| **F0.5** | Constantes com unidades, overlay de debug, pista oval gerada por código | Carro anda e valores aparecem no overlay |
| **F1** | Carro + física de bicicleta + câmera + gamepad + deploy | Pilotagem com sensação boa |
| **F2** | Spline centrípeta, amostragem por arco, `project()` com janela, render por segmentos | Não atravessa borda; `s` estável em 3 voltas |
| **F3** | Setores, volta válida, track limits, reset, pausa, HUD, menu/resultado | Corrida completa de 3 voltas |
| **F4** | Linha de corrida + perfil de velocidade + IA + posições + colisão carro-carro | IA completa 5 voltas sem sair |
| **F5** | 3 pistas, seleção, campeonato, Escola de Pilotagem, medalhas, recordes versionados | Trocar conteúdo sem tocar no código do jogo |
| **F6** | Sprites, partículas, áudio (unlock/mute), ghost, polimento | "Parece um jogo" |

Cada fase termina **jogável**; o MVP fecha na F3.

---

## 17. Riscos e mitigação

| Risco | Mitigação |
|---|---|
| Física "escorregadia" ou travada | Constantes isoladas + overlay/sliders de tuning; metas numéricas |
| Colisão instável em alta velocidade | Passo fixo + subpassos quando `|v| * dt` for grande |
| Progresso `s` saltando | Busca em janela + fallback só em respawn |
| Pista difícil de modelar | Oval/retângulo arredondado gerado por código antes da spline |
| IA burra | Linha de corrida + perfil de velocidade antes de dificuldade |
| Escopo crescendo | Não-objetivos escritos; congelar MVP da F3 |
| Aba em background quebrando a física | Acumulador com `MAX_FRAME`/`MAX_STEPS` e descarte |
| Canvas grande estourando limite | Render por segmentos visíveis, nunca canvas único gigante |
| Melhor tempo via atalho | Setores + volta inválida |

---

## 18. Roteiro de apresentação (10–12 min)

1. **Capa** — GRID ZERO, slogan, autor, disciplina.
2. **O jogo em uma frase** — corrida top-down arcade + 3 pilares.
3. **Objetivo e referências** — Gran Turismo (progressão/UI), arcades (física). O que não copiamos.
4. **Escopo** — MVP, backlog, não-objetivos.
5. **Tecnologias** — TS + Vite + Canvas 2D + Vitest; por que não engine.
6. **Arquitetura** — `sim/` × `render/`, pista como dados.
7. **Física** — SI, modelo de bicicleta, grip, metas numéricas.
8. **Pista e corrida** — spline por arco, projeção em janela, setores, volta inválida.
9. **IA e progressão** — linha de corrida, perfil de velocidade, campeonato, medalhas.
10. **Qualidade e roadmap** — determinismo, replay tests, CI, deploy, fases, riscos, estado atual.

**Pitch de 30 segundos:** *"Grid Zero é um jogo de corrida 2D top-down com física arcade,
feito em TypeScript. A inspiração é a progressão do Gran Turismo com a jogabilidade imediata
dos arcades. O foco não é simular um carro, e sim fazer uma corrida divertida em cinco
segundos — sobre uma base determinística e testável: passo fixo, pista parametrizada por
comprimento de arco e validação de volta por setores."*

### Perguntas prováveis

| Pergunta | Resposta |
|---|---|
| Por que não Unity/Godot? | Núcleo pequeno e top-down; engine adicionaria peso e esconderia a física, que é o que se quer demonstrar. |
| Já tem código? | Planejamento fechado e repositório criado; implementação inicia na F0. |
| Qual a parte mais difícil? | Progresso na pista e determinismo — e as duas soluções estão especificadas. |
| Como sabe que está certo? | Critérios de aceite por fase + testes de replay + CI. |
| Qual a contribuição original? | Progressão estilo GT com arcade top-down, pista parametrizada e simulação determinística testável. |
| E se a física não ficar boa? | Metas numéricas + constantes isoladas e overlay de tuning. |
| Vai dar tempo? | Cada fase termina jogável; MVP fecha na F3. |

---

## 19. Referências

- Steve Swink — *Game Feel: A Game Designer's Guide to Virtual Sensation*
- Robert Nystrom — *Game Programming Patterns* (game loop, component)
- MDN — Canvas API e requestAnimationFrame
- Documentação do Vite e do Vitest
- *Gran Turismo* (Polyphony Digital) — referência de progressão, licenças e UI
- Catmull-Rom centrípeta — Barry & Goldman (1988)
