# 🎭 Week 4 — Behavioral & Scenario Questions

> **Focus:** Shipping under pressure, scope decisions, testing debates, deployment failures, project storytelling, product thinking
>
> **How to use:** Practice these as conversational 2-minute answers. Interviewers want to hear your process, not a perfect outcome.

---

## Q1. The Scope Creep Monster ⭐⭐

**Scenario:** You're building an AI assistant for your portfolio. You planned 4 tools and a basic UI. Your mentor says: "Add voice input, PDF upload, multi-model support, and a Chrome extension." Your deadline is 2 weeks. What do you do?

**Strong answer:**

"I'd thank them for the ideas and then prioritize ruthlessly. The question isn't 'are these good features?' — they all are. The question is 'which ones are necessary for the core value proposition, and which can come in v2?'

My approach: the project needs to demonstrate ONE thing clearly — that I can ship an end-to-end AI product with tools, memory, UI, and tests. That's the bar. Everything beyond that is bonus.

I'd map each suggestion:
- Voice input → cool but doesn't demonstrate AI engineering. Deferred to v2.
- PDF upload → actually useful, connects to RAG which is Episode 5. Deferred to v2 but noted.
- Multi-model support → high value, relatively low effort (provider abstraction). If I can do it in 2 hours, I'd add it. Otherwise, v2.
- Chrome extension → entirely different deployment surface. Hard no for v1.

Then I'd communicate: 'Here's what v1 will include and why. Here's the v2 backlog with your suggestions prioritized.' The person who ships a clean v1 in 2 weeks beats the person who ships a buggy v1.5 in 4 weeks."

---

## Q2. The Testing Argument ⭐⭐⭐

**Scenario:** You're working on an AI project with a teammate. You've written 40+ tests. Your teammate says: "Writing tests for AI is pointless — the outputs are non-deterministic anyway. We should just test manually." How do you respond?

**Strong answer:**

"I understand the frustration — testing non-deterministic outputs IS harder than testing traditional software. But that's exactly why we need automated tests MORE, not less.

Here's my argument:

First, most of our code IS deterministic. The calculator tool, the JSON parser, the memory manager, the export system, the validation logic — these are all testable in the traditional way. That's probably 60% of our codebase.

Second, for the non-deterministic parts, we test properties, not exact outputs. We don't assert 'the response is exactly this string.' We assert 'the response contains a temperature value,' 'the tool selection was correct,' 'the conversation context was maintained.' These tests catch real regressions.

Third, without automated tests, every deployment is a manual QA session. At 40+ scenarios, that's 2 hours of manual testing every time we change anything. Automated tests run in 2 minutes.

I'd propose a compromise: I'll write the deterministic tests (tools, memory, validation), you focus on building 20 golden test cases for the eval suite (input + expected tool + expected properties). Together, we get fast CI tests AND quality measurement.

What I wouldn't do: get defensive or dismiss their concern. The concern is valid — testing AI is genuinely hard. The answer is better testing strategy, not no testing."

---

## Q3. The Friday Deploy ⭐⭐

**Scenario:** It's Friday 3 PM. Your AI assistant is demo-ready. Your team lead says: "Deploy it now so the VP can see it Monday morning." There are no monitoring alerts set up, and you haven't tested the deployment pipeline. Do you deploy?

**Strong answer:**

"I'd deploy — but with guardrails.

Here's my thinking: the VP seeing a live demo Monday morning has real business value. Refusing to deploy is the cautious-but-unhelpful answer. But deploying blindly is reckless.

My plan for the next 2 hours:

1. Deploy to a staging URL (not production) — 30 minutes
2. Run through the 10 critical user flows manually — 30 minutes
3. Set up basic health check monitoring (even just uptime ping) — 15 minutes
4. Send the staging URL to the team lead for a quick smoke test — 15 minutes
5. If staging looks good, promote to the production URL — 15 minutes
6. Write a 'known limitations' doc so the VP isn't surprised — 15 minutes

What I'd communicate to the team lead: 'Deploying now. Here's the link. I set up a basic health check — if it goes down over the weekend, I'll get an alert. Here are 3 things the VP should avoid doing [edge cases we haven't hardened].'

The key insight: deploy to staging first, verify, then promote. It's 30 minutes extra but prevents a Monday morning disaster."

---

## Q4. The Memory Bug ⭐⭐⭐

**Scenario:** Users report that the AI "forgets" what they said 5 messages ago. You check the code — conversation memory is working, all messages are stored. But the AI responses don't reference earlier context. What's happening?

**Strong answer:**

"This is almost always one of three things:

**Most likely — context injection is too short.** The memory stores all messages, but only the last 3 are injected into the routing/generation prompt. The user's message from 5 turns ago is stored but never seen by the model.

Fix: increase context injection window from 3 to 8-10 turns, or switch to a summary-based approach where older context is compressed but still present.

**Second most likely — token budget is too aggressive.** The memory manager is trimming old messages to stay within budget, but the budget is set too low. Messages from 5 turns ago are already evicted.

Fix: increase the token budget, or use a smarter trimming strategy (summarize old turns instead of dropping them).

**Third possibility — the model is ignoring middle context.** If you're injecting 20+ turns, the model pays less attention to messages in the middle (lost in the middle problem). The user's message from 5 turns ago is physically in the prompt but the model doesn't attend to it.

Fix: put the most important context (user's key facts, preferences, previous decisions) at the TOP of the conversation injection, not chronologically in the middle.

My debugging approach: 
1. Log the exact prompt being sent to the LLM (including injected history)
2. Check: is the user's earlier message actually in the prompt? If no → trimming bug
3. If yes → the model is ignoring it → restructure context ordering or add explicit recall instructions"

---

## Q5. The Demo Disaster ⭐⭐

**Scenario:** You're demoing your AI assistant to potential employers. You type "What's the weather in New York?" — and the weather API times out. The screen shows a raw error traceback. How do you handle the moment, and how do you prevent it next time?

**Strong answer:**

**In the moment:** "I'd stay calm and say: 'This is actually a great example of why error handling matters in production AI. The weather API is having a timeout — let me show you how the system handles that.' If I've built graceful degradation, I show the fallback. If I haven't, I say: 'This is something I'd add — a fallback message and retry logic. Let me show you the calculator tool instead, which doesn't depend on external APIs.'

Turn the failure into a teaching moment. Interviewers are more impressed by how you handle failure than by a perfect demo.

**Prevention:**

1. **Fallback for every external API:** Every tool that calls an external service has a timeout (5 seconds) and a fallback response ('Weather data is temporarily unavailable').

2. **Demo-safe defaults:** Have a 'demo mode' flag that uses cached responses instead of live APIs. Enable it before every demo.

3. **Always have a backup tool:** Start the demo with the calculator (zero network dependency), THEN show API-dependent tools.

4. **Test the demo environment before presenting.** 5 minutes before the call, run through 3 queries. If any fail, switch to demo mode."

---

## Q6. The "Why Streamlit?" Question ⭐⭐

**Scenario:** An interviewer asks: "Why did you use Streamlit for the UI? Why not React, or Gradio, or just a CLI?"

**Strong answer:**

"I chose Streamlit specifically because this project's goal was to demonstrate end-to-end AI engineering, not frontend development.

Streamlit let me build a functional, demo-able web interface in about 2 hours. The same UI in React would have taken 2 days — and the frontend engineering wouldn't showcase any AI skills.

But I'm aware of Streamlit's limitations. It re-runs the entire script on every interaction, which means expensive operations need caching. Session state management is more manual than React's component state. And it doesn't scale to complex multi-page applications with custom components.

If this were a production product serving real users, I'd use React for the frontend and FastAPI for the backend. That gives me component-level reactivity, custom styling, proper routing, and the ability to have separate frontend and backend deployment.

For this project's purpose — demonstrating AI architecture, memory management, tool orchestration, and testing — Streamlit was the right tool. It let me spend 90% of my time on the interesting engineering and 10% on the UI, instead of the other way around."

---

## Q7. The Wrong Tool Selection ⭐⭐

**Scenario:** During testing, you find that your router selects the calculator tool when users ask questions like "How many people live in Tokyo?" (should be Wikipedia). The calculator can't answer this. How do you fix it?

**Strong answer:**

"This is a routing accuracy problem — the router sees 'how many' and associates it with math/counting, not knowledge retrieval.

Fix #1 — Improve tool descriptions. The calculator description probably says 'calculate numbers' which is too vague. Better: 'Evaluate mathematical expressions with numbers and operators (+, -, ×, ÷, %). Only for computation, NOT for factual questions.'

Fix #2 — Add negative examples to the routing prompt:
```
NOT calculator: 'How many people live in Tokyo?' (factual question → wikipedia)
NOT calculator: 'How many planets are there?' (factual question → wikipedia)
YES calculator: 'How much is 25% of 480?' (mathematical computation)
```

Fix #3 — Add this exact query to the eval set. Every routing mistake becomes a test case. After fixing, the test ensures it never regresses.

Fix #4 — If the problem persists across many queries, the router might need few-shot examples or a more capable model for the routing step."

---

## Q8. The "No Tests" Codebase ⭐⭐

**Scenario:** You join a team that has a deployed AI chatbot with zero tests. The product works but the team is afraid to change anything because they don't know what might break. Where do you start?

**Strong answer:**

"I wouldn't try to retroactively test everything — that's demoralizing and slow. Instead, I'd use the 'expanding test coverage' strategy:

**Week 1 — Smoke tests (the safety net):**
Write 5-10 end-to-end tests that hit the most critical user paths. These don't test edge cases — they test 'does the app respond at all?' This gives immediate confidence for deployments.

**Week 2 — Regression tests (the catch net):**
Every bug that gets reported becomes a test case BEFORE fixing it. Write the test, watch it fail, fix the bug, watch it pass. Coverage grows organically from real issues.

**Week 3+ — Component tests for new code:**
All NEW code must have tests. Don't rewrite old code, but ensure everything going forward is tested. Over 3-6 months, the tested percentage grows naturally as old code is replaced.

**The golden rule:** Never add a feature without a test. Never fix a bug without a test. Test coverage grows monotonically."

---

## Q9. The Cost Explosion ⭐⭐⭐

**Scenario:** Your AI assistant launches. Usage grows 10x in a week. Your OpenAI bill goes from $50/day to $2,000/day. The CEO says "Fix the cost or shut it down." You have 48 hours.

**Strong answer:**

"48-hour emergency plan:

**Hour 0-4 (stop the bleeding):**
1. Add per-user rate limits immediately (20 requests/hour). This caps maximum cost.
2. Switch the default model from gpt-4o to gpt-4o-mini for simple queries (85% of traffic). Instant 10x cost reduction on those queries.

**Hour 4-12 (measure):**
3. Instrument cost-per-request logging. Find out: which users, which tools, and which queries consume the most tokens. Usually 10% of users cause 50% of cost.
4. Check for conversation context bloat — are some users at 30+ turns with full history injection? That's thousands of input tokens per request.

**Hour 12-24 (optimize):**
5. Add semantic caching. If someone asks the same question (or a very similar one) that was asked before, return the cached response. Expected hit rate: 20-30%.
6. Implement context compression. After 10 turns, summarize old context instead of passing full history. Cuts input tokens by 60-70%.

**Hour 24-48 (sustain):**
7. Add a cost dashboard so we see daily spend in real-time.
8. Set up alerts: $500/day warning, $1000/day page, $1500/day auto-throttle.
9. Present the numbers to the CEO: 'Cost reduced from $2000/day to $400/day, with a path to $200/day through further caching and model routing.'"

---

## Q10. The Multi-Tool Confusion ⭐⭐

**Scenario:** Your assistant has 4 tools. A user says: "What's the weather in 100 degree city?" The router gets confused — is "100 degree" a temperature for the converter, or part of the city name? How do you handle ambiguous queries?

**Strong answer:**

"This is a genuine ambiguity — there IS a city called 'Hundred' and '100 degree' could be a conversion request. The right approach depends on the product context:

**Option 1 — Ask for clarification (best UX for complex queries):**
'Did you mean the weather in a city, or did you want to convert 100 degrees?'

This costs one extra turn but eliminates errors. Good for: assistants where accuracy matters more than speed.

**Option 2 — Default to most likely interpretation + offer alternative:**
Process as a weather query (most likely intent). In the response, add: 'Looking up weather... If you meant to convert 100 degrees, just say "convert 100°F to Celsius."'

Good for: consumer chatbots where one-shot speed matters.

**Option 3 — Confidence-based routing:**
If the router's confidence for both tools is below 0.7, trigger clarification. If one tool is clearly above 0.8, proceed with that one.

The architectural fix: include in the routing prompt: 'If the query is ambiguous between tools, select the most likely interpretation and set confidence below 0.7. The system will ask the user to clarify.'"

---

## Q11. Explaining Technical Decisions to Non-Technical Stakeholders ⭐⭐

**Scenario:** A product manager asks: "Why does the chatbot need 'conversation memory'? Can't AI just remember things?"

**Strong answer:**

"Great question — it's actually one of the most misunderstood things about AI. Let me explain with an analogy.

Imagine you're at a help desk. Every time you ask a question, the agent answers — but then immediately forgets you exist. You ask your next question, and they have no idea who you are or what you just discussed.

That's what an AI does by default. Every message is a fresh start. It doesn't know you asked about Paris 30 seconds ago.

'Conversation memory' is our workaround. Before each new question, we show the AI a transcript of everything said so far in the conversation. So when you say 'What about London?', the AI can see that you were asking about weather in Paris — and it understands that 'what about' means weather.

The cost of this: the transcript gets longer with every message, and we pay per word (token) to send it to the AI. So we have strategies to keep it manageable — like summarizing old messages or only keeping the last 10 exchanges.

The alternative: no memory. The chatbot feels broken. Users ask follow-up questions and get confused responses. That's why we invested in building it right."

---

## Q12. The Deployment Failure ⭐⭐⭐

**Scenario:** You deploy your AI assistant to Streamlit Cloud. It works perfectly locally. On Streamlit Cloud, the weather tool returns errors and the conversation memory resets after every message. Diagnose the issues.

**Strong answer:**

"Two separate issues with different root causes:

**Weather tool errors:** Most likely a missing API key. Locally, the key is in my `.env` file. Streamlit Cloud doesn't have my `.env` — I need to add the key to Streamlit Cloud's Secrets management (Settings → Secrets). Alternatively, the free tier might have outbound network restrictions blocking the weather API call.

Diagnosis: check Streamlit Cloud logs for the exact error. If it's 'API key not found' → add to secrets. If it's 'connection refused' → network issue, might need a different hosting platform.

**Memory resets:** Streamlit Cloud may be running in a mode where `session_state` doesn't persist, OR multiple replicas are serving requests and sessions aren't sticky. More likely: the app is being killed and restarted frequently on the free tier due to inactivity timeout.

Diagnosis: Add logging to `st.session_state` initialization. If 'initializing memory' appears in logs on every message → session state isn't persisting. Fix: ensure the app process stays alive (Streamlit Cloud's free tier sleeps after inactivity) and that no code path accidentally re-initializes the memory."

---

## Q13. "What Would You Do Differently?" ⭐⭐⭐

**Scenario:** An interviewer asks: "If you started this project over, what would you do differently?"

**Strong answer:**

"Three things:

**1. I'd build the eval suite on day one, not day twenty.** I wrote tests after the features were built, which meant I was testing against my assumptions instead of against real user behavior. If I started over, I'd write 20 golden test cases before writing a single line of feature code. That eval set would guide every architectural decision.

**2. I'd implement the provider abstraction layer from the start.** I hardcoded OpenAI calls initially, then had to refactor when I wanted to add Anthropic support. If I'd built the abstraction (like CaseFlow's `llm_client.py`) from day one, adding providers would have been a config change, not a rewrite.

**3. I'd add structured logging from the beginning, not print statements.** Debug logs with print() worked during development but were useless for diagnosing production issues. Structured JSON logs with trace IDs would have saved me hours of debugging."

---

## Q14. The Feature Request vs Technical Debt Trade-Off ⭐⭐

**Scenario:** You have a 2-week sprint. The product owner wants a new "voice input" feature. You want to refactor the memory manager (it's fragile, hard to test, and has a known bug with long conversations). What do you advocate for?

**Strong answer:**

"I'd advocate for the memory refactor, but I'd frame it in product terms, not technical terms.

'The voice input feature adds a new way to input text — it's valuable, but it doesn't change what the system can DO. The memory bug, on the other hand, is causing 5% of conversations to lose context after 15 messages. That's a 5% failure rate on our core value proposition — conversational AI that remembers what you said.

If we ship voice input on top of a fragile memory system, we'll have a shiny new feature that breaks 5% of the time. If we fix memory first, everything we build on top — including voice input next sprint — will be more reliable.

I'd propose: Week 1 = memory refactor with test coverage. Week 2 = start voice input (scope it to be completable in the following sprint). We ship reliability this sprint and the feature next sprint.'

What I wouldn't do: just say 'we need to fix tech debt.' That means nothing to a product owner. Translate it to user impact and business risk."

---

## Q15. Working with Non-Technical Team Members on an AI Project ⭐⭐

**Scenario:** A designer on your team suggests the AI assistant should "think before responding" — showing a visible 2-second thinking animation even when the response is ready instantly. They say it builds trust. You think it adds artificial latency. Who's right?

**Strong answer:**

"The designer is probably right, and there's research to back it up.

Studies show that users trust AI responses MORE when they see evidence of 'thinking' — even if it's artificial. An instant response feels like a random guess. A 1-2 second 'Analyzing your question...' animation feels like deliberation.

But I'd refine the approach:

Instead of ARTIFICIAL delay, use REAL processing steps as the animation:
- 'Understanding your question...' (while the router is classifying)
- 'Searching weather data...' (while the tool is executing)
- 'Formatting response...' (while the LLM generates)

This way, the animation reflects actual work happening, and the user sees transparency into the pipeline — not fake thinking.

If the total processing time is under 500ms (cached response), then yes, I'd add a brief animation. Instant responses feel uncanny. A 500ms-1s animation with a progress indicator feels thoughtful.

The lesson: in AI products, user perception of intelligence matters as much as actual intelligence. This is a legitimate design decision, not a 'fake' one."

---

## Q16. Describing Your AI Project to a 5-Year-Old ⭐

**Scenario:** An interviewer says: "Explain your AI assistant project as if I'm five years old."

**Strong answer:**

"Imagine you have a really smart friend who can do math really fast, knows what the weather is everywhere, and knows a little bit about everything on Wikipedia.

But this friend has a problem — every time you ask them a question, they forget you exist. You ask 'What's the weather in Paris?' and they answer. Then you ask 'What about London?' and they say 'What about London? What are we talking about?'

So I built a notebook that writes down everything you and your friend say to each other. Before each new question, I show the friend the notebook so they can remember what you were talking about.

I also built a TV screen where you can see the conversation happening — like watching a text message thread — instead of just typing into a black box.

And I wrote a bunch of tests — like a teacher's answer key — to make sure the friend gives correct answers and doesn't get confused."

**Why this works in interviews:** It proves you understand the system deeply enough to strip away all jargon. If you can explain conversation memory to a 5-year-old, you definitely understand it at a technical level.
