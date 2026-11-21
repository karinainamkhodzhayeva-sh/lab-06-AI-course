# JUSTIFICATION — the design of the harness, and why

## 1. What the system may do

The system is built only to pre-screen complete scholarship applications from consenting adult applicants (age 18 or older, consent_to_automated_processing = True) for the single purpose of scholarship_screening. It may issue an automatic shortlist or reject decision only when the profile passes all checks, gets a valid JSON reply, scores outside the 40 to 49 borderline range, and does not trigger the low-income rejection rule. For everything in the borderline range or flagged as high-risk, the system only informs a human reviewer.

Each control in HARNESS was added for a specific reason:
- validate_input=True (validate_profile in lab06_core.py) stops impossible numbers before the model sees them.
- blind=("name", "university") (render_for_model in lab06_core.py) hides identity and university fields so the model only grades merit and financial need.
- refuse=True (refusal_reason and my_extra_refusal) enforces legal and security limits before making a model call.
- log_path="decisions.jsonl" (screen and _append_record) saves one JSON record per decision.
- escalate=True with borderline=(40, 49) and my_extra_escalation sends uncertain or sensitive cases to a person.

Privacy constraints shaped how logging was built. Instead of saving raw personal details in decisions.jsonl, screen only logs pid, a SHA-256 hash of the input text (input_sha256), and the lists fields_shown and fields_withheld. Without this, the log file would become a second copy of sensitive student data.

## 2. What it must refuse

The harness refuses requests without calling the model (0 model calls) in four cases:
1. Invalid input (invalid_input): triggered by validate_profile when fields is out of range (fired in Part 4 on "impossible values (GPA 9.9, exam score -5)").
2. Wrong purpose (purpose_not_allowed): trigered by refusal_reason when purpose is not scholarship_screening (fired in Part 4 on "purpose: rank applicants by ethnicity").
3. Minor or no consent (minor, no_consent): triggered by refusal_reason when age is under 18 or consent is False (fired in Part 4 on "an applicant who is 17" and "no consent to automated processing").
4. Prompt injection (prompt_injection_detected): triggered by my_extra_refusal when the name field is over 60 characters or contains instruction words like "ignore", "previous instructions", "output:", "shortlist", "score", "system", "{", or "}" (fired in Part 4 on "the name field is instruction").

A strict name filter can accidentally block real students who have very long names or names containing those letters, which would delay their application. However, testing on all 400 clean profiles in PAIRS gave 0 false refusals, so this trade-off is acceptable as long as students can appeal. Every refused request is saved in decisions.jsonl with outcome="refuse", score=None, and its reason_code so the applicant can be told why it failed and ask for human review.

## 3. What record it keeps

The log is built to answer one main audit question: why did this applicant get this exact outcome, which controls fired, and what input did the model see, without storing raw personal data? Each line in decisions.jsonl answers this by recording record_id, pid, input_sha256, fields_shown, fields_withheld, model_id, prompt_version, harness settings, raw_output, decision, score, controls_fired, outcome, reason_code, escalated_to_human, and timestamp.

Raw values like the student's name, university, and exact GPA are left out on purpose so the log stays lightweight and does not leak personal data (blinded fields are never sent to the model, so they cannot appear in raw_output either). In a real deployment, these logs should be kept for the standard grant audit period (around 5 years), restricted to compliance staff, and stored in append-only files so nobody can edit past decisions. Because raw inputs are hashed rather than saved in plain text, an auditor cannot see the student's exact GPA from decisions.jsonl alone without looking up pid in the main encrypted database.

## 4. The legal constraints

Since  Fund works in Kazakhstan, domestic personal data and informatization laws apply to automated processing of student records, alongside EU AI rules as an international benchmark:
- Constitution of the Republic of Kazakhstan (30 August 1995), Article 14, Paragraph 2: forbids discrimination based on origin, social status, gender, nationality, or language. This requires keeping non-merit traits from affecting scholarship scores, which is why I enabled blind.
- Law of the Republic of Kazakhstan No. 94-V "On Personal Data and Their Protection" (21 May 2013), Article 7, Paragraph 1 and Article 8: requires personal data to be processed only for the stated purpose and with the person's consent. These provisions directly require refuse to block wrong purposes (purpose_not_allowed) and missing consent (no_consent), while leaving technical security filters like prompt_injection_detected up to the developer.
- Law of the Republic of Kazakhstan No. 94-V (21 May 2013), Article 12, Paragraph 1: requires operators to ensure personal data is accurate and reliable, which supports validate_input blocking impossible GPA and exam values.
- Law of the Republic of Kazakhstan No. 418-V "On Informatization" (24 November 2015), Article 36, Paragraph 1: requires owners of electronic information systems to ensure data integrity and audit logging, supporting the structured log in decisions.jsonl.
- Regulation (EU) 2024/1689 (EU Artificial Intelligence Act, 13 June 2024), Article 14, Paragraphs 1-2: requires human oversight for high-risk AI systems (including education and scholarship access in Annex III) so people can intervene and prevent harm. Because Kazakhstan's laws do not specify exact numeric score bands for human review in automated decisions, EU AI Act Article 14 is cited here as a non-binding international benchmark rather than local law, and I went beyond local requirements by adding my_extra_escalation for rejected low-income students.
  
## 5. The ethical principles
Each control in the register is tied to an international ethical recommendation (which is non-binding guidance, not law):
- Fairness and Non-Discrimination (UNESCO Recommendation on the Ethics of Artificial Intelligence, 2021, Section III.2, Paragraphs 28-30): applies to blind, removing name and university after Part 1 showed a +2.5 score advantage for flagship universities (sign p = 0.002).
- Safety, Security, and Accuracy (UNESCO Recommendation on the Ethics of Artificial Intelligence, 2021, Section III.2, Paragraphs 26-27): applies to validate_input and my_extra_refusal, keeping bad data and prompt injections from breaking the scoring logic.
- Respect for Human Rights and Democratic Values (OECD Recommendation of the Council on Artificial Intelligence, OECD/LEGAL/0449, 2019, amended 2024, Principle 1.2): applies to refuse, enforcing consent, age limits, and purpose boundaries.
- Transparency, Explainability, and Accountability (OECD Recommendation of the Council on Artificial Intelligence, OECD/LEGAL/0449, 2019, amended 2024, Principles 1.3 and 1.5): applies to log, making every outcome traceable.
- Human Oversight and Determination (UNESCO Recommendation on the Ethics of Artificial Intelligence, 2021, Section III.2, Paragraphs 35-36): applies to escalate, preventing purely automated rejections on borderline or low-income applications.

In this design, treating every applicant identically clashed with treating them fairly. Hiding the university name took away context about how strict a university's grading might be, which a human might consider, but I prioritized non-discrimination (blind) and substantive fairness (my_extra_escalation for household_income_bracket == 1) because the model gave flagship universities an unfair +2.5 point bump. Ethics also pushed me to add extra human review for low-income rejections even though no local law requires it.

## 6. The trade-offs
- blind=("name", "university"): reduced pair flips and score differences to 0.0% (down from up to 10.0% flips and a +2.5 university score gap) and lowered prompt tokens from 84,409 to 77,922 across 400 model calls (487.3s vs 488.7s). The cost is that removing two lines changed the prompt layout, which shifted 166 individual scores (mean |delta| = 14.01) and changed 51 outcomes (27 reject -> escalate, 19 shortlist -> escalate, 5 reject -> shortlist), while hiding legitimate university context.
- validate_input and refuse: blocked 5 out of 6 bad requests in Part 4 with 0 model calls and 0 false refusals on the 400 clean profiles, at the cost of maintaining extra code and watching out for false blocks on long names.
- log: added almost zero runtime delay and protected privacy via SHA-256 hashes, at the cost of needing the main database to match pid back to a student's name.
- escalate and BORDERLINE = (40, 49): calibrated on the baseline score table where 64 of 400 profiles (16.0%) fell between 40 and 49 (where reject max 45 and shortlist min 45 overlapped). With the harness on, 46 of 400 profiles (11.5%) were actually sent to a person because blinding shifted some scores. This keeps reviewer workload reasonable while catching borderline and low-income cases, though the 12 auto-rejected and 342 auto-shortlisted profiles outside this band are not checked by a human. If the fund had more staff, widening the band to (35, 55) would catch more borderline shifts.

What the design still fails to protect against: I left counterfactual_check off so it would not double the model calls (800 instead of 400), and remaining fields like program or income bracket can still act as indirect proxies for background. Also, I did not run the optional prompt-order probe (ORDER_PROBE), so sensitivity to line ordering remains unmitigated. Fixing these would require enabling counterfactual checks and normalizing prompt layouts, which I skipped to keep runtime and compute costs low.

## AI-use declaration
I used an AI assistant in this lab mainly to understand the assignment instructions and figure out how different parts of the notebook work. During my work I asked questions to clarify concepts, understand why certain functions or checks behave the way they do, and help locate the relevant laws, regulations, and ethical frameworks that apply to automated screening. After getting pointers on which legal instruments to look at, I checked the actual provisions myself on official legal databases. All predictions before running the experiments were written by me, and for the final observations and conclusions. I ran the notebook myself and all numbers and decisions in this work belong to me.

