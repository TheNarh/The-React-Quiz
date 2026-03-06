# The React Quiz App

A timed, interactive quiz application that tests **real React knowledge** — built entirely with React using production-level patterns including a full state machine architecture, countdown timer, live scoring engine and highscore tracking.

> This app doesn't just demonstrate React — it quizzes you on it. Every question covers a core React concept: hooks, state, props, effects, rendering behaviour, and more.

---

##  Live Demo

[**Try it live →**](#) *(Add your Vercel or Netlify link here)*

---

##  Preview

| Start Screen | Quiz In Progress | Results Screen |
|---|---|---|
| <img width="1912" height="924" alt="StartScreen" src="https://github.com/user-attachments/assets/9a93eb60-a991-40f6-baac-b9d30ac4b7cc" />
 | <img width="1831" height="922" alt="QuizInProgress" src="https://github.com/user-attachments/assets/e8516446-cb59-41e6-9381-96295d9d3a25" />
 | <img width="1824" height="920" alt="FinishQuiz" src="https://github.com/user-attachments/assets/234b5914-8783-43f0-8e7e-aeb50a139256" />
 |

---

## What This App Does

- Fetches 15 React-focused quiz questions from a local JSON Server API
- Walks the user through a fully timed, multi-choice quiz
- Awards **different point values** per question based on difficulty (10, 20, or 30 pts)
- Counts down a live timer — **auto-finishes** the quiz if time runs out
- Highlights correct and wrong answers immediately after selection
- Displays a results screen with score, percentage, and emoji feedback
- Tracks and persists the **highscore** across restarts within the session
- Handles loading, error, and empty states gracefully

---

## The Questions

All 15 questions are served from a local JSON Server and cover real, practical React knowledge:

| Topic | Difficulty |
|---|---|
| Most popular JS framework | ⭐ 10pts |
| Who invented React | ⭐ 10pts |
| Fundamental building block of React | ⭐ 10pts |
| What is JSX | ⭐ 10pts |
| How data flows in React | ⭐ 10pts |
| How to pass data to child components | ⭐ 10pts |
| What triggers a UI re-render | ⭐⭐ 20pts |
| When to directly touch the DOM | ⭐⭐ 20pts |
| When does an effect run without a dependency array | ⭐⭐ 20pts |
| Which hook to use for API requests | ⭐ 10pts |
| When to use derived state | ⭐⭐⭐ 30pts |
| When to use a callback to update state | ⭐⭐⭐ 30pts |
| How useState lazy initialisation works | ⭐⭐⭐ 30pts |
| useEffect dependency array rules | ⭐⭐⭐ 30pts |
| Whether effects always run on initial render | ⭐⭐⭐ 30pts |

**Maximum possible score: 320 points**

---

## Architecture — The State Machine

The most deliberate decision in this app is the use of a **finite state machine** pattern powered by `useReducer`.

Rather than scattering `useState` calls across the app, all application state lives in a single reducer with clearly defined transitions:

```
loading → ready → active → finished
                     ↑          |
                     └──────────┘ (restart)
```

Each status represents a distinct phase of the quiz lifecycle. The UI renders conditionally based on the current status — no boolean flags, no tangled state.

```javascript
const initialState = {
  questions: [],
  status: "loading",    // one source of truth for app phase
  index: 0,
  answer: null,
  points: 0,
  highscore: 0,
  secondsRemaining: null,
};
```

### Why This Matters

This app deliberately uses `useReducer` because:

- All state transitions are **explicit and predictable**
- Every action has a **single, traceable effect** on state
- The reducer is a **pure function** — easy to test, easy to reason about
- Complex derived state (points, timer, highscore) is handled in **one place**

---

## Key Technical Decisions

### 1. Centralised State With `useReducer`

Eight action types. One reducer. Every state change in the app — from fetching data to ticking the timer — is a dispatched action. The entire quiz lifecycle is traceable from a single function.

```javascript
function reducer(state, action) {
  switch (action.type) {
    case "dataReceived":  // API returned questions → move to ready
    case "dataFailed":    // API failed → show error state
    case "start":         // user clicked start → begin timer + quiz
    case "newAnswer":     // user selected option → score instantly
    case "nextQuestion":  // advance index → clear answer
    case "finish":        // last question done → save highscore
    case "restart":       // reset everything → keep questions cached
    case "tick":          // every second → decrement + auto-finish at 0
  }
}
```

### 2. Difficulty-Weighted Scoring

Questions carry different point values (10, 20, or 30) stored directly in the JSON data. The reducer evaluates correctness and awards points atomically — no separate scoring logic, no derived calculations in components:

```javascript
case "newAnswer":
  const question = state.questions.at(state.index);
  return {
    ...state,
    answer: action.payload,
    points:
      action.payload === question.correctOption
        ? state.points + question.points  // award question's specific points
        : state.points,                   // wrong answer — no change
  };
```

### 3. Timer With Cleanup Function

The `Timer` component uses `useEffect` with `setInterval` — and critically, **cleans up after itself** to prevent memory leaks:

```javascript
useEffect(function () {
  const id = setInterval(function () {
    dispatch({ type: "tick" });
  }, 1000);

  return () => clearInterval(id);  // cleanup — no memory leaks
}, [dispatch]);
```

The timer dispatches a `tick` action every second. The reducer handles the consequence — including **auto-finishing the quiz** when `secondsRemaining` hits zero. The timer component itself knows nothing about that logic.

### 4. Time Budget Per Quiz

Each quiz session gets `30 seconds × number of questions` — so a 15-question quiz gets 7.5 minutes total. Calculated once at the moment the quiz starts:

```javascript
case "start":
  return {
    ...state,
    status: "active",
    secondsRemaining: state.questions.length * SECS_PER_QUESTION,
  };
```

### 5. Smart Conditional Rendering

The entire app renders from a single status string. No nested ternaries, no boolean spaghetti:

```jsx
{status === "loading"  && <Loader />}
{status === "error"    && <Error />}
{status === "ready"    && <StartScreen />}
{status === "active"   && <Question />}
{status === "finished" && <FinishScreen />}
```

### 6. Dynamic Emoji Feedback

The `FinishScreen` maps score percentage to an emoji — a small detail that makes the UX feel alive and personal:

```javascript
if (percentage === 100) emoji = "🥇";  // perfect score
if (percentage >= 80)   emoji = "🎉";  // great score
if (percentage >= 50)   emoji = "😊";  // decent score
if (percentage >= 0)    emoji = "🤔";  // needs more work
if (percentage === 0)   emoji = "🤦🏻‍♂️"; // back to basics
```

### 7. Self-Contained Button Logic

`NextButton` handles three outcomes internally — hidden before answering, Next mid-quiz, Finish on the last question. The parent passes state down and the child decides what to render:

```javascript
if (answer === null)             return null;
if (index < numQuestions - 1)   return <button>Next</button>;
if (index === numQuestions - 1) return <button>Finish</button>;
```

---

## Component Structure

```
App.jsx                  ← state machine, data fetching, orchestration
├── Header.jsx           ← branding and logo
├── Main.jsx             ← layout wrapper using children pattern
│   ├── Loader.jsx       ← loading state UI
│   ├── Error.jsx        ← error state UI
│   ├── StartScreen.jsx  ← welcome screen + question count + start trigger
│   ├── Progress.jsx     ← visual progress bar + live score tracker
│   ├── Question.jsx     ← question text display
│   │   └── Options.jsx  ← answer buttons + correct/wrong highlighting
│   ├── Footer.jsx       ← layout wrapper using children pattern
│   │   ├── Timer.jsx    ← countdown display + tick dispatch + cleanup
│   │   └── NextButton   ← conditional next / finish button
│   └── FinishScreen.jsx ← results, percentage, emoji, highscore, restart
```

Every component has **one job.** No component does more than it needs to.

---

## Project Structure

```
react-quiz-app/
├── public/
│   └── logo512.png
├── src/
│   ├── components/
│   │   ├── Error.jsx
│   │   ├── FinishScreen.jsx
│   │   ├── Footer.jsx
│   │   ├── Header.jsx
│   │   ├── Loader.jsx
│   │   ├── Main.jsx
│   │   ├── NextButton.jsx
│   │   ├── Options.jsx
│   │   ├── Progress.jsx
│   │   ├── Question.jsx
│   │   ├── StartScreen.jsx
│   │   └── Timer.jsx
│   ├── App.jsx
│   └── index.js
├── data/
│   └── questions.json    ← 15 React questions served via JSON Server
├── package.json
└── README.md
```

---

## Tech Stack

| Technology | Purpose |
|---|---|
| React 18 | UI library |
| useReducer | Centralised state machine |
| useEffect | Data fetching + timer with cleanup |
| JSON Server | Mock REST API for quiz questions |
| CSS3 | Styling and layout |

---

## Getting Started

### Prerequisites
- Node.js v16+
- npm or yarn

### Installation

```bash
# Clone the repo
git clone https://github.com/yourusername/react-quiz-app.git

# Navigate into the project
cd react-quiz-app

# Install dependencies
npm install

# Install JSON Server globally if you don't have it
npm install -g json-server
```

### Running The App

You need **two terminals** running simultaneously.

```bash
# Terminal 1 — start the questions API on port 8000
npm run server

# Terminal 2 — start the React app on port 3000
npm start
```

Then open [http://localhost:3000](http://localhost:3000)

### Add This To Your package.json

```json
"scripts": {
  "start": "react-scripts start",
  "server": "json-server --watch data/questions.json --port 8000"
}
```

---

## Scoring System

| Score | Percentage | Result |
|---|---|---|
| 320 / 320 | 100% | 🥇 Perfect |
| 256 – 319 | 80–99% | 🎉 Excellent |
| 160 – 255 | 50–79% | 😊 Good |
| 1 – 159 | 1–49% | 🤔 Keep studying |
| 0 / 320 | 0% | 🤦🏻‍♂️ Start over |

---

## 💡 What I Learned Building This

- How to model application behaviour as a **finite state machine** using `useReducer`
- Why centralising state in a reducer makes complex apps **easier to reason about** than scattered `useState` calls
- How `useEffect` **cleanup functions** prevent memory leaks from running intervals
- How to architect components around **single responsibility** — each component does one thing well
- How to keep **timer logic decoupled** from the component that displays it
- How to think about **data flow** — dispatching actions rather than mutating state directly

---

## Future Improvements

- [ ] Persist highscore to `localStorage` across browser sessions
- [ ] Add question categories and difficulty filters
- [ ] Animate transitions between questions with Framer Motion
- [ ] Add a leaderboard with player name input
- [ ] Replace JSON Server with a real backend (Node.js + Express)
- [ ] Add a review screen showing correct answers after finishing

---

## Author

**Ludwig Sackey Narh** — Software Engineer

-  GitHub: [github.com/TheNarh](#)
-  LinkedIn: [https://www.linkedin.com/in/ludwig-sackey-narh-670078106/](#)
-  thenarh17@gmail.com

---

## License

MIT — feel free to use this as a reference, a learning tool, or a starting point for your own quiz app.
