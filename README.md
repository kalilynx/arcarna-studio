 (cd "$(git rev-parse --show-toplevel)" && git apply --3way <<'EOF' 
diff --git a/README.md b/README.md
index 3eb830de661b1ccfbcd21e929be2da28a4caae6e..1b2d523b9819bd53336c05f26baaecf1c506ae18 100644
--- a/README.md
+++ b/README.md
@@ -1,38 +1,51 @@
-# Arcarna Studio
+# Arcana Studio
 
-Arcarna Studio is an AI Tarot / Oracle / Angel Card Generator Studio for creating cohesive, publishable spiritual card decks from reference images, inspiration prompts, and reusable deck style DNA.
+Arcana Studio is an AI Tarot / Oracle / Angel Card Generator Studio for creating cohesive, publishable spiritual card decks from reference images, inspiration prompts, and reusable deck style DNA.
 
 ## Feature branch
 
 Suggested branch: `aitarot-ai-generator`
 
 ## Core workflow
 
 1. Create Card
 2. Choose upload, written inspiration, or AI Inspire Me
 3. Analyze reference image and creative intent
 4. Match the request to a genre and visual style
 5. Optimize the final generation prompt
 6. Generate card artwork through FLUX or Stable Diffusion providers
 7. Save the result into a consistent deck system
 
+## Recommended implementation milestones
+
+1. Repository scaffolding with Next.js 15, TypeScript, Tailwind, and shared UI primitives.
+2. Supabase authentication, database migrations, storage buckets, and row-level security policies.
+3. Deck Builder workspace for card metadata, reference uploads, style selection, and Deck DNA presets.
+4. Dynamic prompt engine with reference-image analysis, hidden style rules, negative prompts, and prompt scoring.
+5. FLUX image generation pipeline with provider abstraction, job history, retries, and enhancement tools.
+6. Claude-powered meaning generation for tarot, oracle, angel, numerology, and philosophy decks.
+7. Gallery, card export, print-safe previews, and deck packaging tools.
+8. Marketplace submission flow with copyright provenance, C2PA metadata, licensing, and moderation.
+9. Admin dashboard for users, generations, marketplace review, telemetry, and provider health.
+10. Automated tests, preview deployments, production deployment, and operational runbooks.
+
 ## Repository structure
 
 - `docs/` — product, API, prompt engine, and symbolism architecture.
 - `src/components/` — React component skeletons for the generator studio.
 - `src/ai/prompt-engine/` — prompt construction, style rules, negative prompts, and Deck DNA.
 - `src/ai/image-generation/` — provider client abstractions and enhancement pipeline.
 - `src/ai/interpretation/` — tarot meanings, numerology, and philosophy layers.
 - `src/database/` — Prisma schema and model notes.
 - `public/` — themes, borders, and examples for generated decks.
 
 ## Local development
 
 ```bash
 npm run start
 npm run build
 ```
 
 ## Environment
 
 Copy `.env.example` to `.env` and fill in provider keys before wiring live generation APIs.
 
EOF
)
