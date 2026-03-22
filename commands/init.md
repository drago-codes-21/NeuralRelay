---
name: neuralrelay init
description: Initialize NeuralRelay in the current project
tools:
  - Read
  - Write
  - Bash
  - Glob
---

Initialize NeuralRelay in the current project by creating the required directory structure and default pipeline configuration.

## Behavior

1. Check if `.neuralrelay/` already exists:
   - If yes: print "NeuralRelay is already initialized in this project. Run /neuralrelay start to begin a pipeline." and stop.
   - If no: continue with initialization.

2. Create the directory structure:
   ```
   .neuralrelay/
   └── handoffs/
   ```

3. Create the default `neuralrelay.yaml` in the project root with this content:
   ```yaml
   # NeuralRelay Pipeline Configuration
   # Customize stages, models, and settings for your workflow.

   pipeline:
     - name: requirements
       agent: ba-agent
       model_preference: sonnet
     - name: implementation
       agent: dev-agent
       model_preference: opus
     - name: testing
       agent: test-agent
       model_preference: sonnet
     - name: review
       agent: review-agent
       model_preference: opus

   settings:
     max_rework_cycles: 2
     handoff_validation: strict
     handoff_dir: .neuralrelay/handoffs
     token_tracking: true
   ```

4. Check if `.gitignore` exists. If yes, append `.neuralrelay/handoffs/` to it (handoffs are ephemeral, config is committed). If no `.gitignore`, create one with that entry.

5. Print the welcome message:
   ```
   NeuralRelay initialized successfully.

   Created:
     .neuralrelay/handoffs/   — where handoff chains are stored
     neuralrelay.yaml         — pipeline configuration (edit to customize)

   Quick start:
     /neuralrelay start "add OAuth2 login"   — run a full pipeline
     /neuralrelay status                      — check pipeline progress
     /neuralrelay report                      — view cost & quality analytics
   ```
