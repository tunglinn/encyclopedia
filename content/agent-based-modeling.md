---
title: Agent-Based Modeling
tags:
  - simulation
  - modeling
  - complex-systems
date: 2026-09-16
---

A traffic jam that appears for no visible reason, a market crash that no single trader intended, a flock of starlings turning as one — none of these have a "leader" deciding what the group does. Agent-based modeling (ABM) is the modeling approach built around exactly that observation: instead of writing down equations for how a system behaves as a whole, you program many small, individually simple decision-makers, let them interact, and watch the system-level pattern emerge on its own.

## Background

Most modeling approaches you meet first are **equation-based**: you write a formula for how some aggregate quantity changes over time (the size of an infected population, the average price of a stock, the density of cars on a highway) and solve or simulate that formula directly. This works well when a system is homogeneous enough that "the average behavior" is a good description of everyone in it.

ABM exists for the systems where that assumption breaks: where individuals genuinely differ from each other, where their differences matter to the outcome, and where the interesting behavior of the system isn't in anyone's rulebook — it just falls out of many individual decisions colliding. You don't need a modeling background to follow this page, but it helps to keep one comparison in mind throughout: an equation-based model tells you *what* the system does; an agent-based model tells you *who* does what, and lets you *watch* the system-level answer emerge.

## Intuition

The clearest illustration is a flock of birds. No bird has a map of the flock's shape or a plan for which way to turn. Each bird follows three local rules: stay close to your neighbors, don't collide with them, and match their average direction. Simulate a few hundred birds each following only those three rules, and the flock as a whole swoops, splits, and reforms exactly like a real one — a global pattern with no global rule behind it.

<div style="border:1px solid #ccc; border-radius:8px; padding:16px; margin:16px 0; font-family:sans-serif;">
<strong>Two ways to model the "same" system</strong>
<table style="width:100%; border-collapse:collapse; margin-top:8px;">
<tr style="border-bottom:1px solid #ddd;"><td style="padding:6px; font-weight:bold;">Equation-based</td><td style="padding:6px; font-weight:bold;">Agent-based</td></tr>
<tr><td style="padding:6px; vertical-align:top;">"Flock density spreads according to a diffusion equation with cohesion term λ."<br><br>One formula, describes the whole flock, fast to compute, but λ is an abstraction with no bird actually experiencing it.</td><td style="padding:6px; vertical-align:top;">"Each bird stays near its neighbors, avoids collisions, matches heading."<br><br>Hundreds of simple individuals, slower to compute, but every rule maps onto something a real bird could plausibly be doing.</td></tr>
</table>
</div>

Traffic works the same way (see [[Mobility Simulations]] for the transportation-specific version of this idea): nobody programs a traffic jam, it emerges when enough individual drivers each brake a little more than the driver ahead of them.

## The substance

### Core ingredients

- **Agents** — discrete entities with an internal state (position, beliefs, resources, a strategy) and a set of behavioral rules that determine how they act and how that state updates.
- **Environment** — the space agents inhabit and interact through: a physical grid, a road network, a social network, or an abstract space.
- **Interaction rules** — how agents affect each other and the environment (competing for resources, exchanging information, avoiding crowding).
- **Time steps** — the simulation advances in discrete steps (or continuous time in some variants), with every agent re-evaluating and acting at each step.

### From assumption to running code

This is the part that's easy to gloss over: "each agent decides based on its own rules" sounds simple to say and is genuinely laborious to build. There's a real pipeline between having an assumption about behavior and having a program that enacts it.

<div style="border:1px solid #ccc; border-radius:8px; padding:16px; margin:16px 0; font-family:sans-serif;">
<strong>Example: turning "people avoid crowded buses" into code</strong>
<ol style="margin-top:8px; padding-left:20px;">
<li style="margin-bottom:10px;"><strong>Conceptual assumption</strong> (plain language, what a domain expert believes): "riders are less likely to choose a bus route the more crowded it usually is."</li>
<li style="margin-bottom:10px;"><strong>Formalization</strong> (turn it into math): pick a functional form and give it a number. E.g. a utility function <code>U = β₁·(-travel_time) + β₂·(-crowding_level)</code>, where β₂ is a penalty weight — this is a specific, falsifiable claim, not a vague intuition anymore.</li>
<li style="margin-bottom:10px;"><strong>Calibration</strong> (ground it in data): estimate β₁ and β₂ by statistically fitting the formula against real trip-choice data (travel surveys, smart-card records) using standard techniques like maximum-likelihood estimation for a discrete-choice model. The "assumption" is now a number backed by evidence, not a guess.</li>
<li style="margin-bottom:10px;"><strong>Implementation</strong> (make it executable): write the formula into an agent's decision method in whatever framework you're using — a few lines of Python in a Mesa agent's <code>step()</code>, a small procedure in NetLogo, a Java method in Repast, or (for the specific case of transportation) plugging the coefficients into a pre-built mode-choice module in MATSim/ActivitySim, which needs no new code at all if you're just changing numbers rather than the functional form.</li>
</ol>
</div>

Notice step 4 splits in two, and this distinction matters a lot in practice:

- **Using an existing module with new numbers** — no code required. Established platforms (MATSim, ActivitySim, BEAM) ship pre-built, already-validated decision modules (mode choice, destination choice, departure-time choice) where "encoding an assumption" just means editing a config file with new coefficients.
- **Inventing a genuinely new behavior** — real software engineering required. If your assumption doesn't fit any existing module's shape (say, a totally new kind of decision no one has modeled before), you write actual code implementing that agent's decision procedure inside the framework's API.

So the honest answer to "how much work is it" is: it depends entirely on whether your assumption is a *new number* for an existing rule, or a *new kind of rule* nobody has built yet.

### Is it as simple as a natural-language prompt?

Not in the tools that transportation agencies and most researchers rely on today — but the gap is narrower than it used to be, and it's worth understanding both sides.

**The traditional path (described above) requires formalization.** A vague assumption like "people avoid crowded buses" cannot run as-is; someone has to commit to a specific functional form and specific parameter values before any simulator can execute it. That translation step — turning a fuzzy belief into precise, falsifiable math — is arguably the actual intellectual work of building an ABM, not a formality you can skip.

**A newer path exists that gets closer to "just prompt it."** Since 2023, researchers have built "generative agents": instead of hand-coding a decision function, each agent carries a natural-language memory and persona ("a 34-year-old parent, budget-conscious, dislikes crowds"), and at each simulated timestep a large language model is prompted with that agent's memory and situation to generate what it does next in plain English, which is then parsed back into a concrete action. This is much closer to what you're picturing — you describe the agent in words, and the model improvises its behavior.

> [!warning] Why production transportation models don't do this (yet)
> The LLM-driven approach trades away exactly the properties that regulators and planners need from a transportation model: it's expensive to run at the scale of millions of daily-commute agents (an LLM call per agent per decision vs. a few floating-point operations for a calibrated utility function), it's hard to rigorously calibrate against decades of travel-survey data the way a discrete-choice model can be, it's less reproducible (outputs shift with model version and sampling randomness), and it can produce behavior that's fluent but not actually grounded in evidence. So it shows up in exploratory social-simulation research today, not in the mode-choice models a city uses to justify a transit investment.

In short: today's mainstream ABM tools need assumptions translated into calibrated, executable math, not typed as a sentence — but "prompt an LLM playing a persona" is a real and growing alternative for exploratory, qualitative simulation where rigor matters less than capturing nuanced, hard-to-formalize behavior.

### Contrast with other modeling approaches

- **Equation-based / system dynamics models** describe the system with aggregate variables and differential equations (e.g. "the infected population grows at rate X"). More tractable and easier to analyze mathematically, but assume homogeneity and can miss emergent effects that come from individual variation.
- **Cellular automata** are a constrained special case: agents sit on a fixed grid and their rules depend only on neighboring cells. ABM generalizes this — agents can move, form networks, and carry richer internal state.

### Common uses

- Epidemiology (disease spread through a population of individuals with distinct contact patterns)
- Economics and finance (markets as populations of trading agents with different strategies)
- Traffic and pedestrian flow (see [[Mobility Simulations]] for a deep dive on transportation-specific ABM tools, tradeoffs, and a worked example)
- Ecology (predator-prey dynamics, foraging behavior)
- Social science (opinion dynamics, segregation models like Schelling's model)

### Tools

- **NetLogo** — approachable, widely used in teaching, its own simple scripting language.
- **Mesa** (Python) — a lightweight framework where agent logic is ordinary Python, easy to integrate with the rest of the Python data/ML ecosystem.
- **Repast** — Java-based, used for larger and more performance-sensitive academic models.
- Domain-specific transportation platforms ([[MATSim]], BEAM, ActivitySim) — see [[Mobility Simulations]].

## Quiz

<div id="quiz-abm" style="font-family:sans-serif; max-width:700px;">

<div class="quiz-q" style="margin-bottom:20px; padding:12px; border:1px solid #ddd; border-radius:8px;">
<p><strong>1. What's the key difference between an equation-based model and an agent-based model?</strong></p>
<button onclick="qFeedback(this,false,'Backwards — equation-based models describe the aggregate directly; ABM builds it bottom-up from individuals.')">Equation-based models simulate individuals; agent-based models solve a single aggregate formula</button><br>
<button onclick="qFeedback(this,true,'Right — equation-based models describe the system as a whole with a formula; ABM builds it from many individual agents whose interactions produce the system-level behavior.')">Equation-based models describe the system as a whole with a formula; agent-based models build it up from individual decision-makers</button><br>
<button onclick="qFeedback(this,false,'Both approaches can, and often do, run as computer simulations — that\'s not the distinguishing factor.')">Equation-based models always run on paper; agent-based models always require a computer</button><br>
</div>

<div class="quiz-q" style="margin-bottom:20px; padding:12px; border:1px solid #ddd; border-radius:8px;">
<p><strong>2. In the "riders avoid crowded buses" example, what does the calibration step actually do?</strong></p>
<button onclick="qFeedback(this,false,'That\'s the formalization step (choosing the functional form), which happens before calibration.')">Chooses which mathematical formula to use for the utility function</button><br>
<button onclick="qFeedback(this,true,'Correct — calibration statistically fits the formula\'s free parameters against real data, grounding the assumption in evidence rather than a guess.')">Statistically fits the formula\'s parameters (like the crowding penalty weight) against real travel data</button><br>
<button onclick="qFeedback(this,false,'That\'s the implementation step — writing the formula into code in a specific framework.')">Writes the formula as executable code inside an agent\'s decision method</button><br>
</div>

<div class="quiz-q" style="margin-bottom:20px; padding:12px; border:1px solid #ddd; border-radius:8px;">
<p><strong>3. When does encoding a new behavioral assumption into a platform like MATSim require writing new code, versus just editing a config file?</strong></p>
<button onclick="qFeedback(this,true,'Right — if an existing module (e.g. mode choice) already has the right shape, changing its coefficients is just a config edit; a genuinely new kind of decision requires actual implementation work.')">New code is needed only when the assumption doesn\'t fit any existing decision module\'s shape; new parameter values for an existing module are just a config change</button><br>
<button onclick="qFeedback(this,false,'It\'s not about the number of agents — it\'s about whether the assumption fits an existing module\'s functional form.')">New code is always required, regardless of the assumption, once you have more than a few thousand agents</button><br>
<button onclick="qFeedback(this,false,'The reverse is closer to true — many changes are just config edits, but genuinely new behaviors do require code.')">Config files can only change agent appearance, never their actual decisions</button><br>
</div>

<div class="quiz-q" style="margin-bottom:20px; padding:12px; border:1px solid #ddd; border-radius:8px;">
<p><strong>4. Why don't production transportation planning models typically use LLM-prompted "generative agents" today?</strong></p>
<button onclick="qFeedback(this,false,'The opposite — LLM-driven simulation is newer and less established than calibrated discrete-choice models, not a rejected older method.')">Because LLM-based agents are an older technique that calibrated discrete-choice models have already replaced</button><br>
<button onclick="qFeedback(this,true,'Exactly — cost at scale, weaker calibration against real survey data, poor reproducibility, and fluent-but-ungrounded outputs make LLM-driven agents a worse fit for models that need to be defensible to planners and regulators, even though they lower the barrier to encoding qualitative assumptions.')">Because at the scale needed, they\'re expensive to run, hard to rigorously calibrate against real data, less reproducible, and can produce fluent but ungrounded behavior</button><br>
<button onclick="qFeedback(this,false,'LLMs are broadly capable of generating text describing travel decisions — the issue isn\'t raw technical capability, it\'s cost, calibration, and reproducibility at the scale and rigor transportation planning needs.')">Because large language models are technically incapable of representing a travel decision</button><br>
</div>

<div class="quiz-q" style="margin-bottom:20px; padding:12px; border:1px solid #ddd; border-radius:8px;">
<p><strong>5. In the bird-flocking example, why does the flock's overall shape emerge without being explicitly programmed anywhere?</strong></p>
<button onclick="qFeedback(this,false,'There is no such rule in the model — that\'s the entire point of the example.')">Because one designated "leader bird" is explicitly programmed to plan the flock\'s shape</button><br>
<button onclick="qFeedback(this,true,'Right — the flock-level pattern is an emergent property: it arises from many individual birds each following simple local rules, with no single rule describing the shape directly.')">Because each bird follows only simple local rules (stay near neighbors, avoid collisions, match heading), and the group pattern emerges from their combined interactions</button><br>
<button onclick="qFeedback(this,false,'The simulation isn\'t solving a global equation for the flock\'s shape at all — that\'s exactly what the equation-based approach would do instead.')">Because the simulation solves a hidden global equation for flock shape behind the scenes</button><br>
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
