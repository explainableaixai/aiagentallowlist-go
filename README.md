# aiagentallowlist-go

This module puts an AI agent allow list in front of Go code that lets a model browse. Before your executor fetches a page, submits a form or clicks through a site on the model's behalf, it asks one question: is this URL a page an agent should act on? The service answers with a verdict and the page type it recognised. Policy and data live at [AI agent allow list for autonomous browsers](https://www.aiagentallowlist.com).

```bash
go get github.com/explainableaixai/aiagentallowlist-go
```

## The one call

```go
c := aiagentallowlist.New(os.Getenv("AQ_API_KEY"))
r, err := c.Check(ctx, "https://accounts.example.com/signin")
```

The value can be a URL or a domain, and the answer differs:

**URL.** The response adds a `verdict`, plus a `matched` object with the `layer` that decided and the page type `id`. Built-in path rules report `rules`. Pages known from the catalogue of real sites report `page_type_db`. Typical ids include `login`, `checkout`, `upload` and `wiki_edit`.

**Domain.** You get the catalogue record: `found`, the site's `language`, and `page_types`, which maps each page type to the URL where that page lives on the site.

## Where it goes in an agent

Agent runtimes in Go usually have a tool registry and an executor loop. Wrap the browsing tool, not the model call:

```go
type guardedBrowser struct {
	next  Browser
	guard *aiagentallowlist.Client
}

func (g guardedBrowser) Open(ctx context.Context, url string) (string, error) {
	ctx, cancel := context.WithTimeout(ctx, 4*time.Second)
	defer cancel()

	r, err := g.guard.Check(ctx, url)
	if err != nil {
		return "Navigation paused: the policy check did not answer. A person must approve this step.", nil
	}
	if r["verdict"] != "allow" {
		id := "restricted"
		if m, ok := r["matched"].(map[string]any); ok {
			if s, ok := m["id"].(string); ok {
				id = s
			}
		}
		return fmt.Sprintf("Navigation refused: %s pages are handled by a person.", id), nil
	}
	return g.next.Open(ctx, url)
}
```

Three choices in that snippet are deliberate:

1. **The refusal is returned as tool output, not as a Go error.** The model reads it and adapts. It can pick another source or hand the step back. A Go error usually aborts the whole run.
2. **Only an explicit `allow` proceeds.** Future verdict values fail safe.
3. **A failed check stops the step.** An unreachable guard is not a green light.

## Planning with the domain catalogue

For workflows that visit known sites, look them up once at the start and show a person which steps will need them:

```go
site, _ := c.Check(ctx, "shopify.com")
if found, _ := site["found"].(bool); found {
	pt, _ := site["page_types"].(map[string]any)
	for kind, u := range pt {
		fmt.Printf("%-10s %v\n", kind, u)
	}
}
```

The output doubles as documentation for reviewers: here are the login, checkout and account pages the agent will stay away from.

Sites missing from the catalogue return `found: false`. URL checks still apply the path rules, so `/login`, `/checkout` and similar paths are caught on unknown sites too.

## Client details

`New` returns a `*Client` with exported `APIKey`, `BaseURL` and `HTTPClient` fields. The default HTTP timeout is 30 seconds. For agents, keep a tighter deadline on the context, as in the example, so a slow check never stalls a run.

| Outcome | Returned as |
|---|---|
| Empty key or URL | plain `error`, no request |
| HTTP status 400 or higher | `*aiagentallowlist.APIError` with `Status` and `Body` |
| Timeout or cancellation | the context's error |
| Body that is not a JSON object | JSON decoding error |

Check with `errors.As(err, &apiErr)`. A 429 means slow down. A 401 or 403 means the key or the quota needs attention, which a retry cannot fix.

## Audit trail

Log every decision with the agent run ID, the URL, the verdict and the matched id. After an incident, that log shows what the agent attempted and what stopped it. Day to day, it shows where workflows keep reaching sensitive pages. That usually means a step should be redesigned to include a person, not that the rule should go.

## Tests that keep the guard honest

A guard that silently disappears in a refactor is worse than none. Pin it with a test:

```go
srv := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, _ *http.Request) {
	fmt.Fprint(w, `{"verdict":"deny","matched":{"layer":"rules","id":"checkout"}}`)
}))
defer srv.Close()

guard := aiagentallowlist.New("test")
guard.BaseURL = srv.URL
out, _ := guardedBrowser{next: failIfCalled{}, guard: guard}.Open(ctx, "https://shop.test/checkout")
if !strings.Contains(out, "refused") {
	t.Fatal("checkout was not refused")
}
```

`failIfCalled` is a stub browser that fails the test if the guard lets the request through.

## Why page-level rules

Security guidance for LLM applications, including the OWASP Top 10 for LLM Applications under "excessive agency", recommends limiting what an agent can do without human sign-off. Whole-domain blocking is too coarse for that, because the same site holds harmless reading pages and high-risk action pages. Page types let an agent research widely while the few pages that move money, change accounts or publish content stay behind a person.

## Adjacent controls

- Agents also call other AI services. Keep a register of [which AI services an agent should never reach](https://www.aitoolsblocklist.com).
- Agents and assistants often run without IT's knowledge. [Find unsanctioned AI tools in proxy logs](https://www.shadowaitools.com) before they cause surprises.
- Policies that depend on subject matter can use [topic categories for every URL an agent visits](https://www.websitecategorizationapi.com).

## Rolling it out

Start in observe mode: run the check, log the verdict, but let every navigation through. A week of logs shows which workflows touch sensitive pages and how often. Then switch the guard to enforce, beginning with agents that can spend money or change records. Teams that skip the observe phase tend to discover the blocked steps through failed runs instead of a report.

Python agents can use [the aiagentallowlist PyPI package](https://pypi.org/project/aiagentallowlist/), TypeScript agents [its npm twin](https://www.npmjs.com/package/aiagentallowlist), and Flutter apps [the pub.dev version](https://pub.dev/packages/aiagentallowlist).

## License

MIT
