Use the steps below to run and test the saved NoTitle project on **Windows using Ubuntu through WSL2**. If you already use Ubuntu 24.04, skip the Windows installation steps.

**Current limitation:** the complete database/build/browser test sequence has not yet been verified successfully. These instructions follow the project’s actual scripts; any failures need to be investigated before deployment. Audio and payments remain disabled until their external integrations are configured.

**1. Download the project**

Download:

NoTitle_V1_Integrated_Phases1_to10.zip[NoTitle_V1_Integrated_Phases1_to10.zip](sandbox:/workspace/scratch/22bdd794a4d3/NoTitle_V1_Integrated_Phases1_to10.zip)

Keep the ZIP in your Windows **Downloads** folder for now.

You do not need hosting, a domain, payment credentials, or a cloud database for the initial local tests.

**2. Install Ubuntu through WSL2**

Open **PowerShell as Administrator** and run:

```powershell
wsl --install -d Ubuntu-24.04
```

Restart Windows if prompted. Open **Ubuntu 24.04** from the Start menu and create your Linux username and password.

When entering a Linux password, the terminal displays no characters. This is normal.

Back in PowerShell, check:

```powershell
wsl --list --verbose
```

The Ubuntu distribution should show **VERSION 2**. Microsoft documents this installation and verification process. [Microsoft Learn](https://learn.microsoft.com/en-us/windows/wsl/install?utm_source=chatgpt.com)

**3. Install and configure Docker Desktop**

Install [Docker Desktop for Windows](https://docs.docker.com/desktop/setup/install/windows-install/), then open it.

Configure:

- **Settings → General:** enable the WSL2 engine.
- **Settings → Resources → WSL Integration:** enable Ubuntu 24.04.
- **Settings → Resources → Network:** enable **Host networking**, then apply and restart.

Host networking is required by this project’s local proxy configuration. Docker documents support in Desktop 4.34 and later, with the feature explicitly enabled. Use Linux containers. [Docker Docs](https://docs.docker.com/desktop/features/wsl/?utm_source=chatgpt.com)

Keep Docker Desktop running.

**4. Install the development tools inside Ubuntu**

From this point, run commands in the **Ubuntu terminal**, unless a step explicitly says otherwise.

Install basic utilities and Python:

```bash
sudo apt update
sudo apt install -y python3 python3-venv curl unzip git ca-certificates
```

Check:

```bash
python3 --version
```

It should show **Python 3.12.x**.

Install `uv`, which the project uses to install Python dependencies:

```bash
curl -LsSf https://astral.sh/uv/install.sh -o /tmp/notitle-uv-install.sh
sh /tmp/notitle-uv-install.sh
source "$HOME/.local/bin/env"
```

This uses Astral’s official installer. [uv](https://docs.astral.sh/uv/getting-started/installation/?utm_source=chatgpt.com)

Install Node through `nvm`:

```bash
curl -fsSL https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.8/install.sh -o /tmp/notitle-nvm-install.sh
bash /tmp/notitle-nvm-install.sh
source "$HOME/.nvm/nvm.sh"
nvm install 24
nvm alias default 24
nvm use 24
```

These commands follow the official nvm installation approach. [GitHub](https://github.com/nvm-sh/nvm?utm_source=chatgpt.com)

Check all tools:

```bash
python3 --version
node --version
npm --version
uv --version
docker --version
docker compose version
docker info
```

Expected:

| Check | Expected result |
|---|---|
| Python | `3.12.x` |
| Node | `v24.x.x` |
| npm and uv | Version numbers |
| Docker Compose | Version information |
| `docker info` | Engine details without a connection error |

**5. Extract the project into Ubuntu**

Create a project folder:

```bash
mkdir -p ~/projects
cd ~/projects
explorer.exe .
```

Windows File Explorer should open the Ubuntu folder.

Copy the downloaded ZIP into this folder. Then return to Ubuntu and run:

```bash
unzip NoTitle_V1_Integrated_Phases1_to10.zip
cd ~/projects/notitle-v1
ls
```

You should see files and folders including:

```text
README.md
api
web
ops
infra
tests
package.json
```

**Run all remaining project commands from `~/projects/notitle-v1`.**

**6. Run the first-time setup**

Run:

```bash
python3 ops/nt.py release setup
```

The script is designed to:

- Generate local configuration and private secrets.
- Resolve dependency locks and Docker image digests.
- Install Python dependencies.
- Run core checks.
- Start PostgreSQL and apply database migrations.
- Build the public and admin interfaces.
- Start the application and local proxy.

The first run downloads dependencies and images, so it takes longer than subsequent starts.

**Only continue if the command finishes successfully.** If a package or image cannot be resolved, save the exact error rather than bypassing the lock checks.

For this local setup, the script generates database credentials automatically; you do not need to invent passwords or configure a hosted database.

**7. Check that the application is running**

Run:

```bash
.venv/bin/python ops/nt.py dev status
```

Then check the public health endpoint:

```bash
curl -fsS http://127.0.0.1:8080/health/ready
```

A successful response should contain JSON rather than a connection error or proxy error.

Open these addresses in your Windows browser:

| Interface | Address |
|---|---|
| Public application | [http://127.0.0.1:8080](http://127.0.0.1:8080) |
| Admin application | [http://127.0.0.1:8081](http://127.0.0.1:8081) |

Use **`127.0.0.1` consistently** throughout testing. Switching between it and `localhost` can create separate browser origins and sessions.

**8. Test matching and temporary text with two users**

Use two independent browser sessions:

- User A: Chrome normal window.
- User B: Edge, or Chrome Incognito.

Two ordinary tabs share cookies. Two Incognito windows in the same browser can also share a session.

In **both** browser sessions:

1. Open the public application.
2. Go to **Join**.
3. Choose a declared gender.
4. Read and select the required age, Terms, and processing confirmations.
5. Click **Continue as guest**.
6. Click **Open conversation lobby**.
7. Click **Search**.

The display name is generated automatically.

Check the following:

| Action | Expected result |
|---|---|
| Both users search | They match with each other |
| A sends `Hello from A` | B sees the message |
| B sends a reply | A sees it |
| Either user selects **End conversation** | Both leave the conversation |
| Conversation ends | Temporary text disappears from the conversation interface |
| A searches again | A waits until another available user searches |

Use only synthetic test messages.

**9. Test friends and moderator support**

For friends:

1. Match the two users again.
2. User A clicks **Add friend**.
3. User B clicks **Accept friendship**.
4. End the conversation.
5. Open **Friends** in both sessions.
6. User A selects **Invite**.
7. User B explicitly accepts the invitation.
8. Open the conversation and exchange another message.

Guest friendships should appear as temporary.

For support:

1. Open **Support → Chat with Moderator**.
2. Enter a test subject and message.
3. If prompted, click **Prepare private case receipt**.
4. Download and save the receipt.
5. Confirm you have saved the reference and secret.
6. Click **Save case**.
7. Send a follow-up message.
8. Reload and check that the saved support history remains accessible.

Keep the receipt private: it contains information used to reopen the case.

**10. Create an administrator and test replies**

In Ubuntu, run:

```bash
.venv/bin/python ops/staff.py owner_user --role owner
```

Enter and confirm a strong password of at least 12 characters.

The command creates a private enrollment file. Open its folder:

```bash
explorer.exe .local
```

Open `staff-enrollment.txt`. Add its authenticator secret to your authenticator application and store the recovery codes securely. After confirming enrollment, remove the enrollment file.

Then:

1. Open [http://127.0.0.1:8081](http://127.0.0.1:8081).
2. Sign in with `owner_user`, your password, and the current authenticator code.
3. Open the case queue.
4. Locate the support case you created.
5. Open its conversation and send a moderator reply.
6. Return to the user’s support conversation and check for the reply.
7. Open Analytics and inspect the available metrics.

Some analytics values may be suppressed when the sample is too small. That is expected privacy behavior.

Run the owner creation command **once** for this username.

**11. Run the automated test suite**

Use a disposable local installation with dummy data: integration tests can modify the local test database.

First install browser-test dependencies:

```bash
npm ci --ignore-scripts
npx playwright install --with-deps chromium
```

Playwright’s `--with-deps` option installs browser system dependencies; it may request your Ubuntu password. [Playwright](https://playwright.dev/docs/intro?utm_source=chatgpt.com)

Stop the running application:

```bash
.venv/bin/python ops/nt.py dev down
```

Run the complete local verification sequence:

```bash
.venv/bin/python ops/nt.py release verify --level stack
```

The collector attempts the core tests, real database integration tests, application builds, TypeScript checks, and browser tests.

Inspect the result:

```bash
python3 -m json.tool evidence/release/latest.json
```

Detailed logs are stored under `evidence/release/`.

A failed, blocked, or skipped required check needs investigation. **A successful local test run alone does not establish production readiness.**

The collector stops the application it starts. To resume manual testing:

```bash
.venv/bin/python ops/nt.py dev up --mode product
```

**12. Stop and restart on another day**

To stop:

```bash
cd ~/projects/notitle-v1
.venv/bin/python ops/nt.py dev down
```

To start later:

1. Open Docker Desktop and wait for its engine to start.
2. Open Ubuntu.
3. Run:

```bash
cd ~/projects/notitle-v1
.venv/bin/python ops/nt.py dev up --mode product
```

The normal stop command preserves the database volume. Preserve the project’s `.local` configuration and secrets too.

**If something fails**

| Problem | First action |
|---|---|
| `docker` unavailable in Ubuntu | Enable Docker Desktop’s Ubuntu WSL integration |
| Cannot connect to Docker daemon | Start Docker Desktop and retry `docker info` |
| `node` or `uv` not found | Reopen Ubuntu, then check the installation commands |
| “Existing runner PID” | Run `dev status`, then `dev down` before restarting |
| `502 Bad Gateway` | Check host networking and the backend log |
| Two users do not match | Confirm separate browser sessions and click Search in both |
| Authenticator code rejected | Check your phone’s time and use its current code |
| Package/image version unavailable | Preserve the exact error; the candidate dependency pins need review |
| Audio or subscription purchase disabled | Expected until external integrations and qualification are complete |

Backend log:

```bash
tail -n 80 .local/runner.log
```

Container status:

```bash
docker compose --env-file infra/images.env -f infra/compose.dev.yml ps
```

Proxy log:

```bash
docker compose --env-file infra/images.env -f infra/compose.dev.yml logs --tail=80 edge
```

Share the failed command and relevant error text when troubleshooting. Remove passwords, recovery codes, case receipt secrets, and tokens before sharing logs.
