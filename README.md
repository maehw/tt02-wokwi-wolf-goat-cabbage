*(Please note that is a forked repository. Details of the design are described at the very end of this `README.md`. Go to https://tinytapeout.com for instructions!)*
*(Please also note that this is the last-minute migration of a tt1 project: https://github.com/maehw/wokwi-verilog-gds-wolf-goat-cabbage)*

![](../../workflows/gds/badge.svg) ![](../../workflows/docs/badge.svg)

# What is this whole thing about?

This repo is a template you can make a copy of for your own [ASIC](https://www.zerotoasiccourse.com/terminology/asic/) design using [Wokwi](https://wokwi.com/).

When you edit the Makefile to choose a different ID, the [GitHub Action](.github/workflows/wokwi.yaml) will fetch the digital netlist of your design from Wokwi.

The design gets wrapped in some extra logic that builds a 'scan chain'. This is a way to put lots of designs onto one chip and still have access to them all. You can see [all of the technical details here](https://github.com/mattvenn/scan_wrapper).

After that, the action uses the open source ASIC tool called [OpenLane](https://www.zerotoasiccourse.com/terminology/openlane/) to build the files needed to fabricate an ASIC.

# What files get made?

When the action is complete, you can [click here](https://github.com/mattvenn/wokwi-verilog-gds-test/actions) to see the latest build of your design. You need to download the zip file and take a look at the contents:

* gds_render.svg - picture of your ASIC design
* gds.html - zoomable picture of your ASIC design
* runs/wokwi/reports/final_summary_report.csv  - CSV file with lots of details about the design
* runs/wokwi/reports/synthesis/1-synthesis.stat.rpt.strategy4 - list of the [standard cells](https://www.zerotoasiccourse.com/terminology/standardcell/) used by your design
* runs/wokwi/results/final/gds/user_module.gds - the final [GDS](https://www.zerotoasiccourse.com/terminology/gds2/) file needed to make your design

# What is this specific design about?

The story of the [wolf, goat and cabbage problem](https://en.wikipedia.org/wiki/Wolf,_goat_and_cabbage_problem), quoted from wikipedia with some emojis added for later use in visualization:

>A farmer (🧑‍🌾) went to a market and purchased a wolf (🐺), a goat (🐐), and a cabbage (🥬). On his way home, the 🧑‍🌾 came to the bank of a river (〰〰) and rented a boat (🛶). But crossing the river by 🛶, the 🧑‍🌾 could carry only himself and a single one of his purchases: the 🐺, the 🐐, or the 🥬.
>
>If left unattended together, the 🐺 would eat the 🐐, or the 🐐 would eat the 🥬.
>
>The 🧑‍🌾's challenge was to carry himself and his purchases to the far bank of the 〰〰 leaving each purchase intact.

The solution to the problem is known. However, this project wants to allow the interactive game of this puzzle.

*Disclaimer: I have seen this logic puzzle been realized in plain hardware but I couldn't find it anymore! Any hints are welcome.*


## Input signals

We define four input signals for every relevant object involved in the puzzle:
- **F** for the position of the **f**armer (🧑‍🌾/🚣)
- **W** for the position of the **w**olf (🐺)
- **G** for the position of the **g**oat (🐐)
- **C** for the position of the **c**abbage (🥬),

where

- a 0 means that the object is located on the *left* river bank and
- a 1 means that the object is located on the *right* river bank.

This allows us to use a slider switch for every input signal, so that we can visualize the object's location.
As we will be working with Wokwi, the slider switch can be a [Standard Single Pole Double Throw (SPDT) slide switch](https://docs.wokwi.com/parts/wokwi-slide-switch) with three terminals:

![slider_switch](https://user-images.githubusercontent.com/6305922/187899950-66d318a0-8b79-4f71-8cd2-49a921594b90.png)

The common terminal connects to an input pin of the ASIC (with an internal or external pull-up). The left terminal is connected to GND.

When the switch is in the *left position*, the ASIC pin is connected to GND which we define as logical signal level 0. The object the switch is related to (🧑‍🌾/🚣, 🐺, 🐐 or 🥬) is on the *left river bank*.

When the switch is in the *right position* (right terminal being left unconnected), the signal is pulled high which we define as logical 1. The object the switch is related to (🧑‍🌾/🚣, 🐺, 🐐 or 🥬) is on the *right river bank*.

> **Note**
> TODO: Add example(s).

## Output signals

We want to let the player of the game know when they have won or lost.

The player has lost as soon as

* the 🐺 and the 🐐 are left unattended (i.e. absence of the 🧑‍🌾) *or*
* the 🐐 and the 🥬 are left unattended

because at least one of the objects would be eaten.

The player has won as soon as all objects (🧑‍🌾/🚣, 🐺, 🐐 or 🥬) have reached the *right river bank*.


We can define the following outut signals to be relevant for the game play:

We define four input signals for every relevant object involved in the puzzle:
- **L** for the situation on the **l**eft bank being under control (no object-X-eats-object-Y situation),
  where 1 means "everything is fine" (✔️) and 0 means "situation is out of control" (❌).
- **R** for the situation on the **r**ight bank being under control (no object-X-eats-object-Y situation),
  where 1 means "everything is fine" (✔️) and 0 means "situation is out of control" (❌).
- **E** summary of the situations on both river banks (no object-X-eats-object-Y situation),
  where 1 means "**e**verything is fine" (✔️) and 0 means "situation is out of control" (❌).
  → Game is lost. ❌❌❌
- **A** as an indicator that **a**ll objects have reached the right bank of the river,
  where 1 means "yes" (✔️) and 0 means "no" (❌).
  → Game is won! 🎉🎉🎉

**L** and **R** are intermediate signals indicating on which side of the river a situation has occured why the player has lost the game.

There are some limitiations that are not covered by the current implementation, as described in a separate paragraph below.

We'll work with the following truth table with extended explanations:

|🧑‍🌾/🚣|🐺|🐐|🥬|Scenario          |Situation<br />on the<br />left bank<br />under control?|Situation<br />on the<br />right bank<br />under control?|Every-<br />thing<br />under<br />control?|All on<br />the right<br />bank?|
|----------|----|----|----|------------------|:-------:|:--------:|:-----:|:---:|
|F *(in)* |W *(in)*|G *(in)*|C *(in)*|               |L *(out)*|R *(out)*|E *(out)*|A *(out)*|
|0         |0   |0   |0   |🐺🐐🥬🚣 〰  |✔️       |✔️        |✔️     |❌    |
|0         |0   |0   |1   |🐺🐐🚣 〰 🥬 |✔️       |✔️        |✔️     |❌    |
|0         |0   |1   |0   |🐺🥬🚣 〰 🐐 |✔️       |✔️        |✔️     |❌    |
|0         |0   |1   |1   |🐺🚣 〰 🐐🥬❌|✔️       |❌         |❌      |❌    |
|0         |1   |0   |0   |🐐🥬🚣 〰 🐺 |✔️       |✔️        |✔️     |❌    |
|0         |1   |0   |1   |🐐🚣 〰 🐺🥬 |✔️       |✔️        |✔️     |❌    |
|0         |1   |1   |0   |🥬🚣 〰 🐺🐐❌|✔️       |❌         |❌      |❌    |
|0         |1   |1   |1   |🚣 〰 🐺🐐🥬❌|✔️       |❌         |❌      |❌    |
|1         |0   |0   |0   |🐺🐐🥬❌ 〰 🚣|❌        |✔️        |❌      |❌    |
|1         |0   |0   |1   |🥬🚣 〰 🐺🐐❌|✔️       |❌         |❌      |❌    |
|1         |0   |1   |0   |🐺🥬 〰 🚣🐐 |✔️       |✔️        |✔️     |❌    |
|1         |0   |1   |1   |🐺 〰 🚣🐐🥬 |✔️       |✔️        |✔️     |❌    |
|1         |1   |0   |0   |🐐🥬❌ 〰 🚣🐺|❌        |✔️        |❌      |❌    |
|1         |1   |0   |1   |🐐 〰 🚣🐺🥬 |✔️       |✔️        |✔️     |❌    |
|1         |1   |1   |0   |🥬 〰 🚣🐺🐐 |✔️       |✔️        |✔️     |❌    |
|1         |1   |1   |1   | 〰 🚣🐺🐐🥬🎉|✔️       |✔️        |✔️     |✔️   |

## The Logic

There are nice tools that support finding minimized boolean functions to generate the output signals from the input signals, e.g. http://tma.main.jp/logic/index_en.html. However, you can also do this manually! Note: Multiple minimal function forms are possible! I've picked those which I like best.

The function for output L can be written in the follwoing minimal form, where ...

* ~ means negation (NOT gate)
* f•g means "f logical-AND g" (AND gate)
* h+k means "h logical-OR k" (OR gate)
* just as in multiplication and addition, • has a stronger operator binding/ operator priority than +

L = ~F + C + G

*(In text: everything is under control on the left bank in scenarios where (🚣 is on the left) OR (🥬 is on the right) OR (🐐 is on the right). Note that the operators are _not_ exclusive ORs!)*


The function for output R can be written in the follwoing minimal form, where ...

R = ~F•~G + F•G + ~W•~C + F•W

*(In text: everything is under control on the right bank in scenarios where (the 🐐 is accompanied by the 🧑‍🌾 independent from the side of the river) OR (🧑‍🌾 accompanies the 🥬 on the left bank) OR (🐺 is accompanied by the 🧑‍🌾 on the right side). Note that the operators are _not_ exclusive ORs again!)*


Everything is under control (output E), when the situation is under control on the left **and** on the right river side. The function can be written in the following form:

E = L•R

The game is won (output A) when all objects are on the right river side:

A = F•W•G•C


## Limitiations of the implementation

* **No limit checking**: It's only logic so far. Still, there's no limit checking mechanism that prohibits that more than two objects cross the river at the same time (which of one must be the 🚣).
* **No sync or coupling:** There's also no mechanism that automatically synchronizes a movement of the 🚣 with the movement of another object which is in the same 🛶.
* **No history:** The player must play fair and restart the game (move all objects to the left river bank) when they have lost the game and not keep on moving switches when they have actually lost.


## ASIC hardware I/O pin mapping and Wokwi user interface in the simulation

### Wokwi simulation secreenshot

![Wokwi design simulation](./wokwi-simulation-io-mapping.png)

### Input pins

The design uses 4 input pins for the 4 input signals:

* IN0: *not connected* because it is typically used for clocked designs and may be used in the future of this design
* IN1: input signal **F** for the position of the **f**armer (🧑‍🌾/🚣)
* IN2: input signal **W** for the position of the **w**olf (🐺)
* IN3: input signal **G** for the position of the **g**oat (🐐)
* IN4: input signal **C** for the position of the **c**abbage (🥬)
* IN5: Planned to implement an easter egg
* IN6 and IN7: *not connected* because not required by the current design

All four input signals can be set by using an individual sliding switch.

### Output pins

The design uses 7 output pins for the 4 outut signals.

Why is that? The [TinyTapeout FAQ](https://docs.google.com/document/d/1HeUJ5RWxnGo36LE1jp5CoCfBO91wTzGANzlKC_vVfFI) states:

>Do I have to use the 7 segment?
>No, you can delete it and put whatever you want there. There’s lots of other components you can choose from the + menu. But if you get a PCB, it will only have the 7 segment on it. You’d need to plug the board into a breadboard and add your extra components after.

So I wanted to leave the option to use the 7 segment display on the final PCB - and the [Wokwi simulation](https://docs.wokwi.com/parts/wokwi-7segment).

The design uses the following output pins:

* OUT0+OUT3 (connected): output signal **~E**, i.e. the top and bottom segments light up, when the game is over ❌❌❌ due to an unattended situation on *any* river bank side
* OUT1+OUT2 (connected): output signal **~R** i.e. the top-right and bottom-right segments light up, to indicate an unattended situation on the *right* river bank (game over ❌)
* OUT4+OUT5 (connected): output signal **~L** i.e. the top-left and bottom-left segments light up, to indicate an unattended situation on the *left* river bank (game over ❌)
* OUT6: Planned to implement an easter egg
* OUT7: output signal **A** to light up the "dot LED" of the 7 segment display as an indicator that all objects have reached the right bank of the river and the game is won! 🎉🎉🎉


# Easter egg

There's a nice xkcd comic about the puzzle:

![xkcd_logic_boat](https://user-images.githubusercontent.com/6305922/187967352-cbd9ec3f-7463-44b8-8ff1-38ac7cb35613.png)

*(Source: https://xkcd.com/1134/ - This work is licensed under a Creative Commons Attribution-NonCommercial 2.5 License.)*

And there's right another one: https://xkcd.com/2348/


# Status/TODOs

I am a bit late to the party. I've started to think about the design on August, 31st - and submission deadline is already one day later on September, 1st.

☑️ Describe the design idea

☑️ Implement the design idea using Wokwi

☑️ Edit the [Makefile](Makefile) and change the WOKWI_PROJECT_ID to match the project. → It's here: https://wokwi.com/projects/341614346808328788

☑️ Enable GitHub Actions

☑️ Describe the signal mapping to the ASIC hardware I/O pins/ Wokwi user interface in the simulation

☑️ Hide easter egg (WIP)

☑️ Share your GDS on twitter, tag it #tinytapeout and [@matthewvenn](https://twitter.com/matthewvenn)!

☑️ [Submit it to be made](https://docs.google.com/forms/d/e/1FAIpQLSc3ZF0AHKD3LoZRSmKX5byl-0AzrSK8ADeh0DtkZQX0bbr16w/viewform?usp=sf_link)

☑️ [Join the community](https://discord.gg/rPK2nSjxy8)

🔲 Improve the implementation to work around the current limitations.

🔲 Improve the implementation to work around the hardware limitations (e.g. inputs should be de-bounced as mechanical switches are used).

🔲 ~Add more output signals (e.g. indicating if game was lost because 🐺-has-eaten-🐐 or 🐐-has-eaten-🥬). (Discarded for now as the user interface is slightly limited!)~

