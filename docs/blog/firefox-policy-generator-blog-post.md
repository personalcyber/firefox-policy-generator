**Excerpt:** Firefox Policy Generator (`ffpolicy`) turns `policies.json` from a hand-edited, easy-to-typo config file into a validated, form-driven policy set you can build in a GUI and ship the exact same way through Ansible or Puppet — deterministic output, compliance presets, and every standard Linux install location pre-mapped.

# Firefox Policy Generator: one source of truth for policies.json, from your desktop to your fleet

Firefox's enterprise policy engine is powerful — `policies.json` can lock down telemetry, force-install extensions, pin TLS versions, and dozens of other settings — but writing that file by hand means memorizing Mozilla's schema, getting nesting and casing exactly right, and hoping you didn't typo a policy name that Firefox will silently ignore. **Firefox Policy Generator** (`ffpolicy`) removes that guesswork. It's a small, open-source tool that gives you a real form-driven editor for `policies.json`, and it's just as happy running as a GUI on your laptop as it is running headless in an Ansible playbook or a Puppet manifest.

One codebase, three ways to use it:

- A **PySide6 desktop GUI** for building and tuning a policy set interactively
- A **headless CLI** for validation, generation, and export — the part that plugs into automation
- A **self-contained Linux AppImage** for distributing the GUI without asking anyone to set up a Python environment

The interesting part is that all three share the same validation and generation logic, so a policy set you sanity-checked in the GUI produces byte-identical output to the one your config management tool renders on 500 servers.

## Building a policy set without memorizing the schema

Open the GUI and you get three panes: a category tree of every policy Firefox supports on the left, a form for whichever policy you've selected in the middle, and a live JSON preview on the right that updates as you type.

![Firefox Policy Generator main window — category tree, policy editor, and live JSON preview](https://raw.githubusercontent.com/personalcyber/firefox-policy-generator/claude/blog-post-app-setup-deployment-2pp7ei/docs/blog/ffpolicy-main-window.png)

Every policy comes with an inline description of what it actually does, and — for the settings where it matters — a **Security & Privacy Impact** note explaining the tradeoff of changing it. `DisableTelemetry` only affects data sent to Mozilla; `SSLVersionMin` prevents protocol downgrade. You don't need the Mozilla policy-templates docs open in another tab to know what you're toggling.

![A policy's description and security/privacy impact note above its editor](https://raw.githubusercontent.com/personalcyber/firefox-policy-generator/claude/blog-post-app-setup-deployment-2pp7ei/docs/blog/ffpolicy-policy-description.png)

`ExtensionSettings` — arguably the highest-impact policy in the whole schema, since it decides which extensions can read and modify every page a user visits — gets its own dedicated manager instead of a generic form. Search AMO by name, paste an addons.mozilla.org listing URL directly, or type in a GUID and install URL by hand if a network restriction blocks the AMO API. There's always a manual fallback, so a flaky network never leaves you stuck.

![The Extension Settings manager: search AMO, paste a listing link, or add an extension manually](https://raw.githubusercontent.com/personalcyber/firefox-policy-generator/claude/blog-post-app-setup-deployment-2pp7ei/docs/blog/ffpolicy-extension-manager.png)

For a one-off local install, the GUI's **File → Export to standard location…** dialog picks the right path for you — every standard Linux install location Firefox reads from (the system-wide directory, the per-distro `distribution/` folder, the snap, the Flatpak extension mount point, or a custom path) — and flags whether the write needs elevated privileges, with a one-click retry via `pkexec`/`sudo` if it does.

![The export target dialog resolving the write path and privilege requirement for a chosen target](https://raw.githubusercontent.com/personalcyber/firefox-policy-generator/claude/blog-post-app-setup-deployment-2pp7ei/docs/blog/ffpolicy-export-dialog.png)

Don't want to build a policy set from scratch? `ffpolicy` ships **compliance presets** — DISA STIG, NIST SP 800-171, and a couple of balanced/privacy-focused home-user baselines — that apply a known-good starting point in one click (or one `--preset` flag), which you then layer your own settings on top of.

## From "it works on my machine" to "it works on every machine"

This is where `ffpolicy` earns its keep for anyone managing Firefox across a fleet with Ansible or Puppet. The GUI is for *building* a policy set; the CLI is for *shipping* it. Both read the exact same YAML/JSON input format, so the file you tuned by hand in the GUI is the same file your playbook checks into version control and renders out to every host.

![Validating, generating, and exporting a policy set from the CLI — the same commands a playbook or manifest would run](https://raw.githubusercontent.com/personalcyber/firefox-policy-generator/claude/blog-post-app-setup-deployment-2pp7ei/docs/blog/ffpolicy-cli-ansible.png)

A typical workflow looks like this:

```bash
# Validate the shared YAML source of truth (catches typoed policy names,
# missing install_url on a force-installed extension, version mismatches)
ffpolicy validate my-policies.yaml

# Render it to a policies.json Ansible/Puppet can template out directly
ffpolicy export my-policies.yaml \
    --target custom --custom-path roles/firefox/files/policies.json

# Or apply a compliance baseline with zero manual editing — ideal for CI
ffpolicy generate --preset disa_stig__mac_1_classified -o policies.json
```

A few details that make this genuinely low-friction for automation:

- **Deterministic output.** `ffpolicy` always sorts JSON keys, so re-running `generate` on an unchanged input produces a byte-identical file. That means your Ansible `copy`/`template` task (or Puppet `file` resource) reports "no change" when nothing actually changed, instead of flapping on every run because of key ordering.
- **`--offline` mode.** CI runners without outbound network access can skip the live Mozilla schema sync entirely and validate against the bundled schema fallback — no flaky pipeline failures because a schema fetch timed out.
- **Every standard Linux install location, pre-mapped.** `--target` already knows the right path for Fedora/RHEL, Debian/Ubuntu, ESR, manual tarball installs, the Firefox snap, and both system-wide and per-user Flatpak installs — so your role doesn't need its own lookup table of "which distro puts policies.json where."
- **`discover` and `import` close the loop.** If Firefox is already deployed somewhere, `ffpolicy discover` finds any existing `policies.json`, and `ffpolicy import` turns it back into an editable YAML file — handy for adopting `ffpolicy` on a fleet that already has policies deployed by hand.

Drop `ffpolicy validate` and `ffpolicy generate` into a pre-merge CI check on the repo holding your Ansible role or Puppet module, and a bad policy edit gets caught before it ever reaches a `file`/`template` resource — no more finding out a policy name was misspelled only after it silently did nothing on 500 machines.

## Try it

```bash
pip install -e ".[dev]"
python -m ffpolicy          # GUI
python -m ffpolicy --help   # CLI
```

Firefox Policy Generator is open source — see the project's `README.md` for the full command reference, the export target table, and the bundled compliance presets.
