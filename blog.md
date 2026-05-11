# From Chaos to Clarity: Building the Krkn-AI Visualization Dashboard — My LFX Mentorship Journey

*By Debangi Ghosh | IIT (BHU) Varanasi, MnC '27 | LFX Mentee @ CNCF — krkn-chaos*

---

## The Problem That Started It All

Kubernetes is hard. Chaos engineering on Kubernetes is even harder. And making sense of the results of chaos experiments? That, for a long time, was the unsolved piece.

[Krkn-AI](https://github.com/krkn-chaos/krkn-ai) is a fascinating system — it uses a genetic algorithm to automatically evolve and discover the most effective chaos scenarios to test your cluster's resilience. Over multiple generations, it mutates scenarios, evaluates their fitness (how much chaos they cause while staying within SLO bounds), and keeps the strongest ones. By the end of a run, you have a wealth of data: `results.json`, `all.csv`, `health_check_report.csv`, per-scenario YAML files, logs, and auto-generated graphs.

The trouble? After even a modest experiment run — say 2 generations with 5 scenarios each — you're staring at dozens of files, hundreds of rows of CSV data, multiple YAML configs, and log files that scroll for pages. An engineer trying to answer the question *"Which scenario was the most damaging? Which service degraded first? Did we improve over generations?"* would have to open file after file, mentally correlate timestamps, and piece together a picture that should have been obvious at a glance.

LLM-based text summaries helped, but text alone can't show you a trend line, a heatmap of service degradation, or a side-by-side comparison of generation 0 vs generation 2. Visual pattern recognition is irreplaceable.

That's the problem [Issue #74](https://github.com/krkn-chaos/krkn-ai/issues/74) set out to solve. And it's the problem I spent my LFX Mentorship 2026 building a solution for.

---

## The Proposal: What I Promised

When I applied for the LFX Mentorship under CNCF's krkn-chaos project, I had already done my homework. Before submitting my proposal, I had:

- Deployed the **robot-shop** microservices application locally on Kubernetes
- Generated a valid `krkn-ai.yaml` configuration
- Run Krkn-AI as a Docker container, executed chaos experiments, and explored the `results/` directory
- Built a **proof-of-concept** using a Python Flask backend + React + Vite frontend with Recharts, showing fitness progression charts, health check tables, and even an early AI-summary feature

The proposal outlined a four-layer architecture:

```
Krkn-AI Artifacts (JSON / CSV / YAML / Logs)
           ↓
Data Ingestion & Normalization Layer (Python)
           ↓
Analysis & Insight Engine (Statistics + Anomaly Detection)
           ↓
Visualization & UI Layer (Interactive Web GUI / Generated Report)
           ↓
Optional AI Summary Layer (LLM-assisted Explanations)
```

The plan was methodical: model the data, parse it, analyze it, visualize it, then optionally summarize it with an LLM. I estimated a 12-week timeline, with mid-term evaluations around week 7.

After I got selected, my mentor [Rahul Shetty](mailto:rashetty@redhat.com) shared the onboarding document with the actual week-by-week plan, and we agreed on a key technical decision early on: instead of a Flask + React stack (which would require a separate build process and add complexity), we would use **Streamlit** for the visualization layer — a Python-native framework that keeps everything in one ecosystem, is trivially deployable, and integrates beautifully with pandas and Plotly.

This was the right call, and it shaped everything that followed.

---

## PR #180: The First Dashboard

The first major PR — [#180](https://github.com/krkn-chaos/krkn-ai/pull/180) — established the foundations of the Streamlit dashboard. Let me walk you through exactly what was built.

### The Data Loading & Parsing Layer

Before any chart can be drawn, the raw experiment artifacts need to be read, validated, and transformed. The dashboard parses three primary data sources:

**`results.json`** — the experiment-level summary. This contains the overall run metadata: unique run ID, start/end time, number of generations completed, total and unique scenarios executed, the best fitness score found, the average fitness score, and the full `fitness_progression` array (one entry per generation with `best` and `average` fields).

**`health_check_report.csv`** — the per-service, per-scenario health metrics. Each row captures a scenario × service combination with columns for `min_response_time`, `max_response_time`, `average_response_time`, `success_count`, and `failure_count`.

**`all.csv`** — a consolidated view of all metrics across scenarios and generations, used for cross-generation aggregation.

**Per-generation YAML files** (in `yaml/generation_N/`) — scenario-level configuration data used for labeling and metadata.

One early piece of feedback from my mentor was: *handle the case where health check data is missing*. Not every run completes cleanly. The parser was updated to gracefully degrade — if a file is absent or malformed, the relevant tabs show an informative message rather than throwing an unhandled exception.

### Global Filters — The Unsung Hero of the Dashboard

Before diving into individual tabs, let's talk about the feature that makes everything else useful: the **global filter panel** in the left sidebar.

Rather than having each tab carry its own filter controls (which fragments the UX and makes cross-tab exploration awkward), I implemented a unified sidebar with:

- **Scenario filter**: A multi-select dropdown populated with all scenario names derived from the YAML files. You can filter down to one, a few, or all scenarios. Scenario names are displayed as human-readable labels (e.g., `scenario_1`, `scenario_2`) rather than arbitrary indices.
- **Generation filter**: Filter results to specific generations, useful when you have many generations and want to focus on a particular evolutionary step.
- **Top-K / Top-P filtering**: Select the top K scenarios by fitness score, or the top P percent — great for quickly isolating the "most interesting" chaos scenarios without manually scanning rankings.
- **Service filter**: Filter health check views by specific microservice (cart, catalogue, payment, user, ratings, shipping, etc.).

Every chart and table in the dashboard respects these global filters. Change the scenario selection in the sidebar, and the fitness chart, the health check heatmap, the anomaly tab — all of them update simultaneously. This was one of the design choices I'm most proud of because it turns the dashboard from a collection of charts into a coherent exploration tool.

---

## The Dashboard Tabs: A Tour

### Tab 1: Main Dashboard (Overview)

The first thing a user sees after loading results is the **Overview tab**, which provides the high-level experiment summary.

At the top, a **metrics row** pulled from `results.json` displays:
- Best fitness score across all generations
- Total scenarios executed
- Number of generations completed
- Experiment duration

Below that, the **Krkn-AI configuration panel** shows the genetic algorithm settings from the YAML config: composition rate, crossover rate, mutation rate, scenario mutation rate, and population size. This is critical context — you can't interpret fitness scores without knowing the parameters that generated them.

The **scenario overview table** uses Streamlit's native `st.dataframe` (not a plain table) to display all scenarios with their key metrics side by side. Each row shows the scenario's fitness score, generation, and key health indicators. The "All" option ensures no data is hidden by default.

A **scenario distribution histogram** shows how scenarios are spread across generations — useful for spotting whether the genetic algorithm is exploring broadly or converging early.

The **scenario-wise fitness variation chart** plots fitness scores per scenario as a grouped bar chart, giving an immediate visual comparison of which scenarios pushed the system hardest.

### Tab 2: Fitness Score Evolution

This tab answers the core question of chaos engineering with a genetic algorithm: *Is the system getting better at finding breaking points over time, or has evolution stalled?*

The **fitness progression chart** plots two lines from the `fitness_progression` array in `results.json`:
- **Average fitness per generation** — how the population as a whole is performing
- **Best fitness per generation** — the peak performance of the strongest individual scenario in each generation

A rising best-fitness line with a converging average means the algorithm is successfully evolving. A flat line might mean you need more generations, a higher mutation rate, or that your system is unusually resilient.

The interactive detail added here: **clicking on a generation** drops down a tabular view of all scenarios from that generation, showing their individual scores and metadata. This drill-down capability lets you move from the macro trend to the specific scenarios that drove (or dragged) it.

### Tab 3: Health Check Reports

This is arguably the richest tab in the dashboard, built from `health_check_report.csv`.

**Response time table** — A filterable, sortable dataframe showing min, max, and average response times per service per scenario. At a glance you can see which service in which scenario had the worst degradation.

**Response time heatmap** — Service names on one axis, scenarios on the other, with cell color encoding average response time. Darker cells = slower = more degraded. This is the single most visually immediate way to answer "which service is the weakest link?" You can filter the heatmap by service via a dropdown to zoom into specific components.

**Resilience Radar Chart** — A polar/spider chart where each axis represents a microservice, and the score for each service is computed as `1 / avg_response_time`. Services that remained fast under chaos score high; services that slowed dramatically score low. A balanced, large polygon means overall good resilience. A lopsided polygon immediately shows you where to focus hardening efforts.

**Overlay comparison chart** — All scenarios plotted together on a single response-time chart, both generation-wise and scenario-wise. This is where you see the full picture of how different chaos injections interact with your services across the entire experiment.

### Tab 4: Response Time Over Experiment Duration

Scenario-level YAML files from `yaml/generation_N/` contain the time-series response data captured during each chaos run. This tab renders **generation-wise line charts** showing how response times evolved during each scenario's execution window.

Crucially, chaos injection events are annotated directly on the timeline as vertical markers — so you can see the exact moment traffic spiked, a pod was killed, or network latency was injected, and correlate it with the response time curve. Without these annotations, a spike in the chart is just a spike; with them, it's a story.

Filters for both scenario and service let you isolate exactly the view you need.

### Tab 5: Generation-wise Aggregated Results

This tab provides a generation-level rollup from `all.csv` and `health_check_report.csv`. Rather than looking at individual scenario runs, you see aggregated statistics per generation: mean response times, total failure counts, success rates.

The main value here is spotting **inter-generation trends** that aren't visible in the individual scenario views. Did failures increase from generation 0 to generation 2? Did average response times improve as the algorithm learned which scenarios are most disruptive? This tab answers those questions.

### Tab 6: Anomaly Detection

This tab was introduced in [PR #200](https://github.com/krkn-chaos/krkn-ai/pull/200) and represents one of the more technically interesting pieces of the project.

The premise: not all unusual results are easy to spot by eye. A service whose average response time is 40ms when the norm is 12ms might not look alarming on a color scale — but statistically, it's a 3.3σ outlier. The anomaly detection layer surfaces these automatically.

**Two detection methods are combined:**

1. **Deviation from baseline** — For each service, we compute the baseline average response time from a "calm" scenario (or from the global average across all scenarios). Any scenario where a service's response time deviates from this baseline by more than a configurable threshold is flagged.

2. **Z-score anomaly detection** — For each service across all scenarios, we compute the mean and standard deviation of response times. Scenarios where `|response_time - mean| / std > threshold` (typically 2 or 3) are flagged as statistical outliers.

Flagged anomalies are displayed in a dedicated panel showing: which service was anomalous, in which scenario, the observed response time, the baseline/expected value, and the computed Z-score or deviation percentage.

This tab also flags **fitness score drops** — scenarios where the fitness score dropped significantly compared to the previous generation's best, which may indicate a regression or an instability in the genetic algorithm's evolution path.

The anomaly detection doesn't replace expert judgment — it amplifies it. Rather than asking an engineer to scan hundreds of data points, the dashboard highlights the ten that matter most.

### Tab 7: Generated Graphs

Krkn-AI already produces graphs as part of its standard output — matplotlib-based plots of fitness trends, scenario comparisons, and health check summaries. Rather than duplicate this work or ignore it, the dashboard embeds these **existing generated graphs** directly in a dedicated tab.

This tab reads the `reports/graphs/` directory from the results folder and displays each graph inline. It's a simple but important integration — it means the dashboard is a single destination for all experiment output, not an alternative to the existing artifacts.

### Tab 8: Logs Viewer

Raw log files (`scenario_1.log`, `scenario_2.log`, `run.log`) are notoriously hard to read, especially when you're looking for events that correlate with a spike you saw in the charts.

The Logs tab provides a **structured log viewer** with:
- A file selector to switch between scenario logs and the main run log
- Proper formatting (monospaced, scrollable, with line breaks preserved)
- Color-coded severity levels where detectable

This might seem like a small thing, but having logs available in the same interface as the charts — without switching to a terminal — makes the feedback loop much tighter.

### Tab 9: Failed Scenarios

A deliberate design decision in the checklist was to **separate failed scenarios from valid data**. When a scenario fails (health check returns non-200, scenario execution errors, etc.), including its data in fitness comparisons and response time charts pollutes the analysis.

The Failed Scenarios tab collects and displays these distinctly — showing what failed, which generation it was in, and the reported error where available. This keeps the other tabs clean while ensuring no information is hidden from the user.

---

## PR #200: Baseline Scoring, Anomaly Detection, and HTML Export

The second major PR extended the dashboard with three significant capabilities.

### Baseline Score Evaluation

One of the most powerful additions: a **baseline run concept**. The idea is to establish what "normal" looks like — run the system with no chaos injected, capture the fitness score and response times, and use that as a reference point for all subsequent comparisons.

The dashboard visualizes:
- **Baseline fitness score** alongside the experiment fitness scores per generation
- **Delta charts**: how much better or worse each generation performed relative to baseline
- **Improvement trend**: is the genetic algorithm actually finding scenarios that push beyond the baseline degradation threshold?

This transforms fitness scores from abstract numbers into actionable insights. A best fitness of 3.34 means very little on its own. A best fitness of 3.34 against a baseline of 1.20 tells you the algorithm found scenarios that degrade your system nearly 3× beyond normal — that's a finding worth acting on.

### HTML Report Export

For teams using CI/CD pipelines, a live Streamlit server isn't always the right delivery mechanism. A common workflow is: run Krkn-AI as part of a pipeline, capture artifacts, and attach a report to the build.

The HTML export feature generates a **static, self-contained HTML report** from the dashboard's current state. Every chart is exported as an interactive Plotly figure embedded in the HTML, every table is rendered as an HTML table, and the full anomaly analysis is included. The output is a single `.html` file that can be attached to a CI/CD build artifact, emailed to a team, or committed to a repository.

The CLI integration is clean:

```bash
# Start the dashboard for a live run
krkn_ai run --monitoring

# View results from a completed run
krkn monitor

# Export a static HTML report
krkn monitor --export-html
```

### Dual Monitoring Mode

A thoughtful UX detail: the dashboard auto-detects whether a run is in progress or completed.

- **Live run mode** (triggered by `--monitoring` during `krkn_ai run`): Streamlit polls the results directory at a fixed interval, reloads artifacts, recomputes metrics, and refreshes charts. The localhost URL is printed in the terminal logs. You can watch the fitness evolution in real time as generations complete.
- **Post-run mode** (triggered by `krkn monitor` on a completed results directory): Polling is disabled. The dashboard loads once, renders everything, and stays static.

---

## Architecture: How It All Fits Together

The final dashboard structure under `krkn_ai/dashboard/` reflects the layered architecture from the proposal:

```
krkn_ai/dashboard/
├── dashboard.py          # Main Streamlit entrypoint, tab layout, global filters
├── data_loader.py        # Parsers for JSON, CSV, YAML artifacts
├── data_processor.py     # Aggregation, normalization, baseline computation
├── anomaly_detector.py   # Z-score and baseline deviation logic
├── charts.py             # Plotly chart builders for each visualization
├── html_exporter.py      # Static HTML report generation
└── utils/
    └── logger.py         # Logging utilities
```

The data flows cleanly through each layer: raw files → structured DataFrames → computed metrics → Plotly figures → Streamlit components (or HTML export). The global filter state is maintained in `st.session_state` and threaded through every chart builder call, ensuring consistency across the entire UI.

---

## What I Learned

Building this over 12 weeks taught me things no coursework could:

**Data is messier than you expect.** Partial runs, missing files, malformed CSVs, YAML configs with optional fields — the real world doesn't deliver clean inputs. Defensive parsing with graceful degradation isn't optional; it's the whole game.

**Architecture decisions are permanent.** Choosing Streamlit early meant the entire codebase could be Python-native, making it easier for chaos engineering practitioners (who are already comfortable with Python) to read, modify, and contribute to the dashboard. A React frontend would have added expressive power at the cost of accessibility.

**Visualization is a form of communication.** A chart that requires explanation has failed. Every design choice — which chart type, which axis labels, which color scale — is an argument about what the user should notice first. The anomaly tab exists because I kept asking myself: *if an engineer has 90 seconds to scan this dashboard, what must they not miss?*

**Open source is iterative.** PR #180 was opened "early for feedback," as the description explicitly states. The mentor's code review comments directly shaped the final implementation — handle missing health check data, disable the configurable link-opening behavior. Good mentorship is two-way.

---

## What's Next

The checklist in Issue #166 still has open items:

- Unit tests for the data loading, preprocessing, and chart generation layers
- Support for custom output config names (when users have non-default result file naming)
- Developer guide documentation for the visualization module
- Running with larger synthetic datasets to stress-test the dashboard with many generations and scenarios
- Final PR merge and inclusion in the krkn-ai release

The foundation is solid. The architecture is extensible. And for the first time, engineers running Krkn-AI chaos experiments have a single place to go to understand what their system did under pressure — and what they should fix next.

---

## Links

- **Krkn-AI repo**: [github.com/krkn-chaos/krkn-ai](https://github.com/krkn-chaos/krkn-ai)
- **Feature issue (#74)**: [github.com/krkn-chaos/krkn-ai/issues/74](https://github.com/krkn-chaos/krkn-ai/issues/74)
- **Checklist issue (#166)**: [github.com/krkn-chaos/krkn-ai/issues/166](https://github.com/krkn-chaos/krkn-ai/issues/166)
- **PR #180 (Initial dashboard)**: [github.com/krkn-chaos/krkn-ai/pull/180](https://github.com/krkn-chaos/krkn-ai/pull/180)
- **PR #200 (Anomaly detection + HTML export)**: [github.com/krkn-chaos/krkn-ai/pull/200](https://github.com/krkn-chaos/krkn-ai/pull/200)
- **My GitHub**: [github.com/debangi29](https://github.com/debangi29)
- **LFX Mentorship**: [mentorship.lfx.linuxfoundation.org](https://mentorship.lfx.linuxfoundation.org)

---

*If you're a chaos engineering practitioner using Krkn-AI, or if you're curious about building visualization layers on top of complex experiment outputs, I'd love to hear from you. Drop a comment or connect on LinkedIn.*
