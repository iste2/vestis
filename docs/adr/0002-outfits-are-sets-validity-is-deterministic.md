# Outfits are sets of pieces; validity is deterministic, judgment is scored

An outfit is a set of pieces, identified by its pieces alone, not a path through a graph of what is worn over what. A path cannot hold trousers, shoes and a watch beside the torso's layering, and a sub-DAG would give the same outfit several representations. Whether an outfit is valid is decided by deterministic rules over each piece's body zone and layer and the user's exclusions. Decision models only propose a piece's body zone and layer at import (the user confirms) and score suitability, compatibility and whole outfits; they never decide validity, so generation stays deterministic and explainable. The rules bind generation only: a user may still save and wear an outfit that breaks them.

## Considered options

- **Outfit as a path** of "worn over" edges: rejected, it cannot represent pieces beside the layering chain.
- **Outfit as a sub-DAG** of pieces: rejected, the same outfit gets several representations.
- **Pairwise validity judged by a model** ("can this hoodie go under this jacket?"): rejected, it makes validity non-deterministic and hard to explain; soft pairwise judgment is covered by compatibility.
