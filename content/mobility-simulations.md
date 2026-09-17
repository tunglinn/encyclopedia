---
title: Mobility Simulations
tags: [transportation, simulation, agent-based-modeling]
date: 2026-09-16
---

Every time a city debates whether a new bus lane will cause gridlock, or a self-driving car company tests a merge maneuver a million times before trying it on real pavement, someone is running a mobility simulation. The idea sounds simple — recreate traffic in software — but doing it well means answering a surprisingly deep question: what is the smallest, cheapest model of a road, a driver, and a trip that still behaves like the real thing?

## Background

A mobility (or traffic) simulation is a program that recreates how people and vehicles move through a network of roads, and lets you fast-forward through scenarios that would be too expensive, too slow, or too dangerous to test for real — a new traffic light timing, a highway closure, a fleet of self-driving cars merging into human traffic.

You don't need a transportation background to follow this page, but it helps to know the field splits simulations into three levels of granularity:

- **Macroscopic** — treats traffic like a fluid. You track flow, density, and speed for a whole road segment as aggregate numbers, the way you'd model water through a pipe. Fast, coarse, good for city-wide "what if" questions.
- **Microscopic** — treats every vehicle (sometimes every pedestrian) as an individual agent with its own position, speed, and decisions. Slower, but this is where "does this specific intersection jam up" questions get answered.
- **Mesoscopic** — a compromise: individual vehicles, but simplified interactions, used when you need microscopic-ish detail across a macroscopic-ish area.

This page is mostly about microscopic and agent-based simulation, since that's where most of the interesting modeling — and most of the open-source tooling — lives.

## Intuition

Think of a microscopic traffic simulator as a video game engine, except instead of a game designer scripting NPC behavior, every "character" (car) follows a small set of physics-like rules:

<div style="border:1px solid #ccc; border-radius:8px; padding:16px; margin:16px 0; font-family:sans-serif;">
<strong>What one simulated vehicle actually needs, each timestep:</strong>
<table style="width:100%; border-collapse:collapse; margin-top:8px;">
<tr style="border-bottom:1px solid #ddd;"><td style="padding:6px; font-weight:bold;">Question</td><td style="padding:6px; font-weight:bold;">Model that answers it</td></tr>
<tr style="border-bottom:1px solid #eee;"><td style="padding:6px;">How fast should I go, given the car ahead of me?</td><td style="padding:6px;">Car-following model (e.g. IDM)</td></tr>
<tr style="border-bottom:1px solid #eee;"><td style="padding:6px;">Should I change lanes?</td><td style="padding:6px;">Lane-change model (gap acceptance)</td></tr>
<tr style="border-bottom:1px solid #eee;"><td style="padding:6px;">Which street do I take?</td><td style="padding:6px;">Route choice (shortest path, sometimes learned)</td></tr>
<tr><td style="padding:6px;">Why am I even driving right now?</td><td style="padding:6px;">Demand model (trip: home → work at 8am)</td></tr>
</table>
</div>

Run thousands of these little decision-makers on top of a digital road network, step the simulation forward in small time increments (often 0.1–1 second), and citywide traffic patterns — congestion, spillback, the wave of brake lights that appears for no reason on a highway — emerge on their own. Nobody scripts the traffic jam; it falls out of many simple local decisions, the same way a flock of birds "decides" to turn without a leader.

## The substance

### The four layers of a simulation

**1. The network.** A digital twin of the road system: lanes, intersections, traffic signal phases, speed limits, turn restrictions. Almost nobody draws this by hand anymore — it's converted from [OpenStreetMap](https://www.openstreetmap.org) data or a transportation agency's GIS files using a network-conversion tool (SUMO's `netconvert` is the standard one).

**2. The demand.** Who is traveling, from where, to where, and when. This can be as simple as a random origin-destination matrix, or as rich as a **synthetic population** — thousands of fake-but-statistically-realistic people, each with an age, household, job, and daily activity schedule (wake up → commute → work → gym → home), derived from census and travel-survey data. The more realistic the demand, the more the simulation can answer *behavioral* questions like "if we build this bike lane, how many people actually switch from driving."

**3. The behavior models.** The rules individual agents follow, calibrated against real-world data:

> [!note] Car-following: the Intelligent Driver Model (IDM)
> The most common car-following model computes acceleration as a function of your current speed, your desired speed, and the gap to the car ahead — tuned so vehicles neither collide nor drive unrealistically far apart. It's a handful of parameters (desired speed, reaction time, comfortable deceleration) fit against real trajectory datasets like **[NGSIM](https://catalog.data.gov/dataset/next-generation-simulation-ngsim-vehicle-trajectories-and-supporting-data)** or **[highD](https://levelxdata.com/highd-dataset/)**, which recorded actual highway traffic with high-precision cameras.

Lane-changing adds gap-acceptance logic (will a driver squeeze into that gap or wait), and route choice ranges from simple shortest-path to models that reroute agents in reaction to congestion they encounter, mimicking how a phone's GPS app reroutes you around traffic.

**4. The execution engine.** The loop that actually advances time, resolves conflicts (two cars can't occupy the same space), updates signals, and logs everything for later analysis. This is the "boring" software-engineering part, but it's where performance work happens — running a million agents at city scale requires the same kind of optimization as a real game engine.

### The tools people actually use

<div style="border:1px solid #ccc; border-radius:8px; padding:16px; margin:16px 0; font-family:sans-serif;">
<table style="width:100%; border-collapse:collapse;">
<tr style="border-bottom:1px solid #ddd;"><td style="padding:6px; font-weight:bold;">Tool</td><td style="padding:6px; font-weight:bold;">Type</td><td style="padding:6px; font-weight:bold;">Best for</td></tr>
<tr style="border-bottom:1px solid #eee;"><td style="padding:6px;"><strong><a href="https://eclipse.dev/sumo/">SUMO</a></strong></td><td style="padding:6px;">Open source, microscopic</td><td style="padding:6px;">Easiest entry point; scriptable via Python (TraCI); imports OSM directly</td></tr>
<tr style="border-bottom:1px solid #eee;"><td style="padding:6px;"><strong>[[MATSim]]</strong></td><td style="padding:6px;">Open source, agent-based</td><td style="padding:6px;">City/region-scale demand modeling with synthetic populations</td></tr>
<tr style="border-bottom:1px solid #eee;"><td style="padding:6px;"><strong><a href="https://github.com/LBNL-UCB-STI/beam">BEAM</a></strong></td><td style="padding:6px;">Open source (built on MATSim)</td><td style="padding:6px;">Adding ride-hail, EVs, multimodal choice on top of MATSim</td></tr>
<tr style="border-bottom:1px solid #eee;"><td style="padding:6px;"><strong><a href="https://www.aimsun.com/">Aimsun</a> / <a href="https://www.ptvgroup.com/en/products/ptv-vissim">PTV Vissim</a> / <a href="https://www.caliper.com/transmodeler/default.htm">TransModeler</a></strong></td><td style="padding:6px;">Commercial, GUI-driven</td><td style="padding:6px;">What most transportation agencies and consultants actually use day to day</td></tr>
<tr style="border-bottom:1px solid #eee;"><td style="padding:6px;"><strong><a href="https://www.anl.gov/taps/polaris-transportation-system-simulation-tool">POLARIS</a></strong></td><td style="padding:6px;">Open source (Argonne National Lab)</td><td style="padding:6px;">Large-scale connected/autonomous vehicle research</td></tr>
<tr><td style="padding:6px;"><strong><a href="https://carla.org/">CARLA</a>, <a href="https://flow-project.github.io/">Flow</a>, <a href="https://github.com/cityflow-project/CityFlow">CityFlow</a></strong></td><td style="padding:6px;">Open source, research-grade</td><td style="padding:6px;">Reinforcement learning for driving policy or signal control; CARLA adds full 3D/sensor realism for AV perception research</td></tr>
</table>
</div>

If you just want to see traffic simulated with minimal setup: [SUMO](https://eclipse.dev/sumo/), because `netconvert` + `randomTrips.py`/`duarouter` + `sumo-gui` gets you a running visual simulation of a real place with almost no code.

### How advanced is the field, really?

The honest answer is: very advanced at the mechanics, still catching up on the psychology.

- **Vehicle-level microsimulation is operationally trusted.** Transportation agencies use tools like VISSIM and SUMO to make real infrastructure decisions — signal retiming, lane reconfigurations — because car-following and lane-changing models are well validated against real trajectory data.
- **Activity-based demand modeling is state of the art but data-hungry.** MATSim-style synthetic populations can capture second-order effects like *induced demand* (new road capacity attracts new trips, so congestion doesn't drop as much as naively expected) — but building an accurate synthetic population for a city takes serious data engineering, and calibration against ground truth (real GPS/cellphone traces) is the actual bottleneck, not the simulation engine.
- **Mixed human/AV traffic and rare behaviors remain weak.** Aggressive driving, jaywalking, a driver waving another car through — these are edge cases that most calibrated models still approximate crudely. This is an active research area, often bridged with machine learning: reinforcement learning for adaptive signal control, or driver-behavior models learned directly from trajectory datasets rather than hand-tuned equations.

In short: if your question is "will this intersection back up," simulation tooling can answer it with confidence today. If your question is "how will 10,000 individual humans actually change their behavior," the tools give you a plausible, calibratable answer — not a guaranteed one.

### Is the field shifting toward agent-based modeling?

Yes, and it's not subtle. Three forces are pushing transportation modeling away from aggregate flow equations and even from simple microscopic simulation, toward full agent-based modeling (ABM):

1. **The questions got harder.** Planners used to ask "how much traffic will this road carry." Now they ask "how many people will actually switch to transit if we add congestion pricing," "who benefits and who's burdened by this policy," and "how does a fleet of autonomous vehicles change parking demand." Those are individual-choice questions, and aggregate models can't represent an individual choosing.
2. **The data caught up.** Building a believable synthetic population used to require expensive travel diaries alone. Now cellphone mobility traces, smart-card transit data, and large household travel surveys make it feasible to construct and calibrate millions of realistic synthetic agents.
3. **The compute caught up.** Simulating millions of individual agents with daily schedules was impractical outside a research cluster fifteen years ago. It's now routine — several major US metropolitan planning organizations (San Diego, Puget Sound, others) have already replaced their legacy aggregate ("four-step") travel demand models with activity-based/agent-based ones like ActivitySim, and the same shift is happening in AV, ride-hail, and micromobility research, where the whole point is modeling how individuals decide between modes.

### What agent-based modeling looks like for transportation

[[Agent-Based Modeling|Agent-based modeling]] in general means giving each individual agent its own state and decision rules and letting system-level behavior emerge from their interactions, rather than writing equations for the system as a whole — the same "no one's in charge, but the flock turns together" idea from the Intuition section above, just one instance of a much more general modeling approach used everywhere from epidemiology to economics.

Applied to travel specifically, a transportation ABM agent typically carries: a home and work location, a daily activity schedule (sleep, commute, work, errands, leisure), a set of available modes (car, transit, bike, walk, ride-hail), and preferences (value of time, price sensitivity) — then the simulation lets every agent independently decide, each day, whether to keep its routine or adjust it in response to what changed (a new toll, a canceled bus route, worse-than-usual congestion).

<div style="border:1px solid #ccc; border-radius:8px; padding:16px; margin:16px 0; font-family:sans-serif;">
<table style="width:100%; border-collapse:collapse;">
<tr><td style="padding:6px; vertical-align:top; width:50%;">
<strong>Advantages</strong>
<ul style="margin-top:6px;">
<li>Captures real heterogeneity — a retiree and a two-job parent respond to a toll differently, and the model doesn't need to be told that in advance, it falls out of their different schedules and budgets</li>
<li>Emergent effects (induced demand, mode-shift cascades, second-order congestion) appear on their own instead of needing to be hand-coded as a rule</li>
<li>New behaviors bolt on modularly — add a ride-hail agent type or a new mode without rewriting the whole model</li>
<li>Naturally supports multimodal, whole-day travel patterns (an agent's morning mode choice can depend on what it needs to do that evening)</li>
<li>Results map intuitively onto real people, which makes them easier to explain to policymakers ("12% of low-income households changed their commute") than an aggregate flow number</li>
</ul>
</td><td style="padding:6px; vertical-align:top; width:50%;">
<strong>Disadvantages</strong>
<ul style="margin-top:6px;">
<li>Computationally expensive — millions of agents with individual daily schedules cost far more than solving a handful of flow equations</li>
<li>Hard to calibrate: many free parameters, and different parameter combinations can produce the same aggregate output (<em>equifinality</em>), making it easy to fit the past without actually getting the mechanism right</li>
<li>Harder to validate — there's no individual-level ground truth for most agents, only aggregate counts (traffic sensors, transit ridership) to check the output against indirectly</li>
<li>Stochastic: results vary run to run, so a credible answer needs many Monte Carlo runs, multiplying the already-high compute cost</li>
<li>The behavior rules are still modeler assumptions — an ABM can look far more "realistic" than it actually is, since granular output doesn't guarantee granular accuracy</li>
</ul>
</td></tr>
</table>
</div>

### Thought exercise: Taipei's two-stage moped left turns

> [!note] The behavior
> In Taipei (and much of Taiwan), mopeds turning left at a signalized intersection generally don't cut across oncoming traffic the way a car does. Instead, at the start of the green phase they ride straight through the intersection into a waiting box painted on the far corner, then turn to face the cross street and wait for *that* street's green phase before proceeding. It's a "two-stage" or "hook" turn — trading one complex maneuver (crossing oncoming traffic) for two simple ones (go straight, then go straight again from a different orientation), at the cost of an extra signal cycle.

**Is this already in existing tools?** Not out of the box, but the pieces already exist. This isn't exotic — it's the same underlying mechanism as the "advanced stop line" hook turns used for cyclists in Copenhagen and the tram hook turns in Melbourne, both of which have precedent in the traffic-engineering literature and have been modeled in tools like SUMO. But no mainstream simulator ships a "two-stage moped turn" template, and `netconvert` won't infer one automatically from OpenStreetMap data, because OSM doesn't usually tag this kind of staged-movement restriction — you'd have to hand-author it.

**How hard is it to implement in a microscopic simulator like SUMO?** Moderate — a data-authoring problem, not a code problem:
- **Geometry**: add the physical waiting box as an actual short lane/link at the far corner of the intersection, not just a routing rule. SUMO's network format supports arbitrary lane geometry, so this is drawing, not programming.
- **Vehicle-class restrictions**: SUMO already supports per-lane, per-vehicle-class turn restrictions (`vClass="moped"`), so you can forbid mopeds from taking the direct left-turn connection and force them onto the two-stage path instead.
- **Signal phasing**: the box's release needs to be tied to the *cross* street's green phase, not the phase the moped arrived on — this means hand-writing a traffic-light-logic (`tlLogic`) definition with an extra phase, since default `netconvert`-generated intersections won't have one.

None of this requires touching SUMO's source code — it's all achievable through network XML and traffic-light XML authoring. It's tedious rather than hard, and it's the kind of thing you'd only bother with if you specifically cared about moped behavior at a specific intersection (which is exactly why it isn't a default template — it's a niche need, not a generalizable one).

**What about in an agent-based model?** Here the physical/geometric part is just as easy — same box, same lane — but the *decision logic* is a bigger lift, for a structural reason: ABM platforms like MATSim don't usually reason about intersection-level maneuvers at all. Their default mobility simulation treats a road network as a graph of links with queue dynamics, and turning movements are largely implicit in which link an agent's route uses next — there's no built-in concept of "wait in a box, then re-orient." Getting a MATSim-style agent to actually perform a two-stage turn means either:
- Representing the waiting box as its own short link in the network (same trick as SUMO) and just letting the agent's route pass through it — this part is easy and needs no code, or
- If you want the agent's *choice* of turn style to be a genuine decision, rather than a hardcoded route, you'd need to extend the platform's mobility simulation module with custom logic — non-trivial software engineering, since it means reaching into how the simulator resolves movement at a link, not just editing a config file.

So the honest ranking: representing the *box itself* is easy in both SUMO and ABM tools, because it's just network geometry. Representing the *forced two-stage rule* is a config/data-authoring exercise in SUMO, but requires custom development work in a typical ABM platform, because ABM tools are usually built around trip-level and activity-level decisions, not the intersection-level choreography that this maneuver actually lives in. That mismatch — ABM being strong at *why* someone travels and weak at the fine-grained *how* of a single maneuver — is exactly why real-world studies often pair a mesoscopic/macroscopic ABM for city-wide demand with a microscopic simulator like SUMO or VISSIM for the specific intersection where a maneuver like this actually matters.

### Building your own

This is genuinely a weekend project at the simplest level, and a multi-week one if you want city-scale behavioral realism.

**Fastest path (an afternoon):**
1. Install SUMO.
2. Pull a real street network from OpenStreetMap for your area of interest and convert it with `netconvert`.
3. Generate demand with `randomTrips.py` (or `duarouter` for routed trips).
4. Run `sumo-gui` and watch a visual simulation of real traffic in that spot.

**Adding your own logic (a weekend):** Use SUMO's TraCI Python API to hook into the simulation frame by frame — inject your own vehicles, control a traffic signal with custom logic, or log data for analysis. This is also the on-ramp to RL-based traffic control research (pair it with `Flow` or `CityFlow` if you want a ready-made Gym-style RL environment).

**City-scale behavioral realism (weeks, real data engineering):** [[MATSim]] if you want synthetic populations making daily activity choices instead of pre-scripted trips — expect to spend real time on config XML and demand data preparation before you get an interesting result.

## Quiz

<div id="quiz-mobility-sim" style="font-family:sans-serif; max-width:700px;">

<div class="quiz-q" style="margin-bottom:20px; padding:12px; border:1px solid #ddd; border-radius:8px;">
<p><strong>1. What's the key difference between macroscopic and microscopic traffic simulation?</strong></p>
<button onclick="qFeedback(this,false,'Not quite — that\'s backwards. Macroscopic treats traffic as an aggregate flow (like fluid dynamics); microscopic tracks individual vehicles.')">Macroscopic tracks individual vehicles; microscopic treats traffic as aggregate flow</button><br>
<button onclick="qFeedback(this,true,'Right. Macroscopic models traffic as a continuous flow (density, speed, volume); microscopic gives every vehicle its own position and decisions.')">Macroscopic treats traffic as aggregate flow; microscopic tracks individual vehicles</button><br>
<button onclick="qFeedback(this,false,'Both can model highways and city streets — granularity, not road type, is the distinguishing factor.')">Macroscopic is for highways; microscopic is for city streets only</button><br>
</div>

<div class="quiz-q" style="margin-bottom:20px; padding:12px; border:1px solid #ddd; border-radius:8px;">
<p><strong>2. What does a car-following model like IDM actually compute?</strong></p>
<button onclick="qFeedback(this,false,'That\'s route choice, a separate model. Car-following governs speed relative to the vehicle ahead.')">Which road the vehicle should take to its destination</button><br>
<button onclick="qFeedback(this,true,'Correct — IDM computes acceleration from your speed, desired speed, and the gap to the car in front, so vehicles neither collide nor space unrealistically far apart.')">The vehicle's acceleration based on its speed and the gap to the car ahead</button><br>
<button onclick="qFeedback(this,false,'That\'s lane-changing / gap-acceptance logic, not car-following.')">Whether the vehicle should change lanes</button><br>
</div>

<div class="quiz-q" style="margin-bottom:20px; padding:12px; border:1px solid #ddd; border-radius:8px;">
<p><strong>3. What real-world data is commonly used to calibrate car-following models?</strong></p>
<button onclick="qFeedback(this,true,'Yes — NGSIM and highD are high-precision recorded vehicle trajectory datasets used to fit and validate car-following parameters.')">Recorded vehicle trajectory datasets like NGSIM or highD</button><br>
<button onclick="qFeedback(this,false,'Census data feeds demand modeling (who\'s traveling), not the physics of car-following.')">National census travel surveys</button><br>
<button onclick="qFeedback(this,false,'Signal timing plans describe intersections, not how individual cars accelerate and brake.')">Traffic signal timing plans published by cities</button><br>
</div>

<div class="quiz-q" style="margin-bottom:20px; padding:12px; border:1px solid #ddd; border-radius:8px;">
<p><strong>4. Why is MATSim's agent-based approach able to capture "induced demand" (new road capacity attracting new trips) in a way a simple flow model can't?</strong></p>
<button onclick="qFeedback(this,false,'MATSim is actually slower than flow models — its value isn\'t speed, it\'s behavioral fidelity.')">Because it runs faster, so more scenarios can be tested</button><br>
<button onclick="qFeedback(this,true,'Exactly — because each synthetic agent has its own schedule and can decide to make a new or different trip in response to changed conditions, effects like induced demand emerge naturally instead of needing to be hand-coded.')">Because individual synthetic agents can change their own trip decisions in response to conditions, so new behavior emerges rather than being hardcoded</button><br>
<button onclick="qFeedback(this,false,'MATSim doesn\'t use less network data than flow models — if anything it typically needs more (a full synthetic population).')">Because it needs less network data than flow models</button><br>
</div>

<div class="quiz-q" style="margin-bottom:20px; padding:12px; border:1px solid #ddd; border-radius:8px;">
<p><strong>5. Which is the honest characterization of how "advanced" mobility simulation is today?</strong></p>
<button onclick="qFeedback(this,false,'Vehicle-level microsimulation is actually trusted enough for real agency decisions today — the field isn\'t purely academic.')">Purely a research curiosity — no real agency relies on it for decisions</button><br>
<button onclick="qFeedback(this,true,'Right — the mechanics (car-following, lane-changing, signal effects) are validated and operationally trusted, while capturing realistic human behavior and psychology at scale is still the harder, less-solved half.')">Vehicle-level mechanics are well validated and operationally trusted; capturing realistic large-scale human behavior is still the harder, less-solved problem</button><br>
<button onclick="qFeedback(this,false,'The opposite is closer to true — the physics/mechanics side is the mature part, and behavioral realism is what still lags.')">Human behavior modeling is fully solved; the remaining challenge is just faster computers</button><br>
</div>

</div>

<script>
function qFeedback(btn, correct, explanation) {
  const container = btn.closest('.quiz-q');
  const buttons = container.querySelectorAll('button');
  buttons.forEach(b => b.disabled = true);
  btn.style.background = correct ? '#d4edda' : '#f8d7da';
  btn.style.borderColor = correct ? '#28a745' : '#dc3545';
  const fb = document.createElement('p');
  fb.style.marginTop = '8px';
  fb.style.fontStyle = 'italic';
  fb.textContent = (correct ? '✓ ' : '✗ ') + explanation;
  container.appendChild(fb);
}
</script>
