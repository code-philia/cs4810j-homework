# Homework: improve a GUI testing agent

You get a GUI testing agent built on [browser-use](https://github.com/browser-use/browser-use) and a vision-language model (VLM). It runs natural-language test cases against a real web app, [BookStack](https://www.bookstackapp.com/), and decides whether the app behaves as expected. The test cases come from [WebTestPilot-benchmark](https://github.com/code-philia/WebTestPilot-benchmark).

Every test case runs twice:

| Variant | The app | Correct verdict |
| --- | --- | --- |
| `clean` | works as intended | **PASS** |
| `bug` | has a bug injected into the page (an item missing, a wrong value, a duplicate, ...) | **FAIL** |

A test passes when the agent's verdict is correct. The baseline agent is slow, uses many tokens, and misses most bugs. **Your job is to build an agent that does better on these metrics:**

| Metric | Meaning | Goal |
| --- | --- | --- |
| Accuracy | share of tests where the agent's verdict is correct. A crash or timeout (`ERROR`) counts as wrong | higher |
| Bug recall | share of `bug` tests the agent reported as FAIL | higher |
| False alarm rate | share of `clean` tests the agent reported as FAIL | lower |
| Seconds per test | time the agent spends on one test | lower |
| Tokens per test | LLM tokens (prompt + completion) the agent uses on one test | lower |

## Step 1: Set up the environment

You need:

- [uv](https://docs.astral.sh/uv/getting-started/installation/), which installs Python and the packages for you;
- **Docker**: [Docker Desktop](https://www.docker.com/products/docker-desktop/) on Windows and macOS, Docker Engine on Linux. Start it before you continue;
- an API key for an OpenAI-compatible endpoint whose model accepts images, for example our class-provided endpoints and keys or [OpenRouter](https://openrouter.ai/).

The commands are the same on Windows, macOS and Linux:

```bash
git clone https://github.com/code-philia/cs4810j-homework.git
cd cs4810j-homework
uv run install.py
```

`install.py` installs the Python packages and Chromium, checks that Docker is running, and creates `.env`. Open `.env` and put in your API key. The defaults reproduce the baseline:

```ini
OPENAI_API_KEY=sk-or-...
OPENAI_BASE_URL=https://api.arc-bench.com/v1
LLM_MODEL=deepseek-flash
LLM_REASONING_EFFORT=low
```

## Step 2: Run the baseline and get its test report

```bash
uv run pytest
```

This runs the baseline agent (`agents/baseline.py`) on all 8 tests: 4 test cases × 2 variants. It takes about 13 minutes. The first run also downloads BookStack and MySQL (about 1 GB) and starts them on http://localhost:8080. To try a single test case first, select it with pytest's `-k`: `uv run pytest -k search`.

Before each test, the harness resets BookStack's database, opens a browser and logs in, and for `bug` tests injects the bug. Then it hands the browser to the agent.

When the run finishes, pytest prints the metrics. The baseline's run:

```
test                                       verdict  correct   seconds    tokens  calls
count_recently_created_pages[clean]        PASS     yes          14.2    31,386      4
count_recently_created_pages[bug]          PASS     NO          180.1   145,423     13
create_book[clean]                         PASS     yes          46.5    77,837     10
create_book[bug]                           FAIL     yes          37.7    47,934      6
recently_viewed_book[clean]                PASS     yes          34.7    31,063      4
recently_viewed_book[bug]                  PASS     NO          108.8    48,568      5
search[clean]                              PASS     yes          33.9    40,083      5
search[bug]                                PASS     NO          279.2   211,978     19

metric                baseline (baseline-20261017-124305)
---------------------------------------------------------
Accuracy                                             0.62
Bug recall                                           0.25
False alarm rate                                     0.00
Errors                                                  0
Seconds per test                                     91.9
Tokens per test                                    79,284
LLM calls per test                                    8.2
Total seconds                                         735
Total tokens                                      634,272

Test report: reports/baseline-20261017-124305/report.html
Metrics again, or compared with another run: uv run metrics.py reports/baseline-20261017-124305
```

Each run writes a folder `reports/<agent>-<date>-<time>/`:

| File | What it is |
| --- | --- |
| `report.html` | the test report: open it in a browser. One row per test with the verdict, seconds and tokens, plus links |
| `<test>/trace.zip` | Playwright trace of the agent's browser session: one entry per agent step, each with a snapshot of the page. Open it at [trace.playwright.dev](https://trace.playwright.dev) or with `uv run playwright show-trace <path>` |
| `<test>/agent.gif` | a replay of the agent's steps |
| `<test>/conversation/` | every prompt sent to the LLM and its reply |
| `results.json` | the numbers behind the metrics |

Look at a few tests in the report before you change anything. Where do the time and tokens go? For each bug the baseline missed, what did it see, and why did it still say PASS?

## Step 3: Improve the agent

Copy the baseline and work on your copy:

```bash
cp agents/baseline.py agents/my_agent.py
uv run pytest --agent my_agent -k search        # quick check on one test case
```

The harness calls one function per test:

```python
async def run_test(task: Task, ctx: Context) -> Verdict:
    ...
```

| You get | What it is |
| --- | --- |
| `task.name`, `task.steps` | the test case. Each step has `.action` and `.expectation` |
| `ctx.cdp_url` | the prepared browser, already logged in on the start page. Attach with `BrowserSession(cdp_url=ctx.cdp_url)` |
| `ctx.llm(model=None)` | an LLM on your endpoint whose tokens are counted. `model` defaults to `LLM_MODEL`; pass another name to mix models. Use only LLMs from here |
| `ctx.trace_step(title)` | add a named step to the test's Playwright trace |
| `ctx.out_dir` | this test's report folder, for your own logs and screenshots |
| `ctx.memory_dir` | a folder that persists across tests and runs |

Return `Verdict(passed=True)` if every expectation held, or `Verdict(passed=False, reason=...)` if one did not. Raise an exception if your agent cannot reach a verdict; the test then counts as `ERROR`.

Useful options: `--headed` shows the browser window; `-k bug` runs only the `bug` tests; `-k "create_book and clean"` runs one test.

## Step 4: Rerun and get a new test report

When your agent is ready, run all tests with it:

```bash
uv run pytest --agent my_agent
```

This writes a new report folder, `reports/my_agent-<date>-<time>/`, next to the baseline's.

## Step 5: Compare the metrics

```bash
uv run metrics.py baseline my_agent
```

This takes the latest report of each agent and prints every metric for both side by side, with whether yours is better or worse, and by how much. Make sure both reports are full runs: a quick `-k` run in Step 3 would also count as your latest report. To pick reports yourself, pass their folders instead: `uv run metrics.py reports/<baseline run> reports/<your run>`.
