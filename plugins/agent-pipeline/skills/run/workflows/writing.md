---
name: writing
description: Produce a piece of writing — outline → draft → independent edit (articles, scripts, docs). A template for non-code workflows; copy it to build your own.
handoff-dir: .pipeline
preflight: []
max-fix-rounds: 1
---

## Stage outline
- model: opus
- role: |
    You are an outliner. You do NOT write the finished piece.
    Read the project's CLAUDE.md / README (if any) for audience, voice, length and format.
    Write an outline: goal, audience, key points in order, target length per section, sources or
    figures to use. Put anything the user must decide under "## OPEN QUESTIONS" at the top.
- reads: [request.md, CLAUDE.md]
- writes: outline.md
- stop-if: the file has a non-empty "OPEN QUESTIONS" section

## Stage draft
- model: sonnet
- role: |
    You are a writer. Follow the outline exactly, in the agreed voice and length, adding no points
    outside it. On a fix round, change only what the editor's notes ask for.
- reads: [outline.md]
- writes: draft.md

## Stage edit
- model: opus
- role: |
    You are an independent editor. You do NOT rewrite the piece yourself.
    Compare the draft with the outline and the request: is every point covered, is it right for the
    audience, are any claims false or unclear, is the length right? The first line of your file
    must be exactly one of:
    VERDICT: APPROVE / VERDICT: REVISE / VERDICT: REJECT
    For REVISE or REJECT, list precisely which sentences to change and why.
- reads: [request.md, outline.md, draft.md]
- writes: edit-notes.md
- verdict: {APPROVE: continue, REVISE: back-to draft, REJECT: stop}

## Finish
Give the user the path to `draft.md` and a short summary of the editor's notes. Publish nothing.
