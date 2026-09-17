---
title: MATSim
tags: [transportation, simulation, agent-based-modeling, java]
date: 2026-09-17
---

Open a MATSim scenario for the first time and you'll find no button that says "run traffic simulation." Instead you find a config file, a network file, a population of a million synthetic people each carrying a full daily schedule, and a note that says: run this for a few hundred rounds and check whether it settled down. That's not an accident of bad UX — it's the whole idea. MATSim doesn't compute traffic, it *evolves* it, letting a population of simulated commuters learn their way to a stable pattern the same way real commuters gradually find their own routine.

## Background

If you haven't already, [[Mobility Simulations]] is worth reading first — it covers the general landscape (macroscopic vs. microscopic vs. agent-based simulation) that MATSim is one entry in, and [[Agent-Based Modeling]] covers the general "many simple agents, emergent system behavior" idea MATSim applies specifically to travel.

MATSim (**M**ulti-**A**gent **T**ransport **Sim**ulation) is a Java framework for simulating an entire city or region's daily travel demand using a synthetic population — thousands to millions of fake-but-statistically-realistic people, each with their own daily activity chain (home → work → gym → home). It's not one model but a *framework* of interacting models — network loading, scoring, route/mode/time choice — orchestrated by an iterative loop.

## Intuition

Think of MATSim less like a physics engine and more like a slow-motion evolutionary algorithm played out over commuters' daily lives:

<div style="border:1px solid #ccc; border-radius:8px; padding:16px; margin:16px 0; font-family:sans-serif;">
<strong>One MATSim iteration, for the whole population at once</strong>
<ol style="margin-top:8px; padding-left:20px;">
<li style="margin-bottom:8px;"><strong>Execute.</strong> Every agent's current plan (leave home at 7:45, drive to work, leave work at 5:30, ...) runs simultaneously through the network for one simulated day.</li>
<li style="margin-bottom:8px;"><strong>Score.</strong> Each agent gets a utility score for how that day went — time spent traveling is bad, being stuck in traffic is worse, arriving late to work is worse still, spending quality time at an enjoyable activity is good.</li>
<li style="margin-bottom:8px;"><strong>Replan.</strong> A fraction of agents (not all — this matters, see below) mutate their plan: a different route, a different departure time, a different mode, sometimes a different destination.</li>
</ol>
Repeat a few hundred times, and agents gradually discard bad plans and lean on good ones — not because anyone computed an optimal traffic pattern, but because millions of small, self-interested adjustments settled into one.
</div>

This is why the field calls it "day-to-day" or "co-evolutionary" dynamics rather than an optimization: nobody is solving for the equilibrium directly, the population is *learning* its way toward one, the same emergent-from-many-simple-agents idea as flocking birds in [[Agent-Based Modeling]] — except here the "flock" is a city's worth of commuters, and what emerges is a stable travel pattern rather than a shape in the sky.

## Where MATSim came from

MATSim's lineage traces to **TRANSIMS** (TRansportation ANalysis and SIMulation System), a cellular-automaton microsimulator Kai Nagel worked on at Los Alamos National Laboratory in the 1990s. TRANSIMS already had the core idea — model individual synthetic travelers with daily activity chains, rather than aggregate flow — but it was a closed federal-lab tool chain: standalone C/Fortran programs, not built as an open platform for outside researchers to extend.

When Nagel moved to ETH Zurich around 2004, he set out to reimplement and open-source that approach for a distributed academic community rather than one lab. Kay Axhausen joined shortly after and brought the piece that makes MATSim more than a TRANSIMS clone: the activity-based, utility-scoring perspective — treating a day's plan as something an agent can be scored on and learn to improve, rather than a fixed input. The project coalesced as an open, Java-based framework around 2006.

MATSim was also a reaction against two other mainstream approaches of the time:

- **Static, aggregate four-step travel demand models** (trip generation → distribution → mode choice → assignment, run in tools like EMME, TransCAD, VISUM) — these model flows between coarse zones, not individuals, so they structurally can't represent one person shifting their departure time to dodge congestion, or a single household's actual daily activity chain.
- **Microscopic traffic simulators** (early VISSIM, Aimsun, CORSIM) — excellent at lane-level, car-following detail, but they take demand as a *given* origin-destination matrix. They answer "how does this traffic move" without touching "why is this person traveling right now at all."

MATSim's niche was combining TRANSIMS-style disaggregate realism with an explicit day-to-day learning mechanism for demand itself, at regional scale — a gap neither aggregate planning tools nor microscopic simulators filled.

> [!note] Why Java, specifically?
> Not an arbitrary choice for its era. Nagel's TRANSIMS background was C/Fortran, but MATSim needed something different: a distributed, multi-university open-source project needs a language most incoming contributors (grad students rotating through labs over decades) already know — Java was the dominant CS/engineering teaching language of the 2000s. It also gave platform independence across labs' mixed Windows/Linux/Mac machines, garbage collection so transport researchers (not systems programmers) could extend the codebase without chasing memory bugs, and — most importantly for a framework meant to grow indefinitely — mature OOP and (later) dependency-injection tooling that fits a pluggable, modular architecture. Contemporaneous agent-based modeling toolkits like Repast and MASON made the same call for the same reasons; MATSim wasn't an outlier.

**Has anyone ported it to Python?** No, and the maintainers have addressed this directly: on [a GitHub issue asking exactly this](https://github.com/matsim-org/matsim-code-examples/issues/34), founder Kai Nagel and core maintainer Thibaut Dubernet said it's come up repeatedly, but the team has no capacity or need for it themselves — they offered to sponsor a sub-project under the `matsim-org` GitHub organization if an outside contributor wanted to build and maintain one, and nobody has. The practical workaround they point to is **JPype**, which exposes the JVM classpath to Python almost transparently — you can drive `Controler` and read results from one Python script while the actual simulation still executes on the JVM underneath. `matsim-python-tools`, the official Python package, is I/O and analysis tooling (reading MATSim's XML output into pandas), not a reimplementation of the engine.

## Where MATSim sits in a workflow

MATSim itself is almost never the whole pipeline — it's the expensive middle stage, bracketed by tools that prepare its inputs and tools that make sense of its outputs.

<div style="border:1px solid #ccc; border-radius:8px; padding:16px; margin:16px 0; font-family:sans-serif;">
<strong>A typical MATSim study, end to end</strong>
<table style="width:100%; border-collapse:collapse; margin-top:8px;">
<tr style="border-bottom:1px solid #ddd;"><td style="padding:6px; font-weight:bold;">Stage</td><td style="padding:6px; font-weight:bold;">Typical tool</td></tr>
<tr style="border-bottom:1px solid #eee;"><td style="padding:6px;">Synthetic population generation</td><td style="padding:6px;">eqasim, PopulationSim, or a custom script against census/travel-survey data — usually <em>not</em> MATSim itself</td></tr>
<tr style="border-bottom:1px solid #eee;"><td style="padding:6px;">Network import</td><td style="padding:6px;">OpenStreetMap converted via <code>osm4matsim</code>/<code>matsim-osm</code>, or reuse of an existing regional network (Berlin, Zurich, Sydney are commonly reused open scenarios)</td></tr>
<tr style="border-bottom:1px solid #eee;"><td style="padding:6px;"><strong>MATSim core run</strong></td><td style="padding:6px;">the iterative execute/score/replan loop — the slow, heavy part</td></tr>
<tr style="border-bottom:1px solid #eee;"><td style="padding:6px;">Extensions for anything beyond car + PT</td><td style="padding:6px;">BEAM (ride-hail, EVs, multimodal choice), MATSim-DRT, signal-control contribs</td></tr>
<tr><td style="padding:6px;">Analysis / visualization</td><td style="padding:6px;">matsim-python-tools / matsim-r, Via, SimWrapper (see below)</td></tr>
</table>
</div>

As a **user**, you mostly live in config XML and input file assembly, rarely touching Java — many real projects fork an existing open scenario (open Berlin, open Switzerland) rather than building a region from scratch. As a **developer**, you extend MATSim through its module/plugin system (Guice dependency injection): custom scoring functions, new modes, new replanning strategies. Nearly all real capability growth (DRT/ride-hail, transit assignment, emissions, freight) ships as an optional **contrib** module rather than changes to the core — this is also how the project scales across ~30 active contributors from different universities without the core destabilizing.

## Calling it: inputs, invocation, outputs

There's no CLI with flags — MATSim is invoked as a Java program, either a prebuilt jar driven entirely by a config file, or your own `main()` when you need custom logic:

```bash
java -Xmx16g -cp matsim-<version>.jar org.matsim.run.RunMatsim config.xml
```

```java
Config config = ConfigUtils.loadConfig("config.xml");
Scenario scenario = ScenarioUtils.loadScenario(config);
Controler controler = new Controler(scenario);
controler.addOverridingModule(new MyCustomModule());  // where custom scoring/strategies plug in
controler.run();
```

**Inputs**, all referenced by path from `config.xml`:

<div style="border:1px solid #ccc; border-radius:8px; padding:16px; margin:16px 0; font-family:sans-serif;">
<table style="width:100%; border-collapse:collapse;">
<tr style="border-bottom:1px solid #ddd;"><td style="padding:6px; font-weight:bold;">File</td><td style="padding:6px; font-weight:bold;">Contents</td></tr>
<tr style="border-bottom:1px solid #eee;"><td style="padding:6px;"><code>config.xml</code></td><td style="padding:6px;">Master file — paths to everything below, iteration count, scoring coefficients, replanning strategy weights, qsim (mobility sim) settings</td></tr>
<tr style="border-bottom:1px solid #eee;"><td style="padding:6px;"><code>network.xml(.gz)</code></td><td style="padding:6px;">Nodes + links: capacity, freespeed, lanes, allowed modes — usually converted from OpenStreetMap</td></tr>
<tr style="border-bottom:1px solid #eee;"><td style="padding:6px;"><code>plans.xml(.gz)</code></td><td style="padding:6px;">The synthetic population — one <code>&lt;person&gt;</code> per agent with demographic attributes and an initial daily activity chain</td></tr>
<tr style="border-bottom:1px solid #eee;"><td style="padding:6px;"><code>facilities.xml</code> (optional)</td><td style="padding:6px;">Activity locations (shops, schools) agents can switch between during replanning</td></tr>
<tr style="border-bottom:1px solid #eee;"><td style="padding:6px;"><code>vehicles.xml</code> (optional)</td><td style="padding:6px;">Vehicle type definitions — capacity, emissions class — for freight or fleet modeling</td></tr>
<tr><td style="padding:6px;"><code>transitSchedule.xml</code> + <code>transitVehicles.xml</code> (optional)</td><td style="padding:6px;">Transit lines, stops, timetables, for multimodal runs</td></tr>
</table>
</div>

**Outputs**, written to the configured output directory:

<div style="border:1px solid #ccc; border-radius:8px; padding:16px; margin:16px 0; font-family:sans-serif;">
<table style="width:100%; border-collapse:collapse;">
<tr style="border-bottom:1px solid #ddd;"><td style="padding:6px; font-weight:bold;">File</td><td style="padding:6px; font-weight:bold;">What it's for</td></tr>
<tr style="border-bottom:1px solid #eee;"><td style="padding:6px;"><code>output_events.xml.gz</code></td><td style="padding:6px;">The real payload — every discrete event in the <em>last</em> iteration (link enter/leave, activity start/end, PT boarding). Nearly all downstream analysis is computed from this file.</td></tr>
<tr style="border-bottom:1px solid #eee;"><td style="padding:6px;"><code>output_plans.xml.gz</code></td><td style="padding:6px;">Each agent's final selected plan plus retained alternatives and scores — also the standard file for warm-starting a follow-up run</td></tr>
<tr style="border-bottom:1px solid #eee;"><td style="padding:6px;"><code>output_network.xml.gz</code>, <code>output_config.xml</code></td><td style="padding:6px;">Resolved inputs, for reproducibility</td></tr>
<tr style="border-bottom:1px solid #eee;"><td style="padding:6px;"><code>scorestats.txt</code>, <code>modestats.txt</code>, <code>traveldistancestats.txt</code></td><td style="padding:6px;">One row per iteration — the convergence diagnostics (see below)</td></tr>
<tr><td style="padding:6px;"><code>ITERS/it.N/</code></td><td style="padding:6px;">Periodic full snapshots (events, plans, leg-time histograms) for a subset of iterations, for debugging</td></tr>
</table>
</div>

### What actually happens when you run it

Startup takes seconds to a minute: MATSim parses the config and loads network/population into memory, logging one line per step. Two common early failures: the output directory already exists and isn't empty (MATSim refuses to overwrite by default), or the heap (`-Xmx`) is too small for the population size, producing an `OutOfMemoryError`.

Then the iteration loop dominates the run. Each iteration logs the mobsim advancing through simulated time-of-day (`07:15 → 23:00`), followed by scoring and replanning, and appends one row to `scorestats.txt`/`modestats.txt` — this is what you'd tail to watch convergence live. Every N iterations, a full snapshot lands in `ITERS/`.

Runtime varies by orders of magnitude with population size, not much else:

| Scale | Population | Typical wall time (single machine) |
|---|---|---|
| Toy/tutorial (Sioux Falls, Equil) | ~1k–15k agents | seconds to a few minutes for 100+ iterations |
| Mid-size city | ~100k–1M agents | tens of minutes to a few hours |
| Regional/metro (Berlin, Zurich, Sydney) | multi-million trips | many hours to 1–2 days |

The mobsim and replanning steps parallelize across cores, but MATSim has no built-in multi-node/cluster distribution — a run is bounded by one machine's cores and RAM, which is why regional scenarios run on a beefy server rather than a laptop.

### Why the whole day, not just rush hour

`config.xml`'s `qsim` module sets the simulated window (`startTime`/`endTime`, often pushed past `24:00:00` to `30:00:00` so late-night agents finish their trip rather than being cut off mid-route). But this isn't a "rush hour vs. full day" dial in the way you might expect — MATSim almost always simulates the **entire day**, because its core mechanism is *time-of-day choice*: an agent can only learn to leave earlier to dodge congestion if the simulation actually shows it the consequence of leaving at different times across the full day. Truncating the mobsim to 7–9am would remove the very trade-off the replanning loop is meant to explore.

"Rush hour" analysis, in practice, is a **filter applied after the fact** — you run the full day, then slice `output_events.xml.gz` by timestamp when computing congestion metrics for that window. The simulation window and your analysis window are different things.

## Convergence: no guarantee, and a real failure mode

This is the part that surprises people coming from classical traffic assignment algorithms (like Frank-Wolfe for static user equilibrium), which come with formal convergence proofs. MATSim's co-evolutionary loop has no such guarantee.

> [!warning] "Converged" doesn't mean "stopped changing"
> Because agents choose probabilistically among their stored plans (a logit-style random-utility model), MATSim doesn't settle to a literal fixed point — it relaxes to a **stochastic quasi-equilibrium**, where the population's average score fluctuates within a small band forever. That fluctuation is expected, not a sign something's wrong.

The standard config schedule disables plan *innovation* (new-route search, mode switching, time mutation) for roughly the last 10–20% of iterations, leaving only re-selection among already-discovered plans. **Convergence should be judged on that no-innovation tail specifically** — a flat score curve while innovation is still active can just mean agents haven't found a better plan *yet*, not that none exists.

Reasons a run might genuinely fail to settle, even given more iterations:

- **Oversaturated network** — demand exceeds capacity, so queue spillback shifts congestion from link to link faster than the population can adapt to any one state.
- **Symmetric or tied alternatives** — two routes or departure times that are genuinely equally good can cause agents to oscillate between them indefinitely (a Braess-paradox-flavored failure).
- **Mistuned replanning parameters** — too high a mutation rate, or innovation left enabled too long, keeps injecting competing plans faster than the population settles.
- **Scoring pathologies** — a misconfigured utility function (an activity that's essentially free to prolong indefinitely) can push agents toward an unrealistic extreme instead of a stable pattern.
- **Structural data problems** — disconnected network components, broken references, unrealistic teleported-mode travel times — these often produce a stable-looking but *garbage* equilibrium, which is arguably worse since the stats look fine.
- **Population sample size vs. noise** — a small synthetic population on a large network can make noise dominate any real signal in `scorestats.txt`.

## How many iterations, and warm-starting

There's no formula — this is an empirical, iterative process in its own right, not something you compute up front:

- **Start with a round-number guess**: ~100 for a small/toy scenario, 200–500+ for regional scale with mode/time/destination choice all active (more choice dimensions to explore means more iterations to shake them out).
- **The no-innovation tail needs to be long enough to demonstrate flatness on its own** — if innovation is disabled at 90% of a 100-iteration run, that's only 10 tail iterations, usually too few to trust; people generally want 30–50+ flat tail iterations, which pushes the total count up.
- **Inspect `scorestats.txt`/`modestats.txt`** after the run and decide: flat enough, or extend.

Because a run can always be resumed, "how many iterations" doesn't have to be answered correctly in advance — you can warm-start a continuation using the previous run's `output_plans.xml.gz` as the next run's `plans` input, rather than starting over. Two things matter when doing this:

1. **Don't just re-run the same config from scratch.** If innovation is re-enabled from iteration 0 on an already-good population, you re-inject a fresh round of exploratory mutation, which can temporarily *de-converge* scores before resettling — often not what you want if you're just trying to finish a run that was close.
2. Many workflows instead keep innovation disabled on the continuation (pure re-selection, to confirm stability) or use the `firstIteration`/`lastIteration` config parameters so the mutation-decay schedule continues from where it left off rather than resetting.

This chunked, check-then-extend approach is also just the economically sane way to run something that costs hours to days per attempt: better to run a chunk, inspect, and warm-start another chunk than to guess wrong and either overshoot (wasted compute) or undershoot and have to discard the whole run.

## Analysis and visualization

None of MATSim's raw output is tabular — it's all (gzipped) XML — so every analysis workflow starts by converting it into something more usable:

<div style="border:1px solid #ccc; border-radius:8px; padding:16px; margin:16px 0; font-family:sans-serif;">
<table style="width:100%; border-collapse:collapse;">
<tr style="border-bottom:1px solid #ddd;"><td style="padding:6px; font-weight:bold;">Tool</td><td style="padding:6px; font-weight:bold;">Use it for</td></tr>
<tr style="border-bottom:1px solid #eee;"><td style="padding:6px;"><strong>matsim-python-tools</strong> / <strong>matsim-r</strong></td><td style="padding:6px;">Streaming <code>output_events.xml.gz</code> and plans into pandas/R dataframes — mode share, trip duration/distance, congestion by link and time-of-day</td></tr>
<tr style="border-bottom:1px solid #eee;"><td style="padding:6px;"><strong><a href="https://simunto.com/via/">Via</a></strong> (Simunto)</td><td style="padding:6px;">The de facto standard visualizer — free, drag in network + events, get an animated playback of vehicles moving through the network plus link-volume heatmaps</td></tr>
<tr style="border-bottom:1px solid #eee;"><td style="padding:6px;">OTFVis</td><td style="padding:6px;">MATSim's older built-in visualizer, attachable live during a run — mostly superseded by Via but still used for live debugging</td></tr>
<tr style="border-bottom:1px solid #eee;"><td style="padding:6px;">SimWrapper (VSP)</td><td style="padding:6px;">Web-based dashboards reading an output directory directly — configurable charts/maps without writing plotting code</td></tr>
<tr><td style="padding:6px;">QGIS</td><td style="padding:6px;">Overlay converted network/volume shapefiles onto other GIS layers (land use, census)</td></tr>
</table>
</div>

A typical pipeline: check `scorestats.txt` for convergence → open in Via for a visual sanity check ("does traffic look plausible on the highway?") → pull `output_events.xml.gz` into pandas for the actual study metrics → SimWrapper or a notebook for the final report.

## Ongoing development

MATSim is governed by the nonprofit **MATSim Association**, formed by the contributing universities, rather than owned by one lab — matching its original goal of a distributed academic project. Development happens openly on GitHub (`matsim-org/matsim-libs`, roughly 30 active contributors, 500+ forks). New capability almost always ships as an optional **contrib** module — DRT/ride-hail, transit routing, emissions, SimWrapper integration — rather than growing the core, which is what has let the framework accumulate two decades of extensions from different labs without destabilizing. Releases are now calendar-versioned (`2024.0`, `2025.0`, `2026.0`, one per year), and priorities are steered partly by the annual MATSim User Meeting and community surveys rather than a single roadmap owner.

## Quiz

<div id="quiz-matsim" style="font-family:sans-serif; max-width:700px;">

<div class="quiz-q" style="margin-bottom:20px; padding:12px; border:1px solid #ddd; border-radius:8px;">
<p><strong>1. What does one MATSim iteration actually do?</strong></p>
<button onclick="qFeedback(this,false,'MATSim doesn\'t solve a system of flow equations directly — it simulates individual agents\' days and lets patterns emerge from their choices.')">Solves a system of flow equations for the whole network at once</button><br>
<button onclick="qFeedback(this,true,'Right — execute all agents\' current plans for a day, score how each day went, then let a fraction of agents mutate their plan before the next round.')">Executes every agent\'s current daily plan, scores the outcome, then lets a fraction of agents mutate their plan</button><br>
<button onclick="qFeedback(this,false,'There\'s no single global optimizer step — the "optimization" is emergent, arising from many individual agents\' local plan changes.')">Runs a global optimizer that recomputes every agent\'s ideal route simultaneously</button><br>
</div>

<div class="quiz-q" style="margin-bottom:20px; padding:12px; border:1px solid #ddd; border-radius:8px;">
<p><strong>2. Why does MATSim usually simulate the entire day rather than just a rush-hour window?</strong></p>
<button onclick="qFeedback(this,false,'It\'s the opposite — full-day realism is exactly what the framework is built to capture, not a fallback for something it can\'t do.')">Because MATSim can\'t restrict its simulated time window at all</button><br>
<button onclick="qFeedback(this,true,'Exactly — agents can only learn to shift departure times or modes if the simulation shows them the consequences across the full day; rush-hour analysis is instead a filter applied to the output afterward.')">Because its core mechanism is time-of-day choice, and agents need the full day\'s context to learn to shift when they travel; rush-hour numbers come from filtering the output afterward</button><br>
<button onclick="qFeedback(this,false,'The qsim start/end time is configurable — this isn\'t a hardware limit.')">Because the underlying hardware can\'t simulate a shorter window efficiently</button><br>
</div>

<div class="quiz-q" style="margin-bottom:20px; padding:12px; border:1px solid #ddd; border-radius:8px;">
<p><strong>3. What does "converged" actually mean for a MATSim run?</strong></p>
<button onclick="qFeedback(this,false,'Because of probabilistic (logit-style) plan choice, MATSim doesn\'t settle to a single unchanging state — this describes a literal fixed point, not what actually happens.')">Every agent\'s score becomes numerically identical and never changes again</button><br>
<button onclick="qFeedback(this,true,'Correct — because plan selection is probabilistic, the population settles into a stochastic quasi-equilibrium: scores fluctuate in a small band, and that fluctuation is expected, not a bug.')">The population\'s average score stabilizes into a fluctuating but stable band, especially once plan innovation is disabled for the run\'s final iterations</button><br>
<button onclick="qFeedback(this,false,'MATSim has no such automatic detection by default — convergence is checked manually from the stats files.')">MATSim automatically halts the run once a built-in convergence threshold is detected</button><br>
</div>

<div class="quiz-q" style="margin-bottom:20px; padding:12px; border:1px solid #ddd; border-radius:8px;">
<p><strong>4. What's the correct way to extend a MATSim run that hasn't converged after its configured iterations?</strong></p>
<button onclick="qFeedback(this,false,'This throws away the population\'s learned plans and typically requires far more total iterations than continuing from where it left off.')">Discard the results and restart the whole run from iteration 0 with a higher iteration count</button><br>
<button onclick="qFeedback(this,true,'Right — feeding the prior run\'s learned plans back in as the starting population avoids re-exploring from scratch, though it\'s best paired with a replanning schedule that doesn\'t reset innovation as if starting fresh.')">Warm-start a continuation using the previous run\'s output_plans.xml.gz as the new run\'s population input</button><br>
<button onclick="qFeedback(this,false,'MATSim has no such automatic extension mechanism — you decide this yourself by inspecting the stats files.')">Nothing needs to change — MATSim automatically detects non-convergence and extends the run on its own</button><br>
</div>

<div class="quiz-q" style="margin-bottom:20px; padding:12px; border:1px solid #ddd; border-radius:8px;">
<p><strong>5. Has MATSim been ported to Python?</strong></p>
<button onclick="qFeedback(this,false,'matsim-python-tools is an I/O and analysis library (reading MATSim\'s XML output into pandas) — the simulation engine itself is still Java.')">Yes — matsim-python-tools is a full Python reimplementation of the simulation engine</button><br>
<button onclick="qFeedback(this,true,'Correct — the maintainers have explicitly said they lack the capacity/need to build one themselves, though they\'d support an outside contributor doing it; JPype (driving the JVM from Python) is the practical workaround.')">No — the maintainers have explicitly declined to build one, citing capacity rather than technical difficulty, and point to JPype as a workaround instead</button><br>
<button onclick="qFeedback(this,false,'BEAM extends MATSim\'s capabilities (ride-hail, EVs) but is also JVM-based (Scala/Java), not a Python port.')">Yes — BEAM is a Python rewrite of MATSim\'s core</button><br>
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
