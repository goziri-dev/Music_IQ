# Music IQ

Music IQ is a music-trivia engine that turns any artist into a personalised quiz. The user picks a song, the system pulls real data about its artists from Spotify and Wikipedia, feeds the biography to an LLM to generate trivia questions, and grades free-text answers with fuzzy matching so minor typos and reordered words still count.

The project is a portfolio piece: the goal is to demonstrate clean object-oriented design, third-party API integration, and a pragmatic use of LLMs — not just "call the OpenAI API."

## What it does

1. **Search for a song** via the Spotify Web API.
2. **Resolve the artists** on that track and fetch each artist's biography from Wikipedia.
3. **Generate trivia questions** from the biography using OpenAI (`gpt-3.5-turbo`).
4. **Cache results locally** so the same song or artist is never re-fetched from a paid/rate-limited API.
5. **Grade the user's answer** using token-normalised fuzzy matching so "tokyo oji kita" still matches "Oji, Kita, Tokyo".

## Tech stack

| Area | Choice |
|---|---|
| Language | Python 3.10+ (uses PEP 604) |
| Music data | [Spotipy](https://spotipy.readthedocs.io/) (Spotify Web API) |
| Biographies | [wikipedia-api](https://pypi.org/project/wikipedia-api/) + raw MediaWiki search |
| Question generation | [OpenAI Python SDK](https://github.com/openai/openai-python) |
| Answer grading | [fuzzywuzzy](https://github.com/seatgeek/fuzzywuzzy) + python-Levenshtein |

## Architecture

The codebase is deliberately split into small, single-purpose packages so each concern can be swapped independently — e.g. replacing the OpenAI backend with a local model only touches one class.

```
Music_IQ/
├── models/        # Domain objects: Artist, Song, Question, Model enum
├── builders/      # Turn raw API/local JSON into domain objects (ArtistBuilder, SongBuilder)
├── services/      # Orchestration layer (QuestionService)
├── utils/
│   ├── spotify_api_client.py      # Spotify integration
│   ├── wikipedia_api_client.py    # Wikipedia integration
│   ├── ai_api_client.py           # Abstract AI client
│   ├── question_generator.py      # Abstract generator
│   ├── question_generator_ai.py   # OpenAI implementation
│   ├── question_verifier.py       # Abstract verifier
│   ├── question_verifier_ai.py    # Fuzzy-matching implementation
│   ├── local_storage.py           # JSON-backed cache
│   ├── savable.py / storage.py    # Persistence abstractions
│   └── helper.py
└── data/          # Cached songs, artists and generated questions (JSON)
```

### Design decisions worth calling out

- **Abstract base classes everywhere that matters.** `Savable`, `ModelBuilder`, `QuestionGenerator`, `QuestionVerifier`, and `AIAPIClient` are all ABCs. The concrete OpenAI-backed and fuzzy-matching classes are implementations, so the rest of the app depends on the interface, not the vendor. Replacing OpenAI with Anthropic or a self-hosted model is a one-file change.
- **Read-through cache on external calls.** `SpotifyAPIClient.get_songs` / `get_artists` check `LocalStorage` first and only hit Spotify on a miss. The cache is keyed by query string and persists to `data/song.json` / `data/artist.json`, which keeps development cheap and the app offline-friendly for repeated queries.
- **Separate `from_dict` / `to_dict` contract.** Every domain object round-trips to a dict that matches Spotify's own schema, so the same builder logic works whether the data came from the API or from local cache.
- **Answer normalisation, not just fuzzy matching.** `QuestionVerifierAI` lowercases, strips, and sorts tokens before asking fuzzywuzzy for a score. That handles reordered multi-word answers ("tokyo oji kita" vs "Oji, Kita, Tokyo") which a raw Levenshtein ratio would fail. A guard rejects single-word mismatches to stop "kh" from matching "KHOO" at 80 %+.

## Getting started

### Prerequisites

- Python 3.10+
- A Spotify developer app (client ID + secret)
- An OpenAI API key

### Setup

```bash
git clone https://github.com/Akubuo-F/Music_IQ.git
cd Music_IQ
python -m venv .venv
.venv\Scripts\activate        # Windows
# source .venv/bin/activate   # macOS/Linux
pip install -r requirements.txt
```

Create `utils/.env` with:

```env
SPOTIFY_CLIENT_ID=your_spotify_client_id
SPOTIFY_CLIENT_SECRET=your_spotify_client_secret
OPENAI_CLIENT_SECRET=your_openai_api_key
```

### Try the pieces

Each client has a `__main__` block you can run directly to smoke-test that module:

```bash
python -m utils.spotify_api_client       # search a song and resolve its artists
python -m utils.wikipedia_api_client     # pull biographies for sample artists
python -m utils.question_generator_ai    # generate trivia from a bio
python -m utils.question_verifier_ai     # run the fuzzy-grading test suite
```

## Status & roadmap

This is an active portfolio project. The low-level building blocks (API clients, models, builders, caching, answer grading) are in place; the pieces still being wired up:

- [ ] Complete `QuestionService` — the orchestrator that ties generator + verifier together end-to-end.
- [ ] CLI / web entry point so a user can actually play a round.
- [ ] Migrate from `gpt-3.5-turbo` to a current model and add prompt caching.
- [ ] Swap `fuzzywuzzy` for `rapidfuzz` (drop-in, MIT-licensed, no GPL dependency).
- [ ] Unit tests under `tests/` rather than `if __name__ == "__main__"` blocks.

## License

See [LICENSE.txt](LICENSE.txt).

## Author

Built by **Favour Chigoziri Akubuo** as a demonstration of clean Python architecture, third-party API integration, and applied LLM use. Reach me at `goziri.dev@gmail.com` or `akubuof.work@gmail.com`.
