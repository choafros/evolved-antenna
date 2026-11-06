# Evolving an antenna: my hands on genetic algorithm project

This is my practice project for learning genetic algorithms. I build one from scratch, plug it into an antenna simulator, and let it evolve a working antenna design on my own laptop. It is inspired by NASA's ST5 antenna, the first artificially evolved object to fly in space.

***

## Where the idea comes from: NASA's ST5 antenna

Between 2004 and 2006, a small team at NASA Ames (Greg Hornby, Al Globus, Derek Linden and Jason Lohn) had to design an antenna for Space Technology 5, a mission with three micro satellites studying Earth's magnetosphere. The antenna needed a wide circularly polarised radiation pattern and a wide impedance bandwidth across two X band frequencies. Hand designing something that did both turned out to be very hard.

So they let a computer evolve it.

The requirements were tight:

* Transmit frequency: 8470 MHz
* Receive frequency: 7209.125 MHz
* VSWR on transmit: under 1.2 to 1
* VSWR on receive: under 1.5 to 1
* Gain: at least 0 dBic for 40 to 80 degrees theta, at least minus 5 dBic for 0 to 40 degrees theta, right hand circular polarisation
* Input impedance: 50 ohms
* Volume: inside a cylinder 15.24 cm by 15.24 cm
* Mass: under 165 g

The program started with random wire shapes, kept the best ones, mutated and recombined them, and repeated that for hundreds of generations. The result was an odd, bent wire that no human would sketch, yet it beat the quadrifilar helix antenna the mission contractor had designed by hand. On a spacecraft mock up, two contractor antennas reached about 38% efficiency, one evolved plus one contractor antenna reached about 80%, and two evolved antennas reached about 93%.

The flight antenna (design ID ST5-33-142-7) launched on 22 March 2006 and worked perfectly. When the mission orbit changed during testing, the team adjusted the fitness function and evolved a new design in under a month, which would normally take a human designer much longer. Evolved antennas also flew later on NASA's LADEE lunar mission.

Sources:

* [Wikipedia: Evolved antenna](https://en.wikipedia.org/wiki/Evolved_antenna)
* [Hornby, Globus, Linden and Lohn: Evolution, Re-evolution, and Prototype of an X-band Antenna (PDF)](https://static.aminer.org/pdf/PDF/000/308/624/evolution_re_evolution_and_prototype_of_an_x_band_antenna.pdf)
* [NASA JPL: ST5 evolved antenna](https://www.jpl.nasa.gov/nmp/st5/TECHNOLOGY/antenna.html)
* [NASA NTRS: Computer-Automated Evolution of Spacecraft X-Band Antennas](https://ntrs.nasa.gov/citations/20100023362)
* [Space StackExchange: Did NASA's evolved antennas work?](https://space.stackexchange.com/questions/31453/did-nasas-borg-collective-designed-antennas-work-have-similar-designs-ever-bee)

The point: I can rebuild the whole pipeline (a genetic algorithm, an antenna simulator and a scoring function) on a laptop with free tools, then check the result with about £50 of hardware.

***

## What I will build and learn

By the end I will have:

1. A working genetic algorithm (GA) written from scratch in Python
2. An antenna simulator running inside the evolution loop (NEC2 through PyNEC)
3. A custom fitness function that captures what I want from the design
4. An evolved antenna for a real band, such as 2.4 GHz WiFi
5. Optionally, a physical copy tested with a NanoVNA and some copper wire

Along the way I will pick up evolutionary computation, antenna simulation, VSWR and impedance matching, radiation patterns and gain, circular polarisation, and how to shape an optimisation goal.

I'm starting with no antenna or GA background. Comfortable Python is enough.

***

## Quick start

```bash
# 1. Clone the repo
git clone https://github.com/yourusername/evolved_antenna.git
cd evolved_antenna

# 2. Create a virtual environment (recommended)
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate

# 3. Install dependencies
pip install -r requirements.txt   # deap, pynec, numpy, matplotlib

# 4. Run the toy GA first (Phase 1), which evolves the string "HELLO WORLD"
python src/phase1_ga_toy.py

# 5. Run the real thing and evolve a 2.4 GHz antenna
python src/phase3_evolve_antenna.py
```

***

## My roadmap

I will work through the phases in order. Each one is a milestone on its own.

### Phase 1: evolution without physics

Before touching antennas, I will build a GA that solves something I can check by eye. I will evolve the string "HELLO WORLD".

Concepts I will cover: population, genome, fitness, selection, crossover, mutation, generations, elitism.

```python
# The whole GA loop in a few lines of pseudocode
population = [random_genome() for _ in range(POP_SIZE)]
for generation in range(MAX_GENERATIONS):
    scored = [(fitness(g), g) for g in population]        # 1. evaluate
    scored.sort(reverse=True)                             # 2. best first
    population = [g for _, g in scored[:ELITE]]           # 3. elitism: keep the best
    while len(population) < POP_SIZE:                     # 4. breed the rest
        parent_a, parent_b = select_parents(scored), select_parents(scored)
        child = crossover(parent_a, parent_b)
        child = mutate(child, rate=MUTATION_RATE)
        population.append(child)
```

Milestone: my GA reliably finds "HELLO WORLD" in under about 200 generations. I will change the population size and mutation rate and watch how convergence changes. That is my first taste of GA tuning.

### Phase 2: antenna simulation basics

I will learn to simulate an antenna before I evolve one. I will model a straight wire monopole (a quarter wavelength whip) and check it against textbook results. That teaches me the simulator and gives me a sanity check that everything works.

Key ideas I need to understand:

* VSWR (voltage standing wave ratio) tells me how well the antenna matches my 50 ohm system. 1.0 is perfect and under 2.0 is fine for most hobby work. NASA needed under 1.2.
* S11, or return loss, says the same thing another way. It is what a NanoVNA measures directly.
* The radiation pattern is the gain (in dBi) in each direction (theta, phi).
* NEC (Numerical Electromagnetics Code) is a method of moments solver. NASA used NEC4, and NEC2 is the free version, which suits wire antennas well.

```python
# What simulating an antenna looks like with PyNEC
from pynec import NecContext

nec = NecContext()
nec.geometry.wire(...)   # define wire segments from (x1,y1,z1) to (x2,y2,z2), plus radius
nec.geometry.ground_plane(...)

nec.set_frequency(2.45e9)
nec.calculate_radiation_pattern(theta_count=37, phi_count=72)  # 5 degree steps
nec.calculate_impedance()

gain_db  = nec.pattern_gain_db       # gain at each (theta, phi)
z_in     = nec.z_in                  # complex input impedance
```

Milestone: I simulate a quarter wave monopole for 2.4 GHz (about 31 mm long). I confirm it resonates near 2.4 GHz and that VSWR dips there. Then I plot the gain pattern with matplotlib.

### Phase 3: evolve a wire antenna

Now the real thing. The genome describes the shape of a bent wire, and the fitness function scores how well the simulated antenna performs.

Genome design (the NASA approach): a list of segments, each with a length and some rotation angles. One gene looks like this:

```python
segment = {
    "length":   0.01,    # metres, range [0.005, 0.05]
    "radius":   0.001,   # wire radius (fixed in my simple version)
    "rot_x":    0.3,     # radians
    "rot_y":   -0.5,     # radians
    "rot_z":    0.1,     # radians
}
genome = [segment, segment, segment, segment]   # for example, a 4 segment bent wire
```

To build the antenna, I start at the feed point and apply the segments one after another. I draw a wire of the given length, rotate my pen by the angles, then draw the next one. NASA's winning genome worked the same way, as a tree of `forward(length, radius)` and `rotate_x/y/z(angle)` commands.

Fitness function: this is the heart of the project. NASA's flight run minimised a product of three terms. I will do the same, simplified for a 2.4 GHz goal:

```python
def fitness(genome) -> float:
    """Lower is better. Evolve toward VSWR under 2.0 and gain of at least
    2 dBi for 0 to 60 degrees theta, across 2.40 to 2.50 GHz."""
    try:
        sim = simulate(genome)          # PyNEC call, may fail on silly shapes
    except SimulationError:
        return 1e9                      # a broken antenna gets a terrible score

    # Term 1: VSWR penalty. Push hard above 2.0, no pressure below 1.5.
    # NASA used staged thresholds like this: a strong push above spec,
    # a soft push near it, and a floor below it, so evolution never wastes
    # effort over optimising one metric.
    vswr = 1.5 if sim.vswr < 1.5 else (sim.vswr + 3.0 * (sim.vswr - 2.0))

    # Term 2: gain shortfall against a 2 dBi target, summed over the target angles
    gain_penalty = sum(max(0, 2.0 - g) for g in sim.gain_over_target_angles)

    # Term 3 (optional): pattern smoothness, to penalise very lumpy patterns
    smoothness = sim.azimuthal_std

    return vswr * gain_penalty * smoothness
```

Constraints: I put these in the builder, not in the fitness function. Total height under 60 mm, total length under 100 mm, and no segment may cut through the ground plane. NASA limited the search to a monopole wire antenna for the same reason. A well chosen search space converges much faster.

Milestone: I run the evolution. I expect roughly 1 to 6 hours on my laptop, since each antenna takes a few seconds to simulate, just like NASA's runs. I plot the best fitness per generation and watch it fall as strange bent wire designs appear. Then I export the winner as a plot and as a buildable list of wire lengths and angles.

### Phase 4: tune it like a scientist

This is where I expect most of the learning to happen. I change one thing at a time and write down what I see.

* Population of 20, 50 and 200: the trade off between diversity and speed
* Mutation rate of 1%, 10% and 50%: random search against premature convergence
* Gaussian mutation sigma of 5%, 10% and 30% of range: fine tuning against exploring
* Elitism of 0%, 10% and 50%: never losing my best design against keeping diversity
* Rank against tournament selection: how much selection pressure I apply
* With and without the smoothness term: trade offs between several goals
* Shaped against flat VSWR penalty: fitness shaping, which was NASA's secret sauce

I keep a lab notebook in `experiments.md` and commit one plot per experiment.

Milestone: using my own data, I can explain why NASA chose a population of 50 to 200, a mutation rate near 1% with sigma at 10% of range, and 50% carry over.

### Phase 5: build it and measure it (optional)

NASA validated its antenna in an anechoic chamber. Here is my home version:

1. Buy a NanoVNA (about £50). It measures S11 directly, so VSWR comes straight out.
2. Buy about 1 m of 1 mm copper wire and an SMA connector or an SMA pigtail.
3. Bend the wire to match my evolved design. I print the design at 1:1 scale as a bending guide, or 3D print a former.
4. Measure S11 across 2.4 to 2.5 GHz and compare it with my simulation.
5. As a bonus, use an ESP8266 or ESP32 as a cheap RSSI based pattern checker. I rotate the antenna and log the received signal strength.

Milestone: measured VSWR within about 20% of the simulation. A gap between simulation and reality is normal. The exact wire radius, the connector and even my hand all affect the result. The gap is a lesson in itself, and it is why NASA cross checked with WIPL-D before building.

***

## Planned repo structure

```
evolved_antenna/
├── README.md                    <- this file
├── requirements.txt
├── src/
│   ├── phase1_ga_toy.py         <- toy GA ("HELLO WORLD")
│   ├── phase2_simulate_monopole.py
│   ├── phase3_evolve_antenna.py
│   ├── antenna_builder.py       <- genome to NEC wire segments
│   ├── fitness.py               <- the scoring function
│   └── ga.py                    <- reusable GA engine
├── experiments/
│   └── 001_baseline.md          <- my lab notebook entries
├── designs/
│   └── 20261106_best_genome.json
└── docs/
    └── notes.md                 <- what I learned
```

***

## Cheat sheet: the parameters that matter

Evolutionary parameters (NASA's flight run values in brackets):

* Population size (50 for the GA, 200 for the open ended EA): bigger means more diversity but slower generations
* Mutation rate (1%): the chance that each gene changes
* Mutation strength sigma (10% of the value range): how big each change is
* Selection (rank based, exponential scaling): how strongly the best designs dominate breeding
* Carry over, or elitism (50%): the share of the population kept between generations
* Stopping (100 generations, or 10 to 40 generations with no progress): when to give up
* Run time (6 to 10 hours): a few seconds per antenna simulation

Fitness function parameters, which is where the real tuning happens:

* VSWR penalty shaping: staged thresholds, with a strong push above spec, a soft push near it and a hard floor. NASA used strong above 1.5 and 1.2, soft below 1.25 and 1.15, and a floor at 1.1.
* Gain targets per angle: what counts as good enough in each direction
* Per elevation weights: NASA used weights up to 4.0 in the critical angle band
* Pattern smoothness term: NASA added it because the spacecraft spun at about 40 rpm, and a lumpy pattern causes signal fading
* Grid resolution: 5 degree steps. If it is too coarse, evolution learns to exploit the gaps between sample points.

Design space constraints: volume, mass, wire radius, symmetry, and no branching. NASA banned branching arms because they were hard to build accurately.

***

## What I will read next

Papers:

* Hornby, Globus, Linden and Lohn, *Evolution, Re-evolution, and Prototype of an X-band Antenna* ([PDF](https://static.aminer.org/pdf/PDF/000/308/624/evolution_re_evolution_and_prototype_of_an_x_band_antenna.pdf)). This is the main source for the fitness function and parameters here.
* Hornby et al., *Automated Antenna Design with Evolutionary Algorithms* ([AIAA paper](https://scispace.com/papers/automated-antenna-design-with-evolutionary-algorithms-2wwkqr0z2t))
* Lohn et al., *Computer-Automated Evolution of Spacecraft X-Band Antennas* ([NASA NTRS](https://ntrs.nasa.gov/citations/20100023362))

Tools:

* [PyNEC](https://github.com/tmolteno/python-necpp): Python bindings for NEC2
* [4NEC2](https://www.qsl.net/4nec2/): a free Windows NEC2 GUI, handy for learning simulation interactively
* [DEAP](https://deap.readthedocs.io/): a Python framework for evolutionary algorithms
* [openEMS](https://openems.de/): a free FDTD solver, if I outgrow NEC2
* [NanoVNA](https://nanovna.com/): a cheap vector network analyser for checking real hardware

Background reading:

* [Wikipedia: Evolved antenna](https://en.wikipedia.org/wiki/Evolved_antenna)
* [Wikipedia: Genetic algorithm](https://en.wikipedia.org/wiki/Genetic_algorithm)
* [Wikipedia: Standing wave ratio](https://en.wikipedia.org/wiki/Standing_wave_ratio)
* [NASA JPL: ST5 mission antenna page](https://www.jpl.nasa.gov/nmp/st5/TECHNOLOGY/antenna.html)
* [Space StackExchange: Did NASA's evolved antennas work?](https://space.stackexchange.com/questions/31453/did-nasas-borg-collective-designed-antennas-work-have-similar-designs-ever-bee), which includes the team's own account of the month long re-evolution

***

## Questions I expect to ask

**Do I need to know electromagnetics?** No. I need three numbers: VSWR (I want it low), gain in dBi (I want it high in my target directions) and resonant frequency (I want it in my band). The simulator works all three out and the GA optimises them.

**Why a wire antenna and not a patch antenna?** NEC2 solves wire antennas natively, they simulate in seconds, and they are cheap to build and test. I just bend some copper. I start where NASA started.

**How long does an evolution run take?** Roughly population size times generations times seconds per simulation. With a population of 100, 100 generations and about 2 s per simulation, that is 5 to 6 hours. I debug with a population of 50 and 50 generations, then do an overnight run.

**My evolved design looks like a tangled mess. Is that normal?** Yes, and that is expected. NASA's flight antenna looked like a paperclip that lost a fight. Evolution doesn't care what looks sensible, only what scores well.

**How close will my measured antenna be to the simulation?** I expect VSWR minima to shift by a few percent and gain to differ by a few dB. Connector effects, wire radius tolerances and nearby objects all play a part, and learning to read that gap is one of the best lessons in the project.

***

## Acknowledgements

This project builds on the work of the NASA Ames evolutionary computation group (Hornby, Globus, Linden and Lohn) and the ST5 mission team. The ST5 antenna was the first artificially evolved object flown in space.

Licensed under MIT. Do something cool with it.
