# Postiz Platform Architecture

This is an architecture overview of this Postiz fork. It was generated from the code at upstream `v2.26.0+` (`44df988c`).
All diagrams use [Mermaid](https://mermaid.js.org/) and render natively on GitHub.

---

## 1. System Context

```mermaid
flowchart LR
    user([Users / Teams])
    dev([Developers & AI agents])

    subgraph postiz[Postiz Platform]
        fe[Frontend<br/>Next.js]
        be[Backend API<br/>NestJS]
        orch[Orchestrator<br/>Temporal workers]
    end

    ext[Browser Extension<br/>Chrome MV3]
    sdk["@postiz/node SDK"]
    mcp[MCP clients<br/>ChatGPT / Claude / etc.]

    socials[(35 social channels<br/>X, LinkedIn, Meta, TikTok,<br/>YouTube, Reddit, Discord, ...)]
    saas[(3rd-party services<br/>Stripe, OpenAI, S3/R2,<br/>Resend/SES, Dub/Kutt, HeyGen...)]

    user --> fe
    user --> ext
    dev --> sdk
    dev --> mcp
    fe --> be
    ext -- cookies for cookie-auth channels --> be
    sdk -- Public API v1 --> be
    mcp -- MCP over OAuth --> be
    be --> orch
    orch --> socials
    be --> socials
    be --> saas
    orch --> saas
```

---

## 2. Container / Deployment View

The services come from `docker-compose.yaml`. In production, the `postiz` image runs frontend, backend and orchestrator together under PM2, behind nginx.

```mermaid
flowchart TB
    subgraph app["postiz container (ghcr.io/gitroomhq/postiz-app)"]
        direction LR
        FE["apps/frontend<br/>Next.js 16 · React · Tailwind 3<br/>SWR + useFetch"]
        BE["apps/backend<br/>NestJS REST API<br/>/api · /public/v1 · /mcp"]
        OR["apps/orchestrator<br/>NestJS + Temporal worker<br/>workflows & activities"]
    end

    subgraph data[Data plane]
        PG[("PostgreSQL 17<br/>Prisma ORM<br/>app + Mastra tables")]
        RD[("Redis 7<br/>cache · throttling · locks")]
        UP[("Uploads<br/>local volume or<br/>Cloudflare R2/S3")]
    end

    subgraph temporal[Temporal cluster]
        TS["temporal server 1.28"]
        TPG[("temporal-postgresql 16")]
        TES[("elasticsearch 7<br/>visibility / search attrs")]
        TUI["temporal-ui"]
    end

    SP["spotlight (Sentry dev)"]

    FE -- HTTP --> BE
    BE -- Prisma --> PG
    OR -- Prisma --> PG
    BE --> RD
    OR --> RD
    BE --> UP
    OR --> UP
    BE -- start / signal workflows --> TS
    OR -- poll task queues --> TS
    TS --> TPG
    TS --> TES
    TUI --> TS
    FE -.-> SP
    BE -.-> SP
```

---

## 3. Monorepo Structure

```mermaid
flowchart LR
    subgraph apps
        frontend
        backend
        orchestrator
        commands["commands<br/>(CLI: refresh tokens, agent run, config)"]
        extension["extension<br/>(cookie harvester)"]
        sdk["sdk<br/>(@postiz/node)"]
    end

    subgraph libraries
        NL["nestjs-libraries<br/>integrations · database · dtos · temporal<br/>chat/mcp · upload · emails · encryption<br/>short-linking · 3rdparties · videos · openai"]
        RSL["react-shared-libraries<br/>shared UI, i18n, toaster"]
        HLP["helpers<br/>utils, custom.fetch, decorators"]
    end

    frontend --> RSL
    frontend --> HLP
    backend --> NL
    backend --> HLP
    orchestrator --> NL
    orchestrator --> HLP
    commands --> NL
    sdk -. HTTP .-> backend
    extension -. HTTP .-> backend
```

Layering rule from `CLAUDE.md`: **DTO → Controller → (Manager) → Service → Repository**.
Controllers live in `apps/backend`. Services and repositories live in `libraries/nestjs-libraries`.

---

## 4. Backend Internals

```mermaid
flowchart TB
    subgraph surfaces[API surfaces · apps/backend/src]
        API["/api routes<br/>auth, posts, integrations, media,<br/>analytics, billing, autopost, sets,<br/>signatures, webhooks, copilot, admin..."]
        PUB["/public/v1<br/>API-key auth (SDK, Zapier, n8n)"]
        MCP["MCP server + OAuth apps<br/>chat/start.mcp.ts"]
        HOOK["Inbound webhooks<br/>Stripe · RevenueCat"]
    end

    subgraph core[libraries/nestjs-libraries/src]
        DB["database/prisma/*<br/>repositories + services per domain"]
        IM["IntegrationManager<br/>socialIntegrationList"]
        PROV["integrations/social/*.provider.ts<br/>(35 providers, SocialAbstract)"]
        AG["chat/ + agent/<br/>Mastra agent, tools, MCP relay"]
        TPM["3rdparties/<br/>HeyGen · HubSpot · ReelFarm"]
        VID["videos/<br/>image slides · Seedance"]
        TMP["temporal/<br/>client, register, search attrs"]
        ADP["Pluggable adapters (env-selected)"]
    end

    subgraph adapters[Adapters]
        STO["upload: local | Cloudflare R2<br/>STORAGE_PROVIDER"]
        EM["emails: Resend | SES | Nodemailer<br/>EMAIL_PROVIDER"]
        ENC["encryption: local | AWS KMS<br/>ENCRYPTION_PROVIDER"]
        SL["short-linking: Dub | Kutt | Short.io | LinkDrip"]
        AI["openai/ · deepgram/"]
    end

    API --> DB
    API --> IM
    PUB --> DB
    MCP --> AG
    AG --> DB
    HOOK --> DB
    IM --> PROV
    API --> TPM
    API --> VID
    API --> TMP
    ADP --> STO & EM & ENC & SL & AI
    DB --> ADP
```

---

## 5. Channel (Provider) Architecture

Every channel is one class that implements `SocialProvider` and extends `SocialAbstract`. Generic code never branches on a specific channel. Channel-specific behaviour is added to the provider interface and implemented inside the provider.

```mermaid
classDiagram
    class SocialProvider {
        <<interface>>
        identifier
        name
        scopes
        editor
        generateAuthUrl()
        authenticate()
        refreshToken()
        post()
        postPending()?
        checkPostStatus()?
        comment()?
        analytics()?
        postAnalytics()?
        pages()?
        fetchPageInformation()?
        reConnect()?
        autoRepostPost()?
        autoPlugPost()?
        customFields()?
    }
    class SocialAbstract {
        <<abstract>>
        maxConcurrentJob
        fetch()
        handleErrors()
    }
    SocialAbstract <|-- XProvider
    SocialAbstract <|-- LinkedinProvider
    LinkedinProvider <|-- LinkedinPageProvider
    SocialAbstract <|-- InstagramProvider
    SocialAbstract <|-- InstagramStandaloneProvider
    InstagramStandaloneProvider ..> InstagramProvider : delegates post/comment/analytics
    SocialAbstract <|-- TiktokProvider
    SocialAbstract <|-- TiktokBusinessProvider
    SocialAbstract <|-- MastodonProvider
    MastodonProvider <|-- MastodonCustomProvider : disabled
    SocialAbstract <|-- OtherProviders : 27 more
    SocialProvider <|.. SocialAbstract
```

Each channel touches these places:

```mermaid
flowchart LR
    P["backend provider<br/>integrations/social/&lt;id&gt;.provider.ts"]
    R["registry<br/>integration.manager.ts"]
    D["settings DTO<br/>dtos/posts/providers-settings/"]
    M["DTO map<br/>all.providers.settings.ts"]
    F["frontend editor + preview<br/>new-launch/providers/&lt;id&gt;/"]
    S["frontend registry<br/>show.all.providers.tsx"]
    C["page picker (optional)<br/>continue-provider/&lt;id&gt;"]
    I["icon<br/>public/icons/platforms/&lt;id&gt;.png"]
    P --> R
    D --> M
    F --> S
    P -.identifier.- F
    D -.validates.- F
    C -.- F
    I -.- F
```

Env controls: `HIDDEN_PROVIDERS` hides channels from the add-channel screen. `MIGRATE_PROVIDERS` (e.g. `tiktok:tiktok-business`) migrates channels in place on reconnect.

---

## 6. Post Publishing Flow

```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant FE as Frontend
    participant BE as Backend (PostsController)
    participant DB as PostgreSQL
    participant T as Temporal
    participant W as Orchestrator<br/>postWorkflowV112
    participant A as post.activity
    participant P as SocialProvider
    participant SN as Social Network

    U->>FE: Compose post, pick channels & time
    FE->>BE: POST /posts (DTO per provider validated)
    BE->>DB: create Post rows (group)
    BE->>T: workflow.start("postWorkflowV112")
    T->>W: schedule
    W->>W: sleep until publishDate
    W->>A: getPost / getIntegration
    alt token expired
        A->>P: refreshToken()
        A->>DB: save new tokens (encrypted)
    end
    W->>A: postSocialPending()
    A->>P: post()
    P->>SN: platform API
    SN-->>P: id / pending upload
    opt async platforms (TikTok, IG, YouTube...)
        loop until done
            W->>A: checkPostStatus()
        end
        W->>A: finalizePost()
    end
    W->>A: updatePost(releaseURL) · changeState(PUBLISHED)
    W->>A: postComment() for thread/comments
    W->>A: internalPlugs / globalPlugs (auto-repost, auto-plug)
    W->>A: sendWebhooks() · inAppNotification()
    A->>DB: persist state
    FE-->>U: calendar shows published
```

Errors go through `handleActivityError` and `markUnconfirmed`, and the post moves to the `ERROR` state with a notification.
**Workflows are versioned** (`post.workflow.v1.0.1` … `v1.1.2`). Never edit a workflow that is already on `main`. Add a new version instead.

---

## 7. Background Workflows (Temporal)

```mermaid
flowchart LR
    subgraph wf[apps/orchestrator/src/workflows]
        PW["postWorkflowV1xx<br/>publish a post group"]
        MP["missing.post<br/>sweep stuck posts"]
        RT["refresh.token<br/>proactive token refresh"]
        AP["autopost<br/>RSS → posts"]
        PM["process.media<br/>transcode/resize"]
        GV["generate.video<br/>AI video"]
        CL["clipping<br/>long video → clips"]
        DE["digest.email · send.email"]
        ST["streak<br/>engagement nudges"]
        ET["encrypt.tokens<br/>token migration"]
    end
    subgraph act[activities]
        PA[post.activity]
        IA[integrations.activity]
        MA[media.activity]
        VA[video.activity]
        CA[clipping.activity]
        EA[email.activity]
        AA[autopost.activity]
    end
    PW --> PA
    MP --> PA
    RT --> IA
    ET --> IA
    AP --> AA
    PM --> MA
    GV --> VA
    CL --> CA
    DE --> EA
    ST --> EA
    MA -. RunPod .-> ext1[(RunPod media processor)]
    VA -. Seedance / slides .-> ext2[(AI video)]
```

---

## 8. Data Model (core entities)

There are about 45 app tables plus about 50 `mastra_*` tables for the AI agent. These are the core relationships:

```mermaid
erDiagram
    Organization ||--o{ UserOrganization : has
    User ||--o{ UserOrganization : member
    Organization ||--o{ Integration : "connected channels"
    Customer ||--o{ Integration : "groups channels"
    Organization ||--o{ Post : owns
    Integration ||--o{ Post : "published via"
    Post ||--o{ Post : "parent / thread comments"
    Post ||--o{ TagsPosts : tagged
    Tags ||--o{ TagsPosts : ""
    Organization ||--o{ Media : library
    Organization ||--o| Subscription : billing
    Organization ||--o{ Webhooks : outbound
    Webhooks ||--o{ IntegrationsWebhooks : ""
    Integration ||--o{ IntegrationsWebhooks : ""
    Integration ||--o{ Plugs : "auto-plug rules"
    Organization ||--o{ AutoPost : "RSS feeds"
    Organization ||--o{ Sets : "post presets"
    Organization ||--o{ Signatures : ""
    Organization ||--o{ ThirdParty : "HeyGen/HubSpot..."
    Organization ||--o{ OAuthApp : "MCP / API apps"
    Organization ||--o{ Clipping : ""
    Clipping ||--o{ ClippingClip : ""

    Integration {
        string providerIdentifier
        string internalId
        string token "encrypted"
        string refreshToken "encrypted"
        datetime tokenExpiration
        bool inBetweenSteps
        bool refreshNeeded
        string customerId FK
    }
    Post {
        string group
        string state "QUEUE|PUBLISHED|ERROR|DRAFT"
        datetime publishDate
        string content
        string settings "provider DTO as JSON"
        string releaseURL
    }
```

---

## 9. Extension Points for This Fork

| Goal | Where to extend | Conflict risk with upstream |
|---|---|---|
| New channel | New `*.provider.ts`, DTO, and frontend folder; register in 3 lists | Low for new files, **medium for the registries** |
| Channel variant | Subclass the existing provider (`LinkedinPageProvider` pattern) or delegate to it (`InstagramStandalone` pattern) | Low |
| Generic channel behaviour | Add an optional method to `SocialProvider`, call it generically, and implement it per provider | Medium |
| New background job | New workflow + activity in `apps/orchestrator` | Low |
| Change the publish flow | **New** `postWorkflowV1xx` file, then update the caller in `posts.service` | Medium |
| New storage/email/shortlink backend | New adapter that implements the interface, selected by env var | Low |
| New 3rd-party tool | `3rdparties/<name>/` + `thirdparty.manager.ts` | Low |
| AI agent tools / MCP | `chat/tools/` | Low |

Keeping fork-specific code in **new files** and limiting edits to the shared registries makes `git merge upstream/main` much easier.
