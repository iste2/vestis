# Vestis

Vestis is a digital wardrobe: it catalogues what a person owns and wants, and helps them combine it into outfits, plan trips and decide what to wear each day.

## Language

**Piece**:
Anything wearable catalogued in a wardrobe: clothing, shoes or accessories.
_Avoid_: Garment, item, article, clothing

**Wardrobe**:
The complete collection of pieces a user has catalogued, owned and wished alike.
_Avoid_: Closet, inventory

**Owned** / **Wished**:
The ownership status of a piece. Buying a wished piece changes its status to owned; it stays the same piece.
_Avoid_: Bought, purchased, desired

**Wishlist**:
The wished pieces of a wardrobe.
_Avoid_: Shopping list, favourites

**Starter pieces**:
The pieces every new wardrobe begins with: a white undershirt, white underpants and white socks. Apart from that they are ordinary owned pieces.
_Avoid_: Default pieces, basics

**Outfit**:
A set of pieces worn together at the same time. An outfit is identified by its pieces alone: the same pieces are the same outfit, whether generated, saved or worn.
_Avoid_: Look, combo, candidate, path

**Body zone**:
A region of the body a piece is worn on: head, neck, torso, hands, waist, legs or feet, plus "carried" for bags. A one-piece such as a dress covers several body zones.
_Avoid_: Slot, area, body part

**Layer**:
The position of a piece within a body zone, counted from the body outward. The torso has base (such as an undershirt), main, mid and two outer layers; the legs have base (underpants), hosiery (tights, leggings, long johns) and main; the feet have base (socks) and main.
_Avoid_: Level, tier

**Exclusion**:
A user's statement that two particular pieces never go together.
_Avoid_: Blacklist, incompatibility, forbidden pair

**Valid outfit**:
An outfit with at most one piece per body zone and layer and no exclusion between its pieces. Only valid outfits are generated, and each must also meet its brief; a user may still save and wear any outfit.
_Avoid_: Wearable outfit, complete outfit, legal outfit

**Brief**:
The requirements and preferences a search for outfits must meet, such as required body zones, the weather, the occasion, the wearer (for example rules that depend on gender), which pieces may be used and how different the results must be. By default a brief requires a main torso piece, a main legs piece (or a one-piece covering both) and shoes.
_Avoid_: Query, request, constraint set

**Suitability**:
How well a single piece suits the circumstances at hand, such as the weather or the occasion.
_Avoid_: Fit (that means how a piece sits on the body), context score

**Compatibility**:
How well two pieces go together, independent of any circumstances.
_Avoid_: Match, harmony, edge weight, outfit graph

**Wear**:
A record that a user wore an outfit on a given day. A user's wears together form their **wear history**.
_Avoid_: Log entry, usage, worn event

**Capsule wardrobe**:
A deliberately small selection of pieces chosen so that they combine into many valid outfits.
_Avoid_: Capsule collection, minimal wardrobe

**Trip plan**:
A capsule wardrobe assembled for a specific journey and its circumstances.
_Avoid_: Packing list, travel planner

**Daily recommendation**:
The outfits suggested to a user for a particular day.
_Avoid_: Outfit of the day, OOTD

**Avatar**:
A digital likeness of the user's body used for virtual try-on.
_Avoid_: Model, mannequin, profile picture

**Virtual try-on**:
Rendering pieces or an outfit on the user's avatar.
_Avoid_: Digital fitting, preview
