PRE-001 CORRECTION PROTOCOL — BOT.IA 1.1.1
==============================================

CURRENT STATE (VERIFIED):
-----------------------
Two llama.exe processes running, neither with --n-gpu-layers 0:

PID 9512: "C:\Users\ericb\AppData\Local\Microsoft\WindowsApps\llama.exe" serve 
  --host 127.0.0.1 --port 9931 --jinja --cache-type-k f16 --cache-type-v f16
  → MISSING: --n-gpu-layers 0

PID 8052: C:\Users\ericb\AppData\Local\Microsoft\WindowsApps\llama.exe serve
  --host 127.0.0.1 --jinja --log-colors off --port 53980 
  --alias ggml-org/gemma-3-4b-it-qat-GGUF:Q4_0 --cache-type-k f16 --cache-type-v f16
  --hf-repo ggml-org/gemma-3-4b-it-qat-GGUF:Q4_0
  → MISSING: --n-gpu-layers 0

PRE-001 FAILURE DETAIL (from preflight-report.json):
---------------------------------------------------
Severity: CRITICAL — MANDATORY FAILURE
The --n-gpu-layers 0 argument is NOT present in the process command line.
GUI configuration in Llama-Windows 0.12.0 was not persisted to running process.

REQUIRED CORRECTION:
--------------------
1. OPEN Llama-Windows 0.12.0 GUI
2. Navigate to: Settings → Llama → Custom Serve Arguments
3. ADD --n-gpu-layers 0 to the custom arguments field
4. SAVE the configuration
5. RESTART Llama-Windows server (stop current instance, start new)
6. VERIFY new process command line includes --n-gpu-layers 0
7. CONFIRM via API that model status changes from UNLOADED to LOADED (optional, after CPU config)

POST-CORRECTION VERIFICATION:
-----------------------------
After operator applies fix, verify:
powershell -NoProfile -Command "Get-CimInstance Win32_Process -Filter 'ProcessId = 9512' | Select-Object CommandLine"
Expected: ...serve --host 127.0.0.1 --port 9931 --jinja --cache-type-k f16 --cache-type-v f16 --n-gpu-layers 0

If PID 9512 now includes --n-gpu-layers 0 → PRE-001 PASSED
If not → PRE-001 FAILED, maintain release block

DEPENDENCIES:
-------------
- This correction does NOT load the model
- This correction does NOT require model inference
- This correction only adds an argument to the serve command
- Model loading is a separate step (Phase 2, step 9: "Execute validation inference only after confirming CPU config and obtaining authorization")

NOTES:
------
- The GUI configuration in Llama-Windows 0.12.0 must persist the argument
- Without --n-gpu-layers 0, the agent operates in offline mode (no model loading)
- This is the prerequisite for all 6 pending E2E validations
- After this fix, proceed to Phase 3 (build), Phase 4 (6 validations), Phase 5 (security), Phase 6 (installer), Phase 7 (re-evaluation)