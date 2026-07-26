import { useState, useRef, useCallback } from "react";

const SEVERITY = {
  critical: { color: "#E2685C", label: "Critical", dot: "bg-[#E2685C]" },
  warning: { color: "#E8B04B", label: "Warning", dot: "bg-[#E8B04B]" },
  suggestion: { color: "#6C8EEF", label: "Suggestion", dot: "bg-[#6C8EEF]" },
  good: { color: "#59B399", label: "Nice", dot: "bg-[#59B399]" },
};

const SAMPLE = `function getUser(id) {
  var data = fetch('/api/users/' + id)
  return data.json()
}

function processUsers(users) {
  let result = []
  for (var i = 0; i < users.length; i++) {
    if (users[i].active == true) {
      result.push(users[i])
    }
  }
  return result
}`;

function mockReview(code) {
  const lines = code.split("\n");
  const findings = [];
  lines.forEach((line, i) => {
    const n = i + 1;
    if (/\bvar\b/.test(line)) {
      findings.push({
        line: n,
        severity: "warning",
        category: "Style",
        message: "var creates function-scoped bindings that can leak outside blocks.",
        suggestion: "Use let or const instead.",
      });
    }
    if (/==(?!=)/.test(line)) {
      findings.push({
        line: n,
        severity: "critical",
        category: "Correctness",
        message: "Loose equality (==) coerces types before comparing, which can hide bugs.",
        suggestion: "Use === for predictable comparisons.",
      });
    }
    if (/fetch\(/.test(line) && !/await|\.then/.test(code)) {
      findings.push({
        line: n,
        severity: "critical",
        category: "Async",
        message: "fetch() returns a Promise but the result is used as if it resolved already.",
        suggestion: "Await the fetch call or chain .then() before using the response.",
      });
    }
    if (/for\s*\(/.test(line)) {
      findings.push({
        line: n,
        severity: "suggestion",
        category: "Readability",
        message: "A manual index loop is used to build a filtered list.",
        suggestion: "Array.prototype.filter() expresses this intent more directly.",
      });
    }
  });
  if (findings.length === 0) {
    findings.push({
      line: 1,
      severity: "good",
      category: "Overall",
      message: "No obvious issues spotted in this pass.",
      suggestion: "Consider adding tests to lock in this behavior.",
    });
  }
  const score = Math.max(35, 100 - findings.filter(f => f.severity !== "good").length * 12);
  return {
    score,
    summary: "Heuristic pass (offline fallback) — connect a live model for deeper analysis.",
    findings,
  };
}

async function reviewWithClaude(code) {
  const prompt = `You are a senior software engineer doing a code review. Review the following code and respond with ONLY a raw JSON object, no markdown fences, no preamble, in exactly this shape:
{"score": <0-100 integer>, "summary": "<one sentence overall assessment>", "findings": [{"line": <line number>, "severity": "critical"|"warning"|"suggestion"|"good", "category": "<short category like Correctness, Style, Performance, Security, Readability>", "message": "<what the issue is>", "suggestion": "<how to fix it>"}]}

Give 3-8 findings. If the code is genuinely clean, include one "good" finding instead of inventing problems.

Code to review:
\`\`\`
${code}
\`\`\``;

  const response = await fetch("https://api.anthropic.com/v1/messages", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      model: "claude-sonnet-4-6",
      max_tokens: 1000,
      messages: [{ role: "user", content: prompt }],
    }),
  });

  const data = await response.json();
  const text = data.content
    .map((block) => (block.type === "text" ? block.text : ""))
    .join("")
    .replace(/```json|```/g, "")
    .trim();

  const parsed = JSON.parse(text);
  if (!parsed.findings || !Array.isArray(parsed.findings)) {
    throw new Error("Malformed response");
  }
  return parsed;
}

export default function CodeReviewAssistant() {
  const [code, setCode] = useState(SAMPLE);
  const [loading, setLoading] = useState(false);
  const [result, setResult] = useState(null);
  const [usedFallback, setUsedFallback] = useState(false);
  const [activeLine, setActiveLine] = useState(null);
  const gutterRef = useRef(null);
  const textareaRef = useRef(null);

  const lines = code.split("\n");

  const syncScroll = useCallback(() => {
    if (gutterRef.current && textareaRef.current) {
      gutterRef.current.scrollTop = textareaRef.current.scrollTop;
    }
  }, []);

  const runReview = async () => {
    if (!code.trim()) return;
    setLoading(true);
    setResult(null);
    setUsedFallback(false);
    try {
      const parsed = await reviewWithClaude(code);
      setResult(parsed);
    } catch (e) {
      setResult(mockReview(code));
      setUsedFallback(true);
    } finally {
      setLoading(false);
    }
  };

  const findingsByLine = {};
  (result?.findings || []).forEach((f) => {
    findingsByLine[f.line] = findingsByLine[f.line] || [];
    findingsByLine[f.line].push(f);
  });

  return (
    <div className="min-h-screen bg-[#0F1115] text-[#E7E5E0] font-sans flex flex-col">
      {/* Header */}
      <div className="border-b border-white/10 px-6 py-4 flex items-center justify-between shrink-0">
        <div className="flex items-center gap-2">
          <span className="text-[#E8B04B] font-mono text-lg">▸</span>
          <span className="font-mono text-lg tracking-tight">
            review<span className="text-[#E8B04B]">.</span>ai
          </span>
        </div>
        <span className="text-xs text-white/40 font-mono hidden sm:block">
          AI-assisted code review
        </span>
      </div>

      {/* Main */}
      <div className="flex-1 grid grid-cols-1 lg:grid-cols-5 gap-px bg-white/10 overflow-hidden">
        {/* Editor panel */}
        <div className="lg:col-span-3 bg-[#0F1115] flex flex-col min-h-[320px]">
          <div className="px-4 py-2.5 border-b border-white/10 flex items-center justify-between">
            <span className="text-xs font-mono text-white/40 uppercase tracking-wider">
              Your code
            </span>
            <button
              onClick={runReview}
              disabled={loading || !code.trim()}
              className="text-xs font-mono px-3 py-1.5 rounded bg-[#E8B04B] text-[#0F1115] font-semibold disabled:opacity-40 disabled:cursor-not-allowed hover:bg-[#f0bd63] transition-colors"
            >
              {loading ? "Reviewing…" : "Review code"}
            </button>
          </div>
          <div className="flex flex-1 overflow-hidden">
            <div
              ref={gutterRef}
              className="select-none text-right pr-3 pl-4 py-3 text-white/25 font-mono text-sm leading-6 overflow-hidden bg-[#12151C] shrink-0 relative"
              style={{ minWidth: "3.5rem" }}
            >
              {lines.map((_, i) => {
                const n = i + 1;
                const hasFinding = findingsByLine[n];
                return (
                  <div key={n} className="relative h-6 flex items-center justify-end gap-1.5">
                    {hasFinding && (
                      <span
                        className={`w-1.5 h-1.5 rounded-full ${SEVERITY[hasFinding[0].severity].dot}`}
                      />
                    )}
                    <span>{n}</span>
                  </div>
                );
              })}
            </div>
            <textarea
              ref={textareaRef}
              value={code}
              onChange={(e) => setCode(e.target.value)}
              onScroll={syncScroll}
              spellCheck={false}
              className="flex-1 bg-[#0F1115] text-[#E7E5E0] font-mono text-sm leading-6 py-3 px-4 outline-none resize-none overflow-auto whitespace-pre"
              placeholder="Paste your code here…"
            />
          </div>
        </div>

        {/* Findings panel */}
        <div className="lg:col-span-2 bg-[#171A21] flex flex-col min-h-[320px]">
          <div className="px-4 py-2.5 border-b border-white/10 flex items-center justify-between">
            <span className="text-xs font-mono text-white/40 uppercase tracking-wider">
              Findings
            </span>
            {result && (
              <span
                className="text-xs font-mono px-2 py-0.5 rounded"
                style={{
                  color: result.score >= 70 ? "#59B399" : result.score >= 40 ? "#E8B04B" : "#E2685C",
                  backgroundColor:
                    result.score >= 70
                      ? "rgba(89,179,153,0.12)"
                      : result.score >= 40
                      ? "rgba(232,176,75,0.12)"
                      : "rgba(226,104,92,0.12)",
                }}
              >
                {result.score}/100
              </span>
            )}
          </div>

          <div className="flex-1 overflow-auto p-4 space-y-3">
            {!result && !loading && (
              <p className="text-sm text-white/40 leading-relaxed">
                Paste code on the left and press{" "}
                <span className="text-[#E8B04B] font-mono text-xs">Review code</span> to get
                line-by-line feedback.
              </p>
            )}
            {loading && (
              <p className="text-sm text-white/40 font-mono animate-pulse">
                Reading through your code…
              </p>
            )}
            {result && (
              <>
                <p className="text-sm text-white/70 leading-relaxed pb-1 border-b border-white/10">
                  {result.summary}
                  {usedFallback && (
                    <span className="block mt-1 text-xs text-[#E8B04B]/80">
                      (offline heuristic fallback — live model call didn't return)
                    </span>
                  )}
                </p>
                {result.findings
                  .slice()
                  .sort((a, b) => a.line - b.line)
                  .map((f, idx) => (
                    <div
                      key={idx}
                      onMouseEnter={() => setActiveLine(f.line)}
                      onMouseLeave={() => setActiveLine(null)}
                      className="rounded-md bg-[#0F1115] border-l-2 p-3 transition-colors"
                      style={{
                        borderColor: SEVERITY[f.severity].color,
                        backgroundColor: activeLine === f.line ? "#171A21" : "#0F1115",
                      }}
                    >
                      <div className="flex items-center gap-2 mb-1">
                        <span
                          className="text-[10px] font-mono font-semibold uppercase tracking-wider px-1.5 py-0.5 rounded"
                          style={{
                            color: SEVERITY[f.severity].color,
                            backgroundColor: SEVERITY[f.severity].color + "1F",
                          }}
                        >
                          {SEVERITY[f.severity].label}
                        </span>
                        <span className="text-[10px] font-mono text-white/30">
                          line {f.line} · {f.category}
                        </span>
                      </div>
                      <p className="text-sm text-white/85 leading-snug">{f.message}</p>
                      <p className="text-xs text-white/45 leading-snug mt-1">→ {f.suggestion}</p>
                    </div>
                  ))}
              </>
            )}
          </div>
        </div>
      </div>
    </div>
  );
}
