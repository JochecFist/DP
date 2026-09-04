# Machine Learning Algorithms for Adapting Autonomous Agents in a 3D Action Game

**English** | [Česky](README.cs.md)

Master's thesis (Ing.), Faculty of Nuclear Sciences and Physical Engineering, CTU in Prague, 2026.
Department of Software Engineering. Supervisor: Ing. Josef Nový, Ph.D.

CZ: *Algoritmy strojového učení pro adaptaci autonomních agentů v 3D akční hře*

---

## What this is

A reinforcement-learning agent for a first-person shooter, built in Unreal Engine 5, that learns
to fight the player instead of following a scripted behaviour tree. Between rounds it also picks
its own weapon and armour based on how the previous round went.

The thesis covers the design, three iterations of the agent, and a measured comparison of all
three against a human player, against a scripted NPC, and against each other.

## The game

A first-person shooter set on a ruined city square: a park and a bank building enclosed by
barricades. The player starts inside the bank, weapons and a supply crate are available, and
play is structured into rounds. Each round requires eliminating a set number of enemies before
the next one starts with more of them. Killed enemies drop health and ammo pickups.

The environment and part of the Blueprint code originate from my bachelor's thesis. Most of it
had to be rewritten or removed to be usable for the new agent.

## Tech stack

| | |
|---|---|
| Engine | Unreal Engine 5 (Blueprints) |
| RL bridge | [NevarokML](https://github.com/nevarok/NevarokML) plugin |
| Training backend | Stable-Baselines3 (Python) |
| Algorithm | Proximal Policy Optimization (PPO), `MlpPolicy` |
| Inference | ONNX via Unreal's Neural Network Engine (NNE) |
| Monitoring | TensorBoard |

NevarokML runs client–server: Unreal simulates the environment and the agents, a Python client
does the actual learning with Stable-Baselines3, and the two talk over sockets using JSON to
exchange observations, actions and rewards. Trained weights are exported to ONNX and loaded back
into Unreal as an `NNEModelData` object for inference at runtime.

PPO was chosen over the other algorithms NevarokML exposes (A2C, DDPG, DQN, SAC, TD3) for its
stability in non-stationary environments and its handling of a discrete action space.
`MlpPolicy` rather than `CnnPolicy` because the agent receives plain numeric state values, not
images — convolutional layers would add complexity for nothing.

### Hyperparameters

```
policy          MlpPolicy      gamma           0.9
learning_rate   0.0003         gae_lambda      0.95
n_steps         200            clip_range      0.2
batch_size      54000          ent_coef        0.0
n_epochs        200            vf_coef         0.5
                               max_grad_norm   0.5
```

These were tuned empirically, one parameter at a time, watching convergence speed, value
estimate stability and reward oscillation after each change — not lifted from the literature.
The algorithm and policy choice mattered most; mini-batch size mattered least, affecting
training speed rather than final policy quality.

## The three versions

**V1 — navigation only.** A deliberately minimal agent whose only job was to reach a target
position. Its purpose was validating the framework, not gameplay.

**V2 — combat.** Multi-discrete action space split into movement, rotation and firing.
Observations extended with distance to enemy and normalised yaw difference. This version
exposed the real problem: widening the action and observation space isn't enough. A badly
balanced reward function produced behaviour that scored well and looked wrong — short-term
effective, unnatural to play against, unstable. The reward structure had to be rebuilt.

**Final — combat plus equipment adaptation.** Added an action for swapping weapon and armour
between rounds, and combined reward conditions (correct distance *and* correct rotation rather
than either alone). The exported ONNX model became multi-head: separate outputs for movement,
rotation, firing and equipment choice, coordinated within a single environment step.

## Results

Final version, measured over 50 rounds per opponent type:

| Opponent | Success rate | Round length | Avg. damage dealt |
|---|---|---|---|
| Human player | typically kills the player by round 4 | — | ~40 |
| Scripted NPC | ~55 % | ~10 s | 50–55 |
| Agent V2 | ~55–60 % | 10–12 s | 55–60 |

Additional measurements:

- **Shooting accuracy:** 75–80 %
- **Equipment adaptation:** a favourable weapon/armour combination chosen in 80–90 % of rounds
- **Latency:** introduced latency degraded performance slightly, less than it did for V2
- **Field of view:** changes produced no observable behavioural difference

Training metrics for V1 (1,080,000 simulated steps, ~3,800 model updates): `approx_kl` 0.00098,
`clip_fraction` 0.01, `explained_variance` 0.926, `entropy_loss` −0.157, mean episode reward
0.937. Loss functions stayed within expected bounds with no sign of divergence.

## Limitations

The honest result is that **genuine real-time adaptation during play did not work**, and the
reason is the framework rather than the agent design.

- NevarokML supports only one trainer per dynamically created environment. Static training maps
  can assign multiple environments to a single trainer; the live game map cannot.
- The reward function, action space and observations cannot be changed mid-training while
  keeping an already-learned model. That rules out curriculum learning, so one complex model has
  to be trained from scratch on the full task.
- Consequently the agent cannot adjust its preferred engagement distance or trade off fire rate
  against accuracy in response to a specific player's style.
- Modifying the network architecture directly through NevarokML's C++ interface was attempted
  and did not work. All improvements therefore had to come from the surrounding learning loop:
  observation structure, action set and reward design.

## Repository contents

This repository holds the LaTeX source and the compiled thesis text. The Unreal Engine project
itself is not included — it is largely binary assets and unsuitable for Git.

## Citation

```
JOCHEC, Martin. Algoritmy strojového učení pro adaptaci autonomních agentů
v 3D akční hře. Diplomová práce. Praha: ČVUT, Fakulta jaderná a fyzikálně
inženýrská, 2026. Vedoucí práce Ing. Josef Nový, Ph.D.
```
