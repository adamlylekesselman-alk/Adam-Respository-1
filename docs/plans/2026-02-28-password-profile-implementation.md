# Password Profile & Progress Persistence Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add a password-based profile system to `vocab-flashcards.html` so Learn SR data, quiz history, match best score, and preferences persist across sessions in localStorage.

**Architecture:** All state lives in a single JSON object in localStorage keyed by `vocab_profile_<password>`. A `PasswordScreen` component gates the app on load. Profile state is held at the top level and passed into `VocabApp`; every relevant user action triggers an auto-save.

**Tech Stack:** React 18 (CDN), Tailwind CSS (CDN), Babel standalone, `window.localStorage`

> **Note:** No test framework is available — this is a standalone HTML file. Each task includes manual browser verification steps instead of automated tests.

---

### Task 1: Add localStorage utility functions

**Files:**
- Modify: `vocab-flashcards.html` — add helpers just before the `VocabApp` function definition (around line 184)

**Step 1: Add the two utility functions**

Insert this block immediately before `function VocabApp()`:

```js
// ── Profile persistence ────────────────────────────────────────────────────
function profileKey(password) {
  return `vocab_profile_${password}`;
}

function loadProfile(password) {
  try {
    const raw = localStorage.getItem(profileKey(password));
    return raw ? JSON.parse(raw) : null;
  } catch {
    return null;
  }
}

function saveProfile(password, data) {
  try {
    localStorage.setItem(profileKey(password), JSON.stringify(data));
  } catch {
    // storage full or private browsing — fail silently
  }
}
```

**Step 2: Verify in browser console**

Open `vocab-flashcards.html` in a browser. Open DevTools console and run:

```js
saveProfile("test123", { direction: "es-en" });
console.log(loadProfile("test123")); // should print { direction: "es-en" }
console.log(loadProfile("wrong"));   // should print null
localStorage.removeItem("vocab_profile_test123"); // cleanup
```

Expected: First log prints the object, second prints `null`.

**Step 3: Commit**

```bash
git add vocab-flashcards.html
git commit -m "feat: add localStorage profile utility functions"
```

---

### Task 2: Add PasswordScreen component

**Files:**
- Modify: `vocab-flashcards.html` — add component just after the utility functions from Task 1

**Step 1: Add the PasswordScreen component**

Insert immediately after the `saveProfile` function:

```jsx
function PasswordScreen({ onLogin }) {
  const [pw, setPw] = React.useState("");
  const [shake, setShake] = React.useState(false);

  const handleSubmit = () => {
    const trimmed = pw.trim();
    if (!trimmed) {
      setShake(true);
      setTimeout(() => setShake(false), 500);
      return;
    }
    const existing = loadProfile(trimmed);
    onLogin(trimmed, existing);
  };

  return (
    <div className="min-h-screen bg-gradient-to-br from-orange-50 via-white to-rose-50 flex items-center justify-center p-4">
      <div className="bg-white rounded-2xl shadow-sm border border-gray-100 p-8 w-full max-w-sm text-center">
        <h1 className="text-2xl font-bold text-gray-900 mb-1">Vocabulario</h1>
        <p className="text-sm text-gray-400 mb-6">Enter your password to load your progress</p>
        <input
          type="password"
          value={pw}
          onChange={e => setPw(e.target.value)}
          onKeyDown={e => e.key === "Enter" && handleSubmit()}
          placeholder="Password"
          autoFocus
          className={`w-full px-4 py-3 rounded-xl border-2 border-gray-200 focus:border-orange-400 focus:outline-none text-gray-800 mb-4 transition-all ${shake ? "border-red-400" : ""}`}
        />
        <button
          onClick={handleSubmit}
          className="w-full py-3 rounded-xl bg-orange-500 hover:bg-orange-600 text-white font-semibold shadow-md transition-all"
        >
          Continue →
        </button>
        <p className="text-xs text-gray-400 mt-4">New password? A fresh profile will be created.</p>
      </div>
    </div>
  );
}
```

**Step 2: Add top-level wrapper component**

Replace the final render block at the bottom of the script:

```jsx
// BEFORE (remove this):
const root = ReactDOM.createRoot(document.getElementById("root"));
root.render(<VocabApp />);

// AFTER (replace with this):
function App() {
  const [profile, setProfile] = React.useState(null); // { password, data }

  const handleLogin = (password, existingData) => {
    const data = existingData || {
      direction: "es-en",
      selectedSections: Object.keys(ALL_VOCAB),
      srData: {},
      quizHistory: [],
      matchBest: null,
    };
    setProfile({ password, data });
  };

  if (!profile) return <PasswordScreen onLogin={handleLogin} />;
  return <VocabApp profile={profile} setProfile={setProfile} />;
}

const root = ReactDOM.createRoot(document.getElementById("root"));
root.render(<App />);
```

**Step 3: Update VocabApp signature to accept props**

Change the function signature from:
```jsx
function VocabApp() {
```
to:
```jsx
function VocabApp({ profile, setProfile }) {
```

**Step 4: Verify in browser**

Reload the page. You should see the password screen. Enter any password and press Enter — the menu should appear. Reload again — password screen should appear again (nothing saved yet).

**Step 5: Commit**

```bash
git add vocab-flashcards.html
git commit -m "feat: add PasswordScreen component and App wrapper"
```

---

### Task 3: Initialize VocabApp state from saved profile

**Files:**
- Modify: `vocab-flashcards.html` — update the `useState` initializers inside `VocabApp`

**Step 1: Replace the direction and selectedSections useState calls**

Find these two lines near the top of `VocabApp`:
```jsx
const [selectedSections, setSelectedSections] = useState(new Set(SECTIONS));
const [direction, setDirection] = useState("es-en");
```

Replace with:
```jsx
const [selectedSections, setSelectedSections] = useState(
  () => new Set(profile.data.selectedSections || SECTIONS)
);
const [direction, setDirection] = useState(profile.data.direction || "es-en");
```

**Step 2: Add a save helper at the top of VocabApp**

Add this right after the useState declarations block (after all the `const [x, setX] = useState(...)` lines):

```jsx
// Saves current profile data merged with any updates
const save = React.useCallback((updates) => {
  const newData = { ...profile.data, ...updates };
  saveProfile(profile.password, newData);
  setProfile(p => ({ ...p, data: newData }));
}, [profile]);
```

**Step 3: Verify in browser**

1. Open the app, enter password "adam", confirm you see the menu with all sections selected
2. Open DevTools > Application > Local Storage — you should see `vocab_profile_adam` with the default data after login

**Step 4: Commit**

```bash
git add vocab-flashcards.html
git commit -m "feat: initialize VocabApp state from saved profile"
```

---

### Task 4: Auto-save preferences (sections + direction)

**Files:**
- Modify: `vocab-flashcards.html` — update `toggleSection` and direction button handler

**Step 1: Update toggleSection to call save**

Find the `toggleSection` function:
```jsx
const toggleSection = (sec) => {
  setSelectedSections((prev) => {
    const next = new Set(prev);
    if (next.has(sec)) next.delete(sec);
    else next.add(sec);
    return next;
  });
};
```

Replace with:
```jsx
const toggleSection = (sec) => {
  setSelectedSections((prev) => {
    const next = new Set(prev);
    if (next.has(sec)) next.delete(sec);
    else next.add(sec);
    save({ selectedSections: Array.from(next) });
    return next;
  });
};
```

**Step 2: Update direction buttons to call save**

Find the direction button onClick handlers in the menu render:
```jsx
onClick={() => setDirection(val)}
```

Replace with:
```jsx
onClick={() => { setDirection(val); save({ direction: val }); }}
```

**Step 3: Verify in browser**

1. Enter password "adam", deselect a section, change direction to English→Spanish
2. Reload the page, enter "adam" again — the section should still be deselected and direction English→Spanish
3. Enter a different password — should see fresh defaults (all sections selected, Spanish→English)

**Step 4: Commit**

```bash
git add vocab-flashcards.html
git commit -m "feat: auto-save section and direction preferences"
```

---

### Task 5: Save and restore Learn mode SR data

**Files:**
- Modify: `vocab-flashcards.html` — update `startLearn` and `handleLearnRate`

**Step 1: Update startLearn to merge saved srData**

Find the `startLearn` function:
```jsx
const startLearn = () => {
  const raw = getActiveCards();
  const deck = srInit(raw);
  const queue = deck.map((_, i) => i);
  setLearnDeck(deck);
  setLearnQueue(shuffle(queue));
  setLearnFlipped(false);
  setLearnRound(0);
  setMode("learn");
};
```

Replace with:
```jsx
const startLearn = () => {
  const raw = getActiveCards();
  const savedSr = profile.data.srData || {};
  const deck = srInit(raw).map(card => {
    const saved = savedSr[card.word];
    return saved ? { ...card, ...saved } : card;
  });
  // Put graduated cards at the back (still show them, but deprioritized)
  const notGrad = deck.map((_, i) => i).filter(i => !deck[i].graduated);
  const grad = deck.map((_, i) => i).filter(i => deck[i].graduated);
  setLearnDeck(deck);
  setLearnQueue(shuffle(notGrad).concat(shuffle(grad)));
  setLearnFlipped(false);
  setLearnRound(0);
  setMode("learn");
};
```

**Step 2: Update handleLearnRate to save srData after each rating**

Find the `handleLearnRate` function. After `setLearnDeck(updatedDeck);`, add:

```jsx
// Save SR data keyed by Spanish word
const newSrData = { ...(profile.data.srData || {}) };
updatedDeck.forEach(c => {
  newSrData[c.word] = { interval: c.interval, easeFactor: c.easeFactor, reps: c.reps, graduated: c.graduated };
});
save({ srData: newSrData });
```

**Step 3: Verify in browser**

1. Enter password "adam", start Learn mode, rate a few cards as "Easy"
2. Reload, enter "adam", start Learn — the cards you rated Easy should have their SR state restored (they may be at the back of the queue if graduated)
3. Open DevTools > Local Storage > `vocab_profile_adam` — confirm `srData` has entries for the words you rated

**Step 4: Commit**

```bash
git add vocab-flashcards.html
git commit -m "feat: persist and restore Learn mode SR data across sessions"
```

---

### Task 6: Save quiz history and show on menu

**Files:**
- Modify: `vocab-flashcards.html` — update `quizResults` screen and menu

**Step 1: Save quiz history when quiz completes**

Find the `nextQuiz` function:
```jsx
const nextQuiz = () => {
  setQuizFeedback(null);
  setQuizInput("");
  if (quizIndex + 1 >= quizCards.length) setMode("quizResults");
  else setQuizIndex((i) => i + 1);
};
```

Replace with:
```jsx
const nextQuiz = () => {
  setQuizFeedback(null);
  setQuizInput("");
  if (quizIndex + 1 >= quizCards.length) {
    const entry = {
      date: new Date().toLocaleDateString(),
      score: quizScore + (quizFeedback?.isCorrect ? 1 : 0),
      total: quizCards.length,
    };
    const history = [entry, ...(profile.data.quizHistory || [])].slice(0, 10);
    save({ quizHistory: history });
    setMode("quizResults");
  } else {
    setQuizIndex((i) => i + 1);
  }
};
```

> Note: `quizScore` state updates are async, so we add the current answer's result inline.

**Step 2: Show last quiz score on the menu button**

Find the Written Quiz button in the menu render:
```jsx
<button
  onClick={startQuiz}
  disabled={totalWords === 0}
  className="flex-1 py-3 rounded-xl bg-rose-500 hover:bg-rose-600 text-white font-medium shadow-md shadow-rose-200 transition-all disabled:opacity-40"
>
  Written Quiz (20)
</button>
```

Replace with:
```jsx
<button
  onClick={startQuiz}
  disabled={totalWords === 0}
  className="flex-1 py-3 rounded-xl bg-rose-500 hover:bg-rose-600 text-white font-medium shadow-md shadow-rose-200 transition-all disabled:opacity-40"
>
  Written Quiz (20)
  {profile.data.quizHistory?.length > 0 && (
    <span className="block text-xs font-normal opacity-75">
      Last: {profile.data.quizHistory[0].score}/{profile.data.quizHistory[0].total}
    </span>
  )}
</button>
```

**Step 3: Verify in browser**

1. Enter password "adam", take a quiz to completion
2. Return to menu — the quiz button should show "Last: X/20"
3. Reload, enter "adam" — the last score should still be there

**Step 4: Commit**

```bash
git add vocab-flashcards.html
git commit -m "feat: save quiz history and display last score on menu"
```

---

### Task 7: Save match best score and show on menu

**Files:**
- Modify: `vocab-flashcards.html` — update match done handler and menu button

**Step 1: Save best match score when match completes**

Find where `setMatchDone(true)` is called inside `handleMatchClick`:
```jsx
if (newMatched.size === matchCards.length) setMatchDone(true);
```

Replace with:
```jsx
if (newMatched.size === matchCards.length) {
  const currentBest = profile.data.matchBest;
  if (currentBest === null || matchMisses < currentBest) {
    save({ matchBest: matchMisses });
  }
  setMatchDone(true);
}
```

**Step 2: Show best score on the menu button**

Find the Match Game button in the menu render:
```jsx
<button
  onClick={startMatch}
  disabled={totalWords < 6}
  className="flex-1 py-3 rounded-xl bg-violet-500 hover:bg-violet-600 text-white font-medium shadow-md shadow-violet-200 transition-all disabled:opacity-40"
>
  Match Game
</button>
```

Replace with:
```jsx
<button
  onClick={startMatch}
  disabled={totalWords < 6}
  className="flex-1 py-3 rounded-xl bg-violet-500 hover:bg-violet-600 text-white font-medium shadow-md shadow-violet-200 transition-all disabled:opacity-40"
>
  Match Game
  {profile.data.matchBest !== null && profile.data.matchBest !== undefined && (
    <span className="block text-xs font-normal opacity-75">
      Best: {profile.data.matchBest} miss{profile.data.matchBest !== 1 ? "es" : ""}
    </span>
  )}
</button>
```

**Step 3: Verify in browser**

1. Enter password "adam", play Match Game to completion
2. Return to menu — the match button should show "Best: X misses"
3. Reload, enter "adam" — best score persists

**Step 4: Commit**

```bash
git add vocab-flashcards.html
git commit -m "feat: save match best score and display on menu"
```

---

### Task 8: Add Sign Out button

**Files:**
- Modify: `vocab-flashcards.html` — add sign out button to menu

**Step 1: Add sign out to the menu header**

Find the menu header block:
```jsx
<div className="text-center mb-8">
  <h1 className="text-3xl font-bold text-gray-900 mb-1">Vocabulario</h1>
  <p className="text-lg text-gray-500">Mes Herencia Hispana — Parte II</p>
  <p className="text-sm text-gray-400 mt-1">{totalWords} words selected</p>
</div>
```

Replace with:
```jsx
<div className="text-center mb-8">
  <h1 className="text-3xl font-bold text-gray-900 mb-1">Vocabulario</h1>
  <p className="text-lg text-gray-500">Mes Herencia Hispana — Parte II</p>
  <p className="text-sm text-gray-400 mt-1">{totalWords} words selected</p>
  <button
    onClick={() => setProfile(null)}
    className="text-xs text-gray-400 hover:text-gray-600 mt-2 underline"
  >
    Sign out
  </button>
</div>
```

**Step 2: Verify in browser**

1. Enter password "adam", you should see the menu with a small "Sign out" link under the subtitle
2. Click Sign out — you should return to the password screen
3. Enter "adam" again — all your saved progress should still be there

**Step 3: Commit**

```bash
git add vocab-flashcards.html
git commit -m "feat: add sign out button to return to password screen"
```

---

## Final Verification Checklist

Before considering this complete:

- [ ] Password screen appears on fresh page load
- [ ] New password creates fresh profile; existing password loads saved data
- [ ] Toggling sections + direction persists across reload with same password
- [ ] Learn mode: rated cards have correct SR state after reload
- [ ] Learn mode: graduated cards appear at back of queue on resume
- [ ] Quiz history shows on menu button after completing a quiz
- [ ] Match best score shows on menu button after completing a match
- [ ] Sign out returns to password screen without losing data
- [ ] Different passwords have independent profiles
- [ ] App works normally when no profile data exists for a key (fresh start)
