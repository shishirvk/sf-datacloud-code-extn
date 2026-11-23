# sf-datacloud-code-extn

A Salesforce Data Cloud Code Extension project. It was created with `datacustomcode init` from the [`salesforce-data-customcode`](https://pypi.org/project/salesforce-data-customcode/) SDK.

## Prerequisites

- **Python 3.11.** The SDK requires Python `>=3.10,<3.12`, so Python 3.12 or newer **will not work** (`pip` reports `No matching distribution found for salesforce-data-customcode`). Don't use the system Python. Install 3.11 with **pyenv** or **uv**, as shown below.
- **A Java runtime (JDK 17 recommended).** Running code locally uses PySpark 3.5, which needs a JVM.

You can use either of the two setup options below. Run all commands from this project folder.

## Option A: pyenv + venv

This is how the original setup was done (Python 3.11.0).

1. Install pyenv:
   - macOS: `brew install pyenv`
   - Linux: `curl -fsSL https://pyenv.run | bash`. Install the [Python build dependencies](https://github.com/pyenv/pyenv/wiki#suggested-build-environment) first.
   - Windows: use [pyenv-win](https://github.com/pyenv-win/pyenv-win).

2. Add pyenv to your shell (zsh shown here). Without this step, `python` still points to the system interpreter:

   ```bash
   echo 'eval "$(pyenv init -)"' >> ~/.zshrc
   eval "$(pyenv init -)"
   ```

3. Install Python 3.11 and pin it for this folder:

   ```bash
   pyenv install 3.11
   pyenv local 3.11          # writes .python-version
   python --version          # must print Python 3.11.x
   ```

4. Create a virtual environment and install the SDK:

   ```bash
   python -m venv .venv
   source .venv/bin/activate         # Windows: .venv\Scripts\activate
   pip install --upgrade pip
   pip install salesforce-data-customcode
   ```

## Option B: uv

uv downloads Python 3.11 for you, so you don't need pyenv.

1. Install uv:
   - macOS/Linux: `curl -LsSf https://astral.sh/uv/install.sh | sh`, or `brew install uv`
   - Windows: `powershell -c "irm https://astral.sh/uv/install.ps1 | iex"`

2. Create a Python 3.11 virtual environment and install the SDK:

   ```bash
   uv python install 3.11
   uv venv --python 3.11 .venv
   source .venv/bin/activate         # Windows: .venv\Scripts\activate
   uv pip install salesforce-data-customcode
   ```

## Install project dependencies (optional)

This installs the full local dev toolchain: the SDK, PySpark, pandas, JupyterLab and the rest of `requirements-dev.txt`. It also installs your custom code's own packages from `requirements.txt`.

```bash
pip install -r requirements-dev.txt -r requirements.txt
# or, with uv:
uv pip install -r requirements-dev.txt -r requirements.txt
```

## Verify the install

```bash
datacustomcode version
datacustomcode --help
```

The CLI provides these commands: `configure`, `auth`, `init`, `scan`, `run`, `zip`, `deploy` and `version`.

## Next steps

```bash
datacustomcode configure    # set up Data Cloud org credentials
datacustomcode --help       # see options for run / scan / zip / deploy
```

The custom code entry point is `payload/entrypoint.py`, and its config is in `payload/config.json`.

## Credentials and secrets

Never commit org credentials. `datacustomcode configure` stores credentials in your home directory (`~/.datacustomcode/`), not in this project. Using `--sf-cli-org <alias>` reuses your Salesforce CLI login instead. The `.gitignore` excludes `.env` files, key files, `.sf/`/`.sfdx/` folders, and build outputs such as `deployment.zip`.

The local JupyterLab container (`./jupyterlab.sh start`) runs without a token. It is bound to `127.0.0.1`, so only your own machine can reach it.
