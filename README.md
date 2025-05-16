# GlobalExam Auto-Filler with GPT API

Automate GlobalExam training sessions:  
the script reads every page (context, transcription, images), asks **ChatGPT**
for the best answers, coche the right radios, keeps the page “alive” so the
timer doesn’t pause, waits the required time before **Terminer**, and chains
every activity until the end.

> **Disclaimer**  
> Respect GlobalExam’s Terms of Service — automation may be forbidden on real exams.

---

## Features
|  | Description |
|---|-------------|
| GPT-powered answers | Sends page + transcription + (0-2) images to **GPT-4o** and parses the letters returned. |
| Robust scraping    | Works with inline text *and* pop-up transcriptions; cleans extra SVG / icons out of choices. |
| Anti-idle keep-alive | Simulates scroll & mouse-move every few seconds + optional WakeLock. |
| One-click stop     | Run `stopAutoQCM()` in DevTools to halt the loop gracefully. |

---

## Prerequisites
* A modern Chromium-based browser (Chrome, Edge, Brave…)
* An **OpenAI secret key** with access to GPT-4o or GPT-4o-mini  
  Create one here → <https://platform.openai.com/>

```js

(async () => {
    /* --------------------------- CONFIGURATION ------------------------------ */
    const OPENAI_API_KEY      = '';      // mettre la clé de l'api Open AI : https://platform.openai.com/
    const MODEL               = 'gpt-4o-mini'; // voir la liste des models (prix, qualité...) : https://platform.openai.com/docs/models
  const TEMPERATURE = 0;
  const SCROLL_PIXELS = 6;
  const ACTION_DELAY = 150 * 1000;

  /* ------------------------------ GLOBAUX --------------------------------- */
  let run = true;
  window.stopAutoQCM = () => { run = false; clearInterval(keepAliveId); };

  const sleep = ms => new Promise(r => setTimeout(r, ms));

  /* ---------------------------- ANTI-IDLE --------------------------------- */
  const keepAliveId = setInterval(() => {
    window.scrollBy(0, SCROLL_PIXELS);
    window.scrollBy(0, -SCROLL_PIXELS);
    document.dispatchEvent(new MouseEvent('mousemove', {
      bubbles: true, clientX: Math.random() * innerWidth, clientY: Math.random() * innerHeight
    }));
  }, ACTION_DELAY);

  if (navigator.wakeLock?.request) try { await navigator.wakeLock.request('screen'); } catch { }

  /* ------------------------- NAVIGATION BOUTONS --------------------------- */
  const byRegex = rx => [...document.querySelectorAll('button,input,a')]
    .find(el => rx.test((el.innerText || el.value || '').trim()));
  const clickByRegex = rx => { const b = byRegex(rx); b?.click(); return !!b; };
  const clickChevron = () => { const b = document.querySelector('button svg.fa-chevron-right')?.closest('button'); b?.click(); return !!b; };
  const clickNext = () => clickByRegex(/suivant|continuer|valider|terminer/i) || clickChevron();
  const clickNextActivity = () => clickByRegex(/activité\s+suivante|exercice\s+suivant|examen\s+suivant/i);

  /* ------------------------- FERMETURE POP-UPS ---------------------------- */
  function closeXPopups() {
    const btn = [...document.querySelectorAll('button')].find(b => b.querySelector('svg.fa-times,[data-icon="times"]'));
    if (btn) { btn.click(); console.log('Popup fermée'); }
  }

  /* Remplace l’ancienne getContext() par celle-ci                              */
  function getContext() {

    const SELECTORS = [
      '.exercise-context',
      '.exercise-statement',
      '.exercise-body',
      '.wysiwyg',
      '[class*=wysiwyg]',
      'div[class*=bg-selection]',
      '.text-neutral-80.leading-tight'
    ];

    // 1) Premier sélecteur qui contient vraiment du texte
    for (const sel of SELECTORS) {
      const el = document.querySelector(sel);
      if (el) {
        const txt = el.innerText.trim();
        if (txt.length > 80) {                  // on évite les titres de 3 mots
          console.log(`📄 Contexte capturé (${sel}) : ${txt.length} caractères`);
          return txt;
        }
      }
    }

    // 2) Fallback : le paragraphe le plus long de la page (> 60 car.)
    const para = [...document.querySelectorAll('p')]
      .filter(p => p.offsetParent && p.innerText.trim().length > 60)
      .sort((a, b) => b.innerText.length - a.innerText.length)[0];

    if (para) {
      const txt = para.innerText.trim();
      console.log(`Contexte fallback : ${txt.length} caractères`);
      return txt;
    }

    console.log('Aucun contexte trouvé');
    return '';
  }

  async function getTranscription() {
    /* 1. Clique sur le bouton « Transcription » s’il existe ----------------- */
    const btn = [...document.querySelectorAll('button,a')]
      .find(b => /transcription/i.test(b.textContent));
    if (!btn) return '';

    btn.click();             

    /* 2. Attendre que le texte apparaisse (max 5 s) -------------------------- */
    let text = '';
    for (let i = 0; i < 25; i++) {    
      
      text = document.querySelector('.modal-content')?.innerText?.trim() || '';

      if (!text) text = document.querySelector('.wysiwyg, [class*=wysiwyg]')?.innerText?.trim() || '';

      if (!text) {
        const cand = [...document.querySelectorAll('div,p')]
          .find(el => /heatwave|refer to this talk|according to/i.test(el.innerText));
        text = cand?.innerText.trim() || '';
      }

      if (text) break;
      await sleep(200);
    }

    /* 3. Referme si c’était une modale (croix “X”) --------------------------- */
    document.querySelector('button svg.fa-times, button [data-icon="times"]')
      ?.closest('button')?.click();

    console.log(`📄 Transcription capturée : ${text.length} caractères`);
    return text;
  }

  function getQuestions() {

    /* --- 1. Regrouper les radios par name ---------------------------------- */
    const radios = [...document.querySelectorAll("input[type='radio']")];
    const groups = Object.values(
      radios.reduce((acc, r) => ((acc[r.name] ??= []).push(r), acc), {})
    );

    /* --- 2. Pour chaque groupe, trouver question + choix ------------------- */
    return groups.map((radios, qIdx) => {

      /* --- QUESTION -------------------------------------------------------- */
      let question = 'Question ' + (qIdx + 1);
      // on remonte 6 ancêtres max puis on regarde les éléments précédents
      let node = radios[0].closest('div, li, p');
      for (let i = 0; i < 6 && node; i++, node = node.parentElement) {
        const cand = node.previousElementSibling;
        if (cand && /\?$/.test(cand.innerText.trim()) && cand.innerText.length < 400) {
          question = cand.innerText.trim();
          break;
        }
      }

      /* --- CHOIX ----------------------------------------------------------- */
      const choices = radios.map((r, cIdx) => {
        /*  ▸ label = ancêtre le plus proche, pas de attribut [for="…"] */
        const label = r.closest('label');

        let text = 'Choice ' + (cIdx + 1);
        if (label) {
          const clone = label.cloneNode(true);
          clone.querySelectorAll('input, svg, i').forEach(x => x.remove());
          text = clone.innerText.trim().replace(/^\s*[A-D]\.\s*/, '') || text;
        }

        return {
          letter: String.fromCharCode(65 + cIdx),   // A / B / C / …
          radio: r,
          label: label,
          text: text
        };
      });

      return { question, choices };
    });
  }

  /** Prompt clair */
  function buildPrompt(context, transcript, questions) {
    let p = `You give the right answer for this English multiple-choice quiz.\n`;
    if (context) p += `\nCONTEXT:\n"""${context}"""\n`;
    if (transcript) p += `\nTRANSCRIPTION:\n"""${transcript}"""\n`;
    p += `\nQUESTIONS:\n`;
    questions.forEach((q, idx) => {
      p += `\n${idx + 1}. ${q.question}\n`;
      q.choices.forEach(c => p += `${c.letter}. ${c.text}\n`);
    });
    p += `\nReply with ONLY the chosen letters separated by spaces, e.g. "A C D B".`;
    return p;
  }

  /* ---------------------------- OPENAI CALL ------------------------------- */
  async function askGPT(prompt, count) {
    console.log(prompt);
    try {
      const res = await fetch('https://api.openai.com/v1/chat/completions', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${OPENAI_API_KEY}` },
        body: JSON.stringify({ model: MODEL, temperature: TEMPERATURE, max_tokens: 30, messages: [{ role: 'user', content: prompt }] })
      });
      const data = await res.json();
      const txt = data.choices?.[0]?.message?.content || '';
      const letters = (txt.match(/[A-Z]/gi) || []).slice(0, count).map(x => x.toUpperCase());
      console.log('GPT →', letters.join(' '), '| brut:', txt);
      return letters;
    } catch (e) { console.warn(e); return []; }
  }

  /* -------------------------- COCHER LES RADIOS --------------------------- */
  function applyAnswers(questions, letters) {
    questions.forEach((q, idx) => {
      const letter = letters[idx] || q.choices[Math.floor(Math.random() * q.choices.length)].letter;
      const choice = q.choices.find(c => c.letter === letter) || q.choices[0];
      (choice.label || choice.radio).click();
      (choice.label || choice.radio).style.outline = '2px solid limegreen';
    });
  }

  /* --------------------------- BOUCLE GLOBAL ------------------------------ */
  console.clear(); console.log('% GlobalSuper lancé', 'color:#2fa');

  let page = 1;
  while (run) {
    console.group(`=== Page ${page} ===`);

    const context = getContext();
    const transcript = await getTranscription();
    const questions = getQuestions();

    if (!questions.length) { clickNext(); console.groupEnd(); await sleep(ACTION_DELAY); page++; continue; }

    const prompt = buildPrompt(context, transcript, questions);
    const letters = await askGPT(prompt, questions.length);

    closeXPopups(); //Supprimer pour TOEIC
    applyAnswers(questions, letters);
    await sleep(ACTION_DELAY);

    clickNext();
    clickNextActivity();

    await sleep(ACTION_DELAY);
    console.groupEnd();
  }
})();

```

### How to run
Open a GlobalExam activity (training mode).

Press F12 → Console (or right-click ▸ Inspect ▸ Console).

Paste the entire script once → hit Enter.
You should see:

```console
GlobalSuper lancé
Contexte capturé (…)
Transcription capturée : 872 caractères
GPT → B C A | brut: B C A
```

Sit back. The bot loops through all questions / pages / activities.

Need to cancel? Type stopAutoQCM() in the console and press Enter.
