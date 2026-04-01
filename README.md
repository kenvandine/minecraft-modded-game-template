# minecraft-modded-game-template

A template repository for creating a modded Minecraft game experience using the
[minecraft-server-snap](https://github.com/kenvandine/minecraft-server-snap) framework.

Push a tag → get a GitHub release with a server artifact, a Linux launcher, and a
Windows launcher — automatically.

---

## Using this template

Click **"Use this template"** → **"Create a new repository"** on GitHub, then follow
the steps below.

---

## Setup (5 minutes)

### Step 1 — Edit `pack.yaml`

Open `pack.yaml` and customize it for your game:

```yaml
name: "My Awesome Modpack"     # ← change this
version: "1.0.0"
minecraft_version: "1.21.1"
mod_loader: fabric
mod_loader_version: "latest"
installer_version: "latest"

mods:
  - name: "Fabric API"
    url: "https://cdn.modrinth.com/data/P7dR8mSH/versions/yGAe1owa/fabric-api-0.116.9%2B1.21.1.jar"
    side: both
  # Add your mods here
```

See the [pack YAML reference](https://github.com/kenvandine/minecraft-server-snap/blob/main/docs/pack-yaml-reference.md)
for all available fields.

### Step 2 — (Optional) Set up Microsoft auth

For players to sign in with their Microsoft account for online play, you need a free
Azure app registration. Follow the
[Azure setup guide](https://github.com/kenvandine/minecraft-server-snap/blob/main/docs/azure-setup.md)
(takes about 5 minutes), then add your client ID as a **GitHub Actions secret** —
do not put it in `pack.yaml` or commit it to the repo.

In your repository go to **Settings → Secrets and variables → Actions → New repository secret**:

| Name | Value |
|------|-------|
| `AZURE_CLIENT_ID` | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |

The build workflow injects it into the launcher at build time. If the secret is not
set, the launcher falls back to offline mode (works for LAN and `online-mode=false`
servers).

### Step 3 — Publish your first release

```bash
git add pack.yaml
git commit -m "Initial pack configuration"
git tag v1.0.0
git push origin main --tags
```

GitHub Actions will automatically:
1. Download the Fabric server JAR and your mods
2. Build `server.tar.xz`
3. Build a Linux AppImage launcher
4. Build a Windows exe installer
5. Create a GitHub release with all three artifacts

Watch the progress under **Actions** in your repository.

---

## Adding and updating mods

> See [docs/finding-mods.md](docs/finding-mods.md) for a full guide covering how to
> search Modrinth, verify compatibility, get stable URLs, check dependencies, and choose
> which `side` each mod belongs on.

Find the mod on [modrinth.com](https://modrinth.com), navigate to **Versions**, select
the version matching your `minecraft_version` and `fabric` loader, right-click the
download button and copy the URL.

Add it to `pack.yaml`:

```yaml
mods:
  - name: "Iris Shaders"
    url: "https://cdn.modrinth.com/data/YL57xq9U/versions/.../iris-1.8.0+mc1.21.1.jar"
    side: client
```

Then publish a new release:

```bash
# Edit pack.yaml — bump version, add/update mods
git add pack.yaml
git commit -m "Add Iris Shaders"
git tag v1.1.0
git push origin main --tags
```

---

## Installing on a server

Install the [minecraft-server snap](https://github.com/kenvandine/minecraft-server-snap):

```bash
sudo snap install minecraft-server
```

Then install your modpack from the release URL:

```bash
sudo minecraft-server.install-pack \
  https://github.com/YOUR-ORG/YOUR-REPO/releases/download/v1.0.0/server.tar.xz

sudo snap restart minecraft-server.server
sudo snap logs -f minecraft-server.server
```

---

## Playing

Direct players to your GitHub releases page. They download:

- **Linux**: `Your-Modpack-1.0.0.AppImage` → `chmod +x *.AppImage && ./*.AppImage`
- **Windows**: `Your-Modpack-Setup-1.0.0.exe` → run the installer

On first launch, the launcher automatically downloads Java 21, Minecraft, Fabric, and
installs the bundled mods (~500 MB, one time). After that, it's just click Play.

**Authentication:** If `azure_client_id` is set in your `pack.yaml`, players sign in
with Microsoft for online servers. If it's not set, a **Play Offline** username prompt
appears instead — this works for singleplayer, LAN, and `online-mode=false` servers.

---

## Local testing

### Option A — pre-built binary (quickest)

```bash
# Linux
curl -L -o /usr/local/bin/game-create \
  https://github.com/kenvandine/minecraft-server-snap/releases/latest/download/game-create-linux
chmod +x /usr/local/bin/game-create

game-create build pack.yaml --output ./dist-pack
```

### Option B — from source (no release needed)

```bash
git clone https://github.com/kenvandine/minecraft-server-snap
cd minecraft-server-snap

python3 -m venv .venv
source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -e tools/game-create/

# Run from your game repo
game-create build /path/to/pack.yaml --output ./dist-pack
```

### Test the server artifact

```bash
sudo minecraft-server.install-pack dist-pack/server.tar.xz
sudo snap restart minecraft-server.server
```

### Build and test the AppImage locally

```bash
# Clone the launcher template
git clone https://github.com/kenvandine/minecraft-server-snap
cd minecraft-server-snap/launcher

# Inject your client artifact
mkdir -p resources/mods
tar -xJf /path/to/dist-pack/client.tar.xz -C resources/

# Update app name (or edit electron-builder.yml directly)
sed -i 's/^productName:.*/productName: "My Awesome Modpack"/' electron-builder.yml
sed -i 's/^appId:.*/appId: "com.minecraft.my-awesome-modpack"/' electron-builder.yml

# Install and build
npm install
npm run build:linux             # → dist/*.AppImage

# Run (if FUSE is available)
./"My Awesome Modpack-1.0.0.AppImage"

# Run without FUSE
./"My Awesome Modpack-1.0.0.AppImage" --appimage-extract-and-run
```

---

## Repository structure

```
your-game/
├── pack.yaml                    Your modpack definition — edit this
├── .github/
│   └── workflows/
│       └── release.yml          Triggers on version tags → builds + publishes release
└── .gitignore
```

That's it. The build tools, launcher template, and CI logic all live in
[minecraft-server-snap](https://github.com/kenvandine/minecraft-server-snap).

---

## Workflow reference

The workflow in `.github/workflows/release.yml` calls the reusable workflow from
`minecraft-server-snap`. You generally don't need to edit it, but you can customize:

```yaml
jobs:
  release:
    uses: kenvandine/minecraft-server-snap/.github/workflows/reusable-pack-release.yml@main
    with:
      config: pack.yaml               # path to your YAML
      tag: ${{ github.ref_name }}     # release tag
      tools-version: "v0.2.0"        # pin game-create version (optional)
    secrets:
      GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

See [github-workflows.md](https://github.com/kenvandine/minecraft-server-snap/blob/main/docs/github-workflows.md)
for full documentation.

---

## Troubleshooting

**Workflow fails at "Build artifacts" — mod download error**
Check that all mod URLs in `pack.yaml` are valid direct download links to `.jar` files.
Open the URL in a browser — it should immediately download a JAR.

**Workflow fails at "Build launcher" — electron-builder error**
This is usually a Node.js version issue. The workflow uses Node 20 — ensure you haven't
accidentally pinned an older version in a workflow override.

**Players get "Failed to get Minecraft profile"**
The player authenticated with Microsoft but does not own Minecraft Java Edition.
The game requires a purchased copy.

**Players see "No Azure client ID configured"**
Add `azure_client_id` to `pack.yaml`. See the
[Azure setup guide](https://github.com/kenvandine/minecraft-server-snap/blob/main/docs/azure-setup.md).

**Server won't start after install-pack**
Run `sudo snap logs minecraft-server.server`. Common causes:
- Port 25565 already in use (another server running)
- Insufficient memory (`java_args` in pack.yaml requests more RAM than available)

**AppImage won't launch — "AppImages require FUSE"**
Run with `--appimage-extract-and-run` flag, or install FUSE:
```bash
sudo apt install libfuse2       # Ubuntu/Debian
./"My Game-1.0.0.AppImage" --appimage-extract-and-run
```

**Mod download fails with 404 during `game-create build`**
The mod URL in `pack.yaml` is stale. Modrinth CDN URLs are permanent per version, but
the version ID in the path must match an actual release. Fetch a fresh URL from
[modrinth.com](https://modrinth.com) → Versions → right-click download → Copy link.
