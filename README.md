# AntiSlop Sampler

## Overview

Try the sampler here: [https://colab.research.google.com/drive/11TjqWQCZ8OJBV6Yi2XI1CsOLx0OTtu0t?usp=sharing](https://colab.research.google.com/drive/1Rd3V4AN31cDytfmY9u80rzHXPD_dS6x9?usp=sharing)


Here it is in action (in slow mode so you can see its backtracking & revisions):

https://github.com/user-attachments/assets/aafe267e-adf1-43e6-9622-5e68b08f7fb3

## Update

- Squashed vram leaks, fixed bugs. It should work with any transformers model now.
- Support min_p
- Now using slop_phrase_prob_adjustments.json by default, which has a more intuitive probability adjustment per slop phrase (1 == no change; < 1 means probability is reduced by that factor). It looks like this:
```
[
    ["kaleidoscope", 0.5],
    ["symphony", 0.5],
    ["testament to", 0.5],
    ["elara", 0.5],
    ...
]
```
- I discovered the sampler can squash an annoying habit of LLM writing: overuse of antitheses, e.g. `...not x, but y`, simply by downregulating the string `", not"`. Yay! I think there will be a lot of interesting life hacks to be found like this.
- I've made some generate functions that you can import to deploy the sampler in your code:

### chat_antislop
```python
# Chat generation with streaming
messages = [
    {"role": "user", "content": prompt}
]
for token in chat_antislop(
    model=model,
    tokenizer=tokenizer,
    messages=messages,
    max_new_tokens=400,
    # Antislop sampling may be less reliable at low temperatures.
    temperature=1,    
    min_p=0.1,
    # The adjustment_strength param scales how strongly the probability adjustments are applied.
    # A value of 1 means the values in slop_phrase_prob_adjustments (or the defaults) are used unmodified.
    # Reasonable values are 0 (disabled) thru 10+ (effectively banning the list).
    adjustment_strength=10.0,
    # Optional: Provide a list of slop phrases and probability adjustments
    slop_phrase_prob_adjustments=slop_phrase_prob_adjustments,
    streaming=True
):
    print(tokenizer.decode(token), end='', flush=True)
```

### generate_antislop
```python
# generate without streaming
generated_text = generate_antislop(
    model=model,
    tokenizer=tokenizer,
    prompt=prompt,
    max_new_tokens=300,
    temperature=1,
    min_p=0.1,
    adjustment_strength=2.0,
    slop_phrase_prob_adjustments=slop_phrase_prob_adjustments,
    streaming=False
)        
print(tokenizer.decode(generated_text))
```

## What this does:

You can give it a list of words & phrases to avoid like "a tapestry of", "a testament to", etc., and it will backtrack and try something else if it hits that phrase. It can handle 1000s of slop phrases since the lookups are fast. The phrases and downregulation amounts are user configurable. Previous approaches have done this with per-token logit biasing; but that's quite ineffective since most slop words & phrases are more than one token, and it impairs output quality if we downregulate all those partial-word tokens. So instead, we wait for the whole phrase to appear in the output, then backtrack and downregulate all the tokens that could have produced the slop phrase, and continue from there.

## Why it's interesting:

Samplers typically work at the token level -- but that doesn't work if want to avoid words/phrases that tokenise to >1 tokens. Elara might tokenise to ["El", "ara"], and we don't want to reduce the probs of everything beginning with "El". So, this approach waits for the whole phrase to appear, then backtracks and reduces the probabilities of all the likely tokens that will lead to that phrase being output. Nobody afaik has tried this before. It should produce better results than instructing the model to avoid words & phrases in the prompt.

* Disclaimer: This code has come together over a few days so expect research grade code & possibly bugs.

## GPT-generated details follow:

The **AntiSlop Language Model Sampler** is an advanced text generation tool designed to enhance the quality and diversity of outputs from language models. It addresses the issue of overused words and phrases (referred to as "GPT slop") that are commonly generated by language models due to their training data biases.

By integrating a custom sampling strategy with dynamic adjustments and backtracking mechanisms, the sampler actively downregulates the probability of generating specified overrepresented words or phrases. This results in more varied and engaging text generation that avoids common clichés and repetitive language patterns.

## Functional Explanation

### Motivation

Language models are trained on vast corpora of text, which often leads them to overproduce certain words or phrases that are statistically more frequent in the training data. This can result in outputs that are:

- **Repetitive**: Frequently using the same expressions.
- **Predictable**: Lacking originality due to overreliance on common phrases.
- **Less Engaging**: Failing to capture the reader's interest with fresh language.

The AntiSlop sampler tackles this problem by implementing a dynamic token adjustment system that:

- Monitors the generated tokens in real-time.
- Detects when an overrepresented word or phrase is about to be generated.
- Adjusts the model's output probabilities to discourage the generation of these overused expressions.
- Allows for controlled backtracking to revise the output when necessary.

### Core Components

#### 1. Overrepresented Words List

A JSON file (`over_represented_words.json`) contains a list of words and phrases identified as overrepresented, along with their respective ratios indicating the degree of overrepresentation.

Example format:

```
[
  ["word1", penalty],
  ["phrase two", penalty],
  ...
]
```

#### 2. Token Sequence Preparation

The sampler preprocesses the overrepresented words to:

- Generate multiple variants (e.g., lowercase, uppercase, capitalized, with leading spaces).
- Tokenize each variant using the model's tokenizer.
- Map the token sequences to adjustment factors (inverse of the overrepresentation ratio).

#### 3. Starting Tokens Lookup

To efficiently detect when an overrepresented word is being generated, the sampler:

- Precomputes a lookup table of starting token IDs for each token sequence.
- Includes all possible prefixes of the first token to account for subword tokenizations.

#### 4. Dynamic Logit Adjustment

During generation:

- The sampler monitors the tokens being generated.
- If a sequence matching an overrepresented word is detected, it:

  - **Backtracks**: Removes the tokens associated with the overrepresented word from the generated sequence.
  - **Adjusts Logits**: Modifies the model's logits (pre-softmax output probabilities) to downregulate the probability of generating the overrepresented word in subsequent attempts.
  - **Resamples**: Continues generation from the backtracked position, encouraging the model to choose alternative words.

- This process can repeat if the model continues to attempt generating the overrepresented word, with adjustments becoming progressively stronger due to cumulative effects.

#### 5. Backtracking Mechanism

- The sampler maintains a maximum backtracking limit (`max_backtrack`) to prevent infinite loops.
- When backtracking occurs, the model's cached states (`past_key_values`) are updated to reflect the revised sequence.
- The logit cache is also updated to ensure consistency in subsequent token predictions.

### Technical Workflow

1. **Initialization**:

   - Load the language model and tokenizer.
   - Read the overrepresented words and their penalties.
   - Prepare token sequences and starting tokens lookup.

2. **Generation Loop**:

   - **Token Prediction**:

     - Use the model to predict the next token.
     - Apply temperature scaling and filtering strategies (e.g., top-k, top-p).

   - **Token Sampling**:

     - Sample the next token based on the adjusted probabilities.

   - **Sequence Update**:

     - Append the sampled token to the generated sequence.
     - Update the current position and token counters.

   - **Overrepresented Word Detection**:

     - Check if the recent tokens form a sequence matching any overrepresented word.
     - Utilize the precomputed maximum sequence length for efficient detection.

   - **Adjustment and Backtracking** (if overrepresented word detected):

     - Retrieve the adjustment factor for the detected sequence.
     - Adjust the logits at the position where the sequence started.
     - Backtrack by removing the overrepresented tokens from the sequence.
     - Update the model's cached states and logit cache.
     - Record the downregulation to avoid redundant adjustments.

   - **Termination Conditions**:

     - Check for end-of-sequence tokens.
     - Continue until the maximum length is reached or other stopping criteria are met.

3. **Output**:

   - The final generated text is returned after applying all adjustments.
   - Optional streaming output can display intermediate results during generation.

### Important Notes

- **Overrepresented Words List**: Customize the `over_represented_words.json` file to target specific words or phrases relevant to your use case.
- **Adjusting Parameters**:

  - Increasing `adjustment_strength` will more aggressively downregulate overrepresented words but may affect generation fluency.
  - Adjust `max_backtrack` based on the typical length of phrases you aim to avoid.

- **Performance Considerations**:

  - The sampler introduces additional computational overhead due to dynamic adjustments and backtracking.
  - Ensure that your environment has sufficient resources, especially when using large models.

## Technical Details

- **Logit Adjustment**:

  - The adjustment is applied in the logit space before the softmax function.
  - Adjusted logits are calculated as:

    adjusted_logits = logits + log(adjustment_factor ** adjustment_strength)

  - This method allows for fine-grained control over the probability distribution without outright masking tokens.

- **Caching Mechanism**:

  - The sampler caches the model's outputs (`logits` and `past_key_values`) to avoid redundant computations during backtracking.
  - Efficient cache management ensures that only necessary recalculations are performed.

- **Tokenization Considerations**:

  - The tokenizer's behavior (e.g., subword tokenization) is accounted for by precomputing all possible prefixes.
  - This ensures that partial matches of overrepresented words are detected early in the generation process.

- **Backtracking Limitations**:

  - The backtracking mechanism is designed to prevent infinite loops where the model repeatedly tries to generate an overrepresented word.
  - By limiting the number of backtracking attempts and recording adjustments, the sampler balances between avoiding overused phrases and maintaining generation flow.

---
