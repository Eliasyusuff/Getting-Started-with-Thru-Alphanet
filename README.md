# Getting-Started-with-Thru-Alphanet
This repo is based on Unto Labs' official Thru onboarding guide, plus fixes for a number of bugs encountered during setup as of July 2026 (CLI v0.2.38 / v0.2.39). If you hit an error following the official guide, check the Debug section below before assuming you did something wrong — some of these are upstream problems, not user error.

If you just want to get unblocked fast, jump to fix-thru-toolchain.sh, which automates the steps below.

What is Thru
Thru is a high-performance L1 built by Unto Labs, focused on ultra-fast transactions and low latency. It runs on ThruVM, a custom VM built on the RISC-V instruction set. Smart contracts can be written in C, C++, Rust, or any LLVM-supported language.

Prerequisites
Linux VM (Ubuntu 24.04 LTS verified) or macOS. Windows via WSL2.
~3GB free disk (1.1GB for the RISC-V toolchain, plus build artifacts)
sudo access
Node.js 18+ and npm
OpenSSL, curl, tar, jq, make, standard build tools.

This is mainly the MacOS setup

First, confirm:
uname -s
If it returns Darwin, run:

xcode-select --install

Then install Homebrew, if you do not already have it:

/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

After installation, follow the Homebrew terminal instructions to add brew to your PATH. Then run:

brew update
brew install node curl jq openssl.

npm is included with Node.js. macOS already provides tar and make, while the Xcode Command Line Tools replace Ubuntu’s build-essential.

Verify everything:

node -v
npm -v
curl --version
jq --version
openssl version
make --version
tar --version

Do not run the original sudo apt install command on macOS.

1. Add Homebrew to your terminal

Run:

echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"

nstall the required packages

Once brew update completes:

brew install node curl jq openssl

You do not need to install npm separately because it comes with Node.js. Then verify:

node -v
npm -v
curl --version
jq --version
openssl version

# Install Thru CLI
npm install -g thru
thru --version
First run auto-configures ~/.thru/cli/config.yaml with a random keypair named default.

# List/inspect keys
thru --json keys list
thru --json keys get default
⚠️ The value field in keys get default is your PRIVATE KEY in plaintext. Never use it as your public address.

# Create your on-chain account
thru --json account create default

# Get your public key correctly
YOUR_PUBKEY=$(thru --json getaccountinfo "$(thru --json keys list | jq -r '.keys[0].pubkey // empty')" 2>/dev/null)
# Safer: just copy the pubkey printed by `account create default` directly.

# Verify + fund
thru --json getaccountinfo "$YOUR_PUBKEY"
thru --json faucet withdraw default 10000
thru --json getbalance "$YOUR_PUBKEY"

Installing the Toolchain & SDK
thru dev toolchain install
thru dev sdk install c

Building & Deploying a Program
mkdir -p ~/thru-projects && cd ~/thru-projects
thru dev init c my-first-thru-program --path ~/thru-projects
cd my-first-thru-program

make lib   #build the SDK archive explicitly first
make       # Then link

thru uploader upload default build/thruvm/bin/my_first_thru_program_c.bin

Minting a Token
YOUR_PUBKEY="<your actual public address, NOT keys get default>"
MINT_SEED=$(openssl rand -hex 32)

thru --json token initialize-mint "$YOUR_PUBKEY" CAT "$MINT_SEED" --decimals 6 | tee /tmp/mint.json
MINT=$(jq -r '.token_initialize_mint.mint_account' /tmp/mint.json)

ACCT_SEED=$(openssl rand -hex 32)
thru --json token initialize-account "$MINT" "$YOUR_PUBKEY" "$ACCT_SEED" | tee /tmp/acct.json
TOKEN_ACCT=$(jq -r '.token_initialize_account.token_account' /tmp/acct.json)

# 1,000 tokens x 10^6 decimals
thru --json token mint-to "$MINT" "$TOKEN_ACCT" "$YOUR_PUBKEY" 1000000000

Name Service
ROOT_NAME="myroot$(date +%s | tail -c 5)"
thru --json nameservice init-root "$ROOT_NAME" | tee /tmp/root.json
ROOT_REGISTRAR=$(jq -r '.nameservice_init_root.registrar_account' /tmp/root.json)

thru --json nameservice register-subdomain yourname "$ROOT_REGISTRAR" | tee /tmp/subdomain.json
DOMAIN_ACCT=$(jq -r '.nameservice_register_subdomain.domain_account' /tmp/subdomain.json)


sleep 3
thru --json nameservice append-record "$DOMAIN_ACCT" url "https://example.com"
sleep 3
thru --json nameservice append-record "$DOMAIN_ACCT" com.twitter "@yourhandle"
sleep 3
thru --json nameservice append-record "$DOMAIN_ACCT" thru.pubkey "$YOUR_PUBKEY"

thru --json nameservice resolve "$DOMAIN_ACCT" --key thru.pubkey.


Issues and Fixes

1. thru dev toolchain install / thru dev sdk install c fail on v0.2.39
Symptom:

Error: Failed to download toolchain: Toolchain not found for Linux-x86_64 in release v0.2.39.
Available assets: thru-cli-Linux-x86_64-v0.2.39.tar.gz, thru_0.2.39_amd64.deb, ...
Cause: The v0.2.39 GitHub release (published July 2026) only includes thru-cli-* / .deb / .rpm assets. The thru-toolchain-* and thru-program-sdk-c-* archives present in every prior release (v0.2.35–v0.2.38) are missing from this release. This happens even if your installed CLI is pinned to an older version — toolchain install/sdk install always fetch GitHub's latest release tag, not the version matching your CLI.

Fix: Manually download the last complete release's assets:

mkdir -p ~/.thru/sdk/toolchain ~/.thru/sdk/c

curl -L -o /tmp/toolchain.tar.gz \
  "https://github.com/Unto-Labs/thru/releases/download/v0.2.38/thru-toolchain-Linux-x86_64-v0.2.38.tar.gz"
tar -xzf /tmp/toolchain.tar.gz -C ~/.thru/sdk/toolchain --strip-components=1

curl -L -o /tmp/sdk-c.tar.gz \
  "https://github.com/Unto-Labs/thru/releases/download/v0.2.38/thru-program-sdk-c-v0.2.38.tar.gz"
tar -xzf /tmp/sdk-c.tar.gz -C ~/.thru/sdk/c
(Check current releases at https://github.com/Unto-Labs/thru/releases — this may already be patched by the time you read this.)

2. SDK archive extracts with a mismatched folder name
Symptom: After extracting the SDK tarball plainly (no --strip-components), you get a folder named thru-program-sdk-c-v0.2.38/ instead of the thru-sdk/ the build system expects (GNUmakefile:6: .../thru-sdk/thru_c_program.mk: No such file or directory).

Fix: Rename the extracted folder:

mv ~/.thru/sdk/c/thru-program-sdk-c-v0.2.38 ~/.thru/sdk/c/thru-sdk
3. Compiler can't find thru-sdk/c/tn_sdk.h
Symptom:

examples/my_first_thru_program.c:5:10: fatal error: thru-sdk/c/tn_sdk.h: No such file or directory
Cause: The SDK headers #include <thru-sdk/c/tn_sdk.h> using a self-referencing path — the build passes -I .../thru-sdk/include, and the source expects a thru-sdk/ folder inside that include/ directory, pointing back at the SDK root. A plain extraction doesn't create this.

Fix:

mkdir -p ~/.thru/sdk/c/thru-sdk/include
ln -s ~/.thru/sdk/c/thru-sdk ~/.thru/sdk/c/thru-sdk/include/thru-sdk

4. Linker fails with cannot find -ltn_sdk
Symptom:

.../ld: cannot find -ltn_sdk: No such file or directory
collect2: error: ld returned 1 exit status
Cause: The SDK's Local.mk declares the libtn_sdk.a archive via a custom Make macro (make-lib), but the final link target has no dependency edge on that archive — so make (with or without -j) can jump straight to linking before the archive step ever runs.

Fix: Build the library explicitly first, then link:

make lib
make
5. keys get default returns your PRIVATE KEY, not your address
Symptom: Token/nameservice calls fail with cryptic VM reverts (e.g. TokenError: 7) when you pass the value from thru --json keys get default as $YOUR_PUBKEY.

Cause: keys get default's value field is your private key in plaintext — it is not your public address. Using it as a pubkey argument silently breaks signer/authority checks in the token program.

Fix: Use the actual public address returned by account create default or shown via getaccountinfo/getbalance (format: ta... / taw...), never the output of keys get default.

Support My Work
If this guide helped you, a ⭐ on the repository and sharing it with others is greatly appreciated. Feedback, corrections, and pull requests are always welcome.

Official Links
Website: https://thru.org
Documentation: https://docs.thru.org
X (Twitter): https://x.com/thru_xyz
Careers: https://jobs.ashbyhq.com/unto-labs

Made with ❤️ by Elias

Connect on X(Twitter) : https://x.com/EliasYusuff
