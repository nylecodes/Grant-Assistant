# Grant Application Assistant

---

## 📋 Project Contents

This project contains 1 documents:

1. **GrantDraftAssistant.jsx** _(Created: 7/30/2026)_

---

## 📄 GrantDraftAssistant.jsx

import { useState, useRef, useEffect } from "react";

const STEPS = ["Organization", "Grant Prompt", "Generate", "Results"];

const DEFAULT_ORG = {
  name: "",
  mission: "",
  programs: "",
  peopleServed: "",
  outcomes: "",
  budget: "",
  geography: "",
  yearsActive: "",
};

const ANALYSIS_SYSTEM = `You are an expert grant writing consultant who has helped hundreds of small US nonprofits win funding. You analyze grant prompts to identify every required section, question, and eligeline criterion the funder expects.

Respond ONLY with valid JSON (no markdown fences) in this exact format:
{
  "funderName": "string or 'Unknown'",
  "grantType": "string description",
  "requiredSections": ["list of every section/question the applicant must address"],
  "eligibilityCriteria": ["list of eligibility requirements mentioned"],
  "keyThemes": ["themes or priorities the funder emphasizes"],
  "wordOrPageLimits": "any limits mentioned, or 'None specified'",
  "deadlineInfo": "any deadline info, or 'None specified'"
}`;

const DRAFT_SYSTEM = `You are a senior grant writer specializing in helping small US nonprofits craft compelling, funded narratives. Write in a warm but professional tone. Use specific data when provided. Structure the narrative with clear headers matching what the funder asked for.

Rules:
- Lead each section with the strongest evidence
- Weave quantitative data into human stories
- Use active voice and concrete language
- Keep paragraphs short (3-5 sentences)
- Mirror the funder's language and priorities
- If you lack information for a section, write a [PLACEHOLDER: describe what's needed] marker

Write the full grant narrative draft now. Use markdown headers (##) for each section.`;

const AUDIT_SYSTEM = `You are a grant compliance reviewer. Given a grant analysis and a draft narrative, identify gaps and suggest improvements.

Respond ONLY with valid JSON (no markdown fences):
{
  "missingSections": [
    {"section": "name", "reason": "why it's needed", "priority": "critical|important|recommended"}
  ],
  "strengtheningSuggestions": [
    {"area": "name", "suggestion": "specific actionable suggestion", "dataPointNeeded": "what data to add"}
  ],
  "completenessScore": 0-100,
  "toneAssessment": "brief assessment of tone/voice fit",
  "topPriority": "the single most important thing to fix"
}`;

async function callClaude(messages, system, retries = 2) {
  for (let attempt = 0; attempt <= retries; attempt++) {
    try {
      const r = await fetch("https://api.anthropic.com/v1/messages", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          model: "claude-sonnet-4-6",
          max_tokens: 4000,
          system,
          messages,
        }),
      });
      const data = await r.json();
      if (!r.ok) {
        const msg = data?.error?.message || `API error (${r.status})`;
        if (r.status === 529 && attempt < retries) {
          await new Promise((res) => setTimeout(res, 2000 * (attempt + 1)));
          continue;
        }
        throw new Error(msg);
      }
      if (!data.content || !data.content.length) {
        throw new Error("Empty response from API");
      }
      return data;
    } catch (e) {
      if (attempt === retries) throw e;
      await new Promise((res) => setTimeout(res, 2000 * (attempt + 1)));
    }
  }
}

function parseJSON(text) {
  try {
    const clean = text.replace(/```json\s?|```/g, "").trim();
    return JSON.parse(clean);
  } catch {
    return null;
  }
}

// ── Tiny Markdown → HTML (headers, bold, italic, lists, placeholders) ──
function md(text) {
  if (!text) return "";
  return text
    .replace(/^### (.+)$/gm, '<h4 style="margin:1.1em 0 .4em;font-family:var(--ff-display);font-size:.95rem;color:var(--navy)">$1</h4>')
    .replace(/^## (.+)$/gm, '<h3 style="margin:1.3em 0 .5em;font-family:var(--ff-display);font-size:1.12rem;color:var(--navy);border-bottom:1px solid var(--border);padding-bottom:.3em">$1</h3>')
    .replace(/\*\*(.+?)\*\*/g, "<strong>$1</strong>")
    .replace(/\*(.+?)\*/g, "<em>$1</em>")
    .replace(/\[PLACEHOLDER:\s*(.+?)\]/g, '<span style="background:var(--gold-light);color:var(--gold-dark);padding:2px 8px;border-radius:4px;font-size:.85rem;font-weight:600">⚠ NEEDS: $1</span>')
    .replace(/^- (.+)$/gm, '<li style="margin:.25em 0;margin-left:1.2em">$1</li>')
    .replace(/\n{2,}/g, "</p><p>")
    .replace(/\n/g, "<br/>");
}

// ── Components ──

function Stepper({ step }) {
  return (
    <div style={styles.stepper}>
      {STEPS.map((label, i) => (
        <div key={label} style={{ display: "flex", alignItems: "center", gap: 0 }}>
          <div style={{ display: "flex", flexDirection: "column", alignItems: "center", gap: 4 }}>
            <div
              style={{
                width: 32, height: 32, borderRadius: "50%", display: "flex",
                alignItems: "center", justifyContent: "center", fontSize: ".8rem",
                fontWeight: 700, fontFamily: "var(--ff-body)",
                background: i <= step ? "var(--navy)" : "var(--bg-muted)",
                color: i <= step ? "#fff" : "var(--text-muted)",
                transition: "all .3s",
              }}
            >
              {i < step ? "✓" : i + 1}
            </div>
            <span style={{
              fontSize: ".72rem", fontWeight: i === step ? 700 : 500,
              color: i <= step ? "var(--navy)" : "var(--text-muted)",
              fontFamily: "var(--ff-body)", letterSpacing: ".02em",
            }}>{label}</span>
          </div>
          {i < STEPS.length - 1 && (
            <div style={{
              width: 48, height: 2, margin: "0 6px",
              background: i < step ? "var(--navy)" : "var(--border)",
              transition: "background .3s", marginBottom: 18,
            }} />
          )}
        </div>
      ))}
    </div>
  );
}

function Field({ label, sub, value, onChange, area, placeholder }) {
  const shared = {
    style: area ? styles.textarea : styles.input,
    value, placeholder,
    onChange: (e) => onChange(e.target.value),
  };
  return (
    <label style={styles.label}>
      <span style={styles.labelText}>{label}</span>
      {sub && <span style={styles.labelSub}>{sub}</span>}
      {area ? <textarea rows={area} {...shared} /> : <input {...shared} />}
    </label>
  );
}

function Badge({ priority }) {
  const colors = {
    critical: { bg: "#FDECEA", color: "#B71C1C" },
    important: { bg: "#FFF3E0", color: "#E65100" },
    recommended: { bg: "#E8F5E9", color: "#2E7D32" },
  };
  const c = colors[priority] || colors.recommended;
  return (
    <span style={{
      fontSize: ".68rem", fontWeight: 700, textTransform: "uppercase",
      padding: "2px 8px", borderRadius: 4, background: c.bg, color: c.color,
      letterSpacing: ".04em", fontFamily: "var(--ff-body)",
    }}>{priority}</span>
  );
}

function ScoreRing({ score }) {
  const r = 44, circ = 2 * Math.PI * r;
  const offset = circ - (score / 100) * circ;
  const color = score >= 75 ? "#2E7D32" : score >= 50 ? "#E65100" : "#B71C1C";
  return (
    <svg width="110" height="110" viewBox="0 0 110 110">
      <circle cx="55" cy="55" r={r} fill="none" stroke="var(--bg-muted)" strokeWidth="8" />
      <circle cx="55" cy="55" r={r} fill="none" stroke={color} strokeWidth="8"
        strokeDasharray={circ} strokeDashoffset={offset}
        strokeLinecap="round" transform="rotate(-90 55 55)"
        style={{ transition: "stroke-dashoffset 1s ease" }} />
      <text x="55" y="51" textAnchor="middle" fontSize="1.5rem" fontWeight="800"
        fontFamily="var(--ff-display)" fill={color}>{score}</text>
      <text x="55" y="68" textAnchor="middle" fontSize=".6rem" fontWeight="600"
        fontFamily="var(--ff-body)" fill="var(--text-muted)">COMPLETENESS</text>
    </svg>
  );
}

// ── Main App ──

export default function GrantDraft() {
  const [step, setStep] = useState(0);
  const [org, setOrg] = useState(DEFAULT_ORG);
  const [grantPrompt, setGrantPrompt] = useState("");
  const [loading, setLoading] = useState(false);
  const [loadMsg, setLoadMsg] = useState("");
  const [analysis, setAnalysis] = useState(null);
  const [draft, setDraft] = useState("");
  const [audit, setAudit] = useState(null);
  const [error, setError] = useState("");
  const [copied, setCopied] = useState(false);
  const resultRef = useRef(null);

  useEffect(() => {
    if (step === 3 && resultRef.current) {
      resultRef.current.scrollIntoView({ behavior: "smooth" });
    }
  }, [step]);

  const updateOrg = (key) => (val) => setOrg((o) => ({ ...o, [key]: val }));

  const canProceedStep0 = org.name && org.mission;
  const canProceedStep1 = grantPrompt.trim().length > 30;

  async function runChain() {
    setStep(2);
    setLoading(true);
    setError("");

    try {
      // ── Step 1: Analyze the grant prompt ──
      setLoadMsg("Reading the grant requirements…");
      const a1 = await callClaude(
        [{ role: "user", content: `Analyze this grant prompt and extract every requirement:\n\n${grantPrompt}` }],
        ANALYSIS_SYSTEM
      );
      const analysisText = a1.content?.map((c) => c.text || "").join("") || "";
      const parsedAnalysis = parseJSON(analysisText);
      if (!parsedAnalysis) throw new Error("Could not parse grant analysis. Please try again.");
      setAnalysis(parsedAnalysis);

      // ── Step 2: Generate the narrative draft ──
      setLoadMsg("Drafting your grant narrative…");
      const orgBlock = `ORGANIZATION PROFILE:
Name: ${org.name}
Mission: ${org.mission}
Programs: ${org.programs}
People Served: ${org.peopleServed}
Key Outcomes: ${org.outcomes}
Annual Budget: ${org.budget}
Geography: ${org.geography}
Years Active: ${org.yearsActive}`;

      const analysisBlock = `GRANT ANALYSIS:\n${JSON.stringify(parsedAnalysis, null, 2)}`;

      const a2 = await callClaude(
        [{ role: "user", content: `${orgBlock}\n\n${analysisBlock}\n\nGRANT PROMPT:\n${grantPrompt}\n\nWrite the complete grant narrative draft addressing every required section.` }],
        DRAFT_SYSTEM
      );
      const draftText = a2.content?.map((c) => c.text || "").join("") || "";
      if (!draftText) throw new Error("Draft generation failed. Please try again.");
      setDraft(draftText);

      // ── Step 3: Audit for gaps ──
      setLoadMsg("Reviewing for gaps and suggestions…");
      const a3 = await callClaude(
        [{ role: "user", content: `GRANT ANALYSIS:\n${JSON.stringify(parsedAnalysis, null, 2)}\n\nDRAFT NARRATIVE:\n${draftText}\n\nAudit this draft against the grant requirements. Identify missing sections, suggest data to strengthen it, and score completeness.` }],
        AUDIT_SYSTEM
      );
      const auditText = a3.content?.map((c) => c.text || "").join("") || "";
      const parsedAudit = parseJSON(auditText);
      if (!parsedAudit) throw new Error("Audit parsing failed, but your draft is ready.");
      setAudit(parsedAudit);

      setStep(3);
    } catch (e) {
      setError(e.message || "Something went wrong. Please try again.");
      setLoading(false);
      // If we got a draft but audit failed, show partial results
      if (draft) setStep(3);
      // Otherwise stay on step 2 so the retry button is visible
    } finally {
      setLoading(false);
    }
  }

  function reset() {
    setStep(0);
    setOrg(DEFAULT_ORG);
    setGrantPrompt("");
    setAnalysis(null);
    setDraft("");
    setAudit(null);
    setError("");
  }

  return (
    <div style={styles.root}>
      <style>{cssVars}</style>

      {/* Header */}
      <header style={styles.header}>
        <div style={{ display: "flex", alignItems: "center", gap: 10 }}>
          <div style={styles.logo}>GD</div>
          <div>
            <h1 style={styles.h1}>GrantDraft</h1>
            <p style={styles.tagline}>AI-powered grant narratives for small nonprofits</p>
          </div>
        </div>
        {step === 3 && (
          <button onClick={reset} style={styles.btnGhost}>+ New Draft</button>
        )}
      </header>

      <Stepper step={step} />

      <main style={styles.main}>
        {/* ── STEP 0: Org Profile ── */}
        {step === 0 && (
          <div style={styles.card}>
            <h2 style={styles.cardTitle}>Tell us about your organization</h2>
            <p style={styles.cardSub}>We'll use this to tailor every section of your narrative.</p>
            <div style={styles.grid2}>
              <Field label="Organization Name *" value={org.name} onChange={updateOrg("name")} placeholder="e.g. Bright Futures Youth Center" />
              <Field label="Years Active" value={org.yearsActive} onChange={updateOrg("yearsActive")} placeholder="e.g. 12" />
            </div>
            <Field label="Mission Statement *" sub="Your core purpose in 1–3 sentences" value={org.mission} onChange={updateOrg("mission")} area={3} placeholder="e.g. We empower underserved youth in East Cleveland through after-school STEM education, mentorship, and college readiness programs." />
            <Field label="Key Programs & Services" sub="What you do day-to-day" value={org.programs} onChange={updateOrg("programs")} area={3} placeholder="e.g. After-school STEM lab (M-F), Saturday mentorship circles, annual college prep bootcamp, parent engagement workshops" />
            <div style={styles.grid2}>
              <Field label="People Served Annually" value={org.peopleServed} onChange={updateOrg("peopleServed")} placeholder="e.g. 350 youth ages 10-18" />
              <Field label="Annual Budget" value={org.budget} onChange={updateOrg("budget")} placeholder="e.g. $280,000" />
            </div>
            <Field label="Key Outcomes & Data" sub="Measurable results — graduation rates, test scores, etc." value={org.outcomes} onChange={updateOrg("outcomes")} area={3} placeholder="e.g. 94% high school graduation rate among participants (vs. 71% district avg), 85% report increased STEM confidence" />
            <Field label="Geographic Focus" value={org.geography} onChange={updateOrg("geography")} placeholder="e.g. East Cleveland, Cuyahoga County, OH" />
            <div style={{ display: "flex", justifyContent: "flex-end", marginTop: 12 }}>
              <button style={canProceedStep0 ? styles.btn : styles.btnDisabled} disabled={!canProceedStep0} onClick={() => setStep(1)}>
                Continue →
              </button>
            </div>
          </div>
        )}

        {/* ── STEP 1: Grant Prompt ── */}
        {step === 1 && (
          <div style={styles.card}>
            <h2 style={styles.cardTitle}>Paste the grant prompt</h2>
            <p style={styles.cardSub}>
              Copy the funder's questions, requirements, or RFP narrative section directly. The more detail you include, the better the draft.
            </p>
            <Field
              label="Grant Prompt / RFP Text *"
              sub="Include all questions, evaluation criteria, and requirements"
              value={grantPrompt}
              onChange={setGrantPrompt}
              area={12}
              placeholder={`e.g.\n\nThe XYZ Foundation invites proposals from community-based nonprofits serving youth in Northeast Ohio. Proposals must address:\n\n1. Organizational Background: Describe your organization's history, mission, and experience serving the target population.\n\n2. Statement of Need: Using data, describe the community need your program addresses.\n\n3. Program Description: Detail the proposed program, including activities, timeline, staffing, and number of participants.\n\n4. Evaluation Plan: How will you measure success? Include specific metrics and data collection methods.\n\n5. Budget Narrative: Justify how funds will be used. Maximum award: $50,000.\n\nPriority given to organizations led by communities they serve. Deadline: March 15.`}
            />
            <div style={{ display: "flex", justifyContent: "space-between", marginTop: 12 }}>
              <button style={styles.btnGhost} onClick={() => setStep(0)}>← Back</button>
              <button style={canProceedStep1 ? styles.btn : styles.btnDisabled} disabled={!canProceedStep1} onClick={runChain}>
                Generate Draft →
              </button>
            </div>
          </div>
        )}

        {/* ── STEP 2: Loading / Error with Retry ── */}
        {step === 2 && (
          <div style={styles.card}>
            {loading ? (
              <div style={styles.loadWrap}>
                <div style={styles.spinner} />
                <h2 style={{ ...styles.cardTitle, marginTop: 16 }}>{loadMsg}</h2>
                <p style={styles.cardSub}>This usually takes 30–60 seconds. We're running a 3-step prompt chain to analyze, draft, and audit your narrative.</p>
                <div style={styles.chainViz}>
                  {["Analyze Requirements", "Draft Narrative", "Audit & Suggest"].map((s, i) => (
                    <div key={s} style={{
                      padding: "8px 16px", borderRadius: 8, fontSize: ".8rem",
                      fontWeight: 600, fontFamily: "var(--ff-body)",
                      background: loadMsg.includes("Reading") && i === 0 ? "var(--navy)" :
                        loadMsg.includes("Drafting") && i <= 1 ? "var(--navy)" :
                        loadMsg.includes("Reviewing") ? "var(--navy)" : "var(--bg-muted)",
                      color: loadMsg.includes("Reading") && i === 0 ? "#fff" :
                        loadMsg.includes("Drafting") && i <= 1 ? "#fff" :
                        loadMsg.includes("Reviewing") ? "#fff" : "var(--text-muted)",
                      transition: "all .4s",
                    }}>{s}</div>
                  ))}
                </div>
              </div>
            ) : (
              <div style={styles.loadWrap}>
                <div style={{ fontSize: "2rem", marginBottom: 8 }}>⚠️</div>
                <h2 style={{ ...styles.cardTitle }}>Something went wrong</h2>
                <p style={{ ...styles.cardSub, maxWidth: 420 }}>{error || "The API request failed. This can happen on first load — please try again."}</p>
                <div style={{ display: "flex", gap: 12, marginTop: 12 }}>
                  <button style={styles.btnGhost} onClick={() => { setError(""); setStep(1); }}>← Edit Prompt</button>
                  <button style={styles.btn} onClick={() => { setError(""); runChain(); }}>Retry →</button>
                </div>
              </div>
            )}
          </div>
        )}

        {/* ── STEP 3: Results ── */}
        {step === 3 && (
          <div ref={resultRef}>
            {error && (
              <div style={styles.errorBanner}>{error}</div>
            )}

            {/* Top summary row */}
            <div style={styles.resultGrid}>
              {/* Score + quick stats */}
              <div style={styles.sidePanel}>
                {audit && (
                  <>
                    <div style={{ textAlign: "center" }}>
                      <ScoreRing score={audit.completenessScore || 0} />
                    </div>
                    {audit.topPriority && (
                      <div style={styles.priorityBox}>
                        <span style={{ fontSize: ".68rem", fontWeight: 700, textTransform: "uppercase", color: "var(--gold-dark)", letterSpacing: ".04em" }}>Top Priority</span>
                        <p style={{ margin: "4px 0 0", fontSize: ".85rem", lineHeight: 1.45, color: "var(--navy)" }}>{audit.topPriority}</p>
                      </div>
                    )}
                    {audit.toneAssessment && (
                      <div style={styles.toneBox}>
                        <span style={{ fontSize: ".68rem", fontWeight: 700, textTransform: "uppercase", color: "var(--text-muted)", letterSpacing: ".04em" }}>Tone Check</span>
                        <p style={{ margin: "4px 0 0", fontSize: ".83rem", lineHeight: 1.45 }}>{audit.toneAssessment}</p>
                      </div>
                    )}
                  </>
                )}
                {analysis && (
                  <div style={styles.toneBox}>
                    <span style={{ fontSize: ".68rem", fontWeight: 700, textTransform: "uppercase", color: "var(--text-muted)", letterSpacing: ".04em" }}>Grant Info</span>
                    <p style={{ margin: "4px 0 0", fontSize: ".83rem" }}><strong>Funder:</strong> {analysis.funderName}</p>
                    <p style={{ margin: "2px 0 0", fontSize: ".83rem" }}><strong>Type:</strong> {analysis.grantType}</p>
                    {analysis.deadlineInfo !== "None specified" && (
                      <p style={{ margin: "2px 0 0", fontSize: ".83rem" }}><strong>Deadline:</strong> {analysis.deadlineInfo}</p>
                    )}
                  </div>
                )}
              </div>

              {/* Main content */}
              <div style={{ flex: 1, minWidth: 0 }}>
                {/* Missing sections */}
                {audit?.missingSections?.length > 0 && (
                  <div style={styles.alertCard}>
                    <h3 style={styles.alertTitle}>⚠ Missing or Incomplete Sections</h3>
                    {audit.missingSections.map((ms, i) => (
                      <div key={i} style={styles.alertItem}>
                        <div style={{ display: "flex", alignItems: "center", gap: 8, marginBottom: 2 }}>
                          <Badge priority={ms.priority} />
                          <strong style={{ fontSize: ".88rem", color: "var(--navy)" }}>{ms.section}</strong>
                        </div>
                        <p style={{ margin: 0, fontSize: ".82rem", color: "var(--text-body)", lineHeight: 1.45 }}>{ms.reason}</p>
                      </div>
                    ))}
                  </div>
                )}

                {/* Strengthening suggestions */}
                {audit?.strengtheningSuggestions?.length > 0 && (
                  <div style={styles.suggestCard}>
                    <h3 style={styles.suggestTitle}>💡 Strengthen Your Case</h3>
                    {audit.strengtheningSuggestions.map((s, i) => (
                      <div key={i} style={styles.suggestItem}>
                        <strong style={{ fontSize: ".85rem", color: "var(--navy)" }}>{s.area}</strong>
                        <p style={{ margin: "2px 0", fontSize: ".82rem", lineHeight: 1.45 }}>{s.suggestion}</p>
                        {s.dataPointNeeded && (
                          <p style={{ margin: 0, fontSize: ".78rem", color: "var(--gold-dark)", fontWeight: 600 }}>📊 Data to add: {s.dataPointNeeded}</p>
                        )}
                      </div>
                    ))}
                  </div>
                )}

                {/* The draft itself */}
                {draft && (
                  <div style={styles.draftCard}>
                    <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 12 }}>
                      <h3 style={styles.draftTitle}>Your Grant Narrative Draft</h3>
                      <button style={{
                        ...styles.btnSmall,
                        ...(copied ? { background: "var(--navy)", color: "#fff", borderColor: "var(--navy)" } : {}),
                      }} onClick={() => {
                        try {
                          const ta = document.createElement("textarea");
                          ta.value = draft;
                          ta.style.position = "fixed";
                          ta.style.left = "-9999px";
                          document.body.appendChild(ta);
                          ta.select();
                          document.execCommand("copy");
                          document.body.removeChild(ta);
                          setCopied(true);
                          setTimeout(() => setCopied(false), 2500);
                        } catch (e) {
                          const w = window.open("", "_blank");
                          if (w) {
                            w.document.write("<pre>" + draft.replace(/</g, "&lt;") + "</pre>");
                            w.document.close();
                          }
                        }
                      }}>{copied ? "✓ Copied!" : "Copy to Clipboard"}</button>
                    </div>
                    <div style={styles.draftBody} dangerouslySetInnerHTML={{ __html: md(draft) }} />
                  </div>
                )}
              </div>
            </div>
          </div>
        )}
      </main>

      <footer style={styles.footer}>
        <p>GrantDraft is an AI assistant — always review and personalize before submitting. Not legal or financial advice.</p>
      </footer>
    </div>
  );
}

// ── Styles ──

const cssVars = `
  @import url('https://fonts.googleapis.com/css2?family=Source+Serif+4:wght@400;600;700;800&family=Inter:wght@400;500;600;700&display=swap');
  :root {
    --navy: #1B2A4A;
    --navy-light: #2C3E6B;
    --gold: #C4943E;
    --gold-light: #FDF6E3;
    --gold-dark: #8B6914;
    --bg: #FAFAF8;
    --bg-muted: #F0EFEB;
    --border: #E2E0DA;
    --text-body: #3A3A38;
    --text-muted: #8A8A85;
    --ff-display: 'Source Serif 4', Georgia, serif;
    --ff-body: 'Inter', system-ui, sans-serif;
    --radius: 10px;
  }
  *, *::before, *::after { box-sizing: border-box; }
  @keyframes spin { to { transform: rotate(360deg); } }
`;

const styles = {
  root: {
    fontFamily: "var(--ff-body)",
    background: "var(--bg)",
    color: "var(--text-body)",
    minHeight: "100vh",
    fontSize: "15px",
    lineHeight: 1.55,
  },
  header: {
    display: "flex",
    justifyContent: "space-between",
    alignItems: "center",
    padding: "20px 28px 12px",
    maxWidth: 1060,
    margin: "0 auto",
  },
  logo: {
    width: 40, height: 40, borderRadius: 8,
    background: "var(--navy)", color: "var(--gold)",
    display: "flex", alignItems: "center", justifyContent: "center",
    fontFamily: "var(--ff-display)", fontWeight: 800, fontSize: "1rem",
  },
  h1: {
    margin: 0, fontSize: "1.35rem", fontFamily: "var(--ff-display)",
    fontWeight: 800, color: "var(--navy)", letterSpacing: "-.01em",
  },
  tagline: {
    margin: 0, fontSize: ".78rem", color: "var(--text-muted)", fontWeight: 500,
  },
  stepper: {
    display: "flex", justifyContent: "center", alignItems: "flex-start",
    gap: 0, padding: "16px 28px 8px",
  },
  main: {
    maxWidth: 1060, margin: "0 auto", padding: "8px 28px 40px",
  },
  card: {
    background: "#fff", border: "1px solid var(--border)",
    borderRadius: "var(--radius)", padding: "28px 32px",
    boxShadow: "0 1px 3px rgba(27,42,74,.04)",
  },
  cardTitle: {
    margin: "0 0 4px", fontFamily: "var(--ff-display)",
    fontSize: "1.15rem", fontWeight: 700, color: "var(--navy)",
  },
  cardSub: {
    margin: "0 0 20px", fontSize: ".85rem", color: "var(--text-muted)",
  },
  label: {
    display: "flex", flexDirection: "column", gap: 4, marginBottom: 16,
  },
  labelText: {
    fontSize: ".82rem", fontWeight: 600, color: "var(--navy)",
  },
  labelSub: {
    fontSize: ".74rem", color: "var(--text-muted)", fontWeight: 500, marginTop: -2,
  },
  input: {
    padding: "10px 14px", border: "1px solid var(--border)",
    borderRadius: 8, fontSize: ".9rem", fontFamily: "var(--ff-body)",
    outline: "none", transition: "border .2s",
    background: "var(--bg)",
  },
  textarea: {
    padding: "10px 14px", border: "1px solid var(--border)",
    borderRadius: 8, fontSize: ".9rem", fontFamily: "var(--ff-body)",
    outline: "none", resize: "vertical", lineHeight: 1.55,
    background: "var(--bg)",
  },
  grid2: {
    display: "grid", gridTemplateColumns: "1fr 1fr", gap: 16,
  },
  btn: {
    padding: "10px 28px", borderRadius: 8, border: "none",
    background: "var(--navy)", color: "#fff", fontSize: ".88rem",
    fontWeight: 600, fontFamily: "var(--ff-body)", cursor: "pointer",
    transition: "background .2s",
  },
  btnDisabled: {
    padding: "10px 28px", borderRadius: 8, border: "none",
    background: "var(--bg-muted)", color: "var(--text-muted)",
    fontSize: ".88rem", fontWeight: 600, fontFamily: "var(--ff-body)",
    cursor: "not-allowed",
  },
  btnGhost: {
    padding: "8px 20px", borderRadius: 8,
    border: "1px solid var(--border)", background: "transparent",
    color: "var(--navy)", fontSize: ".84rem", fontWeight: 600,
    fontFamily: "var(--ff-body)", cursor: "pointer",
  },
  btnSmall: {
    padding: "6px 14px", borderRadius: 6,
    border: "1px solid var(--border)", background: "var(--bg)",
    color: "var(--navy)", fontSize: ".76rem", fontWeight: 600,
    fontFamily: "var(--ff-body)", cursor: "pointer",
  },
  loadWrap: {
    display: "flex", flexDirection: "column", alignItems: "center",
    padding: "40px 20px", textAlign: "center",
  },
  spinner: {
    width: 36, height: 36, border: "3px solid var(--bg-muted)",
    borderTopColor: "var(--navy)", borderRadius: "50%",
    animation: "spin 1s linear infinite",
  },
  chainViz: {
    display: "flex", gap: 8, marginTop: 24, flexWrap: "wrap", justifyContent: "center",
  },
  resultGrid: {
    display: "flex", gap: 24, alignItems: "flex-start",
    flexWrap: "wrap",
  },
  sidePanel: {
    width: 200, display: "flex", flexDirection: "column", gap: 16,
    position: "sticky", top: 20,
  },
  priorityBox: {
    background: "var(--gold-light)", border: "1px solid #E8D5A0",
    borderRadius: 8, padding: "12px 14px",
  },
  toneBox: {
    background: "#fff", border: "1px solid var(--border)",
    borderRadius: 8, padding: "12px 14px",
  },
  alertCard: {
    background: "#FFF8F6", border: "1px solid #F5D0C5",
    borderRadius: "var(--radius)", padding: "20px 24px", marginBottom: 16,
  },
  alertTitle: {
    margin: "0 0 12px", fontFamily: "var(--ff-display)",
    fontSize: ".95rem", fontWeight: 700, color: "#B71C1C",
  },
  alertItem: {
    padding: "10px 0", borderBottom: "1px solid #F5D0C5",
  },
  suggestCard: {
    background: "#F6FAFB", border: "1px solid #C8DFE6",
    borderRadius: "var(--radius)", padding: "20px 24px", marginBottom: 16,
  },
  suggestTitle: {
    margin: "0 0 12px", fontFamily: "var(--ff-display)",
    fontSize: ".95rem", fontWeight: 700, color: "var(--navy)",
  },
  suggestItem: {
    padding: "10px 0", borderBottom: "1px solid #C8DFE6",
  },
  draftCard: {
    background: "#fff", border: "1px solid var(--border)",
    borderRadius: "var(--radius)", padding: "24px 28px",
  },
  draftTitle: {
    margin: 0, fontFamily: "var(--ff-display)",
    fontSize: "1.05rem", fontWeight: 700, color: "var(--navy)",
  },
  draftBody: {
    fontSize: ".9rem", lineHeight: 1.7, color: "var(--text-body)",
  },
  errorBanner: {
    background: "#FDECEA", border: "1px solid #F5C6CB",
    borderRadius: 8, padding: "12px 18px", marginBottom: 16,
    fontSize: ".85rem", color: "#B71C1C", fontWeight: 500,
  },
  footer: {
    textAlign: "center", padding: "20px 28px", fontSize: ".74rem",
    color: "var(--text-muted)", borderTop: "1px solid var(--border)",
    maxWidth: 1060, margin: "0 auto",
  },
};


*Created: 7/30/2026, 11:25:10 AM*

