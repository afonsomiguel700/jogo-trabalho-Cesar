# GRID ZERO

> **Comece do zero. Termine na frente.**

Jogo de corrida **2D top-down** com física arcade, feito em **TypeScript + Canvas 2D**.
A inspiração é a camada de progressão do *Gran Turismo* (campeonato, provas de licença,
medalhas) somada à jogabilidade imediata dos arcades de corrida.

O foco não é simular um carro — é fazer uma corrida divertida em cinco segundos, sobre uma
base técnica determinística e testável.

---

## Status

**Planejamento concluído.** A implementação começa na fase F0 (ver roadmap no
[`PLAN.md`](./PLAN.md)).

## Pilares

1. Controle responsivo em 5 segundos de jogo.
2. Curvas recompensam trajetória limpa, não só acelerar.
3. Base pequena, determinística e testável — fácil de evoluir.

## Stack

| Item | Escolha |
|---|---|
| Linguagem | TypeScript (strict) |
| Build/dev | Vite |
| Render | Canvas 2D nativo |
| Loop | `requestAnimationFrame` + passo fixo de 1/60 s |
| Testes | Vitest (simulação headless + replay) |
| Qualidade | ESLint + Prettier + GitHub Actions |

## Roadmap (resumo)

| Fase | Entrega |
|---|---|
| F0 | Base do projeto, loop, CI, deploy |
| F1 | Carro, física, câmera, gamepad |
| F2 | Pista, spline, colisão, render por segmentos |
| F3 | Voltas, setores, HUD, menus, volta válida |
| F4 | IA adversária, posições, colisão carro-carro |
| F5 | 3 pistas, campeonato, Escola de Pilotagem, medalhas |
| F6 | Sprites, áudio, ghost, polimento |

## Documentação

- [`PLAN.md`](./PLAN.md) — planejamento completo: arquitetura, física, pista, corrida, IA,
  qualidade, roadmap com critérios de aceite, riscos e roteiro de apresentação.

## Como rodar

> Ainda não implementado (F0). Quando existir:

```bash
npm install
npm run dev     # servidor de desenvolvimento
npm test        # testes da simulação
```
