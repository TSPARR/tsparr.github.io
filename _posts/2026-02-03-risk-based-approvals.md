---
layout: post
title: "When Every Change Needs Approval, No Change Gets Reviewed"
date: 2026-02-03
categories: [devops, automation, terraform, jamf]
---

## How We Built Risk-Based Merge Request Approvals for a Small Infrastructure Team

There's a moment every small team experiences with code review. You're staring at a five-line change (adding a printer policy, updating a group membership, something routine) and you need someone to approve it before you can merge. Your colleagues are busy. The change is low-risk. You know it's fine. They'll know it's fine. But process is process, so you wait.

Then one of two things happens: either you wait too long and start resenting the process, or your teammate approves without really looking because they trust you and they're busy. Neither outcome is what code review is supposed to achieve.

We're a small team managing multiple customer Jamf Pro environments through Terraform and GitLab. After months of friction, rubber-stamp approvals, and the occasional "just this once" bypass from leadership, we decided to fix the problem at its root. The result is a risk-based approval system that automatically determines which changes need review and which ones don't.

## The False Choice Between Speed and Safety

When we adopted Infrastructure as Code, we inherited assumptions from software development that don't quite fit infrastructure work. The conventional wisdom says all changes should be reviewed. It sounds responsible. It's also impractical for a small team doing operational work.

Consider what we actually do day-to-day: creating policies, updating smart groups, deploying scripts, adjusting configuration profiles. This is routine Jamf administration, the same work we used to do by clicking through the web interface. Nobody reviewed those GUI clicks. Nobody needed to. The risk was low and the changes were scoped to individual customers.

Once we moved to Terraform, suddenly every change required approval. The five-line printer policy sat in the same review queue as the LDAP server reconfiguration. We were treating a paper cut with the same urgency as a broken bone.

Removing approvals entirely wasn't acceptable either. Some changes genuinely benefit from a second pair of eyes. When you're modifying authentication settings or making changes that affect multiple customer environments simultaneously, you want someone to sanity-check your work.

We needed a third option: approval requirements that match actual risk.

## Rethinking What "Risky" Means

Before building anything, we had to define what makes a change risky in our context. We looked at our history: what had caused problems, what made us nervous, what we double-checked even when using the GUI. We identified three factors:

### Blast Radius: How Many Environments Are Affected?

Our Terraform repository manages several Jamf Pro servers, one per customer. A change to a single customer's environment is inherently contained. If something goes wrong, one customer is affected and we can fix it quickly.

A change that touches multiple servers simultaneously is different. The blast radius is larger. A mistake multiplies across environments. These changes deserve more scrutiny regardless of what's being changed.

### Action Type: Can We Undo This Easily?

Terraform's `create` and `update` actions are generally reversible. If you create something wrong, you can delete it. If you update something incorrectly, you can update it again.

Terraform's `delete` and `replace` actions are harder to recover from. Deleted resources are gone. Replaced resources lose their previous state. These destructive actions warrant a pause before execution.

### Resource Sensitivity: Is This Day-to-Day or Foundational?

Some Terraform resources represent routine operational items:

- Policies
- Smart groups and static groups
- Scripts and packages
- Configuration profiles
- Extension attributes
- Categories, buildings, departments

These are the building blocks of Jamf administration. They get created, modified, and retired regularly. Mistakes are easily corrected and typically affect only the specific functionality they control.

Other resources represent foundational infrastructure that rarely changes:

- API roles and integrations
- LDAP server configurations
- Enrollment and reenrollment settings
- SMTP server settings
- Client check-in settings
- Self Service settings
- Device communication settings

These resources have broad impact. An incorrect LDAP configuration affects every user's ability to authenticate. A misconfigured API integration can break automation across the environment. Changes to these resources should always get a second look.

## The Classification System

With our risk factors defined, the logic is simple. A change is high-risk if any of the following are true:

1. It affects more than one server
2. It includes a destructive action (delete or replace)
3. It modifies a high-risk resource type

If none of those conditions apply, the change is low-risk.

The approval decision stays simple: needs review or doesn't. No tiers, no "medium" risk that leaves people wondering whether to block the merge.

### Beyond Resource Types: Attribute-Level Rules

Resource type classification works well for most cases, but sometimes the risk depends on specific attribute values within a resource. A policy is generally low-risk, but a policy scoped to all computers is a different story. Pushing a misconfigured policy to every machine in the fleet is exactly the kind of change that benefits from a second look.*

The Terraform plan JSON includes complete before and after states for each resource, not just a diff. This means we can inspect specific attributes and detect when they change to risky values.

We added attribute-level rules to our classification config:

```yaml
attribute_rules:
  jamfpro_policy:
    - path: scope.0.all_computers
      value: true
      reason: Policy scoped to all computers
```

The classifier compares the before and after states for each rule. If a policy already had `all_computers = true` and we're just updating its name, no flag. But if `all_computers` is changing from `false` to `true`, that gets flagged as high-risk.

```python
before_value = get_nested_value(before_state, rule["path"])
after_value = get_nested_value(after_state, rule["path"])

# Only flag if the value is changing TO the risky value
if after_value == rule["value"] and before_value != rule["value"]:
    reasons.append(rule["reason"])
```

This catches the cases that resource type classification misses. A policy update is routine. A policy update that expands scope to all computers deserves review.

## Why We Analyze the Plan, Not the Files

Our first instinct was path-based classification. Changes to certain directories would be high-risk; changes to others would be low-risk. This approach failed for one structural reason: our repository doesn't separate high-risk and low-risk resources by path.

Our Terraform files contain module blocks for each customer server, with shared defaults and per-server overrides in the same files. A change to `main.tf` might affect one server or all of them, depending on which lines changed. The file path tells us nothing about blast radius.

Analyzing Terraform's plan output solves this problem. The plan shows exactly what will change: which resources, on which servers, with what actions. It's ground truth rather than inference.

```python
for resource_change in plan.get("resource_changes", []):
    actions = resource_change.get("change", {}).get("actions", [])
    address = resource_change.get("address", "")

    # Extract server name from module path
    if match := re.match(r"^module\.([^.]+)\.", address):
        affected_servers.add(match.group(1))

    # Check resource type against high-risk list
    if type_match := re.search(r"(jamfpro_[a-z_]+)\.", address):
        if type_match.group(1) in high_risk_types:
            reasons.append(f"High-risk resource: {type_match.group(1)}")

    # Flag destructive actions
    for action in actions:
        if action in ["delete", "replace"]:
            reasons.append(f"Destructive action: {action}")
```

The classification runs as a CI job after `terraform plan` completes. It reads the plan JSON, applies the rules, and outputs a classification with supporting reasons.

## GitLab Integration

The classification integrates with GitLab's merge request approval system. For high-risk changes, we create an approval rule requiring sign-off from one of the assigned reviewers. For low-risk changes, we create no additional rules, so the author can merge immediately.

Every MR receives a comment explaining its classification:

```markdown
## ⚠️ Risk Classification: HIGH

**Approval required**

**Risk score:** 15 (threshold: 5)

**Affected servers:** acme_corp, globex_inc

**Reasons:**
- Multiple servers affected: acme_corp, globex_inc
- Destructive action: delete on module.acme_corp.jamfpro_policy.old_policy
```

Or for routine changes:

```markdown
## ✅ Risk Classification: LOW

**Merge allowed**

**Risk score:** 1 (threshold: 5)

**Affected servers:** acme_corp
```

This transparency serves two purposes. Authors understand immediately why their change does or doesn't require approval. And if someone disagrees with a classification, the reasoning is visible and the configuration is adjustable.

## What Changed

The quantitative impact is hard to measure since we didn't track approval wait times before. But the qualitative changes are clear.

**Routine work flows faster.** Most of our changes are low-risk by design: single-customer, non-destructive, operational resources. These now merge without waiting, which matches how we worked in the GUI era.

**Reviews are meaningful.** When a reviewer gets pulled in, they know it matters. The change either crosses customer boundaries, involves destructive actions, or touches sensitive infrastructure. This context changes how people review. They actually look at the change instead of assuming it's probably fine.

**Everyone follows the same rules.** The classification is automatic and objective. There's no "I'm senior so I don't need review" or "I'm in a hurry so I'll skip it this time." Leadership follows the same process as everyone else, which matters for team trust.

**The rules are adjustable.** Our risk classifications live in a YAML configuration file. When we onboard a new Terraform resource type or decide something should be reclassified, anyone can update the config. The logic isn't buried in code that only one person understands.

## What We Learned

**Match the tool to your actual workflow.** We manage customer environments; most changes are scoped to single customers. Our classification reflects that reality. A team managing a single large environment would have different risk factors, maybe based on which subsystems are affected or whether changes touch production versus staging.

**Transparency builds trust.** People accept automation they can see. The MR comment showing exactly why something was classified creates accountability for the system itself. If the classification seems wrong, we can examine and improve the rules.

**Use scoring, but keep the output simple.** The final output is still just "needs approval" or "doesn't." But using a point-based scoring system under the hood lets you catch edge cases like "25 low-risk changes is still risky" without adding complexity to the approval workflow itself.

**Start simple and iterate.** Our initial classification used only resource types. We added blast radius checking after realizing multi-server changes deserved review regardless of resource type. We added destructive action checking after a close call with an unreviewed delete. We added attribute-level rules when we realized that a policy scoped to all computers carried more risk than a policy scoped to a single group. And we moved from simple flags to point-based scoring when we noticed that many small changes could add up to something risky. The system evolved based on what we actually encountered.

## Making It Work for Your Team

The specific classifications we use (which Jamf Pro resources are high-risk, how many servers constitute "multiple") won't transfer directly to your situation. But the pattern applies broadly:

1. **Define your risk factors.** What has caused problems historically? What makes your team nervous? What would you want someone to double-check?

2. **Analyze actual changes, not proxies.** File paths, commit messages, and PR titles are indirect signals. The deployment plan or migration script shows what will actually happen.

3. **Integrate with existing workflow.** We didn't add new tools or processes. The classification runs in our existing CI pipeline and uses GitLab's native approval features. Adoption friction was near zero.

4. **Make rules visible and adjustable.** Configuration in version control, comments on every MR, clear documentation of the logic. When someone questions a classification, you can point to the rules and discuss whether they should change.

## Point-Based Scoring

Our first version of the classifier was simpler: any high-risk flag meant the MR needed approval, and no flags meant it didn't. This worked well initially, but we noticed a gap. One low-risk change is fine. But what about 25 low-risk changes in a single MR? Volume matters. A single policy update is routine. Updating 30 policies at once, even if each individual change is low-risk, deserves a second look just because of the scope.

We evolved the system to use point-based scoring.* Rather than treating every risk factor as pass/fail, we assign point values:

```yaml
risk_threshold: 5

points:
  per_resource_change: 1
  destructive_action: 3
  high_risk_resource_type: 5
  attribute_rule_match: 5
  multiple_servers: 10
```

The classifier loops through each resource change and adds up the score:

```python
risk_score = 0

for resource_change in plan.get("resource_changes", []):
    # Every change adds base points
    risk_score += points["per_resource_change"]

    # Destructive actions add points
    for action in change_actions:
        if action in high_risk_actions:
            risk_score += points["destructive_action"]

    # High-risk resource types add points
    if resource_type in high_risk_resource_types:
        risk_score += points["high_risk_resource_type"]

    # Attribute rule matches add points
    for reason in check_attribute_rules(resource_change, resource_type, attribute_rules):
        risk_score += points["attribute_rule_match"]

# Multiple servers adds points once at the end
if len(affected_servers) > 1:
    risk_score += points["multiple_servers"]

risk_level = "high" if risk_score >= threshold else "low"
```

Points accumulate across the entire MR. A single low-risk change scores 1 point. Five low-risk changes hit the threshold. A destructive action on a high-risk resource type scores 9 points by itself.

The MR comment shows the score and threshold so authors understand exactly where they stand:

```markdown
## ✅ Risk Classification: LOW

Merge allowed

**Risk score:** 3 (threshold: 5)

**Affected servers:** acme_corp
```

This catches the "lots of small changes" case that a simple flag-based system would miss, while still allowing routine single-resource updates to flow through without review.

## Conclusion

Mandatory approval for every change is a policy that sounds rigorous but often produces the opposite of its intent. Reviewers rubber-stamp because they're overwhelmed. Authors resent the friction. The process becomes theater.

Risk-based approval recognizes that not all changes deserve equal scrutiny. By automatically classifying changes based on what they actually do (blast radius, destructiveness, sensitivity) we get review where it adds value and speed where it's safe.

Our team now moves faster on routine work while maintaining genuine oversight of changes that matter. The rules are the same for everyone. The reasoning is transparent. And we spend our review capacity on changes that actually benefit from a second opinion.

Infrastructure as Code should make infrastructure management better, not just more bureaucratic. With the right guardrails in the right places, it can be both safe and fast.

---

*Thanks to Neil Martin (@neilmartin83) at Jamf for suggesting both attribute-level rules and point-based scoring.*

*Built with Terraform, GitLab CI, and Python. The classification configuration and scripts are simple enough that any team comfortable with CI pipelines could implement something similar quickly.*
