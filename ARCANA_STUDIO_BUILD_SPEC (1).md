# ARCANA STUDIO v2 — Dynamic Prompt System + Integration Spec

*AI-powered symbolic wisdom studio. Reference-guided, style-consistent, copyright-defensible.*

---

## 0. What changed in v2

1. **LoRA deepened into a full deck-cohesion engine** — style LoRA + character LoRA stacked, Deck DNA persistence, Reference Image Mode, Hidden Style Engine (32 directions), Aesthetic Packs, Smart Style Assistant, AI Prompt Optimizer, and post-generation Image Enhancement pipeline.
2. **Copyright exclusivity workflow** — human-curator step, provenance chain, C2PA metadata. Marketplace decks are defensible as human-authored creative works, not raw AI output.
3. **Philosophy layer default-on** — repositions the product from "AI tarot app" to "reflective wisdom studio." Every card ships with a philosophical reflection unless the user opts out.

---

## 1. Architecture at a glance

```
User selections + optional reference image
        ↓
[Smart Style Assistant]  →  recommends artistic direction
        ↓
[AI Prompt Optimizer]    →  cleans, deconflicts, injects hidden rules
        ↓
[Creation Mode Router]   →  reference / prompt / hybrid / inspire-me
        ↓
[Deck DNA merger]        →  applies persistent deck fingerprint
        ↓
FLUX Schnell / FLUX.2 Pro / FLUX LoRA (fal.ai)  →  card artwork
        ↓
[Image Enhancement chain]  →  upscale, face/hand fix, border, foil
        ↓                                     ↑
[Human Curator Queue]  (marketplace tier only)
        ↓
Claude Sonnet 4.6  →  meanings + philosophy + numerology + astrology
        ↓
Store card + C2PA metadata + provenance chain (Supabase)
        ↓
User: view / journal / export PDF / list to marketplace
```

Two AI providers, four generation modes, one deck. Every card is either fully machine-generated (personal use) or human-curated (marketplace tier).

---

## 2. Extended selection schema

```typescript
type CreationMode = "reference_only" | "prompt_only" | "reference_prompt" | "inspire_me";

type ArtisticDirection =
  | "luxury_tarot" | "golden_renaissance" | "dark_academia" | "cottagecore"
  | "art_nouveau" | "victorian_gothic" | "celestial_fantasy" | "botanical"
  | "ancient_egypt" | "greek_mythology" | "roman_mythology" | "norse"
  | "japanese_ukiyoe" | "chinese_ink" | "watercolour" | "oil_painting"
  | "impressionism" | "digital_painting" | "high_fantasy" | "mystic_realism"
  | "minimal_luxury" | "crystal_magic" | "nature_spirits" | "dreamcore"
  | "ethereal" | "divine_feminine" | "masculine_archetypes" | "sacred_geometry"
  | "cosmic" | "modern_graphic" | "vintage_illustration" | "childrens_storybook";

type AestheticPack =
  | "luxury" | "boho" | "modern" | "minimal" | "spiritual" | "dark"
  | "pastel" | "vintage" | "gold_foil" | "silver_foil" | "crystal"
  | "floral" | "botanical" | "forest" | "celestial" | "gothic"
  | "steampunk" | "cyber_mystic" | "royal" | "ancient" | "feminine"
  | "masculine" | "neutral";

type DeckSelection = {
  // ── Core selections ──────────────────────────────────────────
  deckType: "traditional_tarot" | "oracle" | "angel" | "lenormand"
          | "numerology" | "philosophy" | "astrology" | "hybrid";
  deckSize: "single" | "three" | "major_arcana_22" | "minor_arcana"
          | "full_78" | "custom_oracle";
  customCardCount?: number;

  // ── Creation mode ────────────────────────────────────────────
  creationMode: CreationMode;
  referenceImageUrl?: string;       // required for reference_* modes
  userPrompt?: string;              // required for prompt_* modes

  // ── Style engine ─────────────────────────────────────────────
  artisticDirection: ArtisticDirection;
  aestheticPack?: AestheticPack;

  // ── Reflective layers ────────────────────────────────────────
  // Philosophy is DEFAULT-ON per v2. This is the positioning wedge.
  philosophyEnabled: boolean;                       // default true
  philosophyLayer: PhilosophyTradition;             // default "stoic"
  numerologyEnabled: boolean;                       // default true
  astrologyEnabled: boolean;                        // default true
  chakraSystemEnabled: boolean;                     // default false

  // ── Character consistency ────────────────────────────────────
  characterConsistency: boolean;
  characterAnchor?: {
    archetype: string;        // "wise elder woman", "cloaked wanderer"
    facialProfile: string;    // "high cheekbones, olive skin, silver hair"
    signatureItems: string[]; // ["crescent pendant","embroidered mantle"]
  };

  // ── Deck DNA (populated after first successful card) ─────────
  deckDNA?: DeckDNA;

  // ── Publishing tier ──────────────────────────────────────────
  tier: "personal" | "creator" | "marketplace";
  // marketplace tier triggers the human-curation workflow

  // ── Personalization (for "deck from my Life Path") ───────────
  userBirthDate?: string;
  userFullName?: string;
};

type PhilosophyTradition =
  | "stoic" | "socratic" | "confucian" | "epicurean"
  | "aristotelian" | "platonic" | "hermetic" | "none";

// Frozen at first card. Every subsequent card of the deck inherits.
type DeckDNA = {
  seedCardIds: string[];
  frozenStyle: {
    brushwork: string;
    palette: string[];         // hex codes
    lightingRule: string;
    borderStyle: string;
    symbolMotifs: string[];
    aspectRatio: "2:3";
    typographySafeZone: { top: number; bottom: number };
    visualRhythm: string;
  };
  styleLoraUrl?: string;       // fal LoRA endpoint URL
  characterLoraUrl?: string;   // populated only if characterConsistency
  triggerWord: string;         // e.g. "arcana_deck_abc123"
};

type CardBrief = {
  cardName: string;
  cardNumber: number | string;
  suit?: "cups" | "swords" | "wands" | "pentacles" | null;
  archetype: string;
  coreSymbols: string[];
  element?: "water" | "fire" | "air" | "earth" | "spirit";
  planet?: string;
  zodiac?: string;
  numerology?: number;
};
```

**Defaults for a new DeckSelection:**

```typescript
export const DEFAULT_SELECTION: Partial<DeckSelection> = {
  creationMode: "prompt_only",
  philosophyEnabled: true,        // ← the v2 default
  philosophyLayer: "stoic",
  numerologyEnabled: true,
  astrologyEnabled: true,
  chakraSystemEnabled: false,
  characterConsistency: false,
  tier: "personal",
};
```

---

## 3. The Hidden Style Engine

Users never see prompt engineering. They pick from labelled directions. Behind each label is a rule bundle the app injects into the assembled prompt.

```typescript
type StyleRuleSet = {
  displayName: string;
  brushwork: string;
  paletteHint: string;
  lightingRule: string;
  compositionRule: string;
  symbolismRule: string;
  border: string;
  extras: string[];      // extra tokens joined into the prompt
  hardNegatives: string[]; // extra tokens for the negative prompt
};

export const STYLE_ENGINE: Record<ArtisticDirection, StyleRuleSet> = {
  luxury_tarot: {
    displayName: "Luxury Tarot",
    brushwork: "polished editorial illustration with fine ornamental detail",
    paletteHint: "champagne gold, deep emerald, ivory, midnight",
    lightingRule: "soft directional key light with warm rim glow",
    compositionRule: "centred hero with symmetrical framing and negative space",
    symbolismRule: "restrained iconic symbolism, one dominant motif per card",
    border: "engraved gold border, subtle guilloche pattern",
    extras: ["museum quality", "editorial fashion sensibility", "print ready"],
    hardNegatives: ["cluttered", "childish", "cartoonish"],
  },
  golden_renaissance: {
    displayName: "Golden Renaissance",
    brushwork: "classical oil painting with glazed layers",
    paletteHint: "warm gold, rich crimson, ivory, umber",
    lightingRule: "soft volumetric chiaroscuro from upper left",
    compositionRule: "classical proportions, sacred geometry underlay",
    symbolismRule: "iconographic symbolism grounded in Renaissance tradition",
    border: "decorative gilded border with subtle floral motifs",
    extras: ["museum quality detailing", "no modern objects", "no visible text"],
    hardNegatives: ["contemporary clothing", "modern architecture", "digital effects"],
  },
  dark_academia: {
    displayName: "Dark Academia",
    brushwork: "detailed engraving with cross-hatched shading",
    paletteHint: "aged parchment, sepia, muted forest, oxblood",
    lightingRule: "low-key candlelight and warm library glow",
    compositionRule: "scholarly framing, books, manuscripts, quiet symbolism",
    symbolismRule: "academic and hermetic symbolism, quill, ink, sealed letters",
    border: "antique engraved border with corner ornaments",
    extras: ["aged texture", "Victorian scholarly atmosphere"],
    hardNegatives: ["bright saturated colour", "modern clothing", "flat lighting"],
  },
  cottagecore: {
    displayName: "Cottagecore",
    brushwork: "warm hand-painted illustration with visible texture",
    paletteHint: "sage, buttercream, blush, terracotta",
    lightingRule: "hearth glow and dappled afternoon light",
    compositionRule: "layered natural elements, cosy interior or garden setting",
    symbolismRule: "botanicals, bread, pottery, embroidery motifs",
    border: "embroidered floral border",
    extras: ["hand-embroidered feel", "wildflowers", "linen texture"],
    hardNegatives: ["urban setting", "digital sheen", "hard geometric lines"],
  },
  celestial_fantasy: {
    displayName: "Celestial Fantasy",
    brushwork: "richly painted with fine glowing highlights",
    paletteHint: "midnight blue, silver, starlight, cosmic violet",
    lightingRule: "moonlight with subtle rim glow and starfield ambience",
    compositionRule: "hero framed by constellations and sacred geometry",
    symbolismRule: "stars, nebulae, moons, sacred geometry, flowing fabrics",
    border: "constellation-etched border with subtle gold foil",
    extras: ["gold foil highlights", "soft mystical glow", "floating fabrics"],
    hardNegatives: ["earthly urban", "modern tech", "flat matte"],
  },
  art_nouveau: {
    displayName: "Art Nouveau",
    brushwork: "flowing Mucha-inspired ornamental line work",
    paletteHint: "muted gold, sage, dusty rose, ivory",
    lightingRule: "diffuse ambient light with decorative halo",
    compositionRule: "ornate flowing composition with botanical framing",
    symbolismRule: "botanical borders, decorative halos, flowing hair and fabric",
    border: "ornate art nouveau botanical border",
    extras: ["Mucha-inspired", "decorative panels", "stylised natural forms"],
    hardNegatives: ["photorealism", "harsh contrast", "digital rendering"],
  },
  botanical: {
    displayName: "Botanical",
    brushwork: "herbarium plate illustration with fine detail",
    paletteHint: "vellum, forest, moss, faded rose",
    lightingRule: "flat diffuse daylight, catalogue plate feel",
    compositionRule: "specimen-style layout with labelled elements",
    symbolismRule: "identifiable botanicals, pressed flower motifs",
    border: "vellum edge with subtle pressed flower border",
    extras: ["botanical illustration style", "herbarium plate"],
    hardNegatives: ["fantasy plants", "glowing effects"],
  },
  ancient_egypt: {
    displayName: "Ancient Egypt",
    brushwork: "temple mural style with fine gold detail",
    paletteHint: "lapis, gold, ochre, deep umber",
    lightingRule: "warm sunlit temple glow with deep shadow",
    compositionRule: "hieratic frontal composition with hieroglyphic border",
    symbolismRule: "ankh, eye of Horus, scarab, lotus, papyrus",
    border: "hieroglyphic cartouche border",
    extras: ["temple mural aesthetic", "lapis and gold"],
    hardNegatives: ["Greco-Roman elements", "modern anachronism"],
  },
  greek_mythology: {
    displayName: "Greek Mythology",
    brushwork: "classical Grecian illustration with marble detail",
    paletteHint: "marble white, aegean blue, olive, oxidised bronze",
    lightingRule: "Mediterranean sunlight with cool marble shadows",
    compositionRule: "classical contrapposto or heroic frontal pose",
    symbolismRule: "olive branch, lyre, laurel, amphora, sacred flame",
    border: "meander (Greek key) border in oxidised bronze",
    extras: ["classical Greek atmosphere", "temple architecture"],
    hardNegatives: ["Roman-specific iconography", "Norse elements"],
  },
  norse: {
    displayName: "Norse",
    brushwork: "carved wood and rune-etched detail",
    paletteHint: "cold silver, iron, deep pine, snow white",
    lightingRule: "cold northern light with long shadows",
    compositionRule: "vertical worldtree composition or knotwork frame",
    symbolismRule: "runes, ravens, worldtree, longships, wolves",
    border: "carved knotwork border with runic corners",
    extras: ["Norse rune motifs", "carved wood texture"],
    hardNegatives: ["Mediterranean warm palette", "tropical elements"],
  },
  japanese_ukiyoe: {
    displayName: "Japanese Ukiyo-e",
    brushwork: "traditional Japanese woodblock print",
    paletteHint: "indigo, vermillion, ivory, ink black",
    lightingRule: "flat woodblock lighting with dramatic silhouette",
    compositionRule: "asymmetric composition with strong diagonals",
    symbolismRule: "cherry blossoms, torii, koi, waves, cranes",
    border: "cloud pattern border in ink",
    extras: ["Hokusai-inspired", "gold leaf accents", "ink line"],
    hardNegatives: ["western realism", "digital gradient"],
  },
  chinese_ink: {
    displayName: "Chinese Ink",
    brushwork: "classical Chinese brush painting with expressive strokes",
    paletteHint: "vermillion, jade, ivory, ink black",
    lightingRule: "atmospheric misty light",
    compositionRule: "vertical scroll composition with generous negative space",
    symbolismRule: "dragon, phoenix, lotus, bamboo, plum blossom",
    border: "seal-stamped border with subtle scroll edge",
    extras: ["Chinese calligraphic brushwork", "seal chops"],
    hardNegatives: ["Japanese-specific motifs", "western realism"],
  },
  watercolour: {
    displayName: "Watercolour",
    brushwork: "loose watercolour with granulation and wet-on-wet blooms",
    paletteHint: "soft washes chosen by aesthetic pack",
    lightingRule: "diffuse natural light on paper",
    compositionRule: "loose organic composition with paper white breathing space",
    symbolismRule: "gentle symbolic elements, botanical accents",
    border: "torn paper edge or gentle wash border",
    extras: ["visible paper texture", "watercolour bloom"],
    hardNegatives: ["hard vector lines", "digital sheen"],
  },
  oil_painting: {
    displayName: "Oil Painting",
    brushwork: "classical oil with visible impasto and glaze layers",
    paletteHint: "warm earth tones with jewel accents",
    lightingRule: "Rembrandt or Vermeer-style directional light",
    compositionRule: "classical triangular composition with focused hero",
    symbolismRule: "traditional iconographic symbols",
    border: "gilded classical frame edge",
    extras: ["museum oil painting", "glazed layers"],
    hardNegatives: ["cartoon", "flat digital shading"],
  },
  impressionism: {
    displayName: "Impressionism",
    brushwork: "loose impressionist brushstrokes with broken colour",
    paletteHint: "en plein air natural palette",
    lightingRule: "natural outdoor light with atmospheric colour shifts",
    compositionRule: "atmospheric composition with soft edges",
    symbolismRule: "symbolism dissolved into atmospheric feeling",
    border: "soft feathered border edge",
    extras: ["Monet-adjacent", "broken colour technique"],
    hardNegatives: ["sharp detail", "hyperreal precision"],
  },
  digital_painting: {
    displayName: "Digital Painting",
    brushwork: "polished digital painting with clean rendering",
    paletteHint: "cinematic contemporary palette",
    lightingRule: "cinematic key/fill lighting",
    compositionRule: "concept art composition with clear focal hierarchy",
    symbolismRule: "modern reinterpretation of classic symbols",
    border: "minimal contemporary border",
    extras: ["concept art quality"],
    hardNegatives: ["traditional media texture", "canvas grain"],
  },
  high_fantasy: {
    displayName: "High Fantasy",
    brushwork: "richly detailed high-fantasy illustration",
    paletteHint: "jewel tones, gold, deep forest, mystical purple",
    lightingRule: "dramatic magical light with glowing accents",
    compositionRule: "heroic composition with mythic scale",
    symbolismRule: "swords, staves, dragons, sacred flames, ancient sigils",
    border: "ornate mythic border with gemstone corners",
    extras: ["epic fantasy atmosphere"],
    hardNegatives: ["mundane", "modern setting"],
  },
  mystic_realism: {
    displayName: "Mystic Realism",
    brushwork: "photoreal with subtle surreal mystical touches",
    paletteHint: "grounded natural palette with mystical accent",
    lightingRule: "natural light with subtle otherworldly glow",
    compositionRule: "grounded realistic composition with symbolic element",
    symbolismRule: "one clear mystical symbol in an otherwise realistic scene",
    border: "subtle understated border",
    extras: ["photoreal detail", "single mystical accent"],
    hardNegatives: ["cartoon", "overtly fantasy"],
  },
  minimal_luxury: {
    displayName: "Minimal Luxury",
    brushwork: "clean editorial illustration with generous negative space",
    paletteHint: "cream, matte gold, deep black, single accent",
    lightingRule: "even editorial lighting",
    compositionRule: "extreme negative space with single hero element",
    symbolismRule: "one iconic symbol only",
    border: "hairline metallic border",
    extras: ["editorial minimal", "matte gold accent"],
    hardNegatives: ["busy", "cluttered", "ornate"],
  },
  crystal_magic: {
    displayName: "Crystal Magic",
    brushwork: "prismatic detailed illustration with iridescent detail",
    paletteHint: "amethyst, rose quartz, clear quartz, opal",
    lightingRule: "prismatic light refractions and inner glow",
    compositionRule: "crystalline geometric composition",
    symbolismRule: "crystals, geodes, sacred geometry, prisms",
    border: "faceted crystal border",
    extras: ["iridescent", "prismatic light"],
    hardNegatives: ["dull matte", "muted tones"],
  },
  nature_spirits: {
    displayName: "Nature Spirits",
    brushwork: "organic painterly with mossy texture",
    paletteHint: "moss green, bark brown, mushroom rust, sky",
    lightingRule: "dappled forest light",
    compositionRule: "spirit emerging from natural form",
    symbolismRule: "trees, mushrooms, animals, folk spirits",
    border: "living vine border",
    extras: ["folk spirit atmosphere"],
    hardNegatives: ["urban", "industrial"],
  },
  dreamcore: {
    displayName: "Dreamcore",
    brushwork: "soft dreamlike illustration with hazy edges",
    paletteHint: "pastel dream palette",
    lightingRule: "soft diffuse dream light with faint glow",
    compositionRule: "impossible dream composition",
    symbolismRule: "surreal symbols, floating elements",
    border: "soft cloud-like border",
    extras: ["dreamy hazy atmosphere"],
    hardNegatives: ["harsh reality", "sharp geometric"],
  },
  ethereal: {
    displayName: "Ethereal",
    brushwork: "softly luminous painting with glowing edges",
    paletteHint: "pale luminous palette",
    lightingRule: "otherworldly luminous glow throughout",
    compositionRule: "floating weightless composition",
    symbolismRule: "wings, halos, veils, light rays",
    border: "faint luminous border",
    extras: ["angelic luminous"],
    hardNegatives: ["grounded heavy", "dark shadow"],
  },
  divine_feminine: {
    displayName: "Divine Feminine",
    brushwork: "flowing painterly illustration honouring feminine archetype",
    paletteHint: "rose, gold, cream, deep red",
    lightingRule: "soft embracing light",
    compositionRule: "flowing composition emphasising nurture or sovereignty",
    symbolismRule: "moon, chalice, rose, serpent, womb, flame",
    border: "flowing floral border",
    extras: ["sacred feminine archetype"],
    hardNegatives: ["objectifying", "sexualised"],
  },
  masculine_archetypes: {
    displayName: "Masculine Archetypes",
    brushwork: "strong architectural painting honouring masculine archetype",
    paletteHint: "deep blue, iron, gold, oak",
    lightingRule: "clear strong directional light",
    compositionRule: "grounded architectural composition",
    symbolismRule: "sword, oak, mountain, hammer, sacred flame",
    border: "geometric architectural border",
    extras: ["sacred masculine archetype"],
    hardNegatives: ["aggressive", "toxic masculine tropes"],
  },
  sacred_geometry: {
    displayName: "Sacred Geometry",
    brushwork: "precise geometric line work over painted underlay",
    paletteHint: "cosmic palette with metallic geometry",
    lightingRule: "even light letting geometry lead",
    compositionRule: "geometric mandala framing hero",
    symbolismRule: "Flower of Life, Metatron's cube, sri yantra",
    border: "sacred geometric border",
    extras: ["mathematically precise geometry"],
    hardNegatives: ["random asymmetry"],
  },
  cosmic: {
    displayName: "Cosmic",
    brushwork: "richly painted cosmic scene with fine star detail",
    paletteHint: "nebula purples, deep blue, gold star",
    lightingRule: "starfield ambience with nebula glow",
    compositionRule: "hero framed against galactic backdrop",
    symbolismRule: "planets, comets, constellations, orbits",
    border: "constellation border",
    extras: ["cosmic nebula atmosphere"],
    hardNegatives: ["daylight earth", "urban"],
  },
  modern_graphic: {
    displayName: "Modern Graphic",
    brushwork: "flat modern vector illustration",
    paletteHint: "confident modern palette",
    lightingRule: "flat design without lighting",
    compositionRule: "bold graphic composition",
    symbolismRule: "modernised iconic symbols",
    border: "clean geometric border",
    extras: ["flat vector", "editorial modern"],
    hardNegatives: ["photoreal", "painterly texture"],
  },
  vintage_illustration: {
    displayName: "Vintage Illustration",
    brushwork: "mid-century printed illustration with limited palette",
    paletteHint: "muted vintage palette",
    lightingRule: "flat printed lighting",
    compositionRule: "vintage poster composition",
    symbolismRule: "vintage iconic reinterpretation",
    border: "printed vintage border",
    extras: ["screen-print texture", "aged paper"],
    hardNegatives: ["hyperreal", "digital gradient"],
  },
  childrens_storybook: {
    displayName: "Children's Storybook",
    brushwork: "warm rounded storybook illustration",
    paletteHint: "warm friendly palette",
    lightingRule: "friendly warm light",
    compositionRule: "warm rounded composition",
    symbolismRule: "gentle age-appropriate symbols",
    border: "playful decorative border",
    extras: ["storybook warmth"],
    hardNegatives: ["dark", "gothic", "adult themes"],
  },
  victorian_gothic: {
    displayName: "Victorian Gothic",
    brushwork: "detailed engraved Victorian illustration",
    paletteHint: "oxblood, black, aged brass, ivory",
    lightingRule: "gas-lamp low key lighting",
    compositionRule: "vertical gothic architectural framing",
    symbolismRule: "raven, skeleton key, hourglass, mourning veil",
    border: "gothic arch border with mourning motifs",
    extras: ["Victorian mourning aesthetic"],
    hardNegatives: ["bright cheerful", "modern"],
  },
  roman_mythology: {
    displayName: "Roman Mythology",
    brushwork: "Roman fresco style with warm plaster tones",
    paletteHint: "terracotta, ochre, laurel green, ivory",
    lightingRule: "Mediterranean fresco light",
    compositionRule: "Roman narrative frieze composition",
    symbolismRule: "laurel wreath, eagle, sword, olive, torch",
    border: "Roman fresco border",
    extras: ["Pompeian fresco", "Roman iconography"],
    hardNegatives: ["Greek-specific icons", "Norse elements"],
  },
};
```

**Aesthetic Packs** apply softer overlays — palette, ornamentation, texture — that stack on top of the artistic direction. They tune the *feel* while the direction sets the *style*.

```typescript
export const AESTHETIC_PACKS: Record<AestheticPack, string> = {
  luxury: "matte gold accents, ivory highlights, restrained ornament, rich texture",
  boho: "layered fabric texture, earth tones, tassels, sunfaded palette",
  modern: "clean editorial layout, contemporary palette",
  minimal: "extreme negative space, single accent element",
  spiritual: "soft glow, sacred symbolism gently integrated",
  dark: "low-key lighting, deep shadow, moody atmosphere",
  pastel: "soft pastel palette shift, gentle mood",
  vintage: "aged paper texture, faded edges, sepia undertone",
  gold_foil: "gold foil highlights on ornament and border",
  silver_foil: "silver foil highlights on ornament and border",
  crystal: "iridescent crystalline accents, prismatic light",
  floral: "layered floral botanicals in composition and border",
  botanical: "herbarium-plate botanicals, restrained composition",
  forest: "mossy forest atmosphere, dappled light",
  celestial: "starfield and moon accents, cool night ambience",
  gothic: "gothic architectural framing, sombre atmosphere",
  steampunk: "brass fittings, clockwork elements",
  cyber_mystic: "neon sigils, holographic mysticism",
  royal: "regal ornament, deep jewel palette",
  ancient: "aged patina, archaeological weathering",
  feminine: "flowing forms, softer palette, receptive symbols",
  masculine: "architectural forms, grounded palette, directive symbols",
  neutral: "no additional overlay",
};
```

---

## 4. Reference Image Analyzer

When the user uploads a reference image, Claude Sonnet 4.6 vision extracts style parameters *without copying protected work.*

```typescript
async function analyzeReferenceImage(
  imageUrl: string
): Promise<ReferenceAnalysis> {
  const resp = await anthropic.messages.create({
    model: "claude-sonnet-4-6",
    max_tokens: 800,
    messages: [{
      role: "user",
      content: [
        { type: "image", source: { type: "url", url: imageUrl } },
        {
          type: "text",
          text: `You are a visual style analyst for a tarot studio. Analyse the uploaded image and return strict JSON describing its STYLE parameters ONLY — never its subject matter or identifiable protected elements. We use this to inform our own original artwork.

Do not describe:
- specific characters or identifiable people
- brand logos or trademarks
- text or typography from the image
- any element that would replicate the image's protected content

Do describe:
- lighting direction and quality
- colour harmony and dominant hex-adjacent palette
- brushwork or rendering technique
- compositional structure (rule of thirds, centred, etc.)
- mood and atmosphere
- texture qualities
- visual hierarchy

Return JSON only:
{
  "lighting": string,
  "palette": [string, string, string, string, string],
  "brushwork": string,
  "composition": string,
  "mood": string,
  "texture": string,
  "hierarchy": string,
  "recommendedArtisticDirection": string,
  "recommendedAestheticPack": string
}`,
        },
      ],
    }],
  });
  return JSON.parse((resp.content[0] as any).text);
}

type ReferenceAnalysis = {
  lighting: string;
  palette: string[];
  brushwork: string;
  composition: string;
  mood: string;
  texture: string;
  hierarchy: string;
  recommendedArtisticDirection: ArtisticDirection;
  recommendedAestheticPack: AestheticPack;
};
```

The recommendation fields let the UI auto-select style dropdowns after upload — the user sees "We suggest **Golden Renaissance** with **Luxury** pack" and can confirm or override.

---

## 5. Smart Style Assistant

For prompt-only mode. Reads the user's freeform description and recommends 3–5 artistic directions with reasoning.

```typescript
async function recommendStyles(userPrompt: string): Promise<StyleRecommendation[]> {
  const resp = await anthropic.messages.create({
    model: "claude-sonnet-4-6",
    max_tokens: 600,
    messages: [{
      role: "user",
      content: `A tarot studio user described their vision:

"${userPrompt}"

From this list of artistic directions, pick the 3-5 that best fit their vision. For each, give a one-sentence reason.

${Object.entries(STYLE_ENGINE).map(([k,v]) => `- ${k}: ${v.displayName}`).join("\n")}

Return JSON only:
[
  { "direction": string, "reason": string },
  ...
]`,
    }],
  });
  return JSON.parse((resp.content[0] as any).text);
}

type StyleRecommendation = {
  direction: ArtisticDirection;
  reason: string;
};
```

---

## 6. AI Prompt Optimizer

Runs before every image generation. Cleans contradictions, validates length, injects print-quality tokens, applies default negatives.

```typescript
function optimizePrompt(assembled: string, model: "schnell" | "flux2_pro" | "lora"): string {
  // 1. Strip contradictions (e.g. "photorealistic" + "cartoon")
  const contradictions: [RegExp, string][] = [
    [/photorealistic.*cartoon|cartoon.*photorealistic/gi, "polished illustration"],
    [/minimal.*hyper.?detailed|hyper.?detailed.*minimal/gi, "richly detailed"],
    [/flat.*volumetric|volumetric.*flat/gi, "well-lit"],
  ];
  let cleaned = assembled;
  for (const [pattern, replacement] of contradictions) {
    cleaned = cleaned.replace(pattern, replacement);
  }

  // 2. Deduplicate repeated phrases (assembler can double up)
  const seen = new Set<string>();
  cleaned = cleaned
    .split(/[.,;]\s+/)
    .filter(fragment => {
      const key = fragment.trim().toLowerCase();
      if (!key || seen.has(key)) return false;
      seen.add(key);
      return true;
    })
    .join(". ");

  // 3. Model-specific length caps
  const MODEL_LIMITS = { schnell: 512, flux2_pro: 4000, lora: 512 };
  const limit = MODEL_LIMITS[model];
  if (cleaned.length > limit) {
    cleaned = cleaned.slice(0, limit).replace(/\s+\S*$/, "");
  }

  return cleaned;
}

export const DEFAULT_NEGATIVE_PROMPT = [
  "text", "watermark", "signature", "logo",
  "extra fingers", "extra limbs", "malformed hands", "malformed faces",
  "low resolution", "blurry", "jpeg artifacts",
  "duplicate objects", "cropped subject", "cluttered composition",
  "uneven border", "cheap clipart", "oversaturated", "garish",
  "brand names", "trademarks", "identifiable celebrities",
].join(", ");
```

---

## 7. The Image Prompt Assembler (v2)

Full assembler now integrates style engine, aesthetic pack, reference analysis, Deck DNA, and creation mode.

```typescript
export function assembleImagePrompt(
  deck: DeckSelection,
  card: CardBrief,
  referenceAnalysis?: ReferenceAnalysis
): { prompt: string; negativePrompt: string } {

  const style = STYLE_ENGINE[deck.artisticDirection];
  const pack = deck.aestheticPack ? AESTHETIC_PACKS[deck.aestheticPack] : null;
  const dna = deck.deckDNA?.frozenStyle;

  // ── Layer 1 · Meta rendering rules ──────────────────────────
  const META = `A museum-quality ${deck.deckType.replace(/_/g, " ")} card illustration. Portrait orientation, 2:3 aspect, print-ready with subtle inner border and card frame. High resolution, clean composition, single hero subject centred.`;

  // ── Layer 2 · Style engine (hidden rules) ───────────────────
  const STYLE = `${style.brushwork}. ${style.lightingRule}. ${style.compositionRule}. ${style.symbolismRule}. Border: ${style.border}. ${style.extras.join(", ")}.`;

  // ── Layer 3 · Aesthetic pack overlay ────────────────────────
  const PACK = pack ? `Aesthetic overlay: ${pack}.` : "";

  // ── Layer 4 · Reference-derived cues (if reference mode) ────
  const REF = referenceAnalysis
    ? `Reference cues: lighting ${referenceAnalysis.lighting}; palette ${referenceAnalysis.palette.join(", ")}; mood ${referenceAnalysis.mood}; texture ${referenceAnalysis.texture}. (Style influence only — original composition and subject.)`
    : "";

  // ── Layer 5 · Deck DNA (if this is card 2+) ─────────────────
  const DNA = dna
    ? `Deck DNA (must match): brushwork ${dna.brushwork}; palette ${dna.palette.join(", ")}; lighting ${dna.lightingRule}; border ${dna.borderStyle}; recurring motifs ${dna.symbolMotifs.join(", ")}; visual rhythm ${dna.visualRhythm}.`
    : "";

  // ── Layer 6 · Character anchor (if enabled) ─────────────────
  const CHARACTER = deck.characterConsistency && deck.characterAnchor
    ? `Recurring character: ${deck.characterAnchor.archetype}, ${deck.characterAnchor.facialProfile}, signature items: ${deck.characterAnchor.signatureItems.join(", ")}. This character must appear consistently across the deck.`
    : "";

  // ── Layer 7 · Card-specific symbolism ───────────────────────
  const symbolsLine = card.coreSymbols.join(", ");
  const numerologyLine = deck.numerologyEnabled && card.numerology
    ? `Numerological resonance of ${card.numerology}: subtle symbolic references.`
    : "";
  const astrologyLine = deck.astrologyEnabled && (card.zodiac || card.planet)
    ? `Astrological correspondence: ${[card.zodiac, card.planet].filter(Boolean).join(" and ")} — include a subtle glyph in the border.`
    : "";
  const elementLine = card.element ? `Elemental atmosphere: ${card.element}.` : "";

  const CARD = `Card: ${card.cardName}${card.cardNumber !== undefined ? ` (Number ${card.cardNumber})` : ""}. Archetype: ${card.archetype}. Include the following symbols: ${symbolsLine}. ${elementLine} ${numerologyLine} ${astrologyLine}`.trim();

  // ── Layer 8 · Quality directives ────────────────────────────
  const QUALITY = "Highly detailed, professionally composed, symmetrical framing, ornate but restrained border, portrait orientation, no cropping of hands or faces, no text unless explicitly requested, print-ready quality.";

  // ── Layer 9 · Negative prompt ───────────────────────────────
  const NEGATIVE = [
    DEFAULT_NEGATIVE_PROMPT,
    ...style.hardNegatives,
  ].join(", ");

  const raw = [META, STYLE, PACK, REF, DNA, CHARACTER, CARD, QUALITY]
    .filter(Boolean)
    .join(" ");

  // Run through optimizer before returning
  const model = deck.deckDNA?.styleLoraUrl ? "lora"
              : deck.tier === "marketplace" ? "flux2_pro"
              : "schnell";
  const prompt = optimizePrompt(raw, model);

  return { prompt, negativePrompt: NEGATIVE };
}
```

---

## 8. Creation Mode Router

The four modes route to different assembler paths.

```typescript
async function generateCardFromMode(
  deck: DeckSelection,
  card: CardBrief
): Promise<{ prompt: string; negativePrompt: string }> {

  switch (deck.creationMode) {
    case "reference_only": {
      if (!deck.referenceImageUrl) throw new Error("Reference image required.");
      const analysis = await analyzeReferenceImage(deck.referenceImageUrl);
      // Auto-populate style if user hasn't set one
      const deckWithSuggested: DeckSelection = {
        ...deck,
        artisticDirection: deck.artisticDirection ?? analysis.recommendedArtisticDirection,
        aestheticPack: deck.aestheticPack ?? analysis.recommendedAestheticPack,
      };
      return assembleImagePrompt(deckWithSuggested, card, analysis);
    }

    case "prompt_only": {
      return assembleImagePrompt(deck, card);
    }

    case "reference_prompt": {
      if (!deck.referenceImageUrl) throw new Error("Reference image required.");
      const analysis = await analyzeReferenceImage(deck.referenceImageUrl);
      return assembleImagePrompt(deck, card, analysis);
    }

    case "inspire_me": {
      // Seeded random from curated inspiration list
      const inspirations = [
        "Forest Spirits", "Lost Kingdoms", "Moon Temples", "Forgotten Queens",
        "Sacred Botanicals", "Celestial Gardens", "Dream Library",
        "The Alchemist", "The Wanderer", "The Oracle of Time",
        "The Cartographer of Stars", "The Weaver of Fates",
        "The Keeper of Thresholds", "The Gardener of Endings",
      ];
      const seed = inspirations[Math.floor(Math.random() * inspirations.length)];
      return assembleImagePrompt(
        { ...deck, userPrompt: `${card.cardName} interpreted through the theme: ${seed}` },
        card
      );
    }
  }
}
```

---

## 9. Deck DNA + Multi-LoRA Stacking

The core of deck cohesion. Two LoRAs, stacked, trained per deck.

### 9a. Freeze Deck DNA after first card

```typescript
async function freezeDeckDNA(
  deckId: string,
  firstCardResult: { imageUrl: string; prompt: string },
  deck: DeckSelection
): Promise<DeckDNA> {
  const style = STYLE_ENGINE[deck.artisticDirection];
  const pack = deck.aestheticPack ? AESTHETIC_PACKS[deck.aestheticPack] : "";

  const dna: DeckDNA = {
    seedCardIds: [deckId + ":card1"],
    frozenStyle: {
      brushwork: style.brushwork,
      palette: [], // populated after style LoRA training
      lightingRule: style.lightingRule,
      borderStyle: style.border,
      symbolMotifs: style.extras,
      aspectRatio: "2:3",
      typographySafeZone: { top: 0.08, bottom: 0.08 },
      visualRhythm: `${style.compositionRule}; ${pack}`,
    },
    triggerWord: `arcana_${deckId.slice(0, 8)}`,
  };
  await supabase.from("decks").update({ dna }).eq("id", deckId);
  return dna;
}
```

### 9b. Train the Style LoRA (once per deck, after 12 seed cards)

```typescript
async function trainStyleLoRA(deckId: string, seedImageUrls: string[]) {
  // 1. Bundle seed images into a ZIP
  const zipUrl = await bundleImagesToZip(seedImageUrls);

  // 2. Kick off training (~1000 steps ≈ $8, ~15 minutes)
  const result = await fal.subscribe("fal-ai/flux-2-trainer", {
    input: {
      images_data_url: zipUrl,
      trigger_word: `arcana_${deckId.slice(0, 8)}`,
      steps: 1000,
    },
  });

  const styleLoraUrl = result.data.diffusers_lora_file.url;
  await supabase.from("decks").update({
    "dna->>styleLoraUrl": styleLoraUrl,
  }).eq("id", deckId);
  return styleLoraUrl;
}
```

### 9c. Train the Character LoRA (only if characterConsistency)

```typescript
async function trainCharacterLoRA(deckId: string, characterImageUrls: string[]) {
  // 4-8 renders of the hero character in different poses/scenes
  const zipUrl = await bundleImagesToZip(characterImageUrls);
  const result = await fal.subscribe("fal-ai/flux-2-trainer", {
    input: {
      images_data_url: zipUrl,
      trigger_word: `arcana_char_${deckId.slice(0, 8)}`,
      steps: 800,
    },
  });
  const characterLoraUrl = result.data.diffusers_lora_file.url;
  await supabase.from("decks").update({
    "dna->>characterLoraUrl": characterLoraUrl,
  }).eq("id", deckId);
  return characterLoraUrl;
}
```

### 9d. Generate with stacked LoRAs

```typescript
async function generateWithStackedLoRAs(
  deck: DeckSelection,
  prompt: string,
  negativePrompt: string
) {
  const loras = [];
  if (deck.deckDNA?.styleLoraUrl) {
    loras.push({ path: deck.deckDNA.styleLoraUrl, scale: 1.0 });
  }
  if (deck.characterConsistency && deck.deckDNA?.characterLoraUrl) {
    loras.push({ path: deck.deckDNA.characterLoraUrl, scale: 0.85 });
  }

  const triggerPrefix = deck.deckDNA
    ? `${deck.deckDNA.triggerWord} style, ${
        deck.characterConsistency
          ? `arcana_char_${deck.deckDNA.triggerWord.split("_")[1]}, `
          : ""
      }`
    : "";

  return fal.subscribe("fal-ai/flux-lora", {
    input: {
      prompt: triggerPrefix + prompt,
      negative_prompt: negativePrompt,
      loras,
      image_size: "portrait_4_3",
      num_inference_steps: 28,
      guidance_scale: 3.5,
      num_images: deck.tier === "marketplace" ? 4 : 1, // 4 candidates for curator
      enable_safety_checker: true,
    },
  });
}
```

Stacking a style LoRA at scale 1.0 and a character LoRA at scale 0.85 is the empirically better balance — character LoRAs at 1.0 tend to overpower the deck's visual style.

---

## 10. Image Enhancement Pipeline

Post-generation refinements. Users click a chip and the app chains fal endpoints.

```typescript
type EnhancementAction =
  | "upscale_2x" | "upscale_4x"
  | "fix_faces" | "fix_hands"
  | "remove_artifacts" | "add_border"
  | "add_gold_foil" | "add_embossed_texture"
  | "generate_card_back" | "remove_background"
  | "replace_background" | "harmonize_palette"
  | "cmyk_preview";

const ENHANCEMENT_ENDPOINTS: Record<EnhancementAction, string> = {
  upscale_2x: "fal-ai/clarity-upscaler",
  upscale_4x: "fal-ai/clarity-upscaler",
  fix_faces: "fal-ai/gfpgan",
  fix_hands: "fal-ai/flux-2/edit",         // natural-language edit
  remove_artifacts: "fal-ai/clarity-upscaler",
  add_border: "fal-ai/flux-2/edit",
  add_gold_foil: "fal-ai/flux-2/edit",
  add_embossed_texture: "fal-ai/flux-2/edit",
  generate_card_back: "fal-ai/flux-lora",
  remove_background: "fal-ai/birefnet",
  replace_background: "fal-ai/flux-2/edit",
  harmonize_palette: "fal-ai/flux-2/edit",
  cmyk_preview: "internal:cmyk_preview",   // local Sharp/ICC conversion
};

async function enhance(imageUrl: string, action: EnhancementAction, deck: DeckSelection) {
  switch (action) {
    case "upscale_2x":
      return fal.subscribe("fal-ai/clarity-upscaler", {
        input: { image_url: imageUrl, scale_factor: 2 },
      });
    case "upscale_4x":
      return fal.subscribe("fal-ai/clarity-upscaler", {
        input: { image_url: imageUrl, scale_factor: 4 },
      });
    case "fix_faces":
      return fal.subscribe("fal-ai/gfpgan", {
        input: { image_url: imageUrl },
      });
    case "fix_hands":
      return fal.subscribe("fal-ai/flux-2/edit", {
        input: {
          image_url: imageUrl,
          prompt: "Correct the hands to anatomically accurate proportions with five fingers each. Preserve everything else.",
        },
      });
    case "add_gold_foil":
      return fal.subscribe("fal-ai/flux-2/edit", {
        input: {
          image_url: imageUrl,
          prompt: `Add subtle gold foil highlights along the border and any decorative symbols. Preserve composition and character.`,
        },
      });
    case "generate_card_back":
      return generateCardBack(deck);
    // ...remaining actions follow the same pattern
    default:
      throw new Error(`Unknown enhancement action: ${action}`);
  }
}
```

---

## 11. Copyright Exclusivity Workflow

This is the section that determines whether marketplace decks are defensible IP or unclaimed AI output. **Purely AI-generated images may not qualify for copyright protection.** A human-in-the-loop workflow with recorded creative decisions changes that calculus.

### 11a. Tier gating

| Tier | Copyright status | Workflow |
|---|---|---|
| `personal` | AI-generated, unclaimed | Direct generation, no curator step |
| `creator` | AI-generated with human style choices | Recorded style/prompt selections form the provenance |
| `marketplace` | **Human-curated creative work** | Curator selects, edits, and signs off each card |

### 11b. Provenance chain (recorded on every marketplace card)

```typescript
type ProvenanceChain = {
  cardId: string;
  deckId: string;
  curatorUserId: string;
  createdAt: string;
  steps: ProvenanceStep[];
  c2paMetadata: C2PAManifest;
  humanContributions: string[]; // narrative log of creative decisions
};

type ProvenanceStep =
  | { type: "user_prompt"; content: string; timestamp: string }
  | { type: "reference_upload"; imageHash: string; timestamp: string }
  | { type: "style_selection"; direction: string; pack: string; timestamp: string }
  | { type: "prompt_optimized"; from: string; to: string; timestamp: string }
  | { type: "generation"; model: string; params: any; candidateCount: number; timestamp: string }
  | { type: "curator_selected"; candidateIndex: number; reason: string; timestamp: string }
  | { type: "curator_edited"; editType: EnhancementAction; instruction: string; timestamp: string }
  | { type: "curator_signoff"; curatorUserId: string; timestamp: string };

type C2PAManifest = {
  claim_generator: "Arcana Studio v2";
  ai_model: string;                  // e.g. "FLUX.2 Pro via fal.ai + custom LoRA"
  human_curator_id: string;
  edits_applied: string[];
  created_at: string;
};
```

### 11c. Curator queue

When `tier === "marketplace"`, generation returns 4 candidates rather than 1. Card enters a curator queue.

```typescript
async function submitToCuratorQueue(
  deckId: string,
  cardBrief: CardBrief,
  candidates: string[],   // 4 image URLs
  provenanceSoFar: ProvenanceStep[]
) {
  await supabase.from("curator_queue").insert({
    deck_id: deckId,
    card_brief: cardBrief,
    candidate_urls: candidates,
    provenance: provenanceSoFar,
    status: "awaiting_curation",
    created_at: new Date().toISOString(),
  });
}
```

### 11d. Curator UI (documented for the frontend team)

The curator must, for each card:

1. Select one of 4 candidates. **Reason field required.**
2. Optionally apply 1 or more enhancement actions. Each recorded to provenance.
3. Optionally hand-annotate a prompt refinement and regenerate. Each iteration recorded.
4. Digital sign-off (name + timestamp) written to provenance and C2PA manifest.

**No automatic sign-off**. This is the step that establishes creative contribution and it must be explicit.

### 11e. C2PA metadata embedded on final export

```typescript
import { writeManifestToPng } from "./c2pa-utils";

async function finalizeMarketplaceCard(
  cardId: string,
  imageUrl: string,
  provenance: ProvenanceChain
): Promise<string> {
  const buffer = await fetch(imageUrl).then(r => r.arrayBuffer());
  const manifest = provenance.c2paMetadata;
  const signedPng = await writeManifestToPng(buffer, manifest);
  const finalUrl = await uploadToSupabaseStorage(signedPng, `marketplace/${cardId}.png`);
  return finalUrl;
}
```

C2PA metadata matters because ad platforms and stock sites increasingly require AI-provenance disclosure, and Meta and Google both auto-label detected synthetic media. Baking the provenance into the file means a marketplace deck can be listed on platforms that require disclosure without being auto-flagged as unattributed synthetic content.

### 11f. Marketplace listing badge

Every marketplace card ships with a visible attribution:

```
Human-curated by [Curator Name]
Created [date] · Verified provenance
```

Clicking the badge opens the provenance chain in a modal — this is the visible proof of human authorship contribution.

---

## 12. Meaning Prompt (Philosophy Default-On)

Philosophy is always included unless the user explicitly disables it.

```typescript
export function assembleMeaningPrompt(deck: DeckSelection, card: CardBrief): string {
  const philosophyInstruction = deck.philosophyEnabled
    ? `Include a philosophical reflection from the ${deck.philosophyLayer} tradition. Attribute it to a specific thinker where fitting (Marcus Aurelius, Epictetus, Seneca, Socrates, Confucius, Aristotle, Plato, Hermes Trismegistus). Keep it grounded and specific — no vague mysticism.`
    : "";

  const numerologyInstruction = deck.numerologyEnabled
    ? `Include a numerology field explaining the symbolic resonance of the number ${card.numerology}. Frame as a reflective symbolic lens, not a prediction.`
    : "";

  const astrologyInstruction = deck.astrologyEnabled
    ? `Include an astrology field referencing ${card.zodiac ?? "the zodiacal correspondence"} and ${card.planet ?? "the ruling planetary energy"}. Frame as symbolic correspondence, not predictive fact.`
    : "";

  return `
You are the resident scholar of Arcana Studio — a reflective wisdom studio that treats tarot, oracle, numerology, astrology, and ancient philosophy as interpretive lenses for personal reflection. Never predictive. Never diagnostic. Never a substitute for professional care.

Write the meaning package for a single card. Respond in strict JSON only — no preamble, no markdown fences.

CARD CONTEXT
- Deck type: ${deck.deckType}
- Card name: ${card.cardName}
- Card number: ${card.cardNumber}
- Archetype: ${card.archetype}
- Core symbols: ${card.coreSymbols.join(", ")}
${card.element ? `- Element: ${card.element}` : ""}
${card.suit ? `- Suit: ${card.suit}` : ""}
${card.zodiac ? `- Zodiac: ${card.zodiac}` : ""}
${card.planet ? `- Planet: ${card.planet}` : ""}

FIELDS
- keywords: 5 concrete words. No filler like "energy" or "vibes".
- lightMeaning: 2 sentences. Upright orientation.
- shadowMeaning: 2 sentences. The distortion of the same archetype.
- reversedMeaning: 2 sentences. What blocks or inverts it.
- journalPrompt: 1 open question worth sitting with for a week.
- affirmation: 1 sentence, first-person, present-tense. No toxic positivity.
- meditation: 3-4 sentence micro-meditation.
${numerologyInstruction}
${astrologyInstruction}
${philosophyInstruction}

TONE
- Direct, grounded, warm but not saccharine.
- Concrete imagery over abstract mystic language.
- Reflection framing on everything.

OUTPUT SHAPE
{
  "keywords": [string, string, string, string, string],
  "lightMeaning": string,
  "shadowMeaning": string,
  "reversedMeaning": string,
  "journalPrompt": string,
  "affirmation": string,
  "meditation": string,
  "numerology"?: string,
  "astrology"?: string,
  "philosophy": { "thinker": string, "tradition": string, "reflection": string }
}
  `.trim();
}
```

Note: with philosophy default-on, the `philosophy` field is *not optional* in the schema. This is deliberate — it's the positioning signal on every single card.

---

## 13. The generator route (Next.js API)

`app/api/generate-card/route.ts`:

```typescript
import { NextRequest, NextResponse } from "next/server";
import { fal } from "@fal-ai/client";
import Anthropic from "@anthropic-ai/sdk";
import {
  assembleImagePrompt,
  assembleMeaningPrompt,
  generateCardFromMode,
  optimizePrompt,
  generateWithStackedLoRAs,
  submitToCuratorQueue,
} from "@/lib/prompts";
import { supabase } from "@/lib/supabase";
import { v4 as uuid } from "uuid";

fal.config({ credentials: process.env.FAL_KEY });
const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

export async function POST(req: NextRequest) {
  try {
    const { deck, card } = await req.json() as {
      deck: DeckSelection;
      card: CardBrief;
    };

    const cardId = uuid();
    const provenance: ProvenanceStep[] = [];

    // ── Provenance step 1: capture user inputs ────────────────
    if (deck.userPrompt) {
      provenance.push({
        type: "user_prompt",
        content: deck.userPrompt,
        timestamp: new Date().toISOString(),
      });
    }
    if (deck.referenceImageUrl) {
      provenance.push({
        type: "reference_upload",
        imageHash: await hashUrl(deck.referenceImageUrl),
        timestamp: new Date().toISOString(),
      });
    }
    provenance.push({
      type: "style_selection",
      direction: deck.artisticDirection,
      pack: deck.aestheticPack ?? "none",
      timestamp: new Date().toISOString(),
    });

    // ── Route to correct creation mode & assemble prompt ──────
    const { prompt, negativePrompt } = await generateCardFromMode(deck, card);

    provenance.push({
      type: "prompt_optimized",
      from: "raw",
      to: prompt,
      timestamp: new Date().toISOString(),
    });

    // ── Choose image model based on deck state + tier ─────────
    const hasLoRA = !!deck.deckDNA?.styleLoraUrl;
    const useMarketplaceQuality = deck.tier === "marketplace";
    const candidateCount = useMarketplaceQuality ? 4 : 1;

    let imageResult;
    if (hasLoRA) {
      imageResult = await generateWithStackedLoRAs(deck, prompt, negativePrompt);
    } else if (useMarketplaceQuality) {
      imageResult = await fal.subscribe("fal-ai/flux-2/pro", {
        input: {
          prompt,
          negative_prompt: negativePrompt,
          image_size: "portrait_4_3",
          num_images: candidateCount,
        },
      });
    } else {
      imageResult = await fal.subscribe("fal-ai/flux/schnell", {
        input: {
          prompt,
          negative_prompt: negativePrompt,
          image_size: "portrait_4_3",
          num_inference_steps: 4,
          num_images: 1,
        },
      });
    }

    provenance.push({
      type: "generation",
      model: hasLoRA ? "flux-lora-stacked" : useMarketplaceQuality ? "flux-2-pro" : "flux-schnell",
      params: { prompt_length: prompt.length },
      candidateCount,
      timestamp: new Date().toISOString(),
    });

    const candidates = imageResult.data.images.map((i: any) => i.url);

    // ── Marketplace tier: submit to curator queue, exit early ──
    if (useMarketplaceQuality) {
      await submitToCuratorQueue(deck.deckDNA?.seedCardIds[0] ?? "unknown", card, candidates, provenance);
      return NextResponse.json({
        status: "awaiting_curation",
        candidateUrls: candidates,
        cardId,
        provenance,
      });
    }

    // ── Non-marketplace: kick off meaning generation ──────────
    const meaningPrompt = assembleMeaningPrompt(deck, card);
    const meaningResult = await anthropic.messages.create({
      model: "claude-sonnet-4-6",
      max_tokens: 1200,
      messages: [{ role: "user", content: meaningPrompt }],
    });
    const meanings = JSON.parse((meaningResult.content[0] as any).text);

    // ── Persist ───────────────────────────────────────────────
    await supabase.from("cards").insert({
      id: cardId,
      deck_id: deck.deckDNA?.seedCardIds[0] ?? "personal",
      card_brief: card,
      image_url: candidates[0],
      meanings,
      provenance,
      tier: deck.tier,
    });

    return NextResponse.json({
      cardId,
      imageUrl: candidates[0],
      meanings,
      provenance,
    });

  } catch (err: any) {
    console.error(err);
    return NextResponse.json({ error: err.message }, { status: 500 });
  }
}

async function hashUrl(url: string): Promise<string> {
  const encoder = new TextEncoder();
  const buf = await crypto.subtle.digest("SHA-256", encoder.encode(url));
  return Array.from(new Uint8Array(buf)).map(b => b.toString(16).padStart(2, "0")).join("");
}
```

---

## 14. Cost math (updated for v2)

### Personal tier — 78-card deck, no LoRA
| Component | Unit | Deck total |
|---|---|---|
| 78 images @ FLUX Schnell | ~$0.003 | $0.234 |
| 78 meaning packages @ Sonnet 4.6 | ~$0.008 | $0.624 |
| 1 card back | ~$0.003 | $0.003 |
| **Total** | | **~$0.86** |

### Creator tier — 78-card deck with style LoRA
| Component | Unit | Deck total |
|---|---|---|
| 12 seed images @ FLUX Schnell | ~$0.003 | $0.036 |
| Style LoRA training (1000 steps) | $0.008/step | $8.00 |
| 78 images via LoRA endpoint | ~$0.005 | $0.39 |
| 78 meaning packages | ~$0.008 | $0.624 |
| 1 card back via LoRA | ~$0.005 | $0.005 |
| **Total** | | **~$9.06** |

### Marketplace tier — 78-card deck, dual LoRA, curator queue
| Component | Unit | Deck total |
|---|---|---|
| 12 seed images | ~$0.003 | $0.036 |
| Style LoRA training | $0.008 × 1000 | $8.00 |
| 6 character images | ~$0.005 | $0.030 |
| Character LoRA training | $0.008 × 800 | $6.40 |
| 78 × 4 candidates via stacked LoRA | ~$0.005 × 4 | $1.56 |
| Curator time (estimate 5min/card) | at internal rate | (labour) |
| Enhancement passes (avg 1.5 per card) | ~$0.005 | $0.585 |
| 78 meaning packages | ~$0.008 | $0.624 |
| Card back with LoRA | ~$0.005 | $0.005 |
| C2PA metadata + storage | ~$0.001 | $0.078 |
| **Total (compute only)** | | **~$17.32** |

### Suggested consumer pricing

| Tier | Price | Includes |
|---|---|---|
| Free | $0 | Single card + 3-card spread, no deck save |
| Personal | $8.99/mo | Unlimited personal decks, no LoRA, no marketplace listing |
| Creator | $19.99/mo | Style LoRA per deck, PDF export, print-on-demand link |
| Studio | $49.99/mo | Character LoRA, curator queue access, marketplace listing, C2PA |
| Marketplace revenue share | 70% creator / 30% platform | On each deck sale |

---

## 15. Roadmap (updated)

### Phase 1 — MVP (weeks 1–6)
- Full selection schema + defaults (philosophy default-on)
- Four creation modes wired up
- Style Engine + Aesthetic Packs
- FLUX Schnell single-card generation
- Meaning generation (philosophy always populated)
- Supabase persistence + user library
- Daily card + journal

### Phase 2 — Deck Cohesion (weeks 7–12)
- Deck DNA freeze
- Style LoRA training pipeline
- LoRA-stacked generation route
- Smart Style Assistant
- AI Prompt Optimizer wired into every generation
- Reference Image Analyzer

### Phase 3 — Character + Enhancement (weeks 13–18)
- Character consistency + character LoRA
- Full enhancement pipeline (upscale, face/hand fix, gold foil, borders)
- Card back generator
- PDF guidebook export with bleed marks

### Phase 4 — Marketplace + Copyright (weeks 19–24)
- Curator queue UI
- Provenance chain persistence
- C2PA metadata embedding
- Stripe Connect for creator payouts
- Marketplace listings with attribution badge
- Physical print-on-demand integration (MakePlayingCards API)

### Phase 5 — Depth (weeks 25+)
- "Deck from my Life Path" numerology personalization
- Symbol Explorer (click any symbol → cross-tradition meaning)
- Astrology chart integration (birth chart → deck theme)
- AI-assisted interpretation chat with memory
- Multi-language meaning generation

---

## 16. The non-negotiable framing

Every card, guidebook page, and reading interface displays this on first load and in About:

> Tarot, oracle, numerology, astrology, and philosophical reflection are interpretive lenses for personal insight and reflection. Nothing in Arcana Studio predicts the future, diagnoses conditions, or replaces professional advice from qualified practitioners.

This is what separates a mature reflective wisdom studio from the predictive-claim apps that get pulled from app stores or flagged under consumer protection regulations (including Australian ACL for misleading conduct claims).

---

## 17. Master prompt example (fully v2-assembled)

The High Priestess in a Golden Renaissance direction with Gold Foil pack, philosophy default-on, style LoRA trained:

```
arcana_ab12cd34 style. A museum-quality traditional tarot card illustration. Portrait orientation, 2:3 aspect, print-ready with subtle inner border and card frame. High resolution, clean composition, single hero subject centred. classical oil painting with glazed layers. soft volumetric chiaroscuro from upper left. classical proportions, sacred geometry underlay. iconographic symbolism grounded in Renaissance tradition. Border: decorative gilded border with subtle floral motifs. museum quality detailing, no modern objects, no visible text. Aesthetic overlay: gold foil highlights on ornament and border. Deck DNA (must match): brushwork classical oil painting with glazed layers; palette warm gold, rich crimson, ivory, umber; lighting soft volumetric chiaroscuro from upper left; border decorative gilded border with subtle floral motifs; recurring motifs museum quality detailing, no modern objects, no visible text; visual rhythm classical proportions, sacred geometry underlay. Card: The High Priestess (Number 2). Archetype: The Keeper of Mysteries. Include the following symbols: crescent moon, twin pillars, veil, scroll, stars. Elemental atmosphere: water. Numerological resonance of 2: subtle symbolic references. Astrological correspondence: Cancer and Moon — include a subtle glyph in the border. Highly detailed, professionally composed, symmetrical framing, ornate but restrained border, portrait orientation, no cropping of hands or faces, no text unless explicitly requested, print-ready quality.
```

Negative:
```
text, watermark, signature, logo, extra fingers, extra limbs, malformed hands, malformed faces, low resolution, blurry, jpeg artifacts, duplicate objects, cropped subject, cluttered composition, uneven border, cheap clipart, oversaturated, garish, brand names, trademarks, identifiable celebrities, contemporary clothing, modern architecture, digital effects
```

Deterministic. Reproducible. Deck-coherent. Copyright-defensible when passed through the marketplace curator queue.
