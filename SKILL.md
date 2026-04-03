---
name: autoresearch
description: Run self-improving autoresearch loops on any topic using bilevel optimization. Inner loop optimizes the task; outer loop optimizes how the inner loop searches by generating new search mechanisms.
category: mlops/research
version: 1.0.0
author: Hermes Agent
---

# Bilevel Autoresearch Skill

A self-improving research framework that combines Karpathy's autoresearch pattern with bilevel meta-optimization. The inner loop runs experiments on your topic; the outer loop autonomously discovers and injects new search mechanisms to improve how the inner loop explores.

## Core Concepts

### Inner Loop (Level 1)
The standard Karpathy-style autoresearch loop:
1. Propose a change based on current search mechanism
2. Execute experiment/test
3. Evaluate against measurable metric
4. Keep if improved, discard if not
5. Repeat indefinitely until stopped

### Outer Loop (Level 2) - Meta-Autoresearch
Optimizes HOW the inner loop searches:
1. Analyze inner loop's search patterns
2. Identify systematic blind spots or inefficiencies
3. Generate new search mechanisms as Python code (e.g., Tabu Search, Bandits, Orthogonal Exploration)
4. Inject mechanism at runtime
5. Observe improvement in search quality
6. Keep better mechanisms, discard worse ones

### Key Insight
> "If autoresearch can meta-autoresearch itself, it can, in principle, meta-autoresearch anything with a measurable objective."

## Architecture

```
~/.hermes/skills/bilevel-autoresearch/
├── SKILL.md              # This file
├── runner.py             # Main orchestration script
├── inner_loop.py         # Karpathy-style experiment loop
├── outer_loop.py         # Meta-optimization loop
├── mechanisms/           # Generated search mechanisms
│   ├── __init__.py
│   ├── base.py           # Base mechanism interface
│   ├── random_search.py  # Default: simple random exploration
│   └── ...               # Auto-generated mechanisms
├── eval/                 # Evaluation templates
│   └── README.md         # How to define your metric
└── state/                # Runtime state (not committed)
    ├── results.tsv       # Experiment log
    ├── mechanisms.tsv    # Mechanism performance log
    └── config.yaml       # Current configuration
```

# Usage

### Quick Start

```bash
# Run autoresearch on a topic (Non-blocking background mode)
hermes /bilevel-autoresearch "optimize my email subject line generation"
```

*Note: By default, this skill now initiates as a background process. You can continue chatting while the research loop runs. Use `/research-status` to poll for progress.*

### Interactive Mode

```bash
hermes /bilevel-autoresearch
# Follow the prompts to define:
...
# 2. How to measure success (metric)
# 3. Test inputs/examples
# 4. Constraints and guardrails
```

## Defining Your Task

### Required Components

1. **Task Definition**: What are you optimizing?
   - A prompt/skill to improve
   - A piece of code to optimize
   - A process to refine
   - Any measurable system

2. **Metric**: Single scalar score (higher or lower is better)
   - Must be computable without human judgment
   - Must be unambiguous about direction
   - Examples: accuracy%, loss value, speed ms, conversion rate

3. **Test Inputs**: Representative examples to evaluate against
   - At least 3-5 diverse cases
   - Should cover edge cases
   - Stored in `state/test_inputs.json`

4. **Constraints**: What cannot change
   - File paths that are off-limits (automatically includes `~/hermes-agent/`)
   - Required behaviors that must be preserved
   - Resource limits (time, memory, API calls)

### Example: Optimizing a Claude Skill

```yaml
task: "Improve landing page copy generation skill"
target_file: "~/.hermes/skills/landing-page-copy/SKILL.md"
metric:
  name: "checklist_score"
  direction: "higher"  # or "lower"
  max_value: 100
  eval_script: "evaluate_copy.py"  # Script that returns score 0-100
test_inputs:
  - "Write landing page for AI productivity tool"
  - "Write landing page for fitness app"
  - "Write landing page for B2B SaaS"
constraints:
  forbidden_paths:
    - "~/hermes-agent/"  # Never modify core Hermes code
  required:
    - "Must output under 150 words"
    - "Must include CTA with action verb"
time_budget_per_experiment: 300  # seconds
max_experiments: 100
```

## Search Mechanisms

The outer loop discovers and injects these automatically:

### Default: Random Search
- Proposes random changes within defined parameter ranges
- Simple, unbiased exploration
- Baseline for comparison

### Auto-Discovered Examples

**Tabu Search**: Remembers recent changes, avoids cycling back
```python
class TabuSearch(BaseMechanism):
    def __init__(self, tabu_list_size=10):
        self.tabu_list = deque(maxlen=tabu_list_size)
    
    def propose_change(self, current_state, history):
        # Avoid proposing changes in tabu list
        candidate = self.random_proposal(current_state)
        while candidate in self.tabu_list:
            candidate = self.random_proposal(current_state)
        self.tabu_list.append(candidate)
        return candidate
```

**Multi-Armed Bandit**: Allocates more trials to promising directions
```python
class ThompsonSampling(BaseMechanism):
    def __init__(self):
        self.alpha = defaultdict(lambda: 1)  # successes
        self.beta = defaultdict(lambda: 1)    # failures
    
    def propose_change(self, current_state, history):
        # Sample from posterior, exploit high-performing areas
        best_direction = self.sample_best_direction()
        return self.mutate_along_direction(current_state, best_direction)
```

**Orthogonal Exploration**: Forces exploration of directions LLM would avoid
```python
class OrthogonalSearch(BaseMechanism):
    def propose_change(self, current_state, history):
        # Analyze what the LLM keeps proposing
        common_patterns = self.analyze_common_proposals(history)
        # Propose something deliberately different
        return self.orthogonal_to_patterns(current_state, common_patterns)
```

## Guardrails

### Automatic Protections

1. **Hermes Core Code**: `~/hermes-agent/` is never modified
2. **State Files**: Only `state/` directory is writable for logs
3. **Git-Based**: All changes tracked via git, can revert anytime
4. **Time Budgets**: Experiments auto-kill after timeout
5. **Resource Limits**: Memory and API call caps enforced

### Manual Override

```yaml
# In state/config.yaml
safety:
  strict_mode: true  # Never modify anything outside allowed paths
  allowed_paths:
    - "~/.hermes/skills/"
    - "~/.hermes/workspaces/"
  forbidden_paths:
    - "~/hermes-agent/"
    - "~/.hermes/config.yaml"
    - "~/.hermes/.env"
```

## Results

### Experiment Log (`state/results.tsv`)

```
timestamp	commit	metric_value	status	description	mechanism_used
2026-03-29T10:00:00	a1b2c3d	0.72	keep	baseline	random_search
2026-03-29T10:05:00	b2c3d4e	0.78	keep	increase temperature to 0.8	random_search
2026-03-29T10:10:00	c3d4e5f	0.75	discard	add more examples	random_search
2026-03-29T10:15:00	d4e5f6g	0.82	keep	refine prompt structure	tabu_search
```

### Mechanism Performance (`state/mechanisms.tsv`)

```
mechanism_name	total_experiments	improvements	improvement_rate	avg_gain
random_search	50	12	0.24	0.03
tabu_search	30	10	0.33	0.05
thompson_sampling	20	8	0.40	0.07
```

## Output

After running:

1. **Improved artifact**: The optimized file/prompt/skill
2. **Experiment log**: Full history of what was tried
3. **Mechanism analysis**: Which search strategies worked best
4. **Best configuration**: The winning changes with diff

## Advanced Usage

### Multi-Objective Optimization

```yaml
metric:
  type: "composite"
  objectives:
    - name: "quality_score"
      weight: 0.7
      direction: "higher"
    - name: "generation_time_ms"
      weight: 0.3
      direction: "lower"
  combine: "weighted_sum"
```

### Parallel Experiments

```yaml
parallelism:
  enabled: true
  concurrent_experiments: 3
  strategy: "multi_batch"  # Like AutoResearchClaw
```

### Persistent Memory

```yaml
memory:
  enabled: true  # Like EvoScientist
  type: "experience_replay"
  max_episode_ids: 1000
```

## Integration with Hermes

The skill uses:
- **Subagents**: All experiments run via `delegate_task` for isolation
- **Terminal**: Code execution and testing
- **File tools**: Reading/writing artifacts
- **Git**: Change tracking and revert capability

## References

- Karpathy's Autoresearch: https://github.com/karpathy/autoresearch
- Bilevel Autoresearch Paper: https://arxiv.org/abs/2603.23420
- Ole Lehmann's Guide: https://x.com/itsolelehmann/status/2033919415771713715
