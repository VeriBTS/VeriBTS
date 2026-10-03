# VeriBTS

This repository provides implementations and experimental artifacts for six instances of **VeriBTS: A General LLM-Enabled CEGIS Framework for Behavior Tree Generation**, an LLM-driven, verifier-guided framework for behavior tree synthesis:

- **Examples:** positive and negative example consistency.
- **Goal:** finite-time goal attainment.
- **MITL–TA:** timed-automata verification with BT2Automata/UPPAAL.
- **LTL–BB:** blackboard-based verification with BehaVerify/nuXmv.
- **LTL–CSP:** CSP-based verification with MoVe4BT/PAT.
- **LTL–Direct:** direct BT-to-LTL encoding and verification with Spot.

Our experiments evaluate all six instances on **70 adapted benchmark tasks** and **10 additional stress tasks per instance**, using three LLMs, five repetitions, and a ten-call budget per task. We report synthesis success rates and LLM invocation costs, compare two feedback ablations, and evaluate the Examples instance against BtBot. Experimental data, execution scripts, and analysis scripts are included for reproducibility.
