# 🐻 Lucky Bear: Generative Collectibles Pipeline

A single-file Python pipeline that generates a collection of unique, layered PNG collectibles, with weighted rarity, compatibility rules, a duplicate check, per-image metadata, a CSV export and a rarity report.

> **Note:** This is a sample project I built to demonstrate my generative-collection workflow. The artwork is drawn in code with Pillow, so the project runs with no external assets. It is not client work.

![Preview sheet](docs/preview.png)

---

## Features

- **10 trait groups**: Background, Aura, Back, Body, Clothes, Eyes, Mouth, Face, Head, Items
- **Weighted rarity**: every trait has its own weight, so rare traits (Galaxy body, Gold background, Laser eyes) show up as rarely as you configure
- **Compatibility rules**: pairs of traits that must never appear together (for example gold on gold, or a hood clipping a cape)
- **Uniqueness guaranteed**: every combination is checked against a set of already-used combinations, so there are no duplicates
- **Correct layer order**: clothes sit on the torso but behind the head, held items sit behind the face, and hats are always on top
- **Logo on its own top layer**: nothing in the collection can cover it
- **Reproducible**: a fixed seed gives the same collection every run
- **Full output set**: PNGs, one metadata JSON per image, a CSV of the whole collection, a rarity report and preview sheets

## Quick start

```bash
pip install pillow
python lucky_bear_demo.py 200      # generate 200 unique bears
```

Everything is written to `./out`:

```
out/
├── images/              # 1.png, 2.png, ...
├── metadata/            # 1.json, 2.json, ... (one per image)
├── collection.csv       # every bear with traits, rarity score, rank and tier
├── rarity_report.txt    # trait distribution + top 10 rarest
├── preview.png          # sheet of the first 24 bears
└── preview_rarest.png   # sheet of the 12 rarest bears
```

Re-running deletes and rebuilds `./out`.

## How it works

### 1. Traits and weights

Each group in `TRAITS` is a list of `(name, weight, draw_function)`. A higher weight means a more common trait.

```python
"Body": [
    ("Brown",  30, body(...)),   # common
    ("Gold",    5, body(...)),   # rare
    ("Galaxy",  3, body(...)),   # rarest
]
```

### 2. Compatibility rules

Each rule is a pair that can never appear together:

```python
RULES = [
    (("Background", "Gold"), ("Body", "Gold")),       # gold on gold: invisible
    (("Head", "Wizard"),     ("Items", "Wand")),      # too much magic
    (("Back", "Cape"),       ("Clothes", "Hoodie")),  # hood clips the cape
]
```

### 3. Generation and uniqueness

`main()` rolls a random combination using the weights, then rejects it if it breaks a rule or has already been used. Only valid, unique combinations are rendered. The report prints the number of theoretical combinations (currently 52,684,800 before rules), so you can confirm the collection size is safely below what's available.

### 4. Layer order

The bottom-to-top paint order is defined in one list, `RENDER_ORDER`:

```
Background → Aura → Back → Body (base) → Clothes → Items
→ Body (head) → Eyes → Mouth → Face → Head accessories → Logo
```

The body is split into a `base` part (ears, torso, arms) and a `head` part. This lets clothes fit over the torso while staying behind the head, and keeps hats and the logo on top of everything.

### 5. Rarity

Each bear gets a score equal to the sum of `1 / frequency` of its traits across the collection. Bears are ranked by score and assigned a tier:

| Tier | Top |
|---|---|
| Legendary | 1% |
| Epic | 5% |
| Rare | 20% |
| Common | rest |

### Example metadata

```json
{
  "name": "Lucky Bear #42",
  "image": "42.png",
  "rarity_rank": 7,
  "rarity_score": 31.4,
  "attributes": [
    { "trait_type": "Background", "value": "Night" },
    { "trait_type": "Body", "value": "Panda" },
    { "trait_type": "Head", "value": "Crown" },
    { "trait_type": "Rarity Tier", "value": "Epic" }
  ]
}
```

## Using your own artwork

The drawn layers are placeholders. To use real layer files, replace a draw function with an image loader:

```python
("Happy", 20, lambda: Image.open("layers/Eyes/happy.png").convert("RGBA")),
```

All layer images must share the same canvas size and have transparent backgrounds. Because layers are composited in `RENDER_ORDER`, a layered PSD exported as one PNG per layer drops straight in.

## Configuration

| Setting | Where | Default |
|---|---|---|
| Output size | `S` | `512` (use `2000` for full-size art) |
| Edge smoothing | `K` | `2` (supersampling factor) |
| Logo text | `LOGO_TEXT` | `LUCKY BEAR \| demo.org` |
| Collection size | CLI argument | `100` |
| Seed | `main(seed=...)` | `7` |

## Known limits

- The collection size can't exceed the number of valid unique combinations. Asking for more than exist will loop forever, so keep `n` comfortably below the combinations figure.
- Output is currently saved as RGB PNG with an opaque background. Use RGBA if you need transparency.

## Scaling to a large production run

For a 10,000-image collection at 2000×2000, the same structure extends naturally with:

- traits and rules moved into a JSON/YAML config so non-developers can edit them
- layers loaded from folders instead of drawn in code
- `multiprocessing.Pool` for rendering
- a final verification step that confirms all images are unique and rule-compliant

## Requirements

- Python 3.8+
- [Pillow](https://pypi.org/project/Pillow/)

## License

MIT (or add your preferred license).
