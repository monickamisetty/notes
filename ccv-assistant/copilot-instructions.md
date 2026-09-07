## Multi-rule and aggregate implementation validation

Do not assume that one business requirement must be fully implemented by one
CCV rule.

A requirement may be implemented across:

- multiple CCV JSON Logic rules
- React/UI behavior
- JSON Logic + React/UI behavior

When validating a requested rule:

1. Read the requested rule.
2. Read the associated requirement.
3. Identify which part of the requirement the requested rule implements.
4. Search for companion rules or UI behavior implementing the remaining effects.
5. Evaluate the aggregate implementation.

Do not return CONCERN merely because the requested rule implements only one
part of a multi-effect requirement.

Example:

Requirement:
- Coverage B unavailable
- show info message

Rule A:
- returns x-gw-forbidden

Rule B:
- returns x-gw-cc-rule-info

If A + B together satisfy the requirement, the aggregate implementation may
receive PASS.

The answer should identify the responsibility of each rule.

## Evidence-based concerns

Do not classify speculative risks as CONCERN.

Statements about the following require repository evidence:

- execution order
- rule response merge behavior
- UI rendering level
- component lifecycle behavior
- state-reset sequencing
- one rule overriding another

If the required architecture/runtime behavior is not documented, use
CANNOT VERIFY and state what documentation is missing.

Do not describe a multi-rule implementation as fragile solely because the
requirement is split across multiple rules.

## Validation target

The primary validation question is:

"Does the complete discovered implementation satisfy the business requirement?"

It is not:

"Does this one rule independently implement every sentence in the requirement?"

Still report rule-level defects when an individual rule itself contains incorrect
logic.





SKILL.md
## Aggregate implementation rule

The "Actual Implementation" may contain multiple artifacts.

Before issuing a verdict, search for:

- companion CCV rules
- rules with the same trigger
- rules affecting the same target coverage/term/choice
- documented React/UI behavior

Evaluate the requirement against the combined behavior.

Do not penalize an individual rule simply because another rule intentionally
implements a different effect of the same requirement.