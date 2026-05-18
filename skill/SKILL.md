# Social News Post Pipeline

Posts an AI/tech news post to your social platforms. Image only, no video. Direct API calls, no middleware.

Default platforms are configured in `config.json` under `platforms[]`. The skill posts only to the platforms you list there.

**Do NOT use any MCP tools (Supabase MCP, etc.). Use REST APIs directly with env vars via curl.** Load credentials: `[ -f .env ] && set -a && source .env && set +a`

## When this fires

You provide a news topic or say "run the news post pipeline." Example:

> "post the Nvidia Thinking Machines story"
> "run social-news-post"

The skill fetches a headline from your configured news source, writes the post, generates the image, posts to your configured platforms, and reports the links.

---

## Step 1 — Fetch headline

Fetch the configured news source and extract the top story. Pick the one with the most concrete stats (funding amounts, valuations, percentages, named companies).

The news source URL comes from `config.json` field `news_source_url`.

```bash
NEWS_URL=$(jq -r '.news_source_url' config.json)
curl -s "$NEWS_URL"
```

Extract: headline, key facts/stats, companies involved.

---

## Step 2 — Write the post

Every sentence gets its own line with a blank line below it.

**Template:**

```
[casual hook — 1 line, lowercase, no period OR a quote from the headline]

[Fact 1 with stat.]

[Fact 2 with stat.]

[Fact 3 with stat.]

[Fact 4 with stat.]

[Fact 5 with stat.]

[Fact 6 with stat.]

[Fact 7 — implication or "so what".]

[Fact 8 — bigger picture.]

[Fact 9 — tension or question setup.]

[1-line engagement question?]

--

{{cta_line}}
```

**Rules:**
- Hook: casual or emotional opener, a quoted headline, or a short reaction line. See `voice.md` for examples and how to customize.
- Minimum 8 stat-backed facts — numbers, dollar amounts, percentages, named companies
- One sentence per paragraph, blank line between each
- Engagement question must invite opinion (not yes/no)
- Separator is `--`
- CTA line is read from `config.json` field `cta_line`. If `cta_line` is empty, omit the `--` separator and the CTA entirely.
- No hashtags
- No corporate or AI voice
- No em dashes. Use commas, periods, or rewrite the sentence instead.
- No "X isn't just Y — it's Z" or "It's not about X. It's about Y." sentence structures. These are AI-voice clichés. Write like a human.

**X version:** Same post but strip the `--` and CTA line (external links tank reach on X). After posting, reply to your own post with the CTA line from config, if `cta_url` is set:

```
try it → {{cta_url}}?utm_source=x&utm_medium=social&utm_campaign=news
```

If `cta_url` is empty, skip the reply step entirely.

**Threads version:** Conversational, casual, short. End with the CTA URL if `cta_url` is set.

```
[1-2 sentence casual take on the story]

{{cta_url}}?utm_source=threads&utm_medium=social&utm_campaign=news
```

If `cta_url` is empty, omit the trailing URL.

**Pinterest version:** Same image as other platforms. No caption needed — Pinterest uses the title field only. Board ID is read from `config.json` field `pinterest_board_id`. If `pinterest_board_id` is empty, skip Pinterest entirely.

---

## Step 3 — Generate image

Use `gpt-image-2` model. Save to `/tmp/social-news-post.png`.

Image size and quality come from `config.json` fields `image_size` and `image_quality`.

**Image prompt (verbatim — do not change except INPUT):**

```
Generate a featured blog image for a professional technology or finance article.

INPUT:
- Blog title: [insert the news headline here]

ENTITY DETECTION (STRICT):
- Parse the blog title and extract ONLY entities that are explicitly named.
- Do NOT infer, assume, or introduce additional entities.
- Treat entities as:
  - Company (corporation, startup, platform, product brand)
  - Country (sovereign nation explicitly named)

ENTITY RULES:
1. If EXACTLY ONE company is explicitly named:
   - Show ONLY that company's official logo or wordmark
   - Display it flat on a clean background OR mounted on a real office building
   - Do NOT split the image. Do NOT include flags, maps, or other entities.

2. If EXACTLY ONE country is explicitly named and NO company is named:
   - Show ONLY the official national flag of that country
   - Do NOT split the image.

3. If AND ONLY IF one company AND one country are BOTH explicitly named:
   - Split composition: one side company logo, other side country flag
   - Keep both sides equal in visual weight.

4. If more than one company is explicitly named:
   - Split the image into equal vertical sections, one per company (maximum 3).
   - Each section shows ONLY that company's official logo or wordmark, centered.
   - All logos are the same size and equal visual weight.
   - A thin vertical divider line separates each section.

5. If NO entities are detected:
   - Generate a clean editorial scene relevant to the article subject
   - Professional setting, neutral, factual

ABSOLUTE CONSTRAINTS:
- No illustrations
- No abstract visuals
- No metaphors
- No text overlays
- No charts, graphs, UI, or dashboards
- No glow, gradients, or stylized effects
- No additional logos, flags, or symbols

STYLE & QUALITY:
- Photorealistic or clean editorial realism
- Visual quality comparable to Reuters, Bloomberg, Financial Times, or WSJ
- Neutral, factual, non-promotional tone

FRAMING:
- Horizontal (landscape), blog hero image, clear and centered at all sizes

FINAL INSTRUCTION:
If the title contains only ONE valid entity type, the image must contain ONLY that entity.
If the title contains NO entities, produce an editorial scene.
```

```bash
source .env

IMAGE_SIZE=$(jq -r '.image_size' config.json)
IMAGE_QUALITY=$(jq -r '.image_quality' config.json)

curl -s https://api.openai.com/v1/images/generations \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d "{
    \"model\": \"gpt-image-2\",
    \"size\": \"${IMAGE_SIZE}\",
    \"quality\": \"${IMAGE_QUALITY}\",
    \"prompt\": \"[full prompt above with headline substituted into INPUT]\"
  }" | python3 -c "
import sys, json, base64
data = json.load(sys.stdin)
img = base64.b64decode(data['data'][0]['b64_json'])
open('/tmp/social-news-post.png', 'wb').write(img)
print('Image saved:', len(img), 'bytes')
"
```

---

## Step 4 — Post to configured platforms

Read `platforms[]` from `config.json`. Build a curl call with one `-F "platform[]=<name>"` flag per platform.

Use `/api/upload_photos` with `photos[]`. Direct file upload:

```bash
source .env

CAPTION=$'[caption here]'

# Build platform flags from config
PLATFORM_FLAGS=$(jq -r '.platforms[] | "-F platform[]=" + .' config.json | tr '\n' ' ')

# Build Pinterest flags only if pinterest_board_id is set
PINTEREST_BOARD=$(jq -r '.pinterest_board_id // ""' config.json)
CTA_URL=$(jq -r '.cta_url // ""' config.json)
PINTEREST_FLAGS=""
if [ -n "$PINTEREST_BOARD" ]; then
  PINTEREST_FLAGS="-F pinterest_board_id=${PINTEREST_BOARD}"
  if [ -n "$CTA_URL" ]; then
    PINTEREST_FLAGS="${PINTEREST_FLAGS} -F pinterest_link=${CTA_URL}?utm_source=pinterest&utm_medium=social&utm_campaign=news"
  fi
fi

curl -s -X POST "https://api.upload-post.com/api/upload_photos" \
  -H "Authorization: Apikey ${UPLOAD_POST_API_KEY}" \
  -F "user=${UPLOAD_POST_USER}" \
  -F "photos[]=@/tmp/social-news-post.png" \
  -F "title=${CAPTION}" \
  ${PLATFORM_FLAGS} \
  ${PINTEREST_FLAGS}
```

**If rejected**, fall back to Supabase URL (requires `SUPABASE_URL` and `SUPABASE_KEY` in `.env`):

```bash
source .env

BUCKET=$(jq -r '.supabase_bucket // "social-media"' config.json)
FILENAME="social-news-post-$(date +%s).png"

curl -s -X POST \
  "${SUPABASE_URL}/storage/v1/object/${BUCKET}/${FILENAME}" \
  -H "apikey: ${SUPABASE_KEY}" \
  -H "Authorization: Bearer ${SUPABASE_KEY}" \
  -H "Content-Type: image/png" \
  -H "x-upsert: true" \
  --data-binary "@/tmp/social-news-post.png"

IMAGE_URL="${SUPABASE_URL}/storage/v1/object/public/${BUCKET}/${FILENAME}"

curl -s -X POST "https://api.upload-post.com/api/upload_photos" \
  -H "Authorization: Apikey ${UPLOAD_POST_API_KEY}" \
  -F "user=${UPLOAD_POST_USER}" \
  -F "photos[]=${IMAGE_URL}" \
  -F "title=${CAPTION}" \
  ${PLATFORM_FLAGS} \
  ${PINTEREST_FLAGS}
```

---

## Step 5 — Check async results

If the call returns a `request_id`, poll:

```bash
curl -s "https://api.upload-post.com/api/uploadposts/status?request_id=${REQUEST_ID}" \
  -H "Authorization: Apikey ${UPLOAD_POST_API_KEY}"
```

---

## Step 6 — Report back

List all posted URLs. Example:

```
Posted to N platforms:
- LinkedIn: https://linkedin.com/...
- Threads: https://threads.net/...
```

---

## Config reference

| Key | Source | Notes |
|---|---|---|
| Upload-Post user | `${UPLOAD_POST_USER}` env | Your upload-post.com account handle |
| Upload-Post auth | `Apikey ${UPLOAD_POST_API_KEY}` | From upload-post.com dashboard |
| OpenAI auth | `${OPENAI_API_KEY}` env | From platform.openai.com |
| Image model | `gpt-image-2` | Fixed |
| Image size | `config.json` `image_size` | Default `1536x1024` |
| Image quality | `config.json` `image_quality` | Default `medium` |
| Image temp path | `/tmp/social-news-post.png` | Fixed |
| Supabase URL | `${SUPABASE_URL}` env | Optional, only for fallback |
| Supabase key | `${SUPABASE_KEY}` env | Optional, only for fallback |
| Supabase bucket | `config.json` `supabase_bucket` | Default `social-media`, must be public |
| News source | `config.json` `news_source_url` | Any URL the skill can curl |
| Platforms | `config.json` `platforms[]` | List of `linkedin`, `threads`, `pinterest`, `x`, `instagram` |
| Pinterest board ID | `config.json` `pinterest_board_id` | Empty string skips Pinterest |
| CTA URL | `config.json` `cta_url` | Empty string skips CTAs entirely |
| CTA line | `config.json` `cta_line` | Full CTA paragraph used in the main post template |
| Voice | `skill/voice.md` | Edit this to match how you write |

## Common errors

| Error | Cause | Fix |
|---|---|---|
| image upload rejected | Upload-Post requires URL not file | Fall back to Supabase URL method |
| "Invalid Compact JWS" | Missing `apikey` header on Supabase | Add `-H "apikey: ${SUPABASE_KEY}"` |
| Image looks wrong | Wrong prompt used | Use the verbatim prompt above, only swap INPUT |
| Plain white background | Background spec stripped from prompt | Use verbatim prompt — the "clean background OR mounted on office building" line handles it |
