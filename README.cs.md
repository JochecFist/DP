# Algoritmy strojového učení pro adaptaci autonomních agentů v 3D akční hře

[English](README.md) | **Česky**

Diplomová práce (Ing.), Fakulta jaderná a fyzikálně inženýrská ČVUT v Praze, 2026.
Katedra softwarového inženýrství. Vedoucí práce: Ing. Josef Nový, Ph.D.

EN: *Machine Learning Algorithms for Adapting Autonomous Agents in a 3D Action Game*

---

## O co jde

Agent pro střílečku z pohledu první osoby, postavený v Unreal Engine 5, který se pomocí
zpětnovazebního učení naučí bojovat proti hráči místo toho, aby jel podle předem napsaného
stromu chování. Mezi koly si navíc sám vybírá zbraň a brnění podle toho, jak dopadlo kolo
předchozí.

Práce popisuje návrh, tři verze agenta a měřené porovnání všech tří proti lidskému hráči, proti
skriptovanému NPC a mezi sebou navzájem.

## Hra

Střílečka z pohledu první osoby na zničeném městském náměstí: park a bankovní budova obklopené
barikádami. Hráč začíná uvnitř banky, k dispozici jsou zbraně a zásobovací box, hra je
rozdělená do kol. V každém kole je úkolem eliminovat daný počet nepřátel, pak začíná další kolo
s větším počtem protivníků. Zabití nepřátelé odhazují lékárničky a munici.

Prostředí a část Blueprint kódu pochází z mé bakalářské práce. Většinu bylo nutné přepsat nebo
odstranit, aby byla použitelná pro nového agenta.

## Použité technologie

| | |
|---|---|
| Engine | Unreal Engine 5 (Blueprinty) |
| RL vrstva | plugin [NevarokML](https://github.com/nevarok/NevarokML) |
| Trénovací backend | Stable-Baselines3 (Python) |
| Algoritmus | Proximal Policy Optimization (PPO), `MlpPolicy` |
| Inference | ONNX přes Neural Network Engine (NNE) v Unrealu |
| Monitoring | TensorBoard |

NevarokML funguje na principu klient a server. Unreal simuluje prostředí a chování agentů,
pythonový klient provádí samotné učení pomocí Stable-Baselines3 a obě strany si přes sockety
vyměňují pozorování, akce a odměny ve formátu JSON. Natrénované váhy se exportují do ONNX
a načítají zpět do Unrealu jako objekt `NNEModelData` pro inferenci za běhu.

PPO bylo zvoleno před ostatními algoritmy, které NevarokML nabízí (A2C, DDPG, DQN, SAC, TD3),
kvůli stabilitě v nestacionárním prostředí a práci s diskrétním akčním prostorem. `MlpPolicy`
místo `CnnPolicy` proto, že agent dostává pouze číselné hodnoty popisující stav prostředí, ne
obrazy. Konvoluční vrstvy by tedy jen zvýšily složitost modelu bez užitku.

### Hyperparametry

```
policy          MlpPolicy      gamma           0.9
learning_rate   0.0003         gae_lambda      0.95
n_steps         200            clip_range      0.2
batch_size      54000          ent_coef        0.0
n_epochs        200            vf_coef         0.5
                               max_grad_norm   0.5
```

Hyperparametry nebyly převzaty z literatury, ale laděny experimentálně. Vždy se změnil jeden
parametr a sledovalo se, jestli změna vedla k rychlejší konvergenci, stabilnějším odhadům hodnot
a plynulejšímu růstu odměn, nebo naopak k oscilacím. Největší vliv měla volba algoritmu
a politiky, nejmenší velikost mini batchů, která ovlivňovala spíš rychlost tréninku než kvalitu
výsledné politiky.

## Tři verze agenta

**V1, pouze navigace.** Záměrně minimální agent, jehož jediným úkolem bylo dojít k cílové
pozici. Sloužil k ověření frameworku, ne k hraní.

**V2, boj.** Multi diskrétní akční prostor rozdělený na pohyb, rotaci a střelbu. Pozorování
rozšířena o vzdálenost k nepříteli a normalizovanou úhlovou odchylku (yaw). Tato verze odhalila
skutečný problém: rozšíření akčního a pozorovacího prostoru samo o sobě nestačí. Špatně vyvážená
odměnová funkce vedla k chování, které sbíralo body, ale herně vypadalo nepřirozeně
a nestabilně. Odměnovou strukturu bylo nutné přestavět.

**Finální verze, boj a adaptace výbavy.** Přidána akce pro výměnu zbraně a brnění mezi koly
a kombinované odměnové podmínky (správná vzdálenost a zároveň správná rotace, ne jen jedna
z nich). Exportovaný ONNX model se stal multi head: samostatné výstupy pro pohyb, rotaci,
střelbu a volbu výbavy, koordinované v rámci jednoho kroku prostředí.

## Výsledky

Finální verze, 50 kol proti každému typu protivníka:

| Protivník | Úspěšnost | Délka kola | Průměrné poškození |
|---|---|---|---|
| Lidský hráč | hráče typicky eliminuje ve 4. kole | | ~40 |
| Skriptované NPC | ~55 % | ~10 s | 50 až 55 |
| Agent V2 | ~55 až 60 % | 10 až 12 s | 55 až 60 |

Doplňková měření:

- **Přesnost střelby:** 75 až 80 %
- **Adaptace výbavy:** výhodná kombinace zbraně a brnění zvolena v 80 až 90 % kol
- **Latence:** zavedená latence výkon mírně zhoršovala, méně než u V2
- **Zorné pole (FOV):** změny nevedly k pozorovatelné změně chování

Tréninkové metriky V1 (1 080 000 simulovaných kroků, přibližně 3 800 aktualizací modelu):
`approx_kl` 0,00098, `clip_fraction` 0,01, `explained_variance` 0,926, `entropy_loss` −0,157,
průměrná odměna na epizodu 0,937. Ztrátové funkce se držely v očekávaných mezích bez známek
divergence.

## Omezení

Poctivý výsledek je, že **skutečná adaptace v reálném čase během hraní nefungovala**, a příčinou
je použitý framework, ne návrh agenta.

- NevarokML podporuje pouze jednoho trenéra na jedno dynamicky vytvářené prostředí. U statických
  trénovacích map lze jednomu trenérovi přiřadit více prostředí, u živé herní mapy ne.
- Odměnovou funkci, akční prostor ani pozorování nelze měnit v průběhu tréninku se zachováním
  již naučeného modelu. To vylučuje curriculum learning, takže je nutné trénovat jeden komplexní
  model od nuly na plně složité úloze.
- Agent proto nedokáže dynamicky upravovat optimální vzdálenost od protivníka ani volit mezi
  rychlejší nepřesnou a pomalejší přesnou střelbou podle konkrétního stylu hráče.
- Úprava architektury neuronové sítě přes C++ rozhraní NevarokML se nepodařila. Veškerá
  vylepšení proto musela vzejít z okolní učicí smyčky: struktury pozorování, množiny akcí
  a návrhu odměn.

## Obsah repozitáře

Zdrojový LaTeX a zkompilovaný text práce. Samotný projekt v Unreal Engine součástí není, jde
z velké části o binární data nevhodná pro Git.

## Citace

```
JOCHEC, Martin. Algoritmy strojového učení pro adaptaci autonomních agentů
v 3D akční hře. Diplomová práce. Praha: ČVUT, Fakulta jaderná a fyzikálně
inženýrská, 2026. Vedoucí práce Ing. Josef Nový, Ph.D.
```
