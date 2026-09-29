---
layout: post
title: "Eight Months of Risk-Based Approvals: What We Learned"
date: 2026-09-29
categories: [devops, automation, terraform, jamf]
---

## Theory meets 300 merge requests

Eight months ago, we [wrote about]({% post_url 2026-02-03-risk-based-approvals %}) building a risk-based approval system for managing Jamf Pro with Terraform. The idea was straightforward: changes affecting multiple servers, destructive actions, or high-risk resources needed approval, while single-server changes to operational resources could self-merge.

The system worked, and merge velocity went from roughly 15 MRs per month to over 60. Routine Jamf administration stopped requiring approval for every change. We gave up some of the speed of clicking around in the GUI, but we got version control, audit trails, and the ability to review what mattered. People stopped resenting the process.

But here's what we didn't anticipate: the classifier we shipped in February would have strangled productivity just as badly as mandatory approval for everything. What saved us was treating the classifier itself as infrastructure that needed iteration.

Hundreds of merge requests revealed edge cases we hadn't anticipated. Policy description updates triggered reviews. Dev environment experiments got flagged like production changes. Category reorganization faced the same scrutiny as LDAP modifications. Each problem had an obvious solution in hindsight, but none of them were obvious during initial design.

## Treating everything as a risk

The first issue surfaced within days. Someone opened an MR to update policy descriptions, making them clearer and more specific. The classifier demanded approval because it saw resource changes. Changing `description = "Install Office"` to `description = "Installs Microsoft Office 365 (Word, Excel, PowerPoint, Outlook)"` has zero functional impact, but Terraform detected modifications and the classifier counted them toward risk score.

A similar problem emerged when someone reorganized category assignments. A dozen configuration profiles got moved into better categories for easier navigation in Jamf Pro. The classifier saw twelve resource changes and flagged the MR as high-risk.

We'd assumed resource changes meant functional changes, but that assumption didn't hold. Some changes only affect how things appear in Jamf Pro without changing what they do. Description updates, category assignments, and notes fields were creating noise in the classification system.

We added a list of cosmetic fields:

```yaml
cosmetic_fields:
  universal:
    - category_id
    - description
    - info
    - notes
```

If a resource change only modifies these fields, it doesn't contribute to risk score. The classifier checks both the before and after states. If the only differences are cosmetic, the change doesn't count.

False positives dropped significantly. Organizational changes, documentation updates, and metadata improvements stopped requiring approval.

## Self Service descriptions

Universal cosmetic fields solved the obvious cases, but we discovered that false positives kept appearing from a more subtle source: Self Service descriptions.

Policies, Mac apps, and mobile device apps all have a `self_service_description` field that end users see in the Self Service application. Teams update it regularly to make things clearer. A policy description might go from "Installs Zoom" to "Installs Zoom video conferencing software. Allows joining meetings, screen sharing, and video calls."

Same functionality, better user communication. But the classifier flagged these changes because `self_service_description` is nested inside a `self_service` attribute block rather than being a top-level field like `description`. We taught the classifier to look deeper:

```python
def is_self_service_description_only_change(resource_change, self_service_types):
    """Check if only self_service_description changed."""
    # Verify it's a supported resource type
    if resource_type not in self_service_types:
        return False

    # Check that only the nested self_service_description field changed
    # while all other fields remained identical
```

This catches updates to user-facing documentation that don't affect what the resource actually does.

## Dev servers

We maintain test Jamf Pro servers for experimentation and beta testing. Initially, the classifier treated them identically to production. An engineer testing a new workflow or resource type would create, modify, and delete dozens of test resources, and each change added points to the risk score.

This created a perverse incentive: people avoided using dev environments because it triggered approval friction.

```yaml
excluded_servers:
  - test_server     # Internal test environment
  - dev_server      # Beta testing and experimentation
```

Changes to excluded servers don't contribute to risk scoring, which restored the dev environments to their intended purpose: safe experimentation.

## Bulk cosmetic changes

Point-based scoring was in the original design. A single low-risk change scores 1 point. Five or more points triggers review.

Cosmetic field detection introduced a gap. Someone would open an MR updating 30 policy descriptions. Before cosmetic detection, this scored 30 points and required approval. After cosmetic detection, it scored 0 points because descriptions are cosmetic.

Thirty simultaneous changes to anything represents scope. Even if each individual change is harmless, bulk operations have their own risks: copy-paste errors, automation gone wrong, mass changes based on a misunderstanding. We needed to distinguish between "this change is cosmetic" and "this change doesn't count at all."

Point weights look like this:

```yaml
risk_threshold: 5

points:
  per_resource_change: 1        # Base points for substantive changes
  destructive_action: 3          # Delete or replace
  high_risk_resource_type: 5     # Auth, settings, system config
  attribute_rule_match: 5        # All-computers scope, etc.
  multiple_servers: 10           # Affects blast radius significantly
```

Cosmetic changes don't add points. Substantive changes do. Five low-risk changes hit the threshold. This approach catches bulk operations that deserve review while letting individual routine changes through.

## Scope and blast radius

A configuration profile scoped to a specific computer group is routine. The same profile scoped to all computers is a different story. Push a misconfigured profile to a test group and you affect a dozen machines. Push it to all computers and you might break 1,000+ endpoints.

The original design included attribute-level rules for this. We could flag specific attribute values as high-risk regardless of resource type. We'd only implemented it for policies. Scope detection needed to cover more resource types:

```yaml
attribute_rules:
  jamfpro_policy:
    - path: scope.0.all_computers
      value: true
      reason: Policy scoped to all computers

  jamfpro_macos_configuration_profile_plist:
    - path: scope.0.all_computers
      value: true
      reason: Configuration profile scoped to all computers
    - path: redeploy_on_update
      value: "All"
      reason: Configuration profile set to redeploy to all scoped computers

  jamfpro_mac_application:
    - path: scope.0.all_computers
      value: true
      reason: Mac app scoped to all computers
```

The classifier compares before and after states. If the `all_computers` scope changes to `true`, it adds 5 points and flags the reason. If it's already `true` and staying that way, no flag. This catches the moment when blast radius expands without penalizing resources that already have broad scope.

## Application naming

Beyond scope rules, we discovered another source of false positives rooted in years of inconsistent practices. Different team members had different habits when writing smart group criteria for Application Title. Some wrote `"1Password"`, others wrote `"1Password.app"`. Both work identically in Jamf Pro.

As people standardized old smart groups, Terraform saw these as substantive changes. Someone would update a smart group to add new criteria and standardize the app name format while they were in there. Terraform flagged both changes.

```diff
- value = "1Password"
+ value = "1Password.app"
```

The criteria function identically, but Terraform sees different strings. We taught the classifier that `.app` extension differences are cosmetic:

```python
def is_smart_group_cosmetic_criteria_change(resource_change, cosmetic_config):
    """Check if smart group criteria only changed by .app extension."""
    # Verify it's a smart computer group
    # Check that all criteria differences are just .app extension
    before_normalized = before_value.rstrip(".app")
    after_normalized = after_value.rstrip(".app")

    if before_normalized != after_normalized:
        return False  # Substantive change
```

This eliminates false positives from historical inconsistencies in how criteria were written, allowing teams to gradually standardize their smart group criteria without triggering unnecessary reviews.

## Cosmetic resources

Jamf Pro icons presented yet another challenge. The API allows uploading and modifying icons but doesn't support deletion. Icon changes were adding to risk scores. Because we track icon content hashes using `terraform_data` resources to detect when icons need updating, each icon change would create two resource modifications: the icon itself and the hash tracker.

Someone would update a policy icon to use the new squircle icon from macOS 26 and the classifier would see two substantive changes. Some resource types are cosmetic by nature:

```yaml
cosmetic_resources:
  - jamfpro_icon       # Icons can't be deleted via API, changes are cosmetic
  - terraform_data     # Only used for icon content hash tracking
```

These resources don't add points or count toward volume.

## How the code grew

The classifier went from about 60 lines to over 400. The original version looped through resource changes, checked resource types against a high-risk list, looked for destructive actions, and counted affected servers. If anything triggered, mark it high-risk.

```python
def classify_terraform_plan(terraform_plan, risk_config):
    high_risk_reasons = []

    for resource_change in plan.get("resource_changes", []):
        # Check resource type against high-risk list
        # Check for destructive actions
        # Count affected servers

    risk_level = "high" if high_risk_reasons else "low"
    return risk_level, high_risk_reasons, affected_servers
```

The current version has layers of filtering:

```python
def classify_terraform_plan(terraform_plan, risk_config):
    risk_score = 0

    for resource_change in plan.get("resource_changes", []):
        # Skip excluded servers
        # Skip cosmetic resources entirely
        # Check if cosmetic-only change
        # Add points for substantive changes
        # Add points for high-risk resource types
        # Add points for destructive actions
        # Check attribute-level rules
        # Track servers with substantive changes

    # Add points for multiple servers
    risk_level = "high" if risk_score >= threshold else "low"
    return risk_level, risk_score, reasons, affected_servers
```

Most of the complexity lives in helper functions:
- `is_cosmetic_change()` checks if only universal cosmetic fields changed
- `is_self_service_description_only_change()` handles nested Self Service documentation
- `is_smart_group_cosmetic_criteria_change()` normalizes application title `.app` extensions
- `_get_cosmetic_fields()` builds conditional cosmetic rules based on other attributes
- `check_attribute_rules()` validates scope and other attribute-level patterns

## Configuration as interface

The original configuration was simple:

```yaml
high_risk_resource_types:
  - jamfpro_api_role
  - jamfpro_ldap_server
  # ... 15 items

low_risk_resource_types:
  - jamfpro_policy
  - jamfpro_script
  # ... 15 items

high_risk_actions:
  - delete
  - replace
```

The current configuration has more structure:

```yaml
risk_threshold: 5

points:
  per_resource_change: 1
  destructive_action: 3
  high_risk_resource_type: 5
  attribute_rule_match: 5
  multiple_servers: 10

excluded_servers:
  - test_server
  - dev_server

cosmetic_resources:
  - jamfpro_icon
  - terraform_data

cosmetic_fields:
  universal:
    - category_id
    - description
    - info
    - notes

  resource_specific:
    jamfpro_macos_configuration_profile_plist:
      fields:
        - payload_validate
      conditional:
        redeploy_on_update:
          type: value_not_equal
          value: "All"

attribute_rules:
  jamfpro_policy:
    - path: scope.0.all_computers
      value: true
      reason: Policy scoped to all computers

  jamfpro_macos_configuration_profile_plist:
    - path: scope.0.all_computers
      value: true
      reason: Configuration profile scoped to all computers
  # ... 8 rules total
```

Most tuning happens in the configuration rather than the code. The Python classifier rarely changes at this point. The YAML configuration changes constantly. Anyone on the team can propose classifier adjustments by editing a config file, which keeps the system maintainable and adaptable.

## What We Learned

### Ship and iterate

We thought we'd covered the major scenarios in the original design. But production use revealed gaps we hadn't anticipated: description updates, category reorganization, dev environment experimentation, years of inconsistent naming conventions creating standardization noise. Each edge case was obvious once we saw it.

### Context matters for cosmetic detection

A `description` field is always cosmetic. A `redeploy_on_update` field set to `"All"` makes every other change non-cosmetic because the profile will push to all computers regardless of what you modified. We started with a list of universally cosmetic fields and ended with conditional logic that examines related attributes.

### Track both volume and substance

The point system catches bulk operations. Cosmetic change detection prevents false positives from organizational work. Five substantive changes deserve review. Thirty cosmetic changes might also deserve review for different reasons.

### Exclude test environments from scoring

Treating dev servers the same as production created friction in the wrong place. Excluding test servers from risk scoring made the experimentation space feel safe to use without reducing production safety.

### Use configuration files for tuning

The Python classifier stabilized months ago and rarely changes. The YAML configuration changes constantly. Keeping it in YAML instead of buried in Python code means anyone on the team can propose adjustments without needing to understand the implementation details.

## Classifier Output

The classifier comments on every merge request with its decision and reasoning. High-risk changes look like this:

```markdown
## ⚠️ Risk Classification: HIGH

**Approval required**

**Risk score:** 16 (threshold: 5)

**Affected servers:** prod_server_1, prod_server_2

**Reasons:**
- Multiple servers affected: prod_server_1, prod_server_2
- High-risk resource: jamfpro_ldap_server
- Destructive action: replace on module.prod_server_1.jamfpro_ldap_server.main
```

For a low-risk change:

```markdown
## ✅ Risk Classification: LOW

**Merge allowed**

**Risk score:** 2 (threshold: 5)

**Affected servers:** prod_server_1
```

You see the score, the threshold, and which factors contributed. If you disagree with a classification, you can discuss whether specific rules should change. This transparency turns potential frustration ("why does this need approval?") into productive conversation ("should LDAP server changes always require approval or only when they affect production?").

## The Core Principle Stayed the Same

Approval requirements should match risk. The three original risk factors are still the foundation: blast radius (multiple Jamf Pro servers), destructiveness (deleting or replacing resources), and sensitivity (authentication, API roles, enrollment settings).

Everything else we added reduces false positives. Cosmetic change detection stops description updates from triggering reviews. Point-based scoring catches bulk operations. Attribute rules detect when scope expansion changes blast radius. Server exclusion keeps dev work out of production risk calculations.

## What Changed After Eight Months

The quantitative changes are clear: merge request volume went from roughly 20 per month in the first quarter to over 65 per month by late summer. But the qualitative impact matters more.

Routine work now flows without friction. Documentation updates, category organization, dev environment experimentation, and cosmetic improvements merge without waiting. When someone gets pulled in for review, it matters: multiple servers affected, destructive actions, or high-risk infrastructure changes. The context is right there in the MR comment showing the risk score, the threshold, and exactly which factors contributed.

When someone disagrees with a classification, the conversation shifts from "why do I need approval?" to "should this rule be different?" That shift is important. The system isn't an arbitrary gatekeeper; it's a set of rules we control and can adjust based on what we learn.

The classifier has processed hundreds of merge requests at this point, and people have stopped second-guessing its decisions. The team trusts it because the classifications make sense and the logic is adjustable. Infrastructure automation succeeds when it works well enough that people stop thinking about it.

## Lessons for Teams Using Terraform with Jamf

Teams considering Terraform for Jamf Pro often worry about approval overhead. "We can't wait for code review every time we need to deploy a policy or update a smart group. That'll kill our velocity." They're right to worry—mandatory approval for every change would make Terraform slower than the GUI.

Terraform trades speed for control: clicking in the Jamf Pro GUI is fast but lacks version control, audit trails, peer review, and automated testing. Terraform provides those benefits but introduces friction if every change requires manual approval.

Risk-based classification finds the middle ground. Policy deployments, smart group updates, and configuration profile changes flow without waiting as long as they're scoped to single customers. LDAP reconfigurations, API role changes, and settings that affect authentication or enrollment get reviewed because the risk warrants scrutiny.

If you're managing multiple Jamf Pro servers through Terraform, you'll likely need something like this. The alternative is either approval friction that makes people resent losing GUI speed, or no approval requirements at all, which defeats the purpose of version control and code review.

Start with the basics: blast radius (multiple servers), destructive actions (delete/replace), and resource sensitivity (authentication, settings, system config). The refinements—cosmetic change detection, point-based scoring, excluded servers, attribute rules, and Self Service handling—all emerged from watching how our team used Terraform for Jamf work. None of these were in the original design. They came from iteration.

Keep the configuration accessible. Put rules in YAML, add comments to every MR showing why it was classified that way, and make the logic something anyone on the team can propose changes to. The classifier should feel like a tool you control, not an obstacle you work around.

The classifier has stabilized at this point. We're not planning major architectural changes, just watching for patterns that suggest new rules. As the Jamf Terraform provider adds resource types, we'll classify them and define cosmetic fields. If we discover new scope patterns that indicate high risk, we'll add attribute rules. If classifications feel wrong, we can adjust point weights without touching code.

Terraform will never be as fast as clicking in the GUI. But with the right classification system, you get version control, audit trails, and code review without making every policy update wait in a review queue. Routine work flows freely and risky changes get the scrutiny they deserve.

---

*The core classifier is under 500 lines of Python with YAML configuration. Any team comfortable with CI pipelines and Python could implement something similar.*

*Original blog post: [When Every Change Needs Approval, No Change Gets Reviewed]({% post_url 2026-02-03-risk-based-approvals %})*
