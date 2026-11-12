OPERATOR INSTRUCTIONS — PRE-001 FIX
=====================================

AUTHORITY: Engenheiro Sênior de Software, Arquiteto de Sistemas
TASK: Fix PRE-001 — --n-gpu-layers 0 not applied to running llama.exe process
CURRENT STATUS: PRE-001 FAILED (2 processes confirmed missing --n-gpu-layers 0)
RELEASE BLOCK: YES — Release blocked until PRE-001 passes

CURRENT PROCESS VERIFICATION (by engineer):
-----------------------------------------
PID 9512: command = serve --host 127.0.0.1 --port 9931 --jinja --cache-type-k f16 --cache-type-v f16 (NO --n-gpu-layers 0)
PID 8052: command = serve --host 127.0.0.1 --jinja --log-colors off --port 53980 --alias ... (NO --n-gpu-layers 0)

INSTRUCTIONS FOR OPERATOR:
-------------------------

STEP 1: STOP CURRENT LLAMA SERVER
--------------------------------
1. Open a terminal (cmd or PowerShell)
2. Identify the PID(s) of the running llama.exe process(es):
   tasklist | find "llama.exe"
3. Stop the primary server process (typically the one on port 9931):
   taskkill /F /PID <PID_NUMBER>
   Example: taskkill /F /PID 9512
4. Verify the process has stopped:
   tasklist | find "llama.exe"

STEP 2: CONFIGURE LLAMA-WINDOWS 0.12.0 GUI
-------------------------------------------
1. Open Llama-Windows 0.12.0 application
2. Navigate to: Settings → Llama → Custom Serve Arguments
3. In the "Custom Serve Arguments" field, add: --n-gpu-layers 0
   The field may already have other arguments; ensure --n-gpu-layers 0 is included
   Example final value: --host 127.0.0.1 --port 9931 --jinja --cache-type-k f16 --cache-type-v f16 --n-gpu-layers 0
4. SAVE the configuration
5. CLOSE Llama-Windows 0.12.0

STEP 3: RESTART LLAMA-WINDOWS SERVER
-------------------------------------
1. Launch Llama-Windows 0.12.0 again
2. Verify the new process starts with --n-gpu-layers 0 in its command line:
   - Open a new terminal
   - tasklist | find "llama.exe"
   - Note the new PID(s)
3. Check the command line of the new process:
   powershell -NoProfile -Command "Get-CimInstance Win32_Process -Filter 'ProcessId = <NEW_PID>' | Select-Object CommandLine"
4. Confirm --n-gpu-layers 0 is present in the output

STEP 4: POST-CORRECTION VERIFICATION
------------------------------------
After the operator completes steps 1-3, the engineer verifies:

1. New process command line includes --n-gpu-layers 0
2. If confirmed: PROCEED TO NEXT PHASE (Phase 3: Build diagnosis)
3. If NOT confirmed: MAINTAIN RELEASE BLOCK, investigate why

CRITICAL NOTES:
---------------
- DO NOT load the model during this process
- DO NOT start any inference during this step
- The --n-gpu-layers 0 argument configures CPU-only mode for the serve command
- Model loading is a separate step that happens AFTER PRE-001 passes
- There are TWO processes (PIDs 9512 and 8052); both may need the argument
- After fixing PID 9512 (port 9931), verify PID 8052 as well if it's also serving

EXPECTED OUTCOME:
----------------
After operator applies these steps:
- New llama.exe process command line includes --n-gpu-layers 0
- PRE-001 passes in new preflight execution
- Release block for PRE-001 can be lifted
- Proceed to Phase 3 (build diagnosis) and Phase 4 (6 integration validations)

ACTUAL OUTCOME (to be filled by operator):
-----------------------------------------
[Operator to complete after applying fix]
- PID(s) fixed: _______
- Command line verified: YES / NO
- --n-gpu-layers 0 present: YES / NO
- PRE-001 result: PASSED / FAILED