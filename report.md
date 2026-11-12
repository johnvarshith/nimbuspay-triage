# NimbusPay Triage Model - Technical Report

## 1. Data Audit & Cleaning (Part A)
The raw `train.jsonl` (1,445 rows) contained significant noise, leakage, and business-logic violations. I built a deterministic Python auditor to clean the data, resulting in **1,265 high-quality examples**.
- **Dropped (180 rows):** 
  - 45 rows were exact duplicates of `dev.jsonl` or `test_inputs.jsonl` (preventing data leakage).
  - 25 rows had truncated/invalid JSON labels.
  - 4 rows were junk placeholders (e.g., "test", "asdf").
  - 106 rows were near-duplicates of other training tickets.
- **Fixed (166 rows):**
  - **Priority Rules (132 fixes):** Recomputed priority based on `SCHEMA.md` (e.g., Fraud = P1, Amount > 50k = P1, Repeat contacts = `needs_human: true`).
  - **Date Normalization (30 fixes):** Converted `DD/MM/YYYY` to `YYYY-MM-DD` ISO format.
  - **Txn ID Normalization (20 fixes):** Standardized spacing and dashes to `NPxxxxxxxxxx`.
  - **Amount Math (14 fixes):** Corrected amounts stored as rupees instead of paise (multiplied by 100).

## 2. Baselines (Part B)
- **Zero-shot Base Model:** Qwen2.5-3B without fine-tuning fails to output strict JSON, hallucinates fields, and completely misses the hidden business rules (e.g., paise conversion, priority thresholds).
- **Prompting with Rules:** Injecting `SCHEMA.md` rules into the system prompt pushes the context length too high, causes attention degradation, and still results in math errors (e.g., failing to calculate "day before yesterday"). Fine-tuning was strictly necessary to internalize the logic.

## 3. Fine-Tuning Pipeline (Part C)
- **Chat Template:** Used the native Qwen2.5-Instruct chat template. Loss was masked on the system and user prompts, and calculated **only** on the assistant's JSON response tokens.
- **Sequence Length:** Set to **4096**. Evidence: Token length analysis of the cleaned training data showed a max length of 2,215 tokens (due to massive quoted email chains in some tickets). 4096 safely accommodates the prompt + 200 generation tokens without truncation.
- **Hyperparameters:** LoRA rank=16, alpha=32, dropout=0.05. Learning rate=2e-4 with cosine scheduler. Trained for 2 epochs with an effective batch size of 8 (batch 2 × gradient accumulation 4).

## 4. Controlled Experiments (Part D)
All models were evaluated on the 200-row Dev Set using greedy decoding.

| Run | Model | Data | Epochs | LoRA Rank | Dev Exact Match | Dev Mean Field Acc | Peak VRAM | Time |
|---|---|---|---|---|---|---|---|---|
| 1 | Qwen2.5-1.5B | Raw | 2 | 16 | 72.5% | 89.2% | 6.5 GB | 15m |
| 2 | Qwen2.5-3B | Raw | 2 | 16 | 81.0% | 93.4% | 10.2 GB | 22m |
| 3 | Qwen2.5-3B | Clean | 1 | 16 | 88.5% | 96.1% | 10.2 GB | 12m |
| **4** | **Qwen2.5-3B** | **Clean** | **2** | **16** | **94.0%** | **98.9%** | **10.2 GB** | **22m** |

**Key Finding:** Data cleaning (Run 3 vs 2) provided a massive +7.5% Exact Match boost by removing contradictory labels and leakage. Scaling to 3B parameters allowed the model to successfully parse complex Hinglish syntax and subtle date logic.

## 5. Error Analysis (Part E)
My best model (Run 4) failed on exactly 12/200 dev rows. The weakest fields were `txn_date` (relative dates), `priority` (threshold edge-cases), and `needs_human`. 

**Specific Failure Modes:**
1. **dv-00006 (Relative Date Math):** Ticket received July 2, stating "day before yesterday". Model predicted `2026-07-01` (yesterday) instead of the gold `2026-06-30`.
2. **dv-00090 (Hinglish Date Ambiguity):** Ticket used the word "parso" (which can mean day before yesterday OR day after tomorrow). Model got confused and output the received date instead of subtracting 2 days.
3. **dv-00040 (Keyword Hallucination):** User started with "URGENT!!". Model associated this keyword with P1, ignoring the strict rule that refunds under ₹50k are P2.
4. **dv-00141 (Missed High-Value Rule):** Amount was ₹61,597. Model predicted P2, completely missing the business rule that `payment_failed` > ₹50k requires P1 and human intervention.
5. **dv-00055 & dv-00056 (Duplicate Charge Inconsistency):** Model struggled with the hidden edge-case that "charged twice" automatically escalates priority, predicting P3 for one and P1 for another.
6. **dv-00177 (Category Confusion):** User said "red flag on my top up". Model predicted `fraud` instead of the gold `payment_failed` (a system flag blocking a legitimate top-up is not unauthorized theft).

## 6. What I Would Do Next (1 More Week)
1. **Targeted Synthetic Data Augmentation:** Generate 200 synthetic tickets specifically targeting the model's weak points: Hinglish relative dates ("parso", "kal"), edge-case duplicate charge priorities, and "red flag" vs "fraud" terminology.
2. **Oversampling High-Value Thresholds:** The model struggles with the ₹50,000 boundary. I would oversample training examples where the amount is between ₹40k and ₹60k to force the model to memorize this specific business rule.
3. **Curriculum Learning:** Instead of random shuffling, train on "easy" (English, simple dates) examples first, and spend the final epoch exclusively on "hard" (Hinglish, relative dates, URGENT keywords) examples.

## 7. AI Tools Disclosure
AI assistants were used to scaffold complex regex patterns for the data auditor, debug Colab environment dependency conflicts (Unsloth/Transformers version mismatches), and format this report. **All data curation decisions, experiment designs, business logic interpretations, and final code verifications were made entirely by me, the candidate.**