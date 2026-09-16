---
title: Agent-Based Modeling
tags:
  - simulation
  - modeling
  - complex-systems
date: 2026-09-16
---

Agent-based modeling (ABM) is a simulation approach in which a system is
represented as a collection of autonomous **agents** — individually
programmed entities that follow their own rules and interact with each
other and their environment. Rather than describing a system with a single
set of equations that govern it as a whole, ABM builds the system bottom-up:
each agent decides and acts locally, and the system's overall behavior
emerges from the sum of those interactions.

## Core ingredients

- **Agents** — discrete entities with an internal state (position, beliefs,
  resources, a strategy) and a set of behavioral rules that determine how
  they act and how they update that state.
- **Environment** — the space agents inhabit and interact through, which
  can be a physical grid, a network, or an abstract space.
- **Interaction rules** — how agents affect each other and the environment
  (e.g. competing for resources, exchanging information, moving to avoid
  crowding).
- **Time steps** — the simulation advances in discrete steps (or continuous
  time in some variants), with agents re-evaluating and acting at each step.

## Why use it

ABM is well suited to systems where the aggregate behavior is not easily
captured by a closed-form equation, but does emerge from many simple,
local decisions — traffic jams from individual driving behavior, market
prices from individual trading decisions, disease spread from individual
contact patterns, flocking from individual birds each following a few
simple rules. It trades analytical tractability for the ability to
represent heterogeneity (agents can differ from each other) and emergence
(patterns that aren't explicitly programmed into any single agent, like a
traffic jam appearing without any agent being "programmed" to form one).

## Contrast with other modeling approaches

- **Equation-based / system dynamics models** describe the system with
  aggregate variables and differential equations (e.g. "the infected
  population grows at rate X"). They're often more tractable and easier to
  analyze mathematically, but assume homogeneity and can miss emergent
  effects that come from individual variation.
- **Cellular automata** are a constrained special case: agents sit on a
  fixed grid and their rules depend only on neighboring cells. ABM
  generalizes this — agents can move, form networks, and have richer
  internal state.

## Common uses

- Epidemiology (modeling disease spread through a population of
  individuals with distinct contact patterns)
- Economics and finance (markets as populations of trading agents with
  different strategies)
- Traffic and pedestrian flow
- Ecology (predator-prey dynamics, foraging behavior)
- Social science (opinion dynamics, segregation models like Schelling's
  model)

## Tools

Common frameworks for building ABMs include NetLogo (approachable,
widely used in teaching), Mesa (Python), and Repast.
