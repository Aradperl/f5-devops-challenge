# Generative AI Disclosure

Using AI tools for this challenge is allowed. Be honest here — an accurate
"I used AI heavily" is worth more to us than an inaccurate "I did not".

**Did you use AI tools?** (yes / no): yes

**Which ones?** (e.g. ChatGPT, Claude, Copilot, Cursor): Cursor, Copilot

> If you answered **no**, write "N/A" in sections 1 and 2 and skip to
> section 3. That is a perfectly good answer.

---

## 1. Key prompts

List the main prompts you used. Paraphrasing is fine.

1. "Why does Ubuntu Dockerfile fail interactively during apt-get install nginx when copying a custom nginx.conf first?"
2. "Why does Nginx return 502 connection refused when proxying to 127.0.0.1 in Docker Compose?"
3. "How can I resolve the startup race condition between backend and test-runner without changing backend/app.py?"
4. "How to run Nginx on Alpine as an unprivileged user while binding port 80?"

---

## 2. Where the AI got it wrong

Describe **one concrete case** where the AI's answer was wrong, outdated, or
sub-optimal. Cover: what it suggested, how you noticed it was wrong, and what
you did instead. 3–5 sentences.

> For the startup race, the suggestion was to set CONFIG_FETCH_SECONDS=0. That skips the delay in backend/app.py instead of waiting for port 8080, and depends_on would still only wait for the process to start. app.py sleeps 2 seconds for each of 4 keys, about 8 seconds. What I kept is a backend healthcheck using Python urllib, with start_period set to 30s, and the proxy waiting on service_healthy. The default delay stays, and the first successful /health check ends the wait.

---

## 3. Anything in your submission you could not fully explain?

We would rather know. Write "nothing" if that is the case.

> nothing