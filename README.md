<div align="center">

# Siphon Zone

<img src="media/demo.gif" width="280" alt="Siphon Zone app demo">

</div>

## What is this?

Siphon Zone is a mobile app (iOS/Android) where you can read manga, read novels, and watch live, 24/7 anime channels with no ads. The services, are made up of independent backend services that fetch data, cleans, and enriches content from across the web, stores it on a self-hosted media server, and serves it to the app through a single secure gateway.

It started in 2023 as a single Node.js app called **Siphon Manga** that only did one thing: read manga. Since then it's been rebuilt from the ground up into a 12 service distributed system, that run entirely on self-hosted infrastructure, with its own observability stack, background job queues, and an AI model for generating content descriptions. The rewrite was deliberate: each content type (manga / anime / novels) gets its own backend service to have a proper seperation of concerns.

<table align="center">
  <tr>
    <td align="center"><img src="media/manga-home.png" width="190"><br><sub>Manga home</sub></td>
    <td align="center"><img src="media/manga-detail.png" width="190"><br><sub>Manga detail</sub></td>
    <td align="center"><img src="media/reader.png" width="190"><br><sub>Reader</sub></td>
    <td align="center"><img src="media/browse.png" width="190"><br><sub>Browse</sub></td>
  </tr>
  <tr>
    <td align="center"><img src="media/search.png" width="190"><br><sub>Search</sub></td>
    <td align="center"><img src="media/live-anime.png" width="190"><br><sub>Live anime TV</sub></td>
    <td align="center"><img src="media/library.png" width="190"><br><sub>Library</sub></td>
    <td align="center"><img src="media/library-updates.png" width="190"><br><sub>Library updates</sub></td>
  </tr>
  <tr>
    <td align="center"><img src="media/manga-continue-reading.png" width="190"><br><sub>Continue reading</sub></td>
    <td align="center"><img src="media/library-history.png" width="190"><br><sub>History</sub></td>
    <td align="center"><img src="media/settings-dark.png" width="190"><br><sub>Settings (dark)</sub></td>
    <td align="center"><img src="media/manga-home-light.png" width="190"><br><sub>Light theme</sub></td>
  </tr>
</table>

<p align="center"><sub>Full demo video: <a href="media/demo.mp4">demo.mp4</a></sub></p>

## Why I built it this way

- **Polyglot microservices, on purpose.** 10+ backend services across TypeScript/NestJS, Java/Spring Boot, Go, and C#/.NET, all talking to the same PostgreSQL database and sharing API contracts generated from a single set of **Protocol Buffer** definitions so a schema change in one place regenerates typed clients for every language at once, I did this on purpose because alot of my other apps use different backend languages and this keeps my skills sharp.
- **Data fetching pipeline** Scrapers fall back through multiple strategies (direct HTTP - a pool of headless-Chrome FlareSolverr instances - Playwright) to get past Cloudflare bot-challenges, with tunable worker concurrency and automatic retry/backoff.
- **Background job processing, in two different ecosystems.** The manga scraper runs on **BullMQ/Redis** queues; the novel scraper (written in Go) runs on **Asynq** both handle thousands of chapter-download jobs with bounded concurrency instead of blocking request threads.
- **Cross-source identity resolution.** The same series often exists on multiple source sites under different names. A Jaccard token-similarity matcher reconciles titles against AniList/MyAnimeList/Kitsu IDs, cross-validates conflicting matches, and de-duplicates automatically a small but real data-matching/trust problem.
- **Self-hosted AI enrichment.** A locally-run **Ollama** LLM generates missing plot descriptions during metadata enrichment.
- **Real observability.** Centralized logs via **Grafana + Loki + Promtail**, plus a custom monitoring dashboard/API.
- **Production infra engineering** The whole stack runs on a self-hosted NAS via Docker Compose, with purpose-built deployment tooling: schema-readiness gating so services don't boot before their tables exist, scoped redeploys so a one-line manga fix doesn't rebuild the JVM services.
- **Edge-auth model.** The gateway verifies JWTs and resolves them to a device identity once, then passes a trusted header downstream. The internal services don't each re-implement auth, they trust the edge.

## Architecture

```mermaid
flowchart TB
    subgraph Client["📱 Client"]
        App["Siphon Zone App<br/>React Native · Expo · Tamagui"]
    end

    subgraph Edge["Edge"]
        GW["Gateway - Node/Express<br/>JWT auth · rate limiting · routing"]
    end

    subgraph Content["Content APIs"]
        Manga["Manga Backend<br/>NestJS"]
        Anime["Anime Backend<br/>Spring Boot · Java 21"]
        Novels["Novels Backend<br/>ASP.NET Core 9"]
    end

    subgraph Pipeline["Data Pipeline"]
        MangaData["Manga Fetcher<br/>NestJS + BullMQ"]
        NovelData["Novel Fetcher<br/>Go + Asynq"]
        MetaData["Metadata Fetcher<br/>Spring Boot"]
        Ollama["Ollama LLM<br/>AI descriptions"]
        Flare["FlareSolverr pool<br/>anti-bot bypass"]
    end

    subgraph Storage["Storage"]
        PG[("PostgreSQL")]
        Redis[("Redis")]
        NAS[("Self-hosted NAS<br/>media files")]
    end

    subgraph Obs["Observability"]
        Promtail --> Loki --> Grafana
    end

    App --> GW
    GW --> Manga & Anime & Novels
    Manga & Anime & Novels --> PG
    Manga --> NAS

    MangaData --> Flare
    NovelData --> Flare
    MangaData --> Redis
    NovelData --> Redis
    MangaData --> PG & NAS
    NovelData --> PG & NAS
    MetaData --> PG
    MetaData --> Ollama
```

## The services

| Service | Stack | Role |
|---|---|---|
| **Siphon-Zone-App** | React Native, Expo, Tamagui, TanStack Query | Cross-platform iOS/Android client |
| **Siphon-Zone-Gateway** | Node.js, Express, JWT | Edge API gateway - auth, rate limiting, request routing |
| **Siphon-Zone-Manga-Backend** | NestJS, TypeScript | Manga catalog, search, reader, bookmarks & history |
| **Siphon-Zone-Anime-Backend** | Spring Boot, Java 21 | Anime catalog & live-TV channel API |
| **Siphon-Zone-Novels-Backend** | ASP.NET Core 9, C#, Dapper | Novel catalog & reader API |
| **Siphon-Zone-Data/Siphon-Zone-Manga** | NestJS, BullMQ, Playwright, Sharp | Data fetch, dedupes, and stores manga chapters |
| **Siphon-Zone-Data/Siphon-Zone-Novel** | Go, Asynq, Colly | Data fetch and stores novel chapters |
| **Siphon-Zone-Data/Siphon-Zone-MetaData** | Spring Boot, Java | Enriches series with AniList/MAL/Kitsu data + AI descriptions |
| **Siphon-Zone-Shared-Types** | Protocol Buffers | Single source of truth for API contracts across every language |
| **Siphon-Zone-Config** | Docker Compose, Grafana, Loki, Promtail | Infrastructure, observability, and deployment tooling |

## Distribution

Shipped to real users via [Apple TestFlight](https://developer.apple.com/testflight/) for iOS and signed APKs for Android.