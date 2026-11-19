# Wykop as a source: API access research

*Researched: 2026-11-19. No collector has been written yet.*

## Summary

- **API:** Wykop has an official **API v3 (beta)** that covers everything this pipeline needs: keyword search with date filters, tag streams, and comments on both Wykop content types, *znaleziska* (submitted links) and *wpisy* (Mikroblog microblog posts).
- **Keys:** access needs an **app key and secret**. The official spec says they are "supplied by the administrator", so there may be an approval step. The developer dashboard needs a logged-in Wykop account, so this hasn't been confirmed.
- **Terms:** the site's terms of service (*regulamin*, effective 2026-05-11) say **nothing about the API, scraping or research**. Their licence to user content covers Wykop running the service and other users' *personal use*. Publishing a corpus and quotes goes beyond that, so the API terms shown in the dashboard need to be read before collecting anything.

## 1. Getting keys

| Step | Status |
|---|---|
| Developer portal: [dev.wykop.pl](https://dev.wykop.pl/) → "Przejdź do Panelu" → `dev.wykop.pl/dashboard/app` | Needs a Wykop account login, so **not checked yet** |
| Official spec on the key | `key`: *"Identyfikator aplikacji dostarczony przez administratora"* ("app identifier supplied by the administrator") |
| Community reports, ~2023 (#wykopapiv3) | Access was by invitation or request ("zapraszaliśmy ich do uzyskania dostępu", "o dostęp trzeba teraz prosić?"); one request got the reply that they were "working on performance" |
| Community reports, ~2025 | One user created an app and got a JWT; public endpoints worked, while endpoints needing a logged-in user returned `Access Denied … ROLE_USER`. **Unverified:** one user claims read-only access is paid |

**Next step (manual):** log in at `dev.wykop.pl/dashboard/app` and check:
1. whether you can create an app yourself or must request one;
2. the API terms (*regulamin API*) you have to accept;
3. any mention of cost, quotas or allowed uses (research, storage, publication, AI processing).

## 2. Endpoints relevant to this pipeline

These come from the official OpenAPI spec at [doc.wykop.pl](https://doc.wykop.pl/) (`openapi.yaml`). The base URL is `https://wykop.pl/api/v3/`.

| Need | Endpoint | Useful parameters |
|---|---|---|
| Authenticate the app | `POST /auth` `{"data": {"key", "secret"}}` → JWT (Bearer); `POST /refresh-token` | — |
| Keyword search, Mikroblog | `GET /search/entries` | `q`, `sort`, `votes`, `date_from`, `date_to`, `tags`, `category`, `page`, `limit` (default 25) |
| Keyword search, znaleziska | `GET /search/links` | same, plus `domains` |
| Tag stream (e.g. an AI tag) | `GET /tags/{tagName}/stream` | `page`, `limit`, `sort`, `type`, `date_year`, `date_month` |
| Znalezisko comments | `GET /links/{linkId}/comments`, `…/comments/{commentId}/comments` | `page`, `limit` |
| Wpis comments | `GET /entries/{entryId}/comments` | `page`, `limit` |

- **Errors:** no authorisation returns 403, a blocked endpoint returns 401 with key `3`, and `limit_exceeded` (400) is documented only for writing actions such as posting or voting.
- **Rate limits:** none are published for reading.

## 3. How it would map onto the existing pipeline

- **Collector:** `aiconcerns/collectors/wykop.py`, following the YouTube collector. It would search the 12 keywords, and optionally AI tags, through `/search/entries` and `/search/links` within `date_range`. For each hit it would store:
  - the wpis or znalezisko as `kind=post`;
  - its comments as `comment`/`reply`;
  - `parent_id` set to the wpis or znalezisko, and `engagement` to its votes.
- **Credentials:** `.env.example` already has `WYKOP_KEY` and `WYKOP_SECRET`.
- **Anonymisation:** Wykop usernames appear in text as `@nick`, which the anonymiser already strips. Usernames in the API response's author fields must never be stored.
- **Before anything is enabled:**
  - add Wykop to the classifier prompt in `llm.py` (it currently assumes a "film");
  - add Wykop to the report caveats in `report.py`;
  - add Wykop to the README privacy section.

## 4. Open questions and risks

| Question | Why it matters |
|---|---|
| Is a key self-service, by request, or paid? | Decides whether Wykop is feasible at all |
| What do the API terms allow? | Storing texts, sending them to an LLM (Claude), and publishing an anonymised corpus and quotes in this repo |
| Content licence in the *regulamin* (§ "Jak korzystać z serwisu", pt 7) | It covers Wykop and other users' *personal use*, not research or redistribution. If the API terms don't cover it, quotes and the committed corpus need care or a request to Wykop |
| GDPR (RODO) | Same as for YouTube: comments are personal data, so data must be kept minimal and anonymised, and the research basis documented |
| API is still "beta" | Endpoints and parameters may change |

## Sources

- [Wykop API v3 docs (Swagger) and `openapi.yaml`](https://doc.wykop.pl/)
- [dev.wykop.pl](https://dev.wykop.pl/) and [wykop.pl/dla-programistow](https://wykop.pl/dla-programistow)
- [Regulamin serwisu Wykop.pl (effective 2026-05-11)](https://wykop.pl/regulamin)
- [#wykopapiv3 tag on Wykop (community reports)](https://wykop.pl/tag/wykopapiv3)
- [wykop-sdk-reloaded (Python SDK, says app keys come from dev.wykop.pl)](https://github.com/lukas346/wykop_sdk_reloaded)
- [FakeCop/wykop-api-client (Laravel client, endpoint list)](https://github.com/FakeCop/wykop-api-client)
