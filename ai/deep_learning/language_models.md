##### Intro and Word Vectors

*(Your content here.)*

#### Stanford Intro to LLM's

# What is a Language Model?

A language model (LM) is a probability distribution over sequences of tokens.

<details>
  <summary>Explanation</summary>

  Consider a vocabulary `V` of tokens. A language model `p` assigns a probability (between 0 and 1) to each sequence of tokens `x₁, …, x_L ∈ V`, denoted as:

  $$ p(x_1, \dots, x_L) $$

  This probability indicates how "good" or likely a sequence is. For example, if  
  `V = {ate, ball, cheese, mouse, the}`, then the LM might assign:
  
  $$ p(\{the\}, \{mouse\}, \{ate\}, \{the\}, \{cheese\}) = 0.02 $$
  
  and so on.
</details>

Suppose we have a vocabulary `V`. A language model `p` assigns each sequence of tokens a probability:

$$ p(x_1, \dots, x_L) $$

The model uses **syntactic knowledge** and **world knowledge** to assign these probabilities. For example, *"the mouse ate the cheese"* should have a higher probability than *"the cheese ate the mouse"*.

**Generation:**  
We generate sequences by sampling from the language model:

$$ x_{1:L} \sim p $$

More efficient generation often uses techniques that consider not just raw probabilities but also refined methods to select the best sequence.

<details>
  <summary>Explanation</summary>

  In other words, even though every permutation has a probability, we usually want the best sequence—not just any high-probability sequence. This is achieved using advanced sampling techniques.
</details>

**Autoregressive Language Models**  
We can write the joint probability as:

$$
p(x_1, \dots, x_L) = \prod_{i=1}^L p(x_i \mid x_{1:i-1})
$$

Each term `p(x_i ∣ x_{1:i-1})` is the conditional probability of token `x_i` given the previous tokens.

**Generation Process:**  
To generate a sequence from an autoregressive LM, we sample one token at a time:

$$
\text{for } i = 1, \dots, L: \quad x_i \sim p(x_i \mid x_{1:i-1})^{1/T}
$$

where `T` is the **temperature** parameter controlling randomness:
- `T = 0`: Always pick the most probable token.
- `T = 1`: Sample normally.
- `T = ∞`: Choose uniformly from the vocabulary.

Raising the probabilities to the power `1/T` and then renormalizing gives an **annealed** distribution.

<details>
  <summary>Explanation</summary>

  Adjusting `T` softens or sharpens the peaks in the probability distribution, influencing the diversity of the generated text.
</details>

**Conditional Generation:**  
By providing a prefix (prompt) `x₁:i`, we can generate a completion `x_{i+1:L}`:

$$
\underbrace{\text{the, mouse, ate}}_{\text{prompt}} \quad \rightsquigarrow_{T=0} \quad \underbrace{\text{the, cheese}}_{\text{completion}}
$$

Changing `T` can produce more varied results.

<details>
  <summary>Explanation</summary>

  In summary, using a temperature `T` lets you control how diverse or predictable the output of your language model is.
</details>

### Summary

- A language model assigns a probability `p` over sequences.
- It leverages both syntactic rules and world knowledge.
- Autoregressive models generate text one token at a time.
- The temperature parameter controls output variability.
