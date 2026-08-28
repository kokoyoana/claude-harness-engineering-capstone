# System 1 - Incomplete Claims Diagnosis

## Summary

Three claims terminated with `outcome=incomplete`:

- claim_01_kitchen_fire
- claim_06_low_confidence_escalation
- claim_07_tree_falls_on_car

To investigate, I reran the fixtures individually and added
`assistant_text` to the trace output.

## Claim 01

Run: 20260828_074326

Sequence:

```text
tool_use -> end_turn
```

The model recorded facts successfully but then requested
a damage-cost estimate in free text and terminated without
calling classify_claim, assess_severity, route_to_adjuster
or an escalation tool.

## Claim 06

Run: 20260828_074436

Sequence:

```text
tool_use -> tool_use -> tool_use -> tool_use -> end_turn
```

The model stated that the case should be escalated to a
human adjuster but did not invoke an escalation tool call.
It then requested a damage estimate and terminated.

## Claim 07

Run: 20260828_074533

Sequence:

```text
tool_use -> end_turn
```

The model requested a vehicle repair estimate in free text
instead of invoking classification, routing or escalation.

## Conclusion

The failures occur at the model-to-tool boundary.

The model expresses routing, escalation or clarification
intent in assistant prose but returns `end_turn` before
issuing the required terminal tool call.

The harness behaves correctly because it only continues
for `stop_reason=tool_use` and stops for
`stop_reason=end_turn`.