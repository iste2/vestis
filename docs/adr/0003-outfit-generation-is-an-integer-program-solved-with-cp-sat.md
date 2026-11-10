# Outfit generation is an integer program solved with OR-Tools CP-SAT

Finding the best outfits for a brief is posed as an integer program over the pieces of a wardrobe, not as an enumeration of valid outfits: one boolean per piece, one per pair of pieces that can be worn together, constraints from the valid-outfit rules and the brief, and a linear objective made of suitability, compatibility and the brief's per-zone weights. Every new kind of requirement (an outer layer below 5 °C, only capsule pieces, must include this piece) becomes a constraint or a weight instead of new search code. The solver is Google OR-Tools CP-SAT in Python, chosen for its strength on purely boolean problems.

## Consequences

- Suitability and compatibility are centred on zero (poor is negative, neutral is zero). Summed raw scores would reward filling every body zone, because each extra piece adds pairs; centred scores let an optional piece enter an outfit only when it contributes.
- The best K outfits come from re-solving with earlier solutions cut off; a "differs in at least d pieces" cut, with d from the brief, gives variety.
- Results are deterministic for the same wardrobe, brief and cached scores: fixed solver parameters and a tie-breaker by piece ID.
- Whole-outfit judgments by a decision model are not linear, so they only re-rank the K results afterwards. Counting valid outfits (for capsule wardrobes) is a separate operation, also driven by a brief.
- CP-SAT brings Python into the backend; where it runs is decided separately.

## Considered options

- **Enumerate valid outfits and rank them**: rejected; accessories and base layers multiply the count, and each new requirement needs its own pruning code.
- **HiGHS or GLPK inside the TypeScript backend**: avoids a second language, but CP-SAT and its Python API were preferred for this boolean, pairwise problem.
