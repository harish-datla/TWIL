## Math Foundations

It took very long for humans to invent the concept of counting. It was just 42k years earlier that the [earliest evidence](https://en.wikipedia.org/wiki/History_of_ancient_numeral_systems) for counting was found. Ancient humans used to incise parallel marks on a baboon's bone for each numerical value (e.g., 1 is I, 2 is II, 3 is III, 4 is II|I—you get it).

It took approximately 40k years more (or perhaps even longer, as we can’t pinpoint exactly when counting was invented) to reach the famous Roman numeral system, and a thousand more years to develop the present Arabic numerals (originating in India and later adopted and spread by Arabic mathematicians). It was only about 500 years ago that the world fully adopted the current Arabic numeral system.

**What is addition?**

<details>
  <summary>Show Answer</summary>

  **For Discrete Values:**  
  There isn’t an official definition per se, but addition is just counting the total of discrete quantities from individual collections (like single units, people, books, papers, letters, etc.).

  **For Non-Integer Real Numbers:**  
  It’s about finding the combined scale of the individual items—essentially, placing the collections side by side.

  **For Continuous (Non-Discrete) Values:**  
  Think of it as adding many quantifiably infinitesimal values from each continuous quantity and then scaling the result back to an integer level. For example, to add 1.43 and 2.21, you could split them into units of 0.01 (so 1.43 becomes 143 and 2.21 becomes 221), add to get 364, and then scale back to 3.64.
</details>

**What is subtraction?**

<details>
  <summary>Show Answer</summary>

  **For Discrete Values:**  
  Subtraction can be seen as counting the absence of something. It can also be understood as the opposite of addition; for instance, in `a - b`, you’re essentially asking: “How many units do I need to add to `b` to reach `a`?”

  **For Continuous (Non-Discrete) Values:**  
  The same ideas apply as with addition, but with a focus on the difference between the scaled values.
</details>

**What is multiplication?**

<details>
  <summary>Show Answer</summary>

  **For Discrete Values:**  
  Multiplication is just repeated addition. If you have `x` groups each containing `y` discrete items, multiplying them (`x * y`) gives you the total number of items. It’s a form of scaling.

  **For Continuous (Non-Discrete) Values:**  
  The concept is analogous to the discrete case, with the appropriate adjustments for continuous measures.
</details>

**What is division?**

<details>
  <summary>Show Answer</summary>

  **For Discrete Values:**  
  Division is a bit trickier. You can think of it in several ways:
  1. **Inverse of Multiplication:** How many times does `x` fit into `y` (i.e. `z = y / x`)?
  2. **Splitting Apart**
  3. **Scaling Down**
  4. **Consolidating**
  5. **Repeated Subtraction**

  **For Continuous (Non-Discrete) Values:**  
  The ideas are similar to multiplication, adjusted for continuous measurements.
</details>

*Bonus Question*  
**What is the area of a rectangle and why is it formulated the way it is?**

<details>
  <summary>Show Answer</summary>

  The area of a rectangle is defined as length times breadth (`l * b`) because it represents the number of 1×1 squares that fit into the rectangle. There are `b` squares across the breadth and `l` rows of these squares along the length, so the total number of squares (and thus the area) is `l * b`.

  **Personal Trivia:**  
  I didn’t think about this until I was 18—long after high school—when I couldn’t sleep and began pondering why the area of a rectangle is `l * b`, leading to a 5-day existential crisis.
</details>

#### A Very Trivial Introduction to Algebra

All you need to know about basic algebra is covered by addition, subtraction, multiplication, and division—with the addition of understanding the equality operator and variable naming.

**The Equality Operator:**  
This operator means that everything on the left-hand side is equal to everything on the right-hand side.

**Variable Naming:**  
*... (continue your explanation here)...*

#### NLP with Deep Learning

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
