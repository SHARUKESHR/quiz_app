# quiz_app
# Quiz.APP
```
import React, {
  useReducer,
  useRef,
  useEffect,
  useMemo,
  useCallback,
  useState,
} from "react";
import "./QuizApp.css";

function shuffleArray(array) {
  const arr = [...array];
  for (let i = arr.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [arr[i], arr[j]] = [arr[j], arr[i]];
  }
  return arr;
}

const initialState = {
  currentQuestion: 0,
  answers: [],
  quizOver: false,
  score: 0,
  startTime: null,
  endTime: null,
};

function quizReducer(state, action) {
  switch (action.type) {
    case "START_QUIZ":
      return { ...initialState, startTime: Date.now() };
    case "ANSWER_SELECTED":
      const { selectedIndex, correctIndex, total } = action.payload;
      const isCorrect = selectedIndex === correctIndex;
      const newScore = isCorrect ? state.score + 1 : state.score;
      const nextQ = state.currentQuestion + 1;
      return {
        ...state,
        score: newScore,
        answers: [...state.answers, selectedIndex],
        currentQuestion: nextQ,
        quizOver: nextQ >= total,
        endTime: nextQ >= total ? Date.now() : state.endTime,
      };
    case "RESET":
      return { ...initialState };
    default:
      return state;
  }
}

export default function QuizApp() {
  const [state, dispatch] = useReducer(quizReducer, initialState);
  const [quizQuestions, setQuizQuestions] = useState([]);
  const [quizStarted, setQuizStarted] = useState(false);
  const [answerFeedback, setAnswerFeedback] = useState(null);
  const [elapsedTime, setElapsedTime] = useState(0);
  const timerRef = useRef(null);

  // Fetch Internet-related questions from Computers category
  const fetchQuestions = useCallback(async () => {
    try {
      const resp = await fetch(
        "https://opentdb.com/api.php?amount=20&category=18&type=multiple"
      );
      const data = await resp.json();
      if (data.response_code === 0) {
        const formatted = data.results
          .filter((q) =>
            q.question.toLowerCase().match(/internet|web|http|html|browser|network|url|dns|tcp|ip|email/)
          )
          .map((q) => {
            const options = shuffleArray([...q.incorrect_answers, q.correct_answer]);
            const correct = options.indexOf(q.correct_answer);
            return { question: q.question, options, correct };
          });

        // If too few Internet-specific, fill remaining with general computer ones
        const fullList =
          formatted.length < 10
            ? data.results.map((q) => {
                const opts = shuffleArray([...q.incorrect_answers, q.correct_answer]);
                const correct = opts.indexOf(q.correct_answer);
                return { question: q.question, options: opts, correct };
              })
            : formatted;

        // Make it progressively harder by question length
        const sorted = [...fullList].sort(
          (a, b) => a.question.length - b.question.length
        );
        setQuizQuestions(sorted.slice(0, 20));
      }
    } catch (e) {
      console.error("Error fetching questions:", e);
    }
  }, []);

  // Timer
  useEffect(() => {
    if (state.startTime && !state.quizOver) {
      timerRef.current = setInterval(() => {
        setElapsedTime(Date.now() - state.startTime);
      }, 1000);
    }
    if (state.quizOver) clearInterval(timerRef.current);
    return () => clearInterval(timerRef.current);
  }, [state.startTime, state.quizOver]);

  const handleStartQuiz = useCallback(async () => {
    await fetchQuestions();
    dispatch({ type: "START_QUIZ" });
    setQuizStarted(true);
    setElapsedTime(0);
  }, [fetchQuestions]);

  const handleAnswerSelect = useCallback(
    (idx) => {
      const current = quizQuestions[state.currentQuestion];
      const correctIdx = current.correct;
      const isCorr = idx === correctIdx;
      setAnswerFeedback({ idx, isCorr });

      setTimeout(() => {
        dispatch({
          type: "ANSWER_SELECTED",
          payload: {
            selectedIndex: idx,
            correctIndex: correctIdx,
            total: quizQuestions.length,
          },
        });
        setAnswerFeedback(null);
      }, 1200);
    },
    [quizQuestions, state.currentQuestion]
  );

  const handleReset = useCallback(async () => {
    await fetchQuestions();
    dispatch({ type: "RESET" });
    setQuizStarted(false);
    setElapsedTime(0);
  }, [fetchQuestions]);

  const seconds = Math.floor(elapsedTime / 1000);
  const minutes = Math.floor(seconds / 60);
  const displayTime = `${minutes}:${seconds % 60 < 10 ? "0" : ""}${seconds % 60}`;

  return (
    <div className="quiz-container easy-bg">
      <div className="quiz-card">
        <h1 className="quiz-title">🌐 Internet Quiz Challenge</h1>

        {!quizStarted ? (
          <button className="start-btn" onClick={handleStartQuiz}>
            Start Internet Quiz
          </button>
        ) : !state.quizOver && quizQuestions.length > 0 ? (
          <>
            <div className="status-bar">
              <p>⏱ {displayTime}</p>
              <p>⭐ {state.score}</p>
              <p>
                Q {state.currentQuestion + 1}/{quizQuestions.length}
              </p>
            </div>

            <h2
              className="question"
              dangerouslySetInnerHTML={{
                __html: quizQuestions[state.currentQuestion].question,
              }}
            />
            <div className="options">
              {quizQuestions[state.currentQuestion].options.map((opt, idx) => {
                const correctIdx = quizQuestions[state.currentQuestion].correct;
                let cls = "option-btn";
                if (answerFeedback) {
                  if (idx === correctIdx) cls += " correct";
                  else if (idx === answerFeedback.idx && !answerFeedback.isCorr)
                    cls += " wrong";
                }
                return (
                  <button
                    key={idx}
                    className={cls}
                    onClick={() => handleAnswerSelect(idx)}
                    disabled={!!answerFeedback}
                    dangerouslySetInnerHTML={{ __html: opt }}
                  />
                );
              })}
            </div>
          </>
        ) : state.quizOver ? (
          <div className="result">
            <h2>🎉 Quiz Finished!</h2>
            <p>
              Score: {state.score}/{quizQuestions.length}
            </p>
            <p>Total Time: {displayTime}</p>
            <button className="reset-btn" onClick={handleReset}>
              Try Again
            </button>
          </div>
        ) : (
          <p>Loading internet questions...</p>
        )}
      </div>
    </div>
  );
}

```
# App.css
```
body {
  margin: 0;
  font-family: "Poppins", sans-serif;
  height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
}

.easy-bg {
  background: linear-gradient(135deg, #81c784, #4caf50);
}

.medium-bg {
  background: linear-gradient(135deg, #ffb74d, #fb8c00);
}

.hard-bg {
  background: linear-gradient(135deg, #e57373, #f44336);
}

.quiz-container {
  display: flex;
  justify-content: center;
  align-items: center;
  width: 100%;
}

.quiz-card {
  background: rgb(222, 233, 163);
  padding: 30px;
  border-radius: 15px;
  width: 90%;
  max-width: 600px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.2);
  text-align: center;
}

.quiz-title {
  margin-bottom: 20px;
  color: #6a1b9a;
}

.status-bar {
  display: flex;
  justify-content: space-between;
  margin-bottom: 15px;
  font-weight: bold;
}

.question {
  font-size: 18px;
  margin-bottom: 20px;
}

.options {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.option-btn {
  padding: 10px;
  border-radius: 8px;
  border: 2px solid #ccc;
  cursor: pointer;
  background: #f9f9f9;
  transition: 0.3s;
}

.option-btn:hover {
  background: #e04c2a;
}

.option-btn.correct {
  background-color: #4caf50;
  color: rgb(224, 174, 174);
  border: 2px solid #388e3c;
}

.option-btn.wrong {
  background-color: #f44336;
  color: white;
  border: 2px solid #d32f2f;
}

.start-screen {
  display: flex;
  flex-direction: column;
  gap: 15px;
}

.difficulty-select {
  padding: 10px;
  border-radius: 8px;
  border: 2px solid #9c27b0;
  font-size: 16px;
  cursor: pointer;
}

.start-btn,
.reset-btn {
  background-color: #9c27b0;
  color: white;
  padding: 12px 20px;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-size: 16px;
  transition: 0.3s;
}

.start-btn:hover,
.reset-btn:hover {
  background-color: #7b1fa2;
}

.result h2 {
  color: #4caf50;
}

```
# output
<img width="1727" height="847" alt="image" src="https://github.com/user-attachments/assets/310d676e-78bb-4b93-8999-73c48c212ca0" />

  
