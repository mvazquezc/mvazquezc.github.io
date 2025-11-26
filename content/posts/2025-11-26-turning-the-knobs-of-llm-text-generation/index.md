---
title:  "Turning the Knobs of LLM Text Generation"
author: "Mario"
tags: [ "context engineering", "llm", "AI", "artificial intelligence", "top-k", "top-p", "temperature" ]
url: "/turning-the-knobs-of-llm-text-generation"
draft: false
date: 2025-11-26
lastmod: 2025-11-26
ShowToc: true
ShowBreadCrumbs: true
ShowReadingTime: true
ShowCodeCopyButtons: true
robotsNoIndex: false
searchHidden: false
ShowWordCount: false
---

# Turning the Knobs of LLM Text Generation

Ever wonder how much control you actually have over the text an LLM produces? In this post, we will look at three simple but powerful knobs you can tweak to push a model toward more deterministic output or toward something more creative.

We are talking about `top_k`, `top_p` and `temperature`. But before describing them, we need to understand the two main behaviors we can get from an LLM when it is sampling tokens:

- **Greedy Sampling**: At each step, the model picks the token with the highest probability given the preceding context.

  - Pros:
    - Produces output that is typically coherent and aligned with the most common patterns in the training data.
    - Is deterministic. Same prompt, same output every time.
  - Cons:
    - Often dull or repetitive because it never takes alternative paths.
    - Short-sighted: the "best next token" doesn’t always lead to the best overall sequence.

- **Random (Stochastic) Sampling**: The model draws the next token from the entire probability distribution. Higher probability tokens are more likely, but even low probability tokens can be sampled.

  - Pros:
    - Output is more diverse and creative.
    - Helps avoid repetitive loops that greedy sampling can fall into.
  - Cons:
    - Can reduce coherence or quality.
    - Non-deterministic. Running the same prompt multiple times produces different results.

The influence of `top_k`, `top_p`, and `temperature` lies in how they control the amount of randomness the model is _allowed_ to use during sampling. By adjusting them, you can push the model closer to greedy, deterministic behavior, make it sample from a broader range of possibilities, or land anywhere in between.

## Top-K

This parameter limits the model to the top K most probable tokens at each generation step. Everything outside that shortlist is ignored.

### Example

Let's say we have the prompt `The future of AI is`, and we configure `top_k = 5`. This means the model will pick only tokens among the five tokens with the highest probability.

That list might look like this:

```text
Token: ' in' (ID: 11) | Logprob: -2.1674 | %: 11.45%
Token: ' not' (ID: 45) | Logprob: -3.1830 | %: 4.15%
Token: ' now' (ID: 122) | Logprob: -3.4174 | %: 3.28%
Token: ' here' (ID: 259) | Logprob: -3.4330 | %: 3.23%
Token: ' a' (ID: 10) | Logprob: -3.4955 | %: 3.03%
```

With `top_k = 5`, the model must pick from these five tokens and ignore everything else. How it chooses among them depends on your `temperature` setting.

If `temperature = 0`, the sampling becomes greedy and the model will always select the token with the highest probability,&nbsp;` in` in the example above.

## Top-P (Nucleus Sampling)

This parameter filters out tokens once the cumulative probability threshold is reached. Unlike `top_k`, which always keeps a fixed number of tokens, `top_p` includes as many tokens as necessary until their combined probability reaches the chosen value.

### Example

- `top_k = 5` -> Always selects the 5 highest-probability tokens.
- `top_p = 0.3` -> Collects tokens (from highest to lowest probability) until their cumulative probability reaches 30%.

Given a list of token probabilities, we accumulate them in descending order:

```text
Token: ' the'       | Prob:  8.86% | Cumul:  8.86%
Token: ' a'         | Prob:  7.61% | Cumul: 16.48%
Token: ' not'       | Prob:  3.81% | Cumul: 20.29%
Token: ' in'        | Prob:  3.11% | Cumul: 23.40%
Token: ' being'     | Prob:  2.16% | Cumul: 25.56%
Token: ' one'       | Prob:  1.36% | Cumul: 26.93%
Token: ' home'      | Prob:  1.36% | Cumul: 28.29%
Token: ' now'       | Prob:  1.29% | Cumul: 29.58%
Token: ' getting'   | Prob:  1.24% | Cumul: 30.82%

Top-p threshold reached (30.82% / 30.0%). Remaining tokens are ignored.
```

With `top_p = 0.3`, the model would sample from this set of tokens, regardless of how many there are. In this example, it happened to include nine tokens, 4 more tokens than the fixed limit you’d get from `top_k = 5`. These could have been different if tokens had a high probability, in that case we would have gotten fewer tokens than with `top_k = 5`.

Again, if `temperature = 0`, the sampling becomes greedy and the model will always select the token with the highest probability,&nbsp;` the` in the example above.

## Temperature

This parameter adjusts how "spread out" the probability distribution is during sampling by scaling the model’s logits before selecting the next token.

- Low temperature (closer to 0): Sharpens the distribution, making high-probability tokens even more likely. Output becomes predictable and stable.
- High temperature (closer to 1): Flattens the distribution, increasing the chances of lower-probability tokens. Output becomes more varied and creative, but also more chaotic if pushed too far.

`temperature = 0` removes randomness entirely and forces greedy behavior (always picking the token with the highest probability).

## How can I adjust those parameters?

Good question. How you can tweak `top_k`, `top_p`, and `temperature` depends on how you’re using LLMs.

- If you’re interacting through managed web interfaces (like ChatGPT’s website), you typically cannot adjust these parameters.
- If you’re using API access provided by the service, you can usually control them when sending requests.

Always check the official documentation of your provider for the exact way to set these parameters in API calls.

## What's next?

If you want to experiment with tuning these parameters on open models, check out this Jupyter Notebook, which uses [vLLM](https://docs.vllm.ai/) python bindings to tune LLM text generation.

- [Jupyter Notebook](https://github.com/mvazquezc/ai-helpers/blob/main/notebooks/TopK,TopP,Temperature.ipynb)