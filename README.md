# sf-datacloud-code-extn

A Salesforce Data Cloud Code Extension project. It was created with `datacustomcode init` from the [`salesforce-data-customcode`](https://pypi.org/project/salesforce-data-customcode/) SDK. The entry point is `payload/entrypoint.py`, and its config is `payload/config.json`.

Setup has five steps: Python 3.11 env → Java 17 → shell config (`~/.zshrc`) → Salesforce CLI and org login → verify and run. Do them in order.

## Prerequisites

| Tool | Version | Why |
|---|---|---|
| Python | **3.11** (3.10 also works; **3.12+ does not**) | The SDK requires `>=3.10,<3.12`. On a newer Python, `pip` fails with `No matching distribution found for salesforce-data-customcode` |
| Java (JDK) | **17** | Local runs use PySpark 3.5, which needs a JVM |
| Salesforce CLI (`sf`) | latest | Provides `sf data-code-extension` and the org login |

Don't rely on the system `python3`. On macOS it is often 3.12 or 3.13. Install 3.11 with **pyenv** or **uv**, as shown below.

## 1. Create the Python 3.11 environment

Run these commands from this project folder. Pick **one** option.

### Option A: pyenv + venv

```bash
# install pyenv
brew install pyenv                                   # macOS
# curl -fsSL https://pyenv.run | bash                # Linux (install the build deps first:
#                                                    #  https://github.com/pyenv/pyenv/wiki#suggested-build-environment)
# Windows: https://github.com/pyenv-win/pyenv-win

eval "$(pyenv init -)"              # for this terminal; step 3 makes it permanent
pyenv install 3.11
pyenv local 3.11                    # writes .python-version in this folder
python --version                    # must print Python 3.11.x

python -m venv .venv
source .venv/bin/activate           # Windows: .venv\Scripts\activate
pip install --upgrade pip
pip install salesforce-data-customcode
```

### Option B: uv (no pyenv needed)

```bash
# install uv
curl -LsSf https://astral.sh/uv/install.sh | sh     # macOS/Linux (or: brew install uv)
# Windows: powershell -c "irm https://astral.sh/uv/install.ps1 | iex"

uv python install 3.11
uv venv --python 3.11 .venv
source .venv/bin/activate           # Windows: .venv\Scripts\activate
uv pip install salesforce-data-customcode
```

### Optional: full dev toolchain

This installs PySpark, pandas, JupyterLab and the rest of `requirements-dev.txt`, plus your code's own packages from `requirements.txt`:

```bash
pip install -r requirements-dev.txt -r requirements.txt      # or: uv pip install -r ...
```

## 2. Install Java 17

```bash
brew install openjdk@17                 # macOS
# sudo apt install openjdk-17-jdk       # Debian/Ubuntu
```

On macOS, Homebrew's `openjdk@17` is *keg-only*: it is not put on your PATH automatically. The built-in `/usr/bin/java` is only a placeholder, so without `JAVA_HOME` a run fails with `Unable to locate a Java Runtime`. Step 3 fixes this.

## 3. Configure your shell (`~/.zshrc`)

Add the block below to `~/.zshrc`, and set `DATACLOUD_PROJECT_DIR` to where **you** cloned this project. It does three things:

1. Puts Java 17 on the PATH.
2. Initializes pyenv, if you used Option A.
3. **Activates the project's `.venv` automatically** whenever you `cd` into the project (or any subfolder), and deactivates it when you leave. This matters because `sf data-code-extension` runs whatever `python3` is first on your PATH. If that isn't the 3.11 venv, the run fails with a Python version error.

```zsh
# ---- Salesforce Data Cloud code extension ----

# Java 17 for PySpark (macOS/Homebrew; on Linux use e.g. /usr/lib/jvm/java-17-openjdk-amd64)
export JAVA_HOME="$(brew --prefix)/opt/openjdk@17"
export PATH="$JAVA_HOME/bin:$PATH"

# pyenv (only if you used Option A)
command -v pyenv >/dev/null && eval "$(pyenv init -)"

# Auto-activate the project's Python 3.11 venv when inside the project folder
export DATACLOUD_PROJECT_DIR="$HOME/path/to/sf-datacloud-code-extn"   # <-- change this
_datacloud_venv_hook() {
  local venv="$DATACLOUD_PROJECT_DIR/.venv"
  if [[ "$PWD/" == "$DATACLOUD_PROJECT_DIR/"* ]]; then
    if [[ "$VIRTUAL_ENV" != "$venv" && -f "$venv/bin/activate" ]]; then
      source "$venv/bin/activate"
    fi
  elif [[ "$VIRTUAL_ENV" == "$venv" ]]; then
    deactivate
  fi
}
autoload -U add-zsh-hook
add-zsh-hook chpwd _datacloud_venv_hook
_datacloud_venv_hook
# ---- end Data Cloud ----
```

Then **open a new terminal tab** or run `source ~/.zshrc`. Terminals that were already open don't pick up the change, and that is the most common cause of the Python version error.

Using bash? Put the Java and pyenv lines in `~/.bashrc` and run `source .venv/bin/activate` yourself in each new terminal. The hook above is zsh-only.

## 4. Install the Salesforce CLI and connect your org

```bash
npm install --global @salesforce/cli                        # or see https://developer.salesforce.com/tools/salesforcecli
sf plugins install @salesforce/plugin-data-code-extension   # adds `sf data-code-extension`

sf org login web --alias agent_prod                         # opens a browser; use your Data Cloud org
sf org list                                                 # agent_prod should show as Connected
```

You can use any alias. Use the same one with `--target-org` / `--sf-cli-org` below. The CLI connection is all you need; `datacustomcode configure` (username/password or client credentials) is only an alternative.

## 5. Verify and run

From the project folder:

```bash
cd "$DATACLOUD_PROJECT_DIR"
which python3        # -> .../sf-datacloud-code-extn/.venv/bin/python3
python3 --version    # -> Python 3.11.x
java -version        # -> openjdk version "17..."
datacustomcode version

# run the script locally against org data
sf data-code-extension script run --entrypoint ./payload/entrypoint.py --target-org agent_prod
```

Other useful commands:

```bash
sf data-code-extension script --help     # scan / zip / deploy etc.
sf data-code-extension script scan       # refresh payload/config.json + requirements.txt after editing the entrypoint
datacustomcode run ./payload/entrypoint.py --sf-cli-org agent_prod   # SDK-native equivalent of the run above
```

**Before the first run:** the sample `entrypoint.py` from `init` reads `Account_std__dll`, and that table may not exist in your org. Change `client.read_dlo(...)` and any `write_to_dlo(...)` to Data Lake Objects that do exist. To list them:

```bash
sf api request rest "/services/data/v63.0/ssot/data-lake-objects" --target-org agent_prod
```

This endpoint returns at most 20 objects per page. Add `?offset=20` to get the next page.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Python version error, or `No matching distribution found for salesforce-data-customcode` | `python3` is 3.12+, so the venv isn't active | `source .venv/bin/activate` or open a new terminal; check with `which python3`. In the VS Code terminal, select `.venv/bin/python` via **Python: Select Interpreter** |
| `Unable to locate a Java Runtime` / `JAVA_GATEWAY_EXITED` | `JAVA_HOME` isn't set, so `/usr/bin/java` is the macOS placeholder | Step 2 + step 3, then `java -version` |
| `table "<name>__dll" does not exist` | The entrypoint references a DLO that isn't in the org | Use a DLO from the list above, then run `script scan` |
| `No authorization information found for agent_prod` | The org isn't logged in under that alias | `sf org login web --alias agent_prod` |

## Credentials and secrets

Never commit org credentials. `datacustomcode configure` stores credentials in your home directory (`~/.datacustomcode/`), not in this project. Using `--sf-cli-org <alias>` reuses your Salesforce CLI login instead. The `.gitignore` excludes `.env` files, key files, `.sf/`/`.sfdx/` folders, and build outputs such as `deployment.zip`.

The local JupyterLab container (`./jupyterlab.sh start`) runs without a token. It is bound to `127.0.0.1`, so only your own machine can reach it.
