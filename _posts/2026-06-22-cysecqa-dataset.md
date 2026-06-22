# CySecQA: Building a 120K-Question Dataset for Security Awareness

Most cybersecurity benchmarks test whether you can dissect a packet capture or explain the difference between AES-128 and AES-256. That's useful if you're hiring a SOC analyst. It's useless if you're trying to figure out whether your marketing team will click on a phishing link disguised as a Google Docs share.

That gap is what led us to build [CySecQA](https://huggingface.co/datasets/maleo-ai/CySecQA) — a dataset of 120,000 multiple-choice questions focused entirely on end-user security awareness. Not pentesting. Not cryptography. The mundane, critical stuff: should you plug in a USB you found in the parking lot? What do you do when someone calls claiming to be IT support? How do you handle a colleague tailgating through a secure door?

The dataset covers 415 topics across 22 security domains, split into 100K training and 20K test questions with zero overlap between them.

---

## The Distractor Problem

Here's something that doesn't get enough attention in MCQ dataset construction: the wrong answers matter as much as the right ones.

When we first assembled the topic bank, we ran a frequency analysis on the distractors (the three incorrect options per question). The results were embarrassing. The phrase "It is required" appeared as a wrong answer in **67 different topics**. "Ignore it" showed up in 18. "It is safe" in 15. Nearly 60% of all topics had at least one boilerplate distractor — a generic wrong answer recycled across unrelated questions.

This is a real problem. If a test-taker notices that "Ignore it" is always wrong, they can eliminate it without thinking about the actual security concept. The question stops testing knowledge and starts testing pattern recognition. At that point, your security awareness assessment is measuring memorization of answer patterns, not comprehension of security principles.

We defined a metric for this: **distractor reuse**, the mean number of topics each unique distractor text appears in. The theoretical floor is 1.0 — every distractor unique to its topic. Our baseline was 1.636.

---

## Surgical Enrichment

Fixing this wasn't a matter of regenerating everything from scratch. Many distractors were already good. The problem was specific: identifiable clusters of boilerplate phrases contaminating otherwise sound questions.

We built two mechanisms:

**Full-Replacement Overlay.** For topics where all three distractors were generic garbage ("It is required" / "It is safe" / "It saves time"), we replaced the entire set with topic-specific alternatives. Applied to 42 topics.

**Partial-Swap Overlay.** For topics with a mix of good and bad distractors, we surgically swapped individual phrases. "Ignore it" in a phishing question becomes "Forward it to verify the sender." The other two distractors stay untouched. Applied to 170 topics.

The key insight was working in **clusters** rather than topic-by-topic. Instead of fixing one occurrence of "Ignore it" at a time, we'd identify all 18 topics using that phrase, author 18 topic-specific replacements in one batch, and eliminate the entire cluster at once. Thirteen batches later, every single boilerplate phrase was gone.

The progression looked like this:

| Batch | Target | Reuse After |
|-------|--------|-------------|
| Baseline | — | 1.636 |
| 1–2 | "It is required/safe" cluster | 1.221 |
| 3–5 | "Ignore it", "It is illegal", "A type of X" | 1.126 |
| 6–9 | Action/credential/sharing clusters | 1.062 |
| 10–13 | Remaining pairs | **1.000** |

That final 1.000 is the theoretical floor. Every distractor in the dataset is now globally unique.

---

## What Makes a Good Distractor

The replacements weren't random. Three principles guided every substitution:

**Plausible wrongness.** A distractor has to be something a person with incomplete knowledge might actually believe. For "what is a firewall," replacing the generic "A type of malware" with "A type of web filter" is far more diagnostic — it tests whether the respondent understands the distinction between network-level filtering and content filtering.

**Topic specificity.** Each replacement must only make sense in the context of its assigned topic. "Let them use it for a quick call only" is a plausible wrong answer for a question about strangers borrowing your phone, but it would be nonsensical in a password management question. This prevents creating new boilerplate.

**Pedagogical value.** The best distractors diagnose specific misconceptions. "Cloud providers auto-expire all shared links" sounds reasonable enough that someone who hasn't thought about shared document persistence might select it. That tells you exactly what knowledge gap to address in training.

---

## Paranoid Validation

After 13 enrichment batches, we ran the dataset through 13 independent quality audits to make sure we'd improved actual quality, not just gamed a metric:

- **Stability**: regenerated from scratch, got the exact same numbers. Deterministic.
- **Semantic correctness**: scanned all 1,245 distractors against every correct answer in the dataset. Zero cases where a distractor was accidentally also a correct answer for another question.
- **Within-topic distinctness**: checked whether any question had two distractors testing the same misconception. Found one mild pre-existing case across the entire dataset.
- **Length cues**: identified and fixed 14 distractors whose word count was so different from the correct answer that "pick the shortest/longest option" would work as a strategy.
- **Negation mirrors**: checked for lazy "Do not [correct answer]" distractors. Found zero.
- **Absolutist language**: 193 distractors use words like "always" or "never" — but so do 17 correct answers ("Never share your MFA code"), which means the "extremist answer is wrong" heuristic fails 9% of the time. Good design, not a flaw.

---

## Where It Sits

For context, here's how CySecQA compares to existing cybersecurity MCQ datasets:

| Dataset | Size | Audience |
|---------|------|----------|
| SecQA | 242 | Technical |
| SecEval | 2,000 | Technical |
| SECURE | 5,000 | Technical |
| CyberMetric | 10,000 | Professional |
| SecBench | 44,823 | Technical |
| **CySecQA** | **120,000** | **End-user awareness** |

Every other dataset in this space targets security professionals or LLM benchmarking. CySecQA is the only one designed for the people who actually cause most breaches: regular employees making everyday decisions.

---

## What's Next

The dataset is [on HuggingFace](https://huggingface.co/datasets/maleo-ai/CySecQA) and the code is [on GitHub](https://github.com/anak10thn/CySecQA). Apache 2.0.

There are a few obvious next steps that we haven't done yet and are honest about:

- **IRT calibration.** We don't have item-level difficulty and discrimination parameters because we haven't administered the questions to a human population yet. That's the most important missing piece.
- **LLM benchmarking.** Running GPT-4, Claude, and open models against the test set to see how well they handle everyday security scenarios (as opposed to the technical benchmarks they're usually tested on).
- **Multilingual extension.** The dataset is English-only. Most of the world's employees are not.

The research paper with full methodology details is in the [GitHub repo](https://github.com/anak10thn/CySecQA/blob/main/paper/main.pdf).

If you work in security awareness training or educational NLP and want to collaborate on any of the above, reach out.
