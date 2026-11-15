# Building a Reusable Data Quality Validation Framework

## 1. Interview scenario

Multiple pipelines need consistent checks for schema, nullability, uniqueness, ranges, and referential integrity. Build a lightweight Python validation layer with structured results.

## 2. Design principles

- Separate rule definition from execution.
- Return all actionable failures instead of stopping at the first one.
- Include dataset, rule, severity, failed count, and sample identifiers.
- Distinguish hard failures from warnings.
- Avoid logging full sensitive rows.

## 3. Simple rule engine

```python
from dataclasses import dataclass

@dataclass
class ValidationResult:
    rule: str
    passed: bool
    failed_count: int
    severity: str
    sample_indices: list

def validate_not_null(rows, field, severity="ERROR", sample_limit=5):
    failed = [
        index for index, row in enumerate(rows)
        if row.get(field) is None
    ]
    return ValidationResult(
        rule=f"not_null:{field}",
        passed=not failed,
        failed_count=len(failed),
        severity=severity,
        sample_indices=failed[:sample_limit],
    )

def validate_unique(rows, field, severity="ERROR", sample_limit=5):
    seen = set()
    failed = []
    for index, row in enumerate(rows):
        value = row.get(field)
        if value in seen:
            failed.append(index)
        else:
            seen.add(value)
    return ValidationResult(
        rule=f"unique:{field}",
        passed=not failed,
        failed_count=len(failed),
        severity=severity,
        sample_indices=failed[:sample_limit],
    )

def run_validations(rows, rules):
    return [rule(rows) for rule in rules]
```

## 4. Dry run

Given IDs `[10, 11, 11]`, the uniqueness rule reports one duplicate row (the second occurrence), with `passed=False` and `failed_count=1`. A null check on `customer_id` independently reports missing values.

## 5. Complexity

For N rows:
- Not-null: O(N) time, O(S) sample-index memory where S is capped.
- Uniqueness: average O(N) time, O(U) memory for U unique values.
- A complete framework's cost is the sum of rule costs; avoid repeated full scans when a shared profiling pass is possible.

## 6. Scaling beyond Python lists

For large datasets:
- Push checks into SQL (`COUNT`, `COUNT DISTINCT`, anti-joins).
- Use Spark aggregations for distributed data.
- Sample only for diagnostics, not for hard correctness checks unless policy explicitly allows probabilistic validation.
- Partition checks and compare expected vs observed metrics.
- Treat schema contracts as versioned artifacts.

## 7. Production considerations

- Define thresholds (e.g., null rate under 0.1%) and severity.
- Capture run ID, source partition, event time, and rule version.
- Support quarantine and remediation workflows.
- Avoid treating `COUNT DISTINCT` as free at scale.
- Define referential checks against a consistent snapshot.
- Alert on trend changes, not only absolute failures.
- Do not silently coerce invalid types.

## 8. Interviewer–candidate discussion

**Interviewer:** Should a single bad record fail the whole pipeline?

**Candidate:** It depends on data criticality and contract. A primary key violation may be a hard failure; a small optional-field null rate may be a warning or quarantine. The policy should be explicit and observable.

**Interviewer:** How do you prevent the framework becoming slow?

**Candidate:** Push computation to the execution engine, combine compatible aggregations, bound diagnostic samples, and measure rule-level runtime.

## 9. Tests and edge cases

- Empty dataset.
- All-null column.
- Duplicate nulls: decide whether null counts as a duplicate.
- Unhashable values in uniqueness check.
- Mixed types.
- High-cardinality key.
- Warning threshold vs hard failure.
- Referential check against changing reference data.

```python
def test_unique_rule():
    result = validate_unique(
        [{"id": 1}, {"id": 2}, {"id": 1}], "id"
    )
    assert result.passed is False
    assert result.failed_count == 1
    assert result.sample_indices == [2]
```

## 10. Follow-ups

1. Implement range and referential-integrity rules.
2. Add configurable thresholds.
3. How would you validate a billion-row table?
4. How do you version and roll out data contracts safely?
