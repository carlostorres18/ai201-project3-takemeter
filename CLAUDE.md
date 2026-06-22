# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**TakeMeter** is an AI text classification project (CodePath AI201 Project 3) that classifies Reddit posts from r/soccer into four categories. The classifier is intended for use as a community tool to label post intent/type.

## Classification Labels

| Label | Definition |
|-------|-----------|
| `analysis` | Posts backed by statistics, history, or reasoned argument |
| `hot_take` | Opinions that are unusual or contrarian within the community |
| `reaction` | Posts expressing an in-the-moment emotional response |
| `news` | Transfer news or other factual community-relevant news |

**Edge case handling:** Posts can blend categories (e.g., a reaction to news, or an analysis framed emotionally). When ambiguous between two labels, the primary intent of the post determines the label.

**Data target:** ~200 labeled examples with balanced representation across all four labels. If a label is underrepresented, additional targeted collection is needed before training.

## Evaluation

Accuracy alone is insufficient — also track per-class precision/recall/F1 (especially for the minority class) to catch models that succeed by over-predicting the majority label.

## Status

Project is currently in planning phase. Code structure, tech stack, and commands will be documented here once implementation begins.
