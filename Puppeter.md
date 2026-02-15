
# ✅ Step 1 — Update Server

```bash
sudo apt update
sudo apt upgrade -y
```

---

# ✅ Step 2 — Install Required Dependencies for Chromium

These are required for headless Chromium to run properly.

```bash
sudo apt update && sudo apt upgrade -y

sudo apt install -y \
  chromium \
  fonts-liberation \
  libasound2t64 \
  libatk-bridge2.0-0t64 \
  libatk1.0-0t64 \
  libcups2t64 \
  libdbus-1-3 \
  libdrm2 \
  libgbm1 \
  libgtk-3-0t64 \
  libnspr4 \
  libnss3 \
  libx11-xcb1 \
  libxcomposite1 \
  libxdamage1 \
  libxrandr2 \
  xdg-utils \
  wget \
  curl \
  unzip \
  nodejs \
  npm
```

---

# ✅ Step 3 — Verify Chromium Path

Check where chromium is installed:

```bash
which chromium-browser
```

OR

```bash
which chromium
```

Usually it will return something like:

```
/usr/bin/chromium-browser
```

👉 Save this path. You will need it inside Puppeteer config.

---

# ✅ Step 4 — Install Puppeteer WITHOUT Downloading Chromium

Inside your project folder:

```bash
PUPPETEER_SKIP_DOWNLOAD=true npm install puppeteer
```

⚠️ This prevents Puppeteer from downloading its own Chromium (important for VPS).

---

# ✅ Step 5 — Configure Puppeteer in Your Code

Example production-ready setup:

```js
const puppeteer = require("puppeteer");

const browser = await puppeteer.launch({
  executablePath: "/usr/bin/chromium-browser", // change if different
  headless: "new",
  args: [
    "--no-sandbox",
    "--disable-setuid-sandbox",
    "--disable-dev-shm-usage",
    "--disable-gpu",
    "--no-zygote",
    "--single-process"
  ]
});
```

---

# ✅ Step 6 — Test It Manually

Create a small test file:

```js
const puppeteer = require("puppeteer");

(async () => {
  const browser = await puppeteer.launch({
    executablePath: "/usr/bin/chromium-browser",
    args: ["--no-sandbox", "--disable-setuid-sandbox"]
  });

  const page = await browser.newPage();
  await page.goto("https://example.com");
  await page.pdf({ path: "test.pdf", format: "A4" });

  await browser.close();
})();
```

Run:

```bash
node test.js
```

If `test.pdf` generates successfully ✅ you're done.

