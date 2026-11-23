# Marketing Video Process

> **Scope.** The standard process for an app's first set of marketing videos: one short intro plus one video per headline feature, built as motion graphics timed to a recorded voiceover, then published to YouTube with titles, descriptions, thumbnails and cross-promotion. Reference implementation: **Evident** (five videos, November 2026).

Written 2026-11-22 from the Evident build. Every app in the portfolio should be able to follow this and end up with a matching set.

The one-line version: **the founder records the voice, the picture is a single HTML page rendered frame by frame, and nothing is rendered in full until a still-frame storyboard is approved.**

---

## 1. The principles behind every decision

1. **Voice first, picture second.** The script is recorded before any animation exists. Every on-screen beat is timed to a word in the recording, never the other way round.
2. **Stills before renders.** A full render takes minutes and hides layout problems in motion. Review a storyboard of still frames, fix layout there, then render once. Small spacing tweaks can go straight to render.
3. **Only claim what the product does today.** Trace each claim to working code before it goes in a script. No statistics unless they are ours and current. Numbers inside mock-ups are examples and should look like examples.
4. **The audience is the buyer.** For our apps that is the merchant or operator, not their customer. Open on their problem.
5. **One look across the series.** Same splash, same closing card, same fonts and palette, same music, same thumbnail system. A viewer who sees two videos should know they belong together.
6. **Say what was not checked.** The build agent cannot hear audio. Every audio change is reported as "made, not listened to" and a person listens before it ships.

---

## 2. What to make

| Video | Length | Job |
|---|---|---|
| **Intro** | 35 to 40 s | The problem the app removes and the one-sentence answer. Ends on platforms supported and "bring your data with you". |
| **One per headline feature** | 55 to 80 s | What the feature is for, how it works, who stays in control, how to migrate in. Ends by naming the other features. |

Evident's set: Intro, Reviews, Rewards, Galleries, FAQs. Pick the features a buyer would search for by name.

Publish the intro first, then the features over a couple of hours on the same day.

---

## 3. The stack

| Piece | What it is |
|---|---|
| **`promo.html`** | The whole animation. One 1920x1080 page with a `render(t)` function. Elements carry their own timing as `data-in` / `data-out` seconds. |
| **`render.mjs`** | Playwright opens the page, calls `render(t)` for every frame at 30 fps, and pipes PNG screenshots into ffmpeg. Also writes single stills. |
| **`build.sh`** | Renders the picture, then mixes polished voice and music. `./build.sh mix` remixes without re-rendering. |
| **ffmpeg** | Encoding, voice polish, music edit, final mix, loudness. |
| **faster-whisper** | Word-level timestamps for the voiceover recording (`small.en`, CPU). |
| **Recording app** | Anything that produces a clean mono or dual-mono track. Evident used Ecamm. |
| **Music** | One track for the whole series. Evident generated candidates with the ElevenLabs Music API and kept one. |
| **YouTube Studio** | Publishing. Driven through a browser session signed in as the channel owner. |
| **vidIQ (optional)** | Keyword research and title scoring. Its channel connection is one channel per plan, so do not rely on it for writes to a second channel. |

Each video is a self-contained folder:

```
marketing-videos/<app>-<video>/
  promo.html  render.mjs  build.sh  README.md
  vo.wav  vo-polished.wav
  music/            the series track, edited to this video's length
  recording/        original recording, full.wav, full16k.wav, transcript.json, transcribe.py
  stills/           times.txt, sheet.html, storyboard.jpg
  renders/          <name>-no-music.mp4, <name>-final.mp4
  logo.svg  icon.svg  fonts
```

Requirements: `node`, `ffmpeg`, Playwright (any local install; import it by path), Python 3.12 with `faster-whisper` and `numpy`.

---

## 4. Stage 1. Script

Write the script as short spoken lines, one idea each. A line becomes one headline on screen, so if it does not fit in about ten words it is two lines.

**Feature video shape** (each beat is one to three lines):

1. **Hook.** The buyer's problem, stated plainly. *"Every product page has a question that stops the sale."*
2. **Why it matters.** Two or three consequences.
3. **The turn.** *"That's what [App] [Feature] is for."* Use the same sentence in every video.
4. **How it works.** The main flow in order, as the user meets it.
5. **Control.** What the owner decides and approves. *"Nothing goes live until you approve it."*
6. **Fits what you have.** Integrations, placement options, performance.
7. **Migration.** *"Already have this somewhere else? Import it."*
8. **Hand-off.** *"And [feature] is just the start. [Other features]. All in one app."*
9. **End card.** Logo and the site address. No voice needed.

**Intro shape:** the pile-up of separate tools, what it costs, the single answer, what it covers, platforms, migration.

Rules:

- Benefits get the accent colour; problem statements and questions do not (see section 7).
- No specific statistics.
- Check every claim against the product. If a feature works on one platform only, the script must not imply otherwise.
- Read it aloud before recording. Cut anything that is hard to say.

---

## 5. Stage 2. Record and transcribe

1. The founder records the whole script in one sitting. Restarts and second takes are fine; leave them in the file.
2. Extract audio: a full-quality WAV for the mix and a 16 kHz mono WAV for transcription.
3. Transcribe with word timestamps. Load the audio as a numpy array (passing a file path can fail inside the decoder):

```python
import json, sys, wave, numpy as np
from faster_whisper import WhisperModel
m = WhisperModel("small.en", device="cpu", compute_type="int8")
wf = wave.open(sys.argv[1])
a = np.frombuffer(wf.readframes(wf.getnframes()), np.int16).astype(np.float32) / 32768
segs, _ = m.transcribe(a, word_timestamps=True, vad_filter=False,
                       initial_prompt="<app name, feature names, platform names, domain>")
out = [{"start": s.start, "end": s.end, "text": s.text,
        "words": [{"w": w.word, "s": w.start, "e": w.end} for w in s.words]} for s in segs]
json.dump(out, open(sys.argv[2], "w"), indent=1)
```

4. Where a take was restarted, the model can repeat or invent text. Transcribe each take on its own time range and compare.
5. Choose the takes and write down the mapping in the README: recording seconds to video seconds.

---

## 6. Stage 3. Cut and polish the voice

**Cutting.** Leave about 1.3 to 1.6 s before the first word for the splash. Cut between sentences only.

- A cut placed at the edge of detected silence can clip the tail of a word. Measure the level in 50 ms steps and cut where it has fallen to the room floor.
- When two pieces must join tightly, overlap the tail of the first under the start of the second (`amix`) instead of sliding the timeline. Picture timing then stays put.

**Polish chain** (ffmpeg audio filters, applied before cutting):

```
volume=<gain>dB,highpass=f=80,
equalizer=f=140:t=q:w=0.9:g=1.5,equalizer=f=320:t=q:w=1.0:g=-2.5,
equalizer=f=3500:t=q:w=1.2:g=2.5,highshelf=f=10000:g=1.5,
acompressor=threshold=-20dB:ratio=3:attack=8:release=120:makeup=2.5,
deesser=i=0.4:m=0.5:f=0.5:s=o
```

then, after cutting, `loudnorm=I=-16:TP=-1.5:LRA=11`.

- `<gain>` brings every recording to the same peak before the compressor, so the videos match. Evident used a peak of about -16.8 dB.
- Do not add noise reduction (`afftdn`) to a recording that is already clean. On a quiet recording it produced audible fuzz and a hollow echo.
- Produce a 30-second raw and polished pair for the founder to compare by ear.

---

## 7. Stage 4. Build the picture

**Engine.** Elements declare when they appear and leave; one function sets every element's state for a given time. Because the page is a pure function of `t`, any frame can be rendered alone, which is what makes stills cheap.

```html
<div class="headline side" data-in="12.8" data-out="15.2">Evident puts it all into <b>one fast app.</b></div>
```

```js
const cl = x => Math.max(0, Math.min(1, x));
const p  = (t, a, d) => cl((t - a) / d);            // progress of an animation starting at a, lasting d
const eo = x => 1 - Math.pow(1 - x, 3);             // ease out
const timed = [...document.querySelectorAll('[data-in]')].map(el => ({
  el, a: +el.dataset.in, b: el.dataset.out ? +el.dataset.out : 1e9 }));

function render(t) {
  for (const o of timed) {
    const i = p(t, o.a, .5), out = p(t, o.b, .3);
    o.el.style.opacity = eo(i) * (1 - out);
    o.el.style.transform = `translateY(${(1 - eo(i)) * 26 - out * 14}px)`;
  }
  // scene-specific motion goes here, each also written as a function of t
}
render(0);
```

Evident's page adds a few more attributes on the same idea: `data-pop` (scale in), `data-dx` (slide sideways), `data-fill` (stars lighting up), `data-wire` (a line drawing itself), `data-gw` (a bar growing), `data-type` (text typing out). Add only what a video needs.

**Layout that worked:**

- 1920x1080. Light canvas with a faint dot grid. Product mock-ups on white cards with soft shadows.
- One headline at a time, in a fixed column, timed to the line being spoken.
- Mock-ups are simplified drawings of the real admin and storefront screens, not screenshots. Use the app's own palette and fonts.
- Splash at the start (logo, about 1.4 s). Closing card at the end: dark background, logo, a one-line call to action with the domain, held for about five seconds.

**Accent rule.** In a headline, the emphasised phrase is set in the serif italic. Brand colour for a benefit. Ink (black) only for a problem statement or a question. Keep this consistent; a benefit in black reads as a mistake.

**Per-app tokens to swap:** canvas, ink, muted, line, brand, brand-soft, success, warning, danger, dark; two fonts (a sans for everything, a serif italic for emphasis); `logo.svg` and `icon.svg`; the domain on the closing card.

**CSS traps met:** a short class name colliding with an existing compound selector; clearing `style.borderColor` on an element whose border was set with the inline `border` shorthand turns it dark (put the border in a class).

---

## 8. Stage 5. Storyboard review

1. List one timestamp per beat in `stills/times.txt`.
2. `node render.mjs stills <t1,t2,...>` writes one PNG per timestamp.
3. Lay them out on a numbered contact sheet and save as `storyboard.jpg`.
4. The founder reviews by board number. Fix layout, re-shoot only the boards that changed.
5. Full render only after the storyboard is approved.

Typical notes at this stage: an element fitting too tightly, an accent in the wrong colour, a mock-up image with the wrong proportions. All three are invisible in a script and obvious in a still.

---

## 9. Stage 6. Render and mix

`render.mjs`:

```js
import { chromium } from '<path to playwright>/index.mjs';
import { spawn } from 'node:child_process';
const [,, mode, arg] = process.argv;
const FPS = 30, DUR = 58.2;                       // set per video
const browser = await chromium.launch();
const page = await browser.newPage({ viewport: { width: 1920, height: 1080 } });
await page.goto(new URL('./promo.html', import.meta.url).href);
await page.evaluate(() => document.fonts.ready);
if (mode === 'stills') {
  for (const t of arg.split(',').map(Number)) {
    await page.evaluate(t => render(t), t);
    await page.screenshot({ path: `still_${t.toFixed(2)}.png` });
  }
} else {
  const ff = spawn('ffmpeg', ['-hide_banner', '-loglevel', 'error', '-y', '-f', 'image2pipe', '-framerate', String(FPS), '-i', '-',
    '-i', new URL('./vo.wav', import.meta.url).pathname,
    '-c:v', 'libx264', '-preset', 'slow', '-crf', '18', '-pix_fmt', 'yuv420p', '-c:a', 'aac', '-b:a', '192k',
    '-shortest', '-movflags', '+faststart', arg], { stdio: ['pipe', 'inherit', 'inherit'] });
  for (let f = 0; f < Math.round(DUR * FPS); f++) {
    await page.evaluate(t => render(t), f / FPS);
    const buf = await page.screenshot({ type: 'png' });
    if (!ff.stdin.write(buf)) await new Promise(r => ff.stdin.once('drain', r));
  }
  ff.stdin.end();
  await new Promise(r => ff.on('close', r));
}
await browser.close();
```

`build.sh` (mix step):

```bash
ffmpeg -y -loglevel error -i renders/<name>-no-music.mp4 -i vo-polished.wav -i music/<track>.mp3 -filter_complex "
  [2:a]atrim=0:<DUR>,asetpts=N/SR/TB,volume=-14dB,afade=t=out:st=<DUR-1.6>:d=1.6[m];
  [1:a]asplit=2[vo][sc];
  [m][sc]sidechaincompress=threshold=0.05:ratio=2.5:attack=20:release=400[duck];
  [vo][duck]amix=inputs=2:duration=first:normalize=0,loudnorm=I=-16:TP=-1.5[a]" \
  -map 0:v -map "[a]" -c:v copy -c:a aac -b:a 192k -movflags +faststart renders/<name>-final.mp4
```

Music sits at -14 dB, ducks under the voice, and fades over the last 1.6 s. Final loudness is -16 LUFS.

**One music track for the series.** A freshly generated track for the third video was judged worse than the one already in use, so the first good track became the series track. To fit it to a different length: take the track from near its start, then crossfade (about 0.4 s) into its own ending so the final note lands on the closing card. Make the join on a bar line.

**Check before handing over:** the voice file is not longer than the picture (a longer one silently extends the video); duration matches `DUR`; loudness is about -16 LUFS; pull one frame from the finished file at the point of the last fix and look at it.

---

## 10. Stage 7. Titles, descriptions, thumbnails

**Title.** Under 70 characters. Lead with what the viewer gets, end with the product name.

| Video | Evident's title |
|---|---|
| Intro | Reviews, Rewards, Galleries & FAQs in One Fast App \| Evident |
| Reviews | Collect Verified Product Reviews Automatically \| Evident Reviews |
| Rewards | A Rewards Program That Brings Customers Back \| Evident Rewards |
| Galleries | Turn Customer Photos Into Shoppable Galleries \| Evident Galleries |
| FAQs | Answer the Questions That Stop the Sale \| Evident FAQs |

Run keyword research on the platform names and the category before choosing. For Evident, search demand sat on the larger platform's name; the smaller platform's app searches were too small to measure. Title scores from a tool came back within a point of each other and did not separate candidates, so treat them as a sanity check only.

**Description template.**

```
<Two or three sentences following the video's own script: the problem, then what the feature does.>

<One paragraph on how it works.>

<Control and migration, one or two sentences.>

<Feature> is part of <App>, with <other features> in the same app.

Start a free trial: https://<domain>

More from <App>:
Overview: https://youtu.be/<id>
<Feature>: https://youtu.be/<id>
...

#<category> #<feature> #<platform> #<platform>
```

Write it from the script so every claim is one the video already makes. Leave out anything the site states inconsistently (Evident's trial length differed between pages, so the description says "free trial" with no number).

**Thumbnails.** 1280x720 PNG, built from an HTML template with the video's fonts and palette. Two per video, testing different hooks:

- **A, the problem.** Light canvas. Two-line headline, second line in the serif italic (red for a problem, brand colour for a benefit). A small stack of tilted cards on the right that illustrates it. Logo and a one-line answer bottom left.
- **B, the answer.** Dark background. App icon tile on the left, the benefit headline, four checked feature pills.

Keep the bottom-right corner clear; YouTube puts the duration stamp there. Fit headline lines by shrinking the font until the line fits its column, measuring each line by itself.

---

## 11. Stage 8. Publish on YouTube

**Channel setup, once.** These gate later steps, so do them before the first upload:

- **Phone verification.** Required before any custom thumbnail can be uploaded.
- **Advanced features.** Thumbnail A/B testing ("Test & compare") needs this, and the review can take about a day.
- **External link verification.** The site link in descriptions is not clickable until it is done.
- **Upload defaults.** Set audience (not made for kids), category and language as channel defaults so every upload inherits them.

**Per video.** The founder uploads the file. Then, in Studio:

1. Title and description.
2. Thumbnail A.
3. Audience, category (Science & Technology for our apps), language.
4. AI disclosure. The question covers realistic altered footage, real people, and music as the main focus. Motion graphics with a real voice and background music answers "No".
5. Paid promotion: leave off unless the video carries someone else's promotion.
6. Visibility: schedule. Evident published the intro first, then the four features 30 minutes apart.

**Cross-promotion, after all videos exist:**

| Piece | Setting |
|---|---|
| **Playlist** | One public playlist, intro first, ordered oldest-published first so the publish order is the watch order. |
| **End screens** | "1 video, 1 subscribe" template on the closing card. Each video points to the next in the series; the last points back to the intro. |
| **Cards** | A playlist card at the "just the start" line (features) or at the one-sentence answer (intro). |
| **Description links** | The "More from" block in section 10. YouTube links are clickable without the external-link verification. |

End-screen timing: keep the elements on the closing card. Started earlier, the subscribe button covered the final tagline. Five seconds is the minimum YouTube allows, which is why the closing card is held for five seconds.

Not available: a clickable end-screen link to the site (needs the Partner Program). On Evident's new channel the channel-trailer option was hidden in Studio; check whether it appears for yours.

**Driving Studio through a browser session.** No API or OAuth setup is needed, and it handles uploads of thumbnails. Things that went wrong and how to avoid them:

- Hashtag autocomplete can silently replace the last hashtag typed with a longer suggestion. Read the saved description back and compare.
- Pressing Escape inside the upload wizard closes the wizard.
- A browser extension that injects panels into Studio can interrupt typing and block page scripts for the first seconds after load. Wait for the page to settle, type long text in paragraphs, and verify each field after writing.
- Studio remembers the last end-screen layout, including the chosen video. On the next video it will be pointing at the wrong one, sometimes the video itself. Always re-pick.
- The time field format changes with video length: `seconds:frames` under a minute, `minutes:seconds:frames` above.
- Saving and publishing are public actions. Confirm final values with the founder before the first save, and confirm the schedule before scheduling.

Other routes considered: the YouTube Data API works and is free but needs a Google Cloud project and OAuth consent screen, and videos uploaded through an unaudited project may be locked to private. A third-party tool's connector is the least effort if the channel is the one connected to it.

---

## 12. Per-app checklist

**Before recording**
- [ ] Features chosen; one video each plus the intro
- [ ] Scripts written to the shape in section 4, every claim traced to the product
- [ ] Brand tokens, fonts, logo and icon collected
- [ ] Channel created; phone verified; advanced features requested; upload defaults set

**Per video**
- [ ] Recording transcribed with word timestamps; takes chosen; mapping written in the README
- [ ] Voice cut and polished; raw and polished samples compared by ear
- [ ] `promo.html` built; storyboard approved
- [ ] Full render; duration and loudness checked; one frame inspected at each late fix
- [ ] A person has listened to the final mix
- [ ] Title, description, two thumbnails

**Publishing**
- [ ] Title, description and thumbnail A saved and read back
- [ ] Audience, category, language, AI disclosure set
- [ ] Scheduled; status shows Scheduled with the right time
- [ ] Playlist, end screens, cards and description links added
- [ ] Thumbnail A/B test started once the video is public and advanced features are approved

---

## 13. Status of the reference implementation

| Item | Evident, 2026-11-22 |
|---|---|
| Five videos rendered and approved | Done |
| Published or scheduled with title, description, thumbnail A | Done |
| Playlist, end screens, cards, description links | Done |
| Thumbnail A/B tests | Waiting on YouTube's advanced-features review; B thumbnails are built |
| Channel trailer | Option not offered in Studio |
| Watched on YouTube to confirm end screens and cards play | Not yet |
