# bedazzling-brains
Free study app for students
import { useState, useRef, useEffect } from "react";

const SUBJECTS = ["Math", "Science", "History", "English", "Geography", "Biology", "Chemistry", "Physics"];

const MODES = [
  { id: "chat", label: "💬 Ask Anything", desc: "Get instant answers & explanations" },
  { id: "quiz", label: "🎯 Quiz Me", desc: "Test your knowledge with AI questions" },
  { id: "summarize", label: "📝 Summarize", desc: "Paste notes, get a clean summary" },
  { id: "flashcard", label: "🃏 Flashcards", desc: "Auto-generate flashcards from topics" },
];

const COLORS = {
  bg: "#0D0D1A",
  card: "#13132A",
  border: "#2A2A4A",
  accent: "#7C3AED",
  accentLight: "#A78BFA",
  accentGlow: "rgba(124,58,237,0.25)",
  green: "#10B981",
  pink: "#EC4899",
  text: "#E2E8F0",
  muted: "#64748B",
  inputBg: "#1E1E3A",
};

async function callClaude(messages, systemPrompt) {
  const response = await fetch("https://api.anthropic.com/v1/messages", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      model: "claude-sonnet-4-6",
      max_tokens: 1000,
      system: systemPrompt,
      messages,
    }),
  });
  const data = await response.json();
  return data.content?.[0]?.text || "Sorry, I couldn't get a response.";
}

function TypingDots() {
  return (
    <div style={{ display: "flex", gap: 5, alignItems: "center", padding: "12px 16px" }}>
      {[0, 1, 2].map((i) => (
        <div
          key={i}
          style={{
            width: 8, height: 8, borderRadius: "50%",
            background: COLORS.accentLight,
            animation: `bounce 1.2s ease-in-out ${i * 0.2}s infinite`,
          }}
        />
      ))}
      <style>{`@keyframes bounce { 0%,80%,100%{transform:translateY(0)} 40%{transform:translateY(-8px)} }`}</style>
    </div>
  );
}

function FlashCard({ front, back }) {
  const [flipped, setFlipped] = useState(false);
  return (
    <div
      onClick={() => setFlipped(!flipped)}
      style={{
        cursor: "pointer",
        perspective: 1000,
        height: 160,
        marginBottom: 12,
      }}
    >
      <div style={{
        position: "relative", width: "100%", height: "100%",
        transformStyle: "preserve-3d",
        transition: "transform 0.5s",
        transform: flipped ? "rotateY(180deg)" : "rotateY(0deg)",
      }}>
        {[{ content: front, label: "QUESTION", color: COLORS.accent }, { content: back, label: "ANSWER", color: COLORS.green }].map((side, idx) => (
          <div key={idx} style={{
            position: "absolute", inset: 0,
            backfaceVisibility: "hidden",
            background: COLORS.card,
            border: `1px solid ${side.color}55`,
            borderRadius: 14,
            padding: 20,
            display: "flex", flexDirection: "column", justifyContent: "center",
            transform: idx === 1 ? "rotateY(180deg)" : "none",
            boxShadow: `0 0 20px ${side.color}22`,
          }}>
            <div style={{ fontSize: 10, fontWeight: 700, color: side.color, letterSpacing: 2, marginBottom: 10 }}>{side.label}</div>
            <div style={{ color: COLORS.text, fontSize: 15, lineHeight: 1.5 }}>{side.content}</div>
            <div style={{ fontSize: 11, color: COLORS.muted, marginTop: 10 }}>Tap to flip</div>
          </div>
        ))}
      </div>
    </div>
  );
}

export default function StudyBuddy() {
  const [mode, setMode] = useState("chat");
  const [subject, setSubject] = useState("Math");
  const [messages, setMessages] = useState([]);
  const [input, setInput] = useState("");
  const [loading, setLoading] = useState(false);
  const [flashcards, setFlashcards] = useState([]);
  const [quizState, setQuizState] = useState(null);
  const [selectedAnswer, setSelectedAnswer] = useState(null);
  const [score, setScore] = useState({ correct: 0, total: 0 });
  const bottomRef = useRef(null);

  useEffect(() => {
    bottomRef.current?.scrollIntoView({ behavior: "smooth" });
  }, [messages, loading]);

  useEffect(() => {
    setMessages([]);
    setFlashcards([]);
    setQuizState(null);
    setSelectedAnswer(null);
    setInput("");
  }, [mode, subject]);

  const systemPrompts = {
    chat: `You are an enthusiastic AI study buddy for teenagers studying ${subject}. Explain concepts clearly, use relatable examples, emoji occasionally, and keep it engaging. Be encouraging!`,
    quiz: `You are a quiz master for ${subject}. Generate multiple choice questions with exactly 4 options labeled A, B, C, D. Format your response as JSON: {"question":"...","options":["A) ...","B) ...","C) ...","D) ..."],"answer":"A","explanation":"..."}. Only output valid JSON.`,
    summarize: `You are a note summarizer for ${subject} students. Take messy notes and return a clean, structured summary with key points, important terms, and a quick review checklist. Use markdown formatting.`,
    flashcard: `You are a flashcard creator for ${subject}. Given a topic, generate 5 flashcards. Return ONLY valid JSON array: [{"front":"question","back":"answer"},...]`,
  };

  async function handleSend() {
    if (!input.trim() || loading) return;
    const userText = input.trim();
    setInput("");
    setLoading(true);

    if (mode === "quiz") {
      try {
        const reply = await callClaude([{ role: "user", content: `Generate a ${subject} quiz question about: ${userText}` }], systemPrompts.quiz);
        const clean = reply.replace(/```json|```/g, "").trim();
        const parsed = JSON.parse(clean);
        setQuizState(parsed);
        setSelectedAnswer(null);
      } catch {
        setQuizState({ error: "Couldn't generate quiz. Try a different topic!" });
      }
    } else if (mode === "flashcard") {
      try {
        const reply = await callClaude([{ role: "user", content: `Create flashcards for: ${userText}` }], systemPrompts.flashcard);
        const clean = reply.replace(/```json|```/g, "").trim();
        const cards = JSON.parse(clean);
        setFlashcards(cards);
      } catch {
        setFlashcards([{ front: "Error", back: "Couldn't generate flashcards. Try again!" }]);
      }
    } else {
      const newMessages = [...messages, { role: "user", content: userText }];
      setMessages(newMessages);
      const reply = await callClaude(newMessages, systemPrompts[mode]);
      setMessages([...newMessages, { role: "assistant", content: reply }]);
    }
    setLoading(false);
  }

  function handleAnswerSelect(option) {
    if (selectedAnswer) return;
    const letter = option[0];
    setSelectedAnswer(letter);
    setScore(prev => ({
      correct: prev.correct + (letter === quizState.answer ? 1 : 0),
      total: prev.total + 1,
    }));
  }

  const placeholders = {
    chat: `Ask anything about ${subject}...`,
    quiz: `Enter a ${subject} topic to get quizzed...`,
    summarize: `Paste your ${subject} notes here...`,
    flashcard: `Enter a ${subject} topic for flashcards...`,
  };

  return (
    <div style={{
      minHeight: "100vh",
      background: COLORS.bg,
      fontFamily: "'Inter', 'Segoe UI', sans-serif",
      color: COLORS.text,
      display: "flex",
      flexDirection: "column",
    }}>
      {/* Header */}
      <div style={{
        padding: "20px 24px 0",
        background: `linear-gradient(180deg, ${COLORS.accentGlow} 0%, transparent 100%)`,
        borderBottom: `1px solid ${COLORS.border}`,
      }}>
        <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", marginBottom: 16 }}>
          <div>
            <div style={{ fontSize: 22, fontWeight: 800, letterSpacing: -0.5 }}>
              <span style={{ color: COLORS.accentLight }}>Study</span>
              <span style={{ color: COLORS.text }}>Buddy</span>
              <span style={{ fontSize: 20, marginLeft: 6 }}>✨</span>
            </div>
            <div style={{ fontSize: 12, color: COLORS.muted, marginTop: 2 }}>AI-powered · Always ready</div>
          </div>

          {/* Subject Picker */}
          <select
            value={subject}
            onChange={e => setSubject(e.target.value)}
            style={{
              background: COLORS.inputBg,
              border: `1px solid ${COLORS.border}`,
              color: COLORS.text,
              borderRadius: 10,
              padding: "8px 12px",
              fontSize: 13,
              fontWeight: 600,
              cursor: "pointer",
              outline: "none",
            }}
          >
            {SUBJECTS.map(s => <option key={s} value={s}>{s}</option>)}
          </select>
        </div>

        {/* Mode Tabs */}
        <div style={{ display: "flex", gap: 8, overflowX: "auto", paddingBottom: 16 }}>
          {MODES.map(m => (
            <button
              key={m.id}
              onClick={() => setMode(m.id)}
              style={{
                flexShrink: 0,
                padding: "8px 14px",
                borderRadius: 20,
                border: "none",
                cursor: "pointer",
                fontWeight: 600,
                fontSize: 12,
                transition: "all 0.2s",
                background: mode === m.id ? COLORS.accent : COLORS.card,
                color: mode === m.id ? "#fff" : COLORS.muted,
                boxShadow: mode === m.id ? `0 0 16px ${COLORS.accentGlow}` : "none",
              }}
            >
              {m.label}
            </button>
          ))}
        </div>
      </div>

      {/* Content Area */}
      <div style={{ flex: 1, overflowY: "auto", padding: "20px 24px" }}>

        {/* Welcome State */}
        {mode === "chat" && messages.length === 0 && (
          <div style={{ textAlign: "center", paddingTop: 40 }}>
            <div style={{ fontSize: 48, marginBottom: 12 }}>🧠</div>
            <div style={{ fontSize: 18, fontWeight: 700, marginBottom: 8 }}>What do you want to learn?</div>
            <div style={{ color: COLORS.muted, fontSize: 14, marginBottom: 28 }}>Ask me anything about {subject}</div>
            <div style={{ display: "flex", flexWrap: "wrap", gap: 8, justifyContent: "center" }}>
              {[`Explain quadratic equations`, `What is Newton's 3rd law?`, `Summarize World War 2`, `Help me understand photosynthesis`].map(q => (
                <button
                  key={q}
                  onClick={() => { setInput(q); }}
                  style={{
                    background: COLORS.card,
                    border: `1px solid ${COLORS.border}`,
                    color: COLORS.accentLight,
                    borderRadius: 20,
                    padding: "8px 14px",
                    fontSize: 12,
                    cursor: "pointer",
                    fontWeight: 500,
                  }}
                >{q}</button>
              ))}
            </div>
          </div>
        )}

        {/* Chat Messages */}
        {(mode === "chat" || mode === "summarize") && messages.map((m, i) => (
          <div key={i} style={{
            display: "flex",
            justifyContent: m.role === "user" ? "flex-end" : "flex-start",
            marginBottom: 14,
          }}>
            {m.role === "assistant" && (
              <div style={{ width: 30, height: 30, borderRadius: "50%", background: COLORS.accent, display: "flex", alignItems: "center", justifyContent: "center", fontSize: 14, marginRight: 10, flexShrink: 0, marginTop: 4 }}>✨</div>
            )}
            <div style={{
              maxWidth: "75%",
              padding: "12px 16px",
              borderRadius: m.role === "user" ? "18px 18px 4px 18px" : "4px 18px 18px 18px",
              background: m.role === "user" ? COLORS.accent : COLORS.card,
              border: m.role === "assistant" ? `1px solid ${COLORS.border}` : "none",
              fontSize: 14,
              lineHeight: 1.6,
              color: COLORS.text,
              whiteSpace: "pre-wrap",
            }}>
              {m.content}
            </div>
          </div>
        ))}

        {loading && (mode === "chat" || mode === "summarize") && (
          <div style={{ display: "flex", alignItems: "center" }}>
            <div style={{ width: 30, height: 30, borderRadius: "50%", background: COLORS.accent, display: "flex", alignItems: "center", justifyContent: "center", fontSize: 14, marginRight: 10, flexShrink: 0 }}>✨</div>
            <div style={{ background: COLORS.card, border: `1px solid ${COLORS.border}`, borderRadius: "4px 18px 18px 18px" }}>
              <TypingDots />
            </div>
          </div>
        )}

        {/* Quiz Mode */}
        {mode === "quiz" && !quizState && !loading && (
          <div style={{ textAlign: "center", paddingTop: 40 }}>
            <div style={{ fontSize: 48, marginBottom: 12 }}>🎯</div>
            <div style={{ fontSize: 18, fontWeight: 700, marginBottom: 8 }}>Quiz Mode</div>
            <div style={{ color: COLORS.muted, fontSize: 14 }}>Enter any {subject} topic below and I'll quiz you!</div>
            {score.total > 0 && (
              <div style={{ marginTop: 20, padding: "14px 20px", background: COLORS.card, borderRadius: 14, border: `1px solid ${COLORS.border}`, display: "inline-block" }}>
                <div style={{ fontSize: 24, fontWeight: 800, color: COLORS.accentLight }}>{score.correct}/{score.total}</div>
                <div style={{ fontSize: 12, color: COLORS.muted }}>Score this session</div>
              </div>
            )}
          </div>
        )}

        {mode === "quiz" && loading && (
          <div style={{ textAlign: "center", paddingTop: 60 }}>
            <div style={{ fontSize: 32, marginBottom: 16 }}>🤔</div>
            <div style={{ color: COLORS.muted }}>Generating your question...</div>
          </div>
        )}

        {mode === "quiz" && quizState && !quizState.error && (
          <div style={{ maxWidth: 520, margin: "0 auto" }}>
            {score.total > 0 && (
              <div style={{ textAlign: "right", marginBottom: 12 }}>
                <span style={{ fontSize: 13, color: COLORS.muted }}>Score: </span>
                <span style={{ fontWeight: 700, color: COLORS.accentLight }}>{score.correct}/{score.total}</span>
              </div>
            )}
            <div style={{ background: COLORS.card, border: `1px solid ${COLORS.border}`, borderRadius: 16, padding: 24, marginBottom: 16 }}>
              <div style={{ fontSize: 12, color: COLORS.accentLight, fontWeight: 700, letterSpacing: 1, marginBottom: 12 }}>QUESTION</div>
              <div style={{ fontSize: 16, fontWeight: 600, lineHeight: 1.5 }}>{quizState.question}</div>
            </div>
            <div style={{ display: "flex", flexDirection: "column", gap: 10 }}>
              {quizState.options?.map((opt, i) => {
                const letter = opt[0];
                const isSelected = selectedAnswer === letter;
                const isCorrect = letter === quizState.answer;
                const showResult = !!selectedAnswer;
                let bg = COLORS.card, border = COLORS.border, color = COLORS.text;
                if (showResult && isCorrect) { bg = "#065F46"; border = COLORS.green; }
                else if (showResult && isSelected && !isCorrect) { bg = "#7F1D1D"; border = "#EF4444"; }
                return (
                  <button key={i} onClick={() => handleAnswerSelect(opt)}
                    style={{ background: bg, border: `1px solid ${border}`, color, borderRadius: 12, padding: "14px 18px", textAlign: "left", cursor: selectedAnswer ? "default" : "pointer", fontSize: 14, fontWeight: 500, transition: "all 0.2s" }}>
                    {opt}
                    {showResult && isCorrect && <span style={{ float: "right" }}>✅</span>}
                    {showResult && isSelected && !isCorrect && <span style={{ float: "right" }}>❌</span>}
                  </button>
                );
              })}
            </div>
            {selectedAnswer && (
              <div style={{ marginTop: 16, padding: 16, background: "#1E3A5F", border: "1px solid #3B82F6", borderRadius: 12 }}>
                <div style={{ fontSize: 12, fontWeight: 700, color: "#60A5FA", marginBottom: 6 }}>💡 EXPLANATION</div>
                <div style={{ fontSize: 14, color: COLORS.text, lineHeight: 1.5 }}>{quizState.explanation}</div>
              </div>
            )}
            {selectedAnswer && (
              <button onClick={() => { setQuizState(null); setSelectedAnswer(null); }}
                style={{ marginTop: 16, width: "100%", padding: "14px", background: COLORS.accent, border: "none", color: "#fff", borderRadius: 12, fontWeight: 700, fontSize: 15, cursor: "pointer" }}>
                Next Question →
              </button>
            )}
          </div>
        )}

        {/* Flashcard Mode */}
        {mode === "flashcard" && flashcards.length === 0 && !loading && (
          <div style={{ textAlign: "center", paddingTop: 40 }}>
            <div style={{ fontSize: 48, marginBottom: 12 }}>🃏</div>
            <div style={{ fontSize: 18, fontWeight: 700, marginBottom: 8 }}>Flashcard Generator</div>
            <div style={{ color: COLORS.muted, fontSize: 14 }}>Enter any {subject} topic and I'll create 5 flashcards for you!</div>
          </div>
        )}

        {mode === "flashcard" && loading && (
          <div style={{ textAlign: "center", paddingTop: 60 }}>
            <div style={{ fontSize: 32, marginBottom: 16 }}>✨</div>
            <div style={{ color: COLORS.muted }}>Creating your flashcards...</div>
          </div>
        )}

        {mode === "flashcard" && flashcards.length > 0 && (
          <div>
            <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 16 }}>
              <div style={{ fontWeight: 700 }}>📚 {flashcards.length} Flashcards</div>
              <button onClick={() => setFlashcards([])} style={{ background: "none", border: `1px solid ${COLORS.border}`, color: COLORS.muted, borderRadius: 8, padding: "6px 12px", fontSize: 12, cursor: "pointer" }}>New Topic</button>
            </div>
            {flashcards.map((card, i) => <FlashCard key={i} front={card.front} back={card.back} />)}
          </div>
        )}

        <div ref={bottomRef} />
      </div>

      {/* Input Area */}
      <div style={{
        padding: "16px 24px",
        borderTop: `1px solid ${COLORS.border}`,
        background: COLORS.card,
      }}>
        <div style={{ display: "flex", gap: 10, alignItems: "flex-end" }}>
          <textarea
            value={input}
            onChange={e => setInput(e.target.value)}
            onKeyDown={e => { if (e.key === "Enter" && !e.shiftKey) { e.preventDefault(); handleSend(); } }}
            placeholder={placeholders[mode]}
            rows={mode === "summarize" ? 3 : 1}
            style={{
              flex: 1,
              background: COLORS.inputBg,
              border: `1px solid ${COLORS.border}`,
              borderRadius: 14,
              padding: "12px 16px",
              color: COLORS.text,
              fontSize: 14,
              resize: "none",
              outline: "none",
              fontFamily: "inherit",
              lineHeight: 1.5,
            }}
          />
          <button
            onClick={handleSend}
            disabled={loading || !input.trim()}
            style={{
              width: 46, height: 46,
              borderRadius: 14,
              border: "none",
              background: loading || !input.trim() ? COLORS.border : COLORS.accent,
              color: "#fff",
              fontSize: 20,
              cursor: loading || !input.trim() ? "not-allowed" : "pointer",
              display: "flex", alignItems: "center", justifyContent: "center",
              flexShrink: 0,
              transition: "all 0.2s",
              boxShadow: !loading && input.trim() ? `0 0 16px ${COLORS.accentGlow}` : "none",
            }}
          >
            {loading ? "⏳" : "→"}
          </button>
        </div>
        <div style={{ textAlign: "center", fontSize: 11, color: COLORS.muted, marginTop: 8 }}>
          Press Enter to send · Shift+Enter for new line
        </div>
      </div>
    </div>
  );
}
