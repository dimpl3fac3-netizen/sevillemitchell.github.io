# Weekly Dataset Fusion Overrides

This reference shows how to pass dynamic source trust overrides into `buildWeeklyDataset`.
Project-and-metric overrides use the key format `Project Name:Metric Name` and take precedence over project-only overrides.

```ts
const dataset = await buildWeeklyDataset({
  overrides: {
    // For Alpha, trust Slack completely over Spreadsheet numbers.
    "Project Alpha": {
      spreadsheet: 0.2,
      slack: 0.99,
    },
    // For Beta, only modify trust values specifically for the Velocity metric.
    "Project Beta:Velocity": {
      spreadsheet: 0.5,
      slack: 0.85,
    },
  },
});
```

## Corrected fusion implementation

```ts
interface SourceWeights {
  spreadsheet: number;
  slack: number;
}

type DynamicWeightOverrides = Record<string, Partial<SourceWeights>>;

interface FusionOptions {
  overrides?: DynamicWeightOverrides;
}

const BASELINE_WEIGHTS: SourceWeights = {
  spreadsheet: 0.7,
  slack: 0.6,
};

function fuseMetricsWithProbability(metrics: WeeklyMetric[], options?: FusionOptions): WeeklyMetric[] {
  const groupings = new Map<string, WeeklyMetric[]>();
  const overrides = options?.overrides ?? {};

  for (const item of metrics) {
    const key = `${item.weekStart}_${item.project}_${item.metric}`;
    const group = groupings.get(key) ?? [];
    group.push(item);
    groupings.set(key, group);
  }

  const fusedDataset: WeeklyMetric[] = [];

  for (const items of groupings.values()) {
    if (items.length === 1) {
      fusedDataset.push(items[0]);
      continue;
    }

    const baseTemplate = items[0];
    const { project, metric } = baseTemplate;

    // Resolve weight dynamically: Specific "Project:Metric" -> "Project" -> system default.
    const projectMetricOverride = overrides[`${project}:${metric}`];
    const projectOverride = overrides[project];

    const resolvedWeights: SourceWeights = {
      spreadsheet:
        projectMetricOverride?.spreadsheet ??
        projectOverride?.spreadsheet ??
        BASELINE_WEIGHTS.spreadsheet,
      slack: projectMetricOverride?.slack ?? projectOverride?.slack ?? BASELINE_WEIGHTS.slack,
    };

    const isNumeric = items.every((item) => typeof item.value === "number");

    if (isNumeric) {
      let totalWeightedValue = 0;
      let totalWeight = 0;

      for (const item of items) {
        const weight = resolvedWeights[item.source as keyof SourceWeights] ?? 0.5;
        totalWeightedValue += (item.value as number) * weight;
        totalWeight += weight;
      }

      fusedDataset.push({
        ...baseTemplate,
        value: totalWeight > 0 ? Number((totalWeightedValue / totalWeight).toFixed(2)) : 0,
        source: "fused",
      });
    } else {
      const statusConfidences = new Map<string, number>();

      for (const item of items) {
        const status = String(item.value);
        const weight = resolvedWeights[item.source as keyof SourceWeights] ?? 0.5;
        statusConfidences.set(status, (statusConfidences.get(status) ?? 0) + weight);
      }

      let bestStatus: string | number = String(baseTemplate.value);
      let highestConfidence = -1;

      for (const [status, confidence] of statusConfidences.entries()) {
        if (confidence > highestConfidence) {
          highestConfidence = confidence;
          bestStatus = status;
        }
      }

      fusedDataset.push({
        ...baseTemplate,
        value: bestStatus,
        source: "fused",
      });
    }
  }

  return fusedDataset;
}
```

## Important fixes

- `fusedDataset.push(items)` was changed to `fusedDataset.push(items[0])` because `fusedDataset` stores `WeeklyMetric` objects, not arrays.
- `baseTemplate` now uses `items[0]` so `project`, `metric`, and object spreading reference a single metric record.
- Group creation avoids non-null assertions by initializing the array before pushing.
- Project-and-metric overrides are resolved before project-only overrides, then baseline weights.
