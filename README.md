# Egress Index

**Some neighborhoods look like they have options. Close one street and dozens of blocks have nowhere to go.**

Egress Index maps how many separate ways out each intersection in Santa Rosa, California actually has. Not how many roads you can see from the sidewalk. How many genuinely independent escape routes exist once you follow every one of those roads to the end.

Live: https://egress-index.jerrod-lessel.workers.dev

Part of [Lessel Geospatial](https://lesselgeospatial.com). Someone has to map this mess.

---

## The problem

Stand in a subdivision and count the streets leaving your intersection. Three? Feels fine. You have options.

Follow them, though. In a lot of American suburban design, those three streets curve around, merge, and drain onto the same single collector road. They were never three options. They were one road wearing three costumes.

You cannot see this from a car. You cannot see it from a driveway. You can only see it by tracing every street to its terminus, which is exactly the sort of tedium a computer is good at.

## The question

> How many roads would someone have to close before this spot is stuck?

One means fragile. Three or more means fine. That number is the index.

## How it works

**1. Define "out."** Escaping is not about reaching any road. It is about reaching a road that carries you away: a highway or a major arterial. OpenStreetMap already classifies every road, so motorway, trunk, primary, and secondary get treated as safety roads. Everything else is a local street that only matters for getting you to one.

**2. Add an imaginary room.** Rather than asking "how many of the hundreds of highways can I reach," we invent a single node called the sink and wire every safety road into it. Now the question is "how many separate paths run from here to that one room."

**3. Turn roads into pipes.** Give every road a capacity of one unit. Turn on the tap at an intersection and measure how much water reaches the sink. Three units arriving means three genuinely independent routes. One unit means everything is squeezing through a single pipe.

**4. Count the cuts.** The amount of water that gets through always exactly equals the fewest pipes you would have to sever to stop it. This is max-flow min-cut, proven since the 1950s, and NetworkX ships with it. So one computation answers both "how many ways out" and "how many closures to trap this place."

**5. Measure the pocket.** A score of 1 does not say how much is stuck behind it. A cul-de-sac tip traps one spot, which is obvious and dull. A looping subdivision can trap fifty intersections behind a single street, which is not obvious at all. So every single-egress spot also gets a pocket size: how many other spots share its trap, and the name of the street doing the trapping.

Pocket size, not the score, is what makes the map worth looking at.

## What it found

Santa Rosa, 5,302 intersections, 13,210 road segments.

| Ways out | Count |
|---|---|
| One | 1,941 |
| Two | 1,355 |
| Three or more | 1,196 |
| Already on a safety road | 808 |

Of those 1,941 single-egress spots, 966 are ordinary dead end tips. Boring and correct. But 478 of them have three or more roads leaving and *still* score 1. Those are the merge illusions, and they are the reason this exists.

Ranked by how many blocks strand behind a single street:

| Blocks trapped | Street |
|---|---|
| 51 | Bridgewood Drive |
| 49 | Chatham Drive |
| 32 | Yulupa Avenue |
| 28 | Hall Road |
| 25 | Terra Linda Drive |
| 25 | Summerfield Road |
| 25 | Northpoint Parkway |
| 23 | Burt Street |

These are scattered across the whole city, east, west, and north. This is not a hills story. It is a subdivision design story.

## What this is not

**It is not live navigation.** Every input is precomputed from road geometry. It knows nothing about any current fire, any closure, any traffic. Do not evacuate by it.

**It is a planning and awareness tool.** The useful question it answers is structural: which places are built with one way out, and which street is holding them hostage. That question does not need real time data, which is convenient, because real time data is exactly what a zero backend map cannot have.

## Known limits

- **Roads only.** No slope, no fuel load, no road width, no traffic capacity. A wide arterial and a narrow lane count the same. Fixing that is Phase 2.
- **No population.** Pocket size counts intersections, not people or homes. A pocket of 51 intersections is not necessarily 51 households.
- **Safety is a road class, not a destination.** Reaching a secondary road counts as out. In a large enough fire, that is optimistic.
- **OpenStreetMap is crowd sourced.** Road classifications vary by whoever mapped that block.
- **Directed graph, one direction.** Scores use drivable direction, so one way streets count as one way. Realistic, but during an actual evacuation people drive on shoulders and against traffic. The model does not.

## One bug worth documenting

The first pocket numbers were all inflated by 4.

Four nodes in the OSM extract had no roads leaving them at all, which is impossible in the real world and is just junk geometry. But because they could never reach safety, the minimum cut dumped them onto the trapped side of *every* computation. They rode along in every pocket like stowaways.

The tell was that zero cul-de-sac tips reported a pocket of 1. A true dead end traps exactly itself, so that number should have been large. It was zero, because every tip was reporting 3 or 5 instead.

Evicting them made 966 cul-de-sacs appear exactly where they belonged, and Bridgewood Drive dropped from 53 to an honest 51. The finding survived. The number got smaller. That is usually how it goes.

## Stack

No backend, no build step, no monthly cost.

- **Analysis:** Google Colab, OSMnx, NetworkX
- **Data:** OpenStreetMap via OSMnx
- **Frontend:** MapLibre GL JS, single file, CARTO dark basemap
- **Hosting:** Cloudflare, free tier
- **Output:** one GeoJSON, sitting next to one HTML file

The pipeline stores inputs rather than outputs. Scores, pocket sizes, and street names live in the GeoJSON. Colors and thresholds live in the map. Restyling never requires recomputing anything.

## Rebuilding it

Open the Colab notebook and run the cells top to bottom. Change one line to move the whole project to a different city:

```python
place = "Santa Rosa, California, USA"
```

Scoring takes a few minutes. Pocket measurement takes a few more. Then download the GeoJSON, drop it next to `index.html`, and push.

## Roadmap

- **Phase 2:** draw a hazard polygon and watch the safe routes recalculate, with a user controlled wind direction as a live input
- Fuel load and slope from Google Earth Engine as additional cost layers
- Population weighting so pocket size means households, not intersections# egress-index
A project exploring escapability in an urban environment. 
