# ARCANA STUDIO — Life Path Deck Personalization

*Phase 5 feature. Plugs into v2 architecture. Turns numerology from a decorative field into the deck's genetic code.*

---

## 0. The pitch

Every person deserves a deck that speaks their language.

Enter your birth date and full birth name. Arcana Studio crafts a **27-card personal oracle** drawn from your numerological blueprint — 22 Major Arcana filtered through your Life Path, plus 5 companion cards unique to your **Signature**, **Shadow**, **Path**, **Voice**, and **Heart**.

Your Signature Card is fixed for life. Your Path Card regenerates every birthday.

Every meaning references your specific numbers. Nothing generic. Nothing recycled.

**Why this wins:**
- No competitor does this. TarotAI.art custom decks aren't personalized to the user's numerology.
- Annual Path Card regeneration is a built-in retention hook — users return every birthday for their new card.
- Bridges the numerology audience (large, underserved by tarot apps) with the tarot audience.
- The 5 companion cards create urgency ("your Karmic Debt card is waiting to be seen").
- Highest-ticket single purchase in the product ladder: a lifetime personalized deck is a $49–$199 one-time sale, or the core value of the Studio subscription.

---

## 1. The numerology engine

Pure functions. No side effects. Written for TypeScript, testable in isolation.

`lib/numerology.ts`:

```typescript
// ── Constants ────────────────────────────────────────────────

// Pythagorean letter values
const LETTER_VALUES: Record<string, number> = {
  A: 1, B: 2, C: 3, D: 4, E: 5, F: 6, G: 7, H: 8, I: 9,
  J: 1, K: 2, L: 3, M: 4, N: 5, O: 6, P: 7, Q: 8, R: 9,
  S: 1, T: 2, U: 3, V: 4, W: 5, X: 6, Y: 7, Z: 8,
};

const VOWELS = new Set(["A", "E", "I", "O", "U"]);
// Y is treated as a vowel when it acts as one — simpler rule:
// treat Y as a vowel if surrounded by consonants OR at word start/end.

const MASTER_NUMBERS = new Set([11, 22, 33]);
const KARMIC_DEBT_NUMBERS = new Set([13, 14, 16, 19]);

// ── Reduction ────────────────────────────────────────────────

/**
 * Reduce a number to single digit, preserving master numbers 11, 22, 33.
 * If the number is a karmic debt (13, 14, 16, 19), we track that separately.
 */
export function reduceNumber(n: number): number {
  let current = Math.abs(n);
  while (current > 9 && !MASTER_NUMBERS.has(current)) {
    current = current.toString().split("").reduce((sum, d) => sum + parseInt(d, 10), 0);
  }
  return current;
}

/**
 * Detect karmic debt: if any intermediate two-digit sum equals 13, 14, 16, or 19,
 * the final reduced number carries that karmic weight.
 */
export function detectKarmicDebt(n: number): number | null {
  let current = Math.abs(n);
  while (current > 9 && !MASTER_NUMBERS.has(current)) {
    if (KARMIC_DEBT_NUMBERS.has(current)) return current;
    current = current.toString().split("").reduce((sum, d) => sum + parseInt(d, 10), 0);
  }
  return null;
}

// ── Y-as-vowel detection ─────────────────────────────────────

function isYVowel(name: string, index: number): boolean {
  const letter = name[index];
  if (letter !== "Y") return false;
  const prev = name[index - 1];
  const next = name[index + 1];
  const prevIsVowel = prev && VOWELS.has(prev);
  const nextIsVowel = next && VOWELS.has(next);
  // Y is a vowel if not surrounded by other vowels, or at word boundary
  return !prevIsVowel && !nextIsVowel;
}

// ── Core Numbers ─────────────────────────────────────────────

/**
 * Life Path: sum digits of birth date (YYYY-MM-DD) and reduce.
 * The most important number — reveals the person's overall journey.
 */
export function calculateLifePath(birthDateISO: string): {
  number: number;
  karmicDebt: number | null;
} {
  const digits = birthDateISO.replace(/-/g, "").split("").map(d => parseInt(d, 10));
  const sum = digits.reduce((a, b) => a + b, 0);
  return { number: reduceNumber(sum), karmicDebt: detectKarmicDebt(sum) };
}

/**
 * Birth Day Number: just the day of birth (1-31), reduced if needed.
 */
export function calculateBirthDay(birthDateISO: string): number {
  const day = parseInt(birthDateISO.split("-")[2], 10);
  return reduceNumber(day);
}

/**
 * Expression / Destiny: sum of ALL letters in the full birth name.
 * Reveals natural talents and life direction.
 */
export function calculateExpression(fullName: string): {
  number: number;
  karmicDebt: number | null;
} {
  const clean = fullName.toUpperCase().replace(/[^A-Z]/g, "");
  const sum = clean.split("").reduce((s, ch) => s + (LETTER_VALUES[ch] ?? 0), 0);
  return { number: reduceNumber(sum), karmicDebt: detectKarmicDebt(sum) };
}

/**
 * Heart Desire / Soul Urge: sum of VOWELS only.
 * Reveals inner motivation and what the soul longs for.
 */
export function calculateHeartDesire(fullName: string): {
  number: number;
  karmicDebt: number | null;
} {
  const clean = fullName.toUpperCase().replace(/[^A-Z]/g, "");
  const sum = clean.split("").reduce((s, ch, i) => {
    if (VOWELS.has(ch) || isYVowel(clean, i)) return s + (LETTER_VALUES[ch] ?? 0);
    return s;
  }, 0);
  return { number: reduceNumber(sum), karmicDebt: detectKarmicDebt(sum) };
}

/**
 * Personality: sum of CONSONANTS only.
 * Reveals outward persona and first impression.
 */
export function calculatePersonality(fullName: string): {
  number: number;
  karmicDebt: number | null;
} {
  const clean = fullName.toUpperCase().replace(/[^A-Z]/g, "");
  const sum = clean.split("").reduce((s, ch, i) => {
    if (!VOWELS.has(ch) && !isYVowel(clean, i)) return s + (LETTER_VALUES[ch] ?? 0);
    return s;
  }, 0);
  return { number: reduceNumber(sum), karmicDebt: detectKarmicDebt(sum) };
}

/**
 * Maturity: Life Path + Expression, reduced.
 * Reveals the person one grows into after age 35.
 */
export function calculateMaturity(lifePath: number, expression: number): number {
  return reduceNumber(lifePath + expression);
}

// ── Karmic Lessons ──────────────────────────────────────────

/**
 * Karmic Lessons: numbers 1-9 that DO NOT appear in the name letters.
 * These are the growth edges the person is here to work on.
 */
export function calculateKarmicLessons(fullName: string): number[] {
  const clean = fullName.toUpperCase().replace(/[^A-Z]/g, "");
  const present = new Set<number>();
  for (const ch of clean) {
    if (LETTER_VALUES[ch]) present.add(LETTER_VALUES[ch]);
  }
  const missing: number[] = [];
  for (let n = 1; n <= 9; n++) {
    if (!present.has(n)) missing.push(n);
  }
  return missing;
}

// ── Cycles ──────────────────────────────────────────────────

/**
 * Personal Year: birth month + birth day + current year, reduced.
 * Reveals the theme of the current calendar year for this person.
 */
export function calculatePersonalYear(birthDateISO: string, currentYear: number): number {
  const [_, month, day] = birthDateISO.split("-").map(Number);
  const sum = month + day + currentYear;
  return reduceNumber(sum);
}

/**
 * Personal Month: Personal Year + current month.
 * Theme of the current calendar month.
 */
export function calculatePersonalMonth(personalYear: number, currentMonth: number): number {
  return reduceNumber(personalYear + currentMonth);
}

/**
 * Personal Day: Personal Month + current day.
 * Theme of the current calendar day — powers the daily draw.
 */
export function calculatePersonalDay(personalMonth: number, currentDay: number): number {
  return reduceNumber(personalMonth + currentDay);
}

/**
 * Challenge Numbers: four challenges derived from birth date differences.
 * These reveal the obstacles the person will meet across life stages.
 */
export function calculateChallenges(birthDateISO: string): [number, number, number, number] {
  const [year, month, day] = birthDateISO.split("-").map(Number);
  const m = reduceNumber(month);
  const d = reduceNumber(day);
  const y = reduceNumber(year);

  const first = reduceNumber(Math.abs(m - d));
  const second = reduceNumber(Math.abs(d - y));
  const third = reduceNumber(Math.abs(first - second));
  const fourth = reduceNumber(Math.abs(m - y));

  return [first, second, third, fourth];
}

/**
 * Pinnacle Cycles: four life cycles derived from birth date sums.
 * These reveal the dominant energies of each life stage.
 */
export function calculatePinnacles(birthDateISO: string): [number, number, number, number] {
  const [year, month, day] = birthDateISO.split("-").map(Number);
  const m = reduceNumber(month);
  const d = reduceNumber(day);
  const y = reduceNumber(year);

  const first = reduceNumber(m + d);
  const second = reduceNumber(d + y);
  const third = reduceNumber(first + second);
  const fourth = reduceNumber(m + y);

  return [first, second, third, fourth];
}

// ── The full profile ─────────────────────────────────────────

export type NumerologyProfile = {
  lifePath: number;
  lifePathKarmicDebt: number | null;
  birthDay: number;
  expression: number;
  expressionKarmicDebt: number | null;
  heartDesire: number;
  heartDesireKarmicDebt: number | null;
  personality: number;
  personalityKarmicDebt: number | null;
  maturity: number;
  karmicLessons: number[];
  personalYear: number;
  personalMonth: number;
  personalDay: number;
  challenges: [number, number, number, number];
  pinnacles: [number, number, number, number];
};

export function buildNumerologyProfile(
  birthDateISO: string,
  fullName: string,
  today: Date = new Date()
): NumerologyProfile {
  const lifePath = calculateLifePath(birthDateISO);
  const expression = calculateExpression(fullName);
  const heartDesire = calculateHeartDesire(fullName);
  const personality = calculatePersonality(fullName);

  const personalYear = calculatePersonalYear(birthDateISO, today.getFullYear());
  const personalMonth = calculatePersonalMonth(personalYear, today.getMonth() + 1);
  const personalDay = calculatePersonalDay(personalMonth, today.getDate());

  return {
    lifePath: lifePath.number,
    lifePathKarmicDebt: lifePath.karmicDebt,
    birthDay: calculateBirthDay(birthDateISO),
    expression: expression.number,
    expressionKarmicDebt: expression.karmicDebt,
    heartDesire: heartDesire.number,
    heartDesireKarmicDebt: heartDesire.karmicDebt,
    personality: personality.number,
    personalityKarmicDebt: personality.karmicDebt,
    maturity: calculateMaturity(lifePath.number, expression.number),
    karmicLessons: calculateKarmicLessons(fullName),
    personalYear,
    personalMonth,
    personalDay,
    challenges: calculateChallenges(birthDateISO),
    pinnacles: calculatePinnacles(birthDateISO),
  };
}
```

---

## 2. Number → deck attribute mapping

Every core number carries meaning across five dimensions the deck cares about: **archetype**, **element**, **planet**, **zodiac**, and **philosophy**. This mapping table is the bridge from numerology to the v2 selection schema.

`lib/numerology-mappings.ts`:

```typescript
export type NumberProfile = {
  archetype: string;
  shadowArchetype: string;
  element: "fire" | "water" | "air" | "earth" | "spirit";
  planet: string;
  zodiac: string;
  philosophy: "stoic" | "socratic" | "confucian" | "epicurean" | "aristotelian" | "platonic" | "hermetic";
  color: string;              // hex-friendly seed for the palette
  keyword: string;            // one word essence
  giftLine: string;           // what this number offers
  shadowLine: string;         // what to watch for
  suggestedArtisticDirection: string;    // from v2 STYLE_ENGINE
  suggestedAestheticPack: string;        // from v2 AESTHETIC_PACKS
  symbolMotifs: string[];     // recurring visual symbols
};

export const NUMBER_PROFILES: Record<number, NumberProfile> = {
  1: {
    archetype: "The Sovereign",
    shadowArchetype: "The Tyrant",
    element: "fire",
    planet: "Sun",
    zodiac: "Aries",
    philosophy: "stoic",
    color: "#C9A24A",
    keyword: "Sovereignty",
    giftLine: "You are here to lead, to initiate, to walk your own road.",
    shadowLine: "The shadow is domination, isolation, and refusing help.",
    suggestedArtisticDirection: "minimal_luxury",
    suggestedAestheticPack: "royal",
    symbolMotifs: ["crown", "flame", "single pillar", "sunrise", "sword"],
  },
  2: {
    archetype: "The Diplomat",
    shadowArchetype: "The Doormat",
    element: "water",
    planet: "Moon",
    zodiac: "Cancer",
    philosophy: "confucian",
    color: "#8FA5C4",
    keyword: "Harmony",
    giftLine: "You are here to listen, to bridge, to sense what others cannot.",
    shadowLine: "The shadow is self-erasure and losing yourself in others.",
    suggestedArtisticDirection: "divine_feminine",
    suggestedAestheticPack: "celestial",
    symbolMotifs: ["crescent moon", "twin pillars", "chalice", "veil", "mirror"],
  },
  3: {
    archetype: "The Creator",
    shadowArchetype: "The Scatterer",
    element: "fire",
    planet: "Jupiter",
    zodiac: "Sagittarius",
    philosophy: "epicurean",
    color: "#D97A5A",
    keyword: "Expression",
    giftLine: "You are here to make, to speak, to bring joy into visible form.",
    shadowLine: "The shadow is scatter, superficiality, and abandoning the deep work.",
    suggestedArtisticDirection: "art_nouveau",
    suggestedAestheticPack: "floral",
    symbolMotifs: ["lyre", "quill", "blooming flower", "birds", "open mouth"],
  },
  4: {
    archetype: "The Builder",
    shadowArchetype: "The Prisoner",
    element: "earth",
    planet: "Saturn",
    zodiac: "Capricorn",
    philosophy: "aristotelian",
    color: "#6B5A3E",
    keyword: "Foundation",
    giftLine: "You are here to build what lasts — structure, discipline, mastery.",
    shadowLine: "The shadow is rigidity, self-punishment, and building your own cage.",
    suggestedArtisticDirection: "sacred_geometry",
    suggestedAestheticPack: "ancient",
    symbolMotifs: ["oak", "pillar", "compass", "brick", "mountain"],
  },
  5: {
    archetype: "The Seeker",
    shadowArchetype: "The Escapist",
    element: "air",
    planet: "Mercury",
    zodiac: "Gemini",
    philosophy: "socratic",
    color: "#4A8A6B",
    keyword: "Freedom",
    giftLine: "You are here to question, to travel, to refuse the given answer.",
    shadowLine: "The shadow is restlessness, addiction, and running from depth.",
    suggestedArtisticDirection: "high_fantasy",
    suggestedAestheticPack: "forest",
    symbolMotifs: ["winged sandals", "open road", "key", "compass", "wanderer's staff"],
  },
  6: {
    archetype: "The Nurturer",
    shadowArchetype: "The Martyr",
    element: "earth",
    planet: "Venus",
    zodiac: "Libra",
    philosophy: "confucian",
    color: "#B85D6A",
    keyword: "Care",
    giftLine: "You are here to hold, to heal, to make a hearth of your life.",
    shadowLine: "The shadow is martyrdom, control disguised as care, and losing self in service.",
    suggestedArtisticDirection: "cottagecore",
    suggestedAestheticPack: "floral",
    symbolMotifs: ["rose", "hearth", "chalice", "hands cupped", "wheat sheaf"],
  },
  7: {
    archetype: "The Mystic",
    shadowArchetype: "The Recluse",
    element: "water",
    planet: "Neptune",
    zodiac: "Pisces",
    philosophy: "hermetic",
    color: "#4A3B5C",
    keyword: "Depth",
    giftLine: "You are here to seek, to study, to hear what the surface hides.",
    shadowLine: "The shadow is isolation, cynicism, and hiding behind the search.",
    suggestedArtisticDirection: "dark_academia",
    suggestedAestheticPack: "spiritual",
    symbolMotifs: ["hermit's lantern", "open book", "cave mouth", "crescent moon", "still water"],
  },
  8: {
    archetype: "The Steward of Power",
    shadowArchetype: "The Tyrant of Wealth",
    element: "earth",
    planet: "Saturn",
    zodiac: "Scorpio",
    philosophy: "stoic",
    color: "#6E4A2F",
    keyword: "Mastery",
    giftLine: "You are here to command material and use it well.",
    shadowLine: "The shadow is greed, control, and mistaking possessions for worth.",
    suggestedArtisticDirection: "luxury_tarot",
    suggestedAestheticPack: "royal",
    symbolMotifs: ["scales", "crown", "eagle", "ouroboros", "coin"],
  },
  9: {
    archetype: "The Elder",
    shadowArchetype: "The Bitter Judge",
    element: "fire",
    planet: "Mars",
    zodiac: "Sagittarius",
    philosophy: "aristotelian",
    color: "#9A5D4A",
    keyword: "Completion",
    giftLine: "You are here to give, to complete cycles, to serve the whole.",
    shadowLine: "The shadow is bitterness, sacrifice-as-identity, and refusing to receive.",
    suggestedArtisticDirection: "ethereal",
    suggestedAestheticPack: "celestial",
    symbolMotifs: ["wheel", "halo", "world tree", "outstretched hands", "circle"],
  },
  11: {
    archetype: "The Illuminator",
    shadowArchetype: "The Overwhelmed Antenna",
    element: "spirit",
    planet: "Neptune",
    zodiac: "Aquarius",
    philosophy: "hermetic",
    color: "#8DA8D9",
    keyword: "Vision",
    giftLine: "You are here to see beyond and translate what you see for others.",
    shadowLine: "The shadow is anxiety, sensory overwhelm, and prophet's fatigue.",
    suggestedArtisticDirection: "celestial_fantasy",
    suggestedAestheticPack: "crystal",
    symbolMotifs: ["twin pillars", "star", "mirror", "third eye", "flame in glass"],
  },
  22: {
    archetype: "The Master Builder",
    shadowArchetype: "The Grand Failure",
    element: "earth",
    planet: "Saturn",
    zodiac: "Capricorn",
    philosophy: "platonic",
    color: "#556B4E",
    keyword: "Manifestation",
    giftLine: "You are here to build things that outlast you.",
    shadowLine: "The shadow is grandiosity, paralysis, and refusing the practical step.",
    suggestedArtisticDirection: "sacred_geometry",
    suggestedAestheticPack: "ancient",
    symbolMotifs: ["cathedral", "blueprint", "world tree", "pillars of light", "keystone"],
  },
  33: {
    archetype: "The Teacher",
    shadowArchetype: "The Burnt Guru",
    element: "spirit",
    planet: "Jupiter",
    zodiac: "Pisces",
    philosophy: "stoic",
    color: "#B87D5C",
    keyword: "Compassion",
    giftLine: "You are here to teach, to model, to hold the room.",
    shadowLine: "The shadow is spiritual bypassing, burnout, and messiah complex.",
    suggestedArtisticDirection: "divine_feminine",
    suggestedAestheticPack: "spiritual",
    symbolMotifs: ["heart in hands", "dove", "olive branch", "chalice overflowing", "open palms"],
  },
};

/**
 * Karmic Debt profiles — appear as shadow cards in the deck.
 */
export const KARMIC_DEBT_PROFILES: Record<number, { name: string; teaching: string; symbol: string }> = {
  13: {
    name: "The Debt of Effort",
    teaching: "Work you avoid in past patterns must now be met head-on. Structure over shortcut.",
    symbol: "a plow standing in unturned earth",
  },
  14: {
    name: "The Debt of Discipline",
    teaching: "Excess of sensation in the past requires temperance now. Freedom through form.",
    symbol: "a chalice balanced on a knife's edge",
  },
  16: {
    name: "The Debt of Ego",
    teaching: "Pride toppled towers in the past. Now the work is humility without collapse.",
    symbol: "a broken tower against dawn light",
  },
  19: {
    name: "The Debt of Solitude",
    teaching: "Independence taken to isolation in the past must now become interdependence.",
    symbol: "a hermit turning toward the road",
  },
};
```

---

## 3. The 27-card personalized deck

Every user gets:

- **The 22 Major Arcana** — but each card's brief is filtered through their Life Path's element, palette, and shadow lens.
- **5 Companion Cards** — bespoke to them:

| Companion | Sourced from | Regenerates? |
|---|---|---|
| **The Signature Card** | Life Path Number | Never — lifetime card |
| **The Voice Card** | Expression Number | Never |
| **The Heart Card** | Heart Desire Number | Never |
| **The Shadow Card** | Karmic Debt (or Challenge 1 if none) | Never |
| **The Path Card** | Personal Year Number | Every birthday |

### 3a. The Major Arcana as briefs

`lib/major-arcana.ts`:

```typescript
import { CardBrief } from "./types";

export const MAJOR_ARCANA: CardBrief[] = [
  { cardName: "The Fool",            cardNumber: 0,  archetype: "The Innocent Beginner",  coreSymbols: ["cliff edge","white rose","small dog","knapsack","rising sun"],       element: "air",   zodiac: "Aquarius",   planet: "Uranus",  numerology: 0 },
  { cardName: "The Magician",        cardNumber: 1,  archetype: "The Focused Will",       coreSymbols: ["infinity symbol","four suit tools","altar","red robe","white lily"], element: "air",   zodiac: "Gemini",     planet: "Mercury", numerology: 1 },
  { cardName: "The High Priestess",  cardNumber: 2,  archetype: "The Keeper of Mysteries",coreSymbols: ["crescent moon","twin pillars","veil","scroll","stars"],              element: "water", zodiac: "Cancer",     planet: "Moon",    numerology: 2 },
  { cardName: "The Empress",         cardNumber: 3,  archetype: "The Great Mother",       coreSymbols: ["crown of stars","pomegranates","wheat field","flowing river","rose"], element: "earth", zodiac: "Taurus",     planet: "Venus",   numerology: 3 },
  { cardName: "The Emperor",         cardNumber: 4,  archetype: "The Sovereign Father",   coreSymbols: ["stone throne","ram heads","ankh","desert mountains","red robe"],     element: "fire",  zodiac: "Aries",      planet: "Mars",    numerology: 4 },
  { cardName: "The Hierophant",      cardNumber: 5,  archetype: "The Bridge to Tradition",coreSymbols: ["triple crown","crossed keys","two acolytes","stone temple","staff"], element: "earth", zodiac: "Taurus",     planet: "Venus",   numerology: 5 },
  { cardName: "The Lovers",          cardNumber: 6,  archetype: "The Sacred Choice",      coreSymbols: ["angel","sun","two figures","tree of knowledge","tree of life"],      element: "air",   zodiac: "Gemini",     planet: "Mercury", numerology: 6 },
  { cardName: "The Chariot",         cardNumber: 7,  archetype: "The Directed Will",      coreSymbols: ["armoured driver","two sphinxes","canopy of stars","laurel","city"],  element: "water", zodiac: "Cancer",     planet: "Moon",    numerology: 7 },
  { cardName: "Strength",            cardNumber: 8,  archetype: "The Gentle Power",       coreSymbols: ["lion","woman","infinity symbol","white robe","flowers"],             element: "fire",  zodiac: "Leo",        planet: "Sun",     numerology: 8 },
  { cardName: "The Hermit",          cardNumber: 9,  archetype: "The Inner Light",        coreSymbols: ["lantern with star","staff","grey robe","mountain peak","cloak"],     element: "earth", zodiac: "Virgo",      planet: "Mercury", numerology: 9 },
  { cardName: "Wheel of Fortune",    cardNumber: 10, archetype: "The Turning",            coreSymbols: ["wheel","four fixed signs","sphinx","serpent","anubis"],              element: "fire",  zodiac: "Sagittarius",planet: "Jupiter", numerology: 1 },
  { cardName: "Justice",             cardNumber: 11, archetype: "The Balanced Truth",     coreSymbols: ["scales","sword upright","crown","pillars","red robe"],               element: "air",   zodiac: "Libra",      planet: "Venus",   numerology: 11 },
  { cardName: "The Hanged Man",      cardNumber: 12, archetype: "The Willing Suspension", coreSymbols: ["T-cross","inverted figure","halo","serene face","tied leg"],         element: "water", zodiac: "Pisces",     planet: "Neptune", numerology: 3 },
  { cardName: "Death",               cardNumber: 13, archetype: "The Great Transformer",  coreSymbols: ["skeleton in armour","white rose banner","rising sun","fallen king"], element: "water", zodiac: "Scorpio",    planet: "Pluto",   numerology: 4 },
  { cardName: "Temperance",          cardNumber: 14, archetype: "The Alchemist",          coreSymbols: ["angel","two chalices","water flowing","iris flowers","triangle"],    element: "fire",  zodiac: "Sagittarius",planet: "Jupiter", numerology: 5 },
  { cardName: "The Devil",           cardNumber: 15, archetype: "The Bound Shadow",       coreSymbols: ["horned figure","two chained figures","inverted pentagram","torch"],  element: "earth", zodiac: "Capricorn",  planet: "Saturn",  numerology: 6 },
  { cardName: "The Tower",           cardNumber: 16, archetype: "The Sudden Awakening",   coreSymbols: ["struck tower","lightning","crown falling","two figures falling"],    element: "fire",  zodiac: "Aries",      planet: "Mars",    numerology: 7 },
  { cardName: "The Star",            cardNumber: 17, archetype: "The Renewed Hope",       coreSymbols: ["kneeling figure","seven small stars","one great star","two urns"],   element: "air",   zodiac: "Aquarius",   planet: "Uranus",  numerology: 8 },
  { cardName: "The Moon",            cardNumber: 18, archetype: "The Deep Unconscious",   coreSymbols: ["moon","two towers","wolf and dog","crayfish","winding path"],       element: "water", zodiac: "Pisces",     planet: "Neptune", numerology: 9 },
  { cardName: "The Sun",             cardNumber: 19, archetype: "The Radiant Joy",        coreSymbols: ["sun with face","child on horse","sunflowers","banner","wall"],      element: "fire",  zodiac: "Leo",        planet: "Sun",     numerology: 1 },
  { cardName: "Judgement",           cardNumber: 20, archetype: "The Great Awakening",    coreSymbols: ["angel with trumpet","rising figures","cross banner","mountains"],    element: "fire",  zodiac: "Scorpio",    planet: "Pluto",   numerology: 2 },
  { cardName: "The World",           cardNumber: 21, archetype: "The Complete Cycle",     coreSymbols: ["dancing figure","laurel wreath","four fixed signs","two wands"],     element: "earth", zodiac: "Capricorn",  planet: "Saturn",  numerology: 3 },
];
```

### 3b. The 5 Companion Cards

`lib/companion-cards.ts`:

```typescript
import { NumerologyProfile } from "./numerology";
import { NUMBER_PROFILES, KARMIC_DEBT_PROFILES } from "./numerology-mappings";
import { CardBrief } from "./types";

export function buildCompanionCards(profile: NumerologyProfile): CardBrief[] {
  const cards: CardBrief[] = [];

  // 1. The Signature Card — Life Path
  const sig = NUMBER_PROFILES[profile.lifePath];
  cards.push({
    cardName: `The Signature — ${sig.archetype}`,
    cardNumber: `Life Path ${profile.lifePath}`,
    archetype: sig.archetype,
    coreSymbols: sig.symbolMotifs,
    element: sig.element,
    zodiac: sig.zodiac,
    planet: sig.planet,
    numerology: profile.lifePath,
  });

  // 2. The Voice Card — Expression
  const voice = NUMBER_PROFILES[profile.expression];
  cards.push({
    cardName: `The Voice — ${voice.archetype}`,
    cardNumber: `Expression ${profile.expression}`,
    archetype: `Your outward gift: ${voice.keyword}`,
    coreSymbols: [...voice.symbolMotifs.slice(0, 3), "open mouth", "outstretched hand"],
    element: voice.element,
    zodiac: voice.zodiac,
    planet: voice.planet,
    numerology: profile.expression,
  });

  // 3. The Heart Card — Heart Desire
  const heart = NUMBER_PROFILES[profile.heartDesire];
  cards.push({
    cardName: `The Heart — ${heart.archetype}`,
    cardNumber: `Heart Desire ${profile.heartDesire}`,
    archetype: `Your inner longing: ${heart.keyword}`,
    coreSymbols: [...heart.symbolMotifs.slice(0, 3), "chalice", "flame within"],
    element: heart.element,
    zodiac: heart.zodiac,
    planet: heart.planet,
    numerology: profile.heartDesire,
  });

  // 4. The Shadow Card — Karmic Debt if any, else Challenge 1
  const debt = profile.lifePathKarmicDebt
            ?? profile.expressionKarmicDebt
            ?? profile.heartDesireKarmicDebt
            ?? profile.personalityKarmicDebt;

  if (debt && KARMIC_DEBT_PROFILES[debt]) {
    const dp = KARMIC_DEBT_PROFILES[debt];
    cards.push({
      cardName: `The Shadow — ${dp.name}`,
      cardNumber: `Karmic Debt ${debt}`,
      archetype: dp.teaching,
      coreSymbols: [dp.symbol, "veil", "half-lit face"],
      element: "water",
      numerology: debt,
    });
  } else {
    const challenge = profile.challenges[0];
    const cp = NUMBER_PROFILES[challenge];
    cards.push({
      cardName: `The Shadow — ${cp.shadowArchetype}`,
      cardNumber: `First Challenge ${challenge}`,
      archetype: cp.shadowLine,
      coreSymbols: [...cp.symbolMotifs.slice(0, 2), "veil", "shadow figure"],
      element: cp.element,
      numerology: challenge,
    });
  }

  // 5. The Path Card — Personal Year (regenerates annually)
  const path = NUMBER_PROFILES[profile.personalYear];
  cards.push({
    cardName: `The Path — ${path.keyword} Year`,
    cardNumber: `Personal Year ${profile.personalYear}`,
    archetype: `This year's invitation: ${path.giftLine}`,
    coreSymbols: [...path.symbolMotifs.slice(0, 3), "open door", "path forward"],
    element: path.element,
    zodiac: path.zodiac,
    planet: path.planet,
    numerology: profile.personalYear,
  });

  return cards;
}
```

---

## 4. The personalization decision — how numerology configures the deck

Given a full profile, we derive a `DeckSelection` that's ready for v2's generator.

`lib/personalize-deck.ts`:

```typescript
import { NumerologyProfile } from "./numerology";
import { NUMBER_PROFILES } from "./numerology-mappings";
import { DeckSelection, ArtisticDirection, AestheticPack, PhilosophyTradition } from "./types";

export function personalizeDeckSelection(
  profile: NumerologyProfile,
  userOverrides: Partial<DeckSelection> = {}
): DeckSelection {
  const sig = NUMBER_PROFILES[profile.lifePath];

  const base: DeckSelection = {
    deckType: "hybrid",             // major arcana + companion cards
    deckSize: "custom_oracle",
    customCardCount: 27,
    creationMode: "prompt_only",

    // Style pulled from Life Path, but respect explicit user override
    artisticDirection: (sig.suggestedArtisticDirection as ArtisticDirection),
    aestheticPack: (sig.suggestedAestheticPack as AestheticPack),

    // Reflective layers — all on for personalized decks
    philosophyEnabled: true,
    philosophyLayer: (sig.philosophy as PhilosophyTradition),
    numerologyEnabled: true,
    astrologyEnabled: true,
    chakraSystemEnabled: false,

    // Character consistency ON — the deck has a recurring archetype
    characterConsistency: true,
    characterAnchor: {
      archetype: `${sig.archetype.toLowerCase()}, embodying ${sig.keyword.toLowerCase()}`,
      facialProfile: "expressive face aligned with the archetype, gender-neutral by default unless user specifies otherwise",
      signatureItems: sig.symbolMotifs.slice(0, 3),
    },

    tier: "creator",                // default; upgrade paths to marketplace
    userBirthDate: userOverrides.userBirthDate,
    userFullName: userOverrides.userFullName,
  };

  return { ...base, ...userOverrides };
}
```

The user can override anything on the final review screen — but the defaults are theirs.

---

## 5. The onboarding flow

Not a form dump. A conversation.

### Step-by-step UX

**Screen 1 — Invitation**

> Every deck we make can be shaped by who you are.
>
> Enter your birth date and full birth name, and Arcana Studio will craft a 27-card personal oracle drawn from your numerological blueprint.
>
> Your data stays private. Nothing is shared.
>
> [Begin]

**Screen 2 — Birth date**

Date picker.

> Your birth date carries your Life Path — the shape of your journey.

**Screen 3 — Full birth name**

Free text.

> Your birth name carries three more numbers: Expression, Heart Desire, and Personality. Use the name on your birth certificate for the fullest reading. If your name has changed, we can regenerate later.

**Screen 4 — Reveal (animated, one number at a time)**

For each of Life Path, Birth Day, Expression, Heart Desire, Personality, Maturity:

> Your Life Path Number is **[N]** — **[keyword]**
>
> _[gift line]_
>
> The shadow to hold: _[shadow line]_

If a Karmic Debt was detected, a special reveal:

> A Karmic Debt has surfaced in your numbers: **[N]** — **[name]**
>
> _[teaching]_
>
> Your deck will include this as a Shadow Card. Meet it directly.

**Screen 5 — Signature preview**

Before generation, show a placeholder card:

> Your Signature Archetype is **The [Archetype]**.
>
> This character will appear across your deck — a recurring figure who embodies your Life Path.
>
> Style: **[Suggested Direction]** with **[Aesthetic Pack]** overlay.
> Philosophy tradition: **[Philosophy]**
>
> [Confirm and Generate] · [Adjust Style]

**Screen 6 — Adjust style (optional)**

Full style engine dropdowns from v2, pre-populated with the suggestions but freely changeable.

**Screen 7 — Generation**

Progress screen:

> Weaving your deck...
>
> Card 3 of 27 — The High Priestess
>
> _[soft animated progress]_

Full deck takes ~5-10 minutes with LoRA training (first-time users) or ~90 seconds with LoRA cached (return users regenerating annual Path Card).

**Screen 8 — Deck reveal**

Grid of all 27 cards. Signature Card highlighted with a subtle glow. Tap any card to view meanings.

---

## 6. The API route

`app/api/personalize-deck/route.ts`:

```typescript
import { NextRequest, NextResponse } from "next/server";
import { buildNumerologyProfile } from "@/lib/numerology";
import { personalizeDeckSelection } from "@/lib/personalize-deck";
import { MAJOR_ARCANA } from "@/lib/major-arcana";
import { buildCompanionCards } from "@/lib/companion-cards";
import { supabase } from "@/lib/supabase";
import { v4 as uuid } from "uuid";
import { generatePersonalizedCard } from "@/lib/personalized-generator";

export async function POST(req: NextRequest) {
  try {
    const { birthDate, fullName, styleOverrides } = await req.json();

    if (!birthDate || !fullName) {
      return NextResponse.json({ error: "birthDate and fullName are required" }, { status: 400 });
    }

    // ── 1. Compute the numerology profile ─────────────────────
    const profile = buildNumerologyProfile(birthDate, fullName);

    // ── 2. Derive the personalized DeckSelection ──────────────
    const deck = personalizeDeckSelection(profile, {
      userBirthDate: birthDate,
      userFullName: fullName,
      ...styleOverrides,
    });

    // ── 3. Persist the deck record ────────────────────────────
    const deckId = uuid();
    await supabase.from("decks").insert({
      id: deckId,
      user_id: (await getCurrentUser(req)).id,
      selection: deck,
      numerology_profile: profile,
      deck_type: "life_path_personalized",
      created_at: new Date().toISOString(),
      // Path Card regeneration marker
      path_card_generated_for_year: new Date().getFullYear(),
    });

    // ── 4. Build the full card list ───────────────────────────
    const majorArcana = MAJOR_ARCANA;
    const companions = buildCompanionCards(profile);
    const allCards = [...majorArcana, ...companions];

    // ── 5. Kick off generation (async — returns job id) ───────
    const jobId = uuid();
    await supabase.from("generation_jobs").insert({
      id: jobId,
      deck_id: deckId,
      total_cards: allCards.length,
      completed_cards: 0,
      status: "queued",
    });

    // Fire-and-forget the background worker
    startBackgroundGeneration(jobId, deckId, deck, allCards, profile);

    return NextResponse.json({
      deckId,
      jobId,
      profile,
      cardCount: allCards.length,
      estimatedMinutes: 8,
    });

  } catch (err: any) {
    console.error(err);
    return NextResponse.json({ error: err.message }, { status: 500 });
  }
}

/**
 * Background worker. In production, use a proper queue (Inngest, Trigger.dev,
 * or Vercel Cron). For MVP, an async fn kicked off from the route works.
 */
async function startBackgroundGeneration(
  jobId: string,
  deckId: string,
  deck: DeckSelection,
  cards: CardBrief[],
  profile: NumerologyProfile
) {
  try {
    // Step A. Generate 12 seed cards for LoRA training
    const seedCards = cards.slice(0, 12);
    const seedImages: string[] = [];
    for (const card of seedCards) {
      const url = await generatePersonalizedCard(deck, card, profile, { useLoRA: false });
      seedImages.push(url);
    }

    // Step B. Train the style LoRA
    const styleLoraUrl = await trainStyleLoRA(deckId, seedImages);
    await supabase.from("decks")
      .update({ "selection->deckDNA->>styleLoraUrl": styleLoraUrl })
      .eq("id", deckId);

    // Step C. Train the character LoRA if enabled
    if (deck.characterConsistency) {
      // Use 6 character-focused renders from seed batch
      const charImages = seedImages.slice(0, 6);
      const charLoraUrl = await trainCharacterLoRA(deckId, charImages);
      await supabase.from("decks")
        .update({ "selection->deckDNA->>characterLoraUrl": charLoraUrl })
        .eq("id", deckId);
    }

    // Step D. Regenerate seed cards with LoRA for consistency
    // Step E. Generate remaining cards with LoRA
    const deckWithLoras = await refetchDeckWithLoras(deckId);
    const remaining = cards.slice(0);
    let completed = 0;
    for (const card of remaining) {
      const url = await generatePersonalizedCard(deckWithLoras, card, profile, { useLoRA: true });
      await supabase.from("cards").insert({
        deck_id: deckId,
        card_brief: card,
        image_url: url,
      });
      completed++;
      await supabase.from("generation_jobs")
        .update({ completed_cards: completed })
        .eq("id", jobId);
    }

    await supabase.from("generation_jobs")
      .update({ status: "complete", completed_at: new Date().toISOString() })
      .eq("id", jobId);

  } catch (err: any) {
    await supabase.from("generation_jobs")
      .update({ status: "failed", error: err.message })
      .eq("id", jobId);
  }
}
```

---

## 7. The personalized meaning prompt

The v2 `assembleMeaningPrompt` gets an override for personalized decks — meanings reference the user's specific numbers.

`lib/personalized-generator.ts` (excerpt):

```typescript
export function assemblePersonalizedMeaningPrompt(
  deck: DeckSelection,
  card: CardBrief,
  profile: NumerologyProfile
): string {
  return `
You are the resident scholar of Arcana Studio, writing a meaning package for a personalized deck.

The reader is walking a specific numerological path. Reference their profile subtly — never generically. Every meaning should feel written for them.

THEIR PROFILE
- Life Path: ${profile.lifePath} (${NUMBER_PROFILES[profile.lifePath].keyword})
- Expression: ${profile.expression} (${NUMBER_PROFILES[profile.expression].keyword})
- Heart Desire: ${profile.heartDesire} (${NUMBER_PROFILES[profile.heartDesire].keyword})
- Personality: ${profile.personality}
- Maturity: ${profile.maturity}
- Current Personal Year: ${profile.personalYear} (${NUMBER_PROFILES[profile.personalYear].keyword})
${profile.lifePathKarmicDebt ? `- Karmic Debt: ${profile.lifePathKarmicDebt}` : ""}
${profile.karmicLessons.length ? `- Karmic Lessons (missing numbers): ${profile.karmicLessons.join(", ")}` : ""}

CARD CONTEXT
- Card name: ${card.cardName}
- Card number: ${card.cardNumber}
- Archetype: ${card.archetype}
- Core symbols: ${card.coreSymbols.join(", ")}
${card.element ? `- Element: ${card.element}` : ""}
${card.zodiac ? `- Zodiac: ${card.zodiac}` : ""}

INSTRUCTIONS
- Weave references to their Life Path or Personal Year into the light and shadow meanings where meaningful.
- The journal prompt should feel written for someone on Life Path ${profile.lifePath}.
- The philosophy reflection should draw from the ${NUMBER_PROFILES[profile.lifePath].philosophy} tradition (their Life Path's home tradition).
- Never dump their numbers as a list. Weave them.
- Never predictive. Never diagnostic. Reflection framing throughout.

OUTPUT SHAPE (JSON only)
{
  "keywords": [string, string, string, string, string],
  "lightMeaning": string,
  "shadowMeaning": string,
  "reversedMeaning": string,
  "journalPrompt": string,
  "affirmation": string,
  "meditation": string,
  "numerology": string,
  "astrology": string,
  "philosophy": { "thinker": string, "tradition": string, "reflection": string },
  "personalResonance": string
}

The "personalResonance" field is a 2-sentence note specifically addressing why this card matters for someone on Life Path ${profile.lifePath} in Personal Year ${profile.personalYear}.
  `.trim();
}
```

The `personalResonance` field is the money field. It's what makes users feel seen. It's what turns a $9 personal deck into a $49 lifetime keepsake.

---

## 8. Annual Path Card regeneration

The Path Card regenerates every birthday. This is the retention hook.

### Cron job

`app/api/cron/regenerate-path-cards/route.ts`:

```typescript
// Runs daily. Finds users whose birthday is today AND who have a personalized deck.
// Regenerates their Path Card for the new Personal Year.

export async function GET(req: NextRequest) {
  const today = new Date();
  const monthDay = `${String(today.getMonth() + 1).padStart(2, "0")}-${String(today.getDate()).padStart(2, "0")}`;

  const { data: eligibleDecks } = await supabase
    .from("decks")
    .select("id, user_id, selection, numerology_profile")
    .eq("deck_type", "life_path_personalized")
    .lt("path_card_generated_for_year", today.getFullYear())
    .filter("numerology_profile->>birthDateMonthDay", "eq", monthDay);

  for (const deck of eligibleDecks ?? []) {
    // Rebuild profile with today's date
    const newProfile = buildNumerologyProfile(
      deck.selection.userBirthDate,
      deck.selection.userFullName,
      today,
    );
    const newCompanions = buildCompanionCards(newProfile);
    const pathCard = newCompanions[4]; // The Path Card is index 4

    const url = await generatePersonalizedCard(deck.selection, pathCard, newProfile, { useLoRA: true });

    // Update the existing Path Card record
    await supabase.from("cards")
      .update({
        image_url: url,
        card_brief: pathCard,
        regenerated_at: today.toISOString(),
      })
      .eq("deck_id", deck.id)
      .eq("card_brief->>cardNumber", `Personal Year ${newProfile.personalYear}`);

    await supabase.from("decks")
      .update({ path_card_generated_for_year: today.getFullYear() })
      .eq("id", deck.id);

    // Send notification
    await sendBirthdayEmail(deck.user_id, newProfile.personalYear);
  }

  return NextResponse.json({ regenerated: eligibleDecks?.length ?? 0 });
}
```

### The birthday email

Subject: **Your new Path Card is ready — Personal Year [N]**

Body:
> Today you enter Personal Year **[N]** — [keyword].
>
> _[gift line]_
>
> Your Path Card has been redrawn. Open the deck to meet it.
>
> [Open my deck]

This is your annual retention event. Users who bought a personal deck 3 years ago still get a new card every year. That's a lifetime relationship.

---

## 9. Cost per personalized deck

| Component | Cost |
|---|---|
| 12 seed images @ FLUX Schnell | $0.04 |
| Style LoRA training (1000 steps) | $8.00 |
| Character LoRA training (800 steps) | $6.40 |
| 27 cards via stacked LoRA | $0.14 |
| 27 personalized meaning packages | $0.30 |
| 1 card back | $0.005 |
| Numerology computation | $0 (pure functions) |
| **Total first generation** | **~$14.89** |

**Annual Path Card regeneration:** ~$0.02 per year per user (one image + one meaning).

### Pricing rationale

At $14.89 in compute, a personalized deck sold at:
- **$49 one-time** = 3.3× margin
- **$99 one-time (with printed physical deck via print-on-demand)** = still healthy after ~$18 print+ship
- **Studio subscription** ($49.99/mo) — one personalized deck justifies the first month's fee

The annual Path Card at $0.02 is essentially free — it's pure retention.

---

## 10. The marketing hook (what to write on the landing page)

> **A deck as unique as your birth chart.**
>
> Enter your birth date. Enter your birth name.
>
> Arcana Studio calculates your Life Path, Expression, Heart Desire, and Personality, then crafts a 27-card personal oracle drawn from your numerological blueprint — 22 Major Arcana filtered through the shape of your journey, plus 5 companion cards unique to you:
>
> — Your **Signature Card** (your Life Path archetype, yours for life)
> — Your **Voice Card** (how you're here to express)
> — Your **Heart Card** (what your soul longs for)
> — Your **Shadow Card** (the pattern you're here to meet)
> — Your **Path Card** (this year's invitation, regenerated every birthday)
>
> Every card carries a philosophical reflection drawn from the tradition that matches your Life Path — Stoic, Confucian, Hermetic, or Platonic — because insight without practice is just decoration.
>
> Your data stays private. Your deck stays yours.
>
> **From $49 one-time. Every birthday, a new Path Card, free.**

---

## 11. Non-negotiable framing (again, always)

Add to the onboarding flow, the About page, and every personalized reading interface:

> Numerology and tarot are reflective symbolic systems. Nothing in your Arcana Studio deck predicts the future, diagnoses conditions, or replaces professional advice. Your deck is a mirror for insight — no more, no less.

This matters more for personalized decks than any other product tier, because the personalization creates a stronger illusion of accuracy. Frame reflection, not prediction, from the first screen.

---

## 12. The two decisions this unlocks

**1. Do you want Life Path personalization to be Studio-only, or an add-on purchase for lower tiers?**
- Studio-only: cleaner ladder, drives subscription upgrades.
- Add-on: broader access, one-time revenue.

Recommendation: **one-time $49 purchase available at all tiers**, plus **unlimited regenerations included with Studio**. Studio users can regenerate with different name variants (birth name, married name, chosen name) as their identity shifts. That's a meaningful benefit and a real reason to subscribe.

**2. Do you want the Signature Card printable as merch (mug, art print, tarot-sized art card) via print-on-demand?**
- Yes: the Signature Card becomes physical, personalized, sharable. Massive gifting angle.
- Prices at $19–$39 depending on format via Printful or Printify.

This is a natural upsell after the reveal screen: "Your Signature Card as a framed print — [Order]". Some users will only ever buy the print. That's fine. It's the same margin.

---

## 13. Roadmap slot

This slots into Phase 5 of the v2 roadmap but is meaty enough to launch as its own campaign:

- **Weeks 25-26**: Numerology engine + mappings (pure functions, unit tests)
- **Weeks 27-28**: Personalization decision + Major Arcana briefs + Companion generation
- **Weeks 29-30**: Onboarding flow UI + reveal animations
- **Weeks 31-32**: API route + background job worker + LoRA training pipeline
- **Weeks 33-34**: Annual regeneration cron + birthday email
- **Week 35**: Print-on-demand integration for Signature Card merch
- **Week 36**: Marketing campaign launch, coordinated with a birthday/new-year moment

Ship the numerology engine as a free public tool ("What's my Life Path?" calculator) *before* the deck feature launches. It's free lead-gen for the personalized deck, and it's a real utility that gets shared.
