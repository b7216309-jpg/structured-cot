# Unicode Answer Grammar Notes

This branch adds `grammars/fsm_grammar_unicode_answer.gbnf`, a small variant
of the original Structured-CoT grammar.

The original grammar keeps the final answer in printable ASCII:

```gbnf
code ::= [\x09\x0A\x0D\x20-\x7E]+
```

That is a good fit for Python/code benchmark outputs, but it is awkward for
normal chat because it cannot emit accented characters, emoji, or non-ASCII math
symbols in the final answer channel. The Unicode variant keeps the compact
thinking format while broadening only the final answer region:

```gbnf
root ::= think answer
think ::= "<think>\n" "GOAL: " line "APPROACH: " line "EDGE: " line "</think>\n\n"
line ::= [^\n]+ "\n"
answer ::= [^\x00]+
```

## Local Test Setup

Tests were run against a local `llama.cpp` server using:

- model: `Qwen3.6-35B-A3B-Uncensored-HauhauCS-Aggressive-Q4_K_P.gguf`
- server: `llama-server`, OpenAI-compatible `/v1/chat/completions`
- reasoning mode: `--reasoning on --reasoning-format deepseek`
- temperature: `0`
- max tokens: `2048` for code tests, `10000` for harder logic/philosophy tests

The goal was not a publishable benchmark. It was a practical local check for
whether the grammar made the model noticeably worse, and whether a Unicode
answer region reduced normal-chat awkwardness.

## Code Reliability Check

Clean A/B, with the server temporarily booted without a default grammar for the
FREE baseline and the FSM grammar applied per request:

| Test | FREE | FSM |
| --- | ---: | ---: |
| HumanEval+, first 20 tasks | 18/20 | 20/20 |
| Mean total completion tokens | 1779 | 268 |
| Mean think tokens | 1720 | 102 |

The FSM-only wins were `HumanEval/1` and `HumanEval/9`. There were no FREE-only
wins in this slice.

A second run with the grammar as the server-wide default passed the same 20/20
tasks with the same short token profile.

## Logic And Philosophy Check

Five non-code prompts were tested with a 10k token cap:

- Cheryl's birthday
- blue-eyed islanders
- Wason selection task
- Fitch's paradox
- Gettier case

Manual assessment: both FREE and Structured-CoT solved the set. The structured
answer was clearer on the Gettier case; the FREE proof sketch was a little
cleaner on Fitch's paradox. No clear reasoning loss was observed.

## Awkwardness Check

The original ASCII-answer grammar was intentionally tested on casual/exact
output prompts:

| Prompt type | Original ASCII grammar | Unicode answer grammar |
| --- | --- | --- |
| Exact French with accents | accents were stripped | accents preserved |
| Emoji-only reply | emitted text like `Smile` | emitted the emoji |
| Math symbols | degraded symbols such as <=/>= | preserved Unicode symbols |
| Casual supportive text | used ASCII substitutes like `<3` | used real emoji |
| Code prompt | worked | worked |

The Unicode grammar still inherits one known quirk from the broad answer region:
the model can occasionally emit extra tags such as `</think>` in the final
answer. Attempts to filter closing tags inside the grammar caused worse
behavior, including loops on short answers, so the safer middle ground was to
keep the original stopping behavior and only widen the answer character class.

## Practical Takeaway

For code-only evaluation, the original ASCII grammar is fine. For a local
llama.cpp endpoint used by general chat clients, the Unicode answer variant is a
better default because it keeps the compact Structured-CoT behavior while
allowing normal human text.
