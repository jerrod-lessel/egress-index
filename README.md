# Egress Index

**Some neighborhoods look like they have options. Close one street and dozens of blocks have nowhere to go.**

Egress Index maps how many separate ways out each intersection in Santa Rosa, California actually has. Not how many roads you can see from the sidewalk. How many genuinely independent escape routes exist once you follow every one of those roads to the end.

Then it lets you break the network and watch the way out rethink itself.

Live: https://egress-index.jerrod-lessel.workers.dev

Part of [Lessel Geospatial](https://lesselgeospatial.com).

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

**5. Measure the pocket.** A score of 1 does not say how much is stuck behind it. A cul-de-sac tip traps one spot, which is obvious and dull. A looping subdivision can trap dozens of intersections behind a single street, which is not obvious at all. So every single-egress spot also gets a pocket size: how many other spots share its trap, and the name of the street doing the trapping.

Pocket size, not the score, is what makes the map worth looking at.

**6. Route it live.** The scoring is precomputed, but the routing is not. The browser holds a graph of 6,749 nodes and 16,254 directed edges and runs A* against it, which finishes in well under a millisecond. That is what makes the hazards worth having.

## What it found

Santa Rosa, 5,718 intersections displayed, routed on a graph of 6,749 nodes and 16,254 directed edges. The extra nodes are a 1,500 m halo outside the city limits, which the math routes through but the map does not draw.

| Ways out | Count |
|---|---|
| One | 2,060 |
| Two | 1,475 |
| Three or more | 1,371 |
| Already on a safety road | 812 |

Of those 2,060 single-egress spots, 1,094 trap only themselves. Ordinary dead end tips, boring and correct. The other **966 have company behind the same street**, and those are the merge illusions this exists to find.

Ranked by how many intersections strand behind one street:

| Blocks trapped | Street |
|---|---|
| 49 | Chatham Drive |
| 38 | Stone Bridge Road |
| 25 | Northpoint Parkway |
| 25 | Summerfield Road |
| 25 | Terra Linda Drive |
| 22 | Village Parkway |
| 15 | San Ramon Way |
| 15 | La Mar Way |

Scattered across the whole city, east, west, and north. This is not a hills story. It is a subdivision design story.

## Break it yourself

The map ships with three ways to damage the network. All three do the same thing under the hood: they stamp a cost onto edges, and the router never asks which kind of hazard produced it.

- **Obstruction.** Close a specific street. The penalty is large but finite, so a boxed-in origin still returns a route, flagged in amber, rather than failing silently.
- **Fire.** A spreading footprint with a wind direction you drag to aim. Reach is yours to set.
- **Flood.** A volume of water, not a depth. You say how much got loose and the terrain decides where it goes. Roads close when water sits more than six inches over the lowest point of the pavement.

Elevation for the flood model comes from 3DEP, sampled at roughly 9 m spacing along each road rather than at its endpoints. That is the underpass catch: a road dipping under a rail line floods at the dip, and endpoint elevations miss it entirely.

## What this is not

**It is not live navigation.** Every input is precomputed from road geometry. It knows nothing about any current fire, any closure, any traffic. Do not evacuate by it.

**It is a planning and awareness tool.** The useful question it answers is structural: which places are built with one way out, and which street is holding them hostage. That question does not need real time data, which is convenient, because real time data is exactly what a zero backend map cannot have.

## Known limits

- **Roads only.** No slope, no fuel load, no road width, no traffic capacity. A wide arterial and a narrow lane count the same.
- **No population.** Pocket size counts intersections, not people or homes. A pocket of 49 intersections is not necessarily 49 households.
- **Safety is a road class, not a destination.** Reaching a secondary road counts as out. In a large enough fire, that is optimistic.
- **OpenStreetMap is crowd sourced.** Road classifications vary by whoever mapped that block.
- **Directed graph, one direction.** Scores use drivable direction, so one way streets count as one way. Realistic, but during an actual evacuation people drive on shoulders and against traffic. The model does not.
- **The flood model spends its water through an assumed corridor along the roads,** not across real terrain. It is a reasonable approximation at small volumes and stops being one at large ones, which is why the slider is capped at 40,000 m³ rather than left open. A terrain based version exists and is discussed below.

## Two bugs worth documenting

### The city boundary was eating neighborhoods

The first version clipped the road network at the Santa Rosa city limits, which is the obvious thing to do and is wrong.

A neighborhood whose only way out runs through unincorporated county before curving back into town reads as trapped, because the road it uses was cut off the edge of the data. The model was not finding fragility. It was finding the edge of its own extract.

The fix was to build on the city plus a 1,500 m halo, with boundary holes filled, and then score only the intersections meant for display. Compute wide, show narrow.

This changed the answer, not just the presentation. Bridgewood Drive was the single worst trap in the first run at 51 blocks. With the halo it stops being a trap at all, because the road it depends on leaves the city and comes back. Chatham Drive survived the rebuild at 49 and is now the worst in the city.

Any result that moves that much when you widen the frame was never a result. It was an artifact.

### Four stowaways in every pocket

The first pocket numbers were all inflated by 4.

Four nodes in the OSM extract had no roads leaving them at all, which is impossible in the real world and is just junk geometry. But because they could never reach safety, the minimum cut dumped them onto the trapped side of *every* computation. They rode along in every pocket like stowaways.

The tell was that zero cul-de-sac tips reported a pocket of 1. A true dead end traps exactly itself, so that number should have been large. It was zero, because every tip was reporting 3 or 5 instead. Today it is 1,094, which is what a healthy version of that number looks like.

## The notebooks

The full pipeline is in [`notebooks/`](notebooks/). GitHub renders them inline with their outputs, so they read end to end, including the run logs.

| Notebook | Produces |
|---|---|
| `01_egress_scoring.ipynb` | `egress_santa_rosa.geojson`, the index itself |
| `02_routing_graph.ipynb` | `graph_santa_rosa.json`, what the browser routes on |
| `03_elevation.ipynb` | `elev` and `emin` written back into that graph |

They run in order and need nothing but a free Colab session. Step 1 takes about 50 minutes, almost all of it in two loops: one max-flow per intersection, then one min-cut per trapped spot.

To move the whole project to a different city, change one line in step 1:

```python
place = "Santa Rosa, California, USA"
```

## Stack

No backend, no build step, no monthly cost.

- **Analysis:** Google Colab, OSMnx, NetworkX, py3dep
- **Data:** OpenStreetMap via OSMnx, USGS 3DEP elevation
- **Frontend:** MapLibre GL JS, single file, CARTO dark basemap, A* routing in the browser
- **Hosting:** Cloudflare, free tier
- **Output:** one GeoJSON and one graph JSON, sitting next to one HTML file

The pipeline stores inputs rather than outputs. Scores, pocket sizes, and street names live in the data. Colors and thresholds live in the map. Restyling never requires recomputing anything.

## What is in here but not used

`dem_santa_rosa.png`, `basins_santa_rosa.png` and `basins_santa_rosa.json` support a terrain based flood model that fills real depressions rather than an assumed corridor. It was built, verified against a measured benchmark, and deliberately not shipped: it is better physics attached to the third demonstration of a mechanism the tool already demonstrates twice.

The postmortem will probably live at [Null Island](https://lesselgeospatial.com) soon, I just have to find some time to write it up at some point. The files stay here because they cost nothing and the work was real (and I spent more time on them then I probably should have). 😂
