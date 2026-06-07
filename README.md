# Open Source Contribution Log — AI301

## Status: Phase I Complete

---

## Phase I — Issue Selection

### Issue

**[py-vetlog-calendar #110 — List all surgeries from the past 7 days](https://github.com/josdem/py-vetlog-calendar/issues/110)**

Project: [josdem/py-vetlog-calendar](https://github.com/josdem/py-vetlog-calendar)
Language: Python
Label: `good first issue`, `help wanted`, `enhancement`

---

### Problem Summary

`py-vetlog-calendar` is a Python tool that reads a veterinary clinic's Google Calendar and performs tasks like listing pets with pending vaccinations. Currently, there is no way to see which surgeries occurred in the last 7 days. The issue asks for a `list_surgeries` method to be added to `calendar.py` that filters calendar events from the past week where the event title contains "Surgery" (English) or "Cirugia" (Spanish), and prints them. Without this feature, clinic staff have no automated way to pull a weekly surgery report from the calendar. "Fixed" means: a working method exists, it correctly filters by title keyword and date range, and tests verify both behaviors.

---

### Why I Chose This Issue

I chose issue #110 in `py-vetlog-calendar` because it is the right size and shape for a first open source contribution as someone still building my Python skills.

The task is one clearly-bounded function: read events from a Google Calendar API response, filter by date (past 7 days) and by title keyword ("Surgery" / "Cirugia"), and return or print the results. I can explain the problem in one sentence and I can picture exactly what done looks like. The issue's acceptance criteria name the specific file (`calendar.py`), the specific method (`list_surgeries`), and even which keywords to check — that kind of specificity is rare and valuable for a first contributor.

The project's README shows a clean setup with a single `uv sync` command and a test suite I can run with `uv run pytest tests/unit -v`. There are 9 prior contributors listed in the README, which tells me the maintainer has a track record of accepting and crediting outside PRs. The issue was opened today with no assignee and no competing pull requests, so the path is clear.

I am specifically interested in learning how real-world Python projects handle date arithmetic and filtering over external API data (Google Calendar), which are patterns I will use repeatedly if I go into backend or data engineering work.

---

## Phase II — Reproduce & Plan

*(To be filled in during Week 2)*

### Local Environment Setup

### Bug Reproduction

### Solution Approach

---

## Phase III — Build

*(To be filled in during Weeks 3+)*

### Testing Strategy

### Implementation Notes

---

## Phase IV — Submit & Iterate

*(To be filled in during Weeks 4+)*

### Pull Request

### PR Summary

### Maintainer Feedback Log

| Date | Feedback | Response | Status |
|------|----------|----------|--------|
|      |          |          |        |
