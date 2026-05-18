# social-news-post

> **500K LinkedIn impressions in 2 months. I didn't post a single one.**
>
> [![Watch the 60-second story](https://img.youtube.com/vi/eI_V83pR6cI/hqdefault.jpg)](https://youtube.com/shorts/eI_V83pR6cI)
>
> ↑ Click to watch on YouTube

A Claude Code skill that turns AI and tech news into a posted update on your social platforms. One command, one image, one set of captions, posted everywhere you list.

## What this does

You say "run social-news-post" or "post the [topic] story" inside Claude Code. The skill pulls a headline from a news source you choose, writes the post in your voice with at least 8 stat-backed facts, generates a clean editorial image with GPT Image, and posts it to the platforms you configured. Direct REST API calls, no middleware.

## What you need

- A social account on at least one of: LinkedIn, Threads, Pinterest, X, Instagram
- An [upload-post.com](https://upload-post.com) account (paid plan, around $9/month) connected to those social accounts
- An [OpenAI API key](https://platform.openai.com) — each image costs about $0.04
- Optional: a free [Supabase](https://supabase.com) project, only as a fallback for when upload-post rejects direct file uploads
- [Claude Code](https://docs.claude.com/en/docs/claude-code) installed and signed in
- `jq`, `curl`, `python3` on your machine (standard on macOS and most Linux distros)

## Install

```bash
git clone https://github.com/YOUR_USERNAME/social-news-post.git
cd social-news-post

cp config.example.json config.json
cp .env.example .env
```

Fill in `.env` with your API keys. Fill in `config.json` with your platforms and CTA.

Then copy the skill into your Claude Code skills folder:

```bash
mkdir -p ~/.claude/skills/social-news-post
cp -r skill/* ~/.claude/skills/social-news-post/
```

## Run it

From the project folder, start Claude Code:

```bash
claude
```

Then type:

```
/social-news-post
```

Or describe what you want:

```
post the latest AI funding story
```

The skill runs end to end and reports the posted URLs back to you.

## Configure

Edit `config.json`:

| Field | What it does |
|---|---|
| `cta_url` | Your call-to-action URL. If empty, no CTA is added to any post. |
| `cta_line` | The full CTA paragraph used at the bottom of the main post. Leave empty to skip. |
| `news_source_url` | Where the skill pulls the headline from. Default is deepai.vc/signals. |
| `pinterest_board_id` | Your Pinterest board ID. If empty, Pinterest is skipped. |
| `platforms` | Array of platform names the skill posts to. Pick from `linkedin`, `threads`, `pinterest`, `x`, `instagram`. |
| `image_size` | `1536x1024` (landscape), `1024x1024` (square), or `1024x1536` (portrait). |
| `image_quality` | `low`, `medium`, or `high`. Higher quality costs more. |
| `supabase_bucket` | Bucket name for the optional fallback. Must be public. Default `social-media`. |

## Customize the voice

The skill reads `skill/voice.md` before writing any post. Open that file and edit it to match how you actually talk — hook examples, engagement question shapes, tone notes. Whatever shape you write becomes the default.

## Change the news source

Set `news_source_url` in `config.json` to any URL the skill can `curl`. A few that work well:

- AI funding and signals: `https://www.deepai.vc/signals`
- Hacker News front page: `https://news.ycombinator.com`
- Any news aggregator that renders headlines and stats in plain HTML

If the source is JavaScript-rendered and `curl` returns a stub, the skill will fall back to its WebFetch tool.

## Troubleshooting

- **Upload-post rejects the image.** Set `SUPABASE_URL` and `SUPABASE_KEY` in `.env`. The skill will host the image on Supabase Storage and pass the public URL instead.
- **Image looks generic or has no logo.** The headline didn't contain a named entity the prompt could detect. Try a more specific headline.
- **Plain white background on the image.** Make sure the skill is using the verbatim prompt in `skill/SKILL.md`. The "clean background OR mounted on office building" line is what gives the image its scene.
- **Posts don't appear on a platform.** Check that the platform is in your `platforms[]` array AND that the same account is connected on upload-post.com.

## License

MIT. See `LICENSE`.
