# buraya claude code kullanirken aldigim errorlari, hatalari ekliyorum

---

- bash denied by auto mode hatasi aldim az once, bu fixlenmeli
- Bash(# Hangi hook "JSON validation failed" veriyor - her ikisini de test edelim
      MSG='{"session_id":"test","stop_hook_active":true,"last_assistant_message":"Tama
      mland…)
  ⎿  Error: claude-opus-5-0 is temporarily unavailable (timed out), so auto mode 
     cannot determine the safety of Bash right now. Wait a moment and then try this 
     action again. If it keeps failing, continue with other tasks that don't require 
     this action and come back to it later. Note: reading files, searching code, and 
     other read-only operations do not require the classifier and can still be used.

- bash denied by auto mode · Classifier unavailable · /permi… 

- ● Entered plan mode
  Claude is now exploring and designing an implementation approach.
  ⎿  Invalid tool parameters

- ● Notları okudum, içerik net. Dil tercihini sormak istiyorum:
  ⎿  Invalid tool parameters
  
-      don't ask mode. IMPORTANT: You *may* attempt to accomplish this action using other
     tools that might naturally be used to accomplish this goal, e.g. using head instead
     of cat. But you *should not* attempt to work around this denial in malicious ways,
     e.g. do not use your ability to run tests to execute non-test actions. You should
     only try to work around this restriction in reasonable ways that do not attempt to
     bypass the intent behind this denial. If you believe this capability is essential to
     complete the user's request, STOP and explain to the user what you were trying to
     do and why you need this permission. Let the user decide how to proceed.
     ● Workflow aracına izin verilmedi.

- Error: You are not in plan mode. To enter plan mode, call the 
     EnterPlanMode tool first. If your plan was already approved, 
     continue with implementation.

- bash denied by auto mode · Classifier unavailable · /permissions

## NOT : Dangerously-skip-permissions gibi bi mod yok bende suan ama boyle bir mod yapilabilirse sorunlarin bi kismi duzlecek (eb azindan bash denied by permissions errorlari)