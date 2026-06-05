# Contribution 1: Lint unused funcs

**Contribution Number:** 1  
**Student:** Shamshad Ansari 
**Issue:** https://github.com/kubernetes-sigs/cluster-api/issues/7599  
**Status:** Phase I [Complete]

---

## Why I Chose This Issue

I chose this issue because I want to start learning Kubernetes development through a contribution that is still connected to skills I already have. Cluster API is a large Kubernetes-related project, so I wanted an issue that would help me get familiar with the codebase without needing very deep Kubernetes knowledge right away.

This issue stood out to me because it is mainly about Go linting and code quality. I have worked on a project in Go before, so I feel comfortable reading Go code and learning more about Go tooling. I also think this is a useful issue because unused functions can make a codebase harder to maintain. Through this contribution, I hope to learn more about how large open-source Go projects use linters, static analysis, and CI checks to keep their code clean.

---

## Understanding the Issue

### Problem Description

The issue is asking for a linter or linting setup that can detect unused functions in the Cluster API codebase. Some unused code was previously found manually, and the maintainers want a better way to catch this automatically in the future. The issue mentions two cases: functions that are not used anywhere, and functions that are only used by test code. The first case seems easier to detect, while the second may be harder because it could create false positives.

### Expected Behavior

The project should have a linting check that can detect functions that are not used anywhere in the codebase. This would help contributors catch unused code earlier before it stays in the project or reaches a pull request.

### Current Behavior

Right now, unused functions may not always be caught automatically. In the example from the issue, unused code was found and removed manually. This means the project may still need better tooling to detect this kind of problem consistently.

### Affected Components

The main affected components are the Go source code and the project’s linting/static analysis setup. This likely involves the repository’s lint configuration and any CI or developer workflow that runs lint checks. The issue also specifically mentions unused code found in the KCP, or Kubeadm Control Plane, area, so that part of the codebase is relevant too.

---

## Reproduction Process

### Environment Setup

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. [Step 1]
2. [Step 2]
3. [Observed result]

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
