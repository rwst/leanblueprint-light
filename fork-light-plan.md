# Implementation Plan: Documentation-Free Lean Blueprint

This document outlines the changes needed to fork `leanblueprint` and create a variant that DOES NOT build the Mathlib documentation locally, removing the doc-gen4 dependency and the hover features that rely on it.

## 1. Modifications to `leanblueprint/client.py`
The project creation script (`leanblueprint new`) currently forces/prompts for `doc-gen4` integration. We will remove this to ensure the project doesn't try to pull or build Mathlib's documentation infrastructure.

*   **Remove `doc-gen4` Injection:**
    *   In the `LakefileLean` class, delete the `add_docgen` method.
    *   In the `LakefileToml` class, delete the `add_docgen` method.
*   **Remove CLI Prompts for `doc-gen4`:**
    *   In the `new()` function, remove the prompt section asking: `"Modify lakefile and lake-manifest to allow building the documentation?"`.
    *   Remove the corresponding `lake update doc-gen4` subprocess command.

## 2. Modifications to `leanblueprint/Packages/blueprint.py`
The PlasTeX template uses a hover feature to display Lean declarations. Because we are removing the documentation build, we should strip out the tooltips that expect local HTML documentation to be present.

*   **Simplify Lean Links Template:**
    *   Modify `LEAN_LINKS_TPL`. Instead of generating a `<div class="tooltip">` with a list of links (which rely on the local `#doc/` router), simplify it to a basic text or label, or link directly to the global Mathlib documentation (`https://leanprover-community.github.io/mathlib4_docs/find/?pattern=...`).
    *   *Example Change:*
        ```html
        <!-- Remove this: -->
        <div class="tooltip">
            <span class="lean_link">Lean</span>
            <ul class="tooltip_list">...
        <!-- Replace with a simple inline label or a direct global link. -->
        ```
*   **Simplify Lean Declarations Template:**
    *   Modify `LEAN_DECLS_TPL` to remove the modal dialog entirely or replace the URLs with global Mathlib paths, avoiding the assumption of a locally hosted `doc-gen4` site.

## 3. Modifications to `leanblueprint/templates/blueprint.yml`
The generated GitHub Actions CI workflow currently uses `leanprover-community/docgen-action`. This action intrinsically triggers a build of Mathlib documentation via `doc-gen4`. We need to replace this action with raw commands to build *only* the blueprint and check the declarations without invoking `doc-gen4`.

*   **Replace `docgen-action` in the `Compile blueprint and documentation` step:**
    *   Instead of using `leanprover-community/docgen-action`, write explicit run commands:
        ```yaml
        - name: Install dependencies
          run: |
            sudo apt-get update
            sudo apt-get install -y graphviz texlive texlive-latex-extra latexmk
            pip install leanblueprint
        - name: Build Blueprint PDF
          run: leanblueprint pdf
        - name: Build Blueprint Web
          run: leanblueprint web
        - name: Check Declarations
          run: leanblueprint checkdecls
        ```
*   **Adjust Home Page Deployment:**
    *   Ensure the GitHub action still deploys the `web`, `print`, and `home_page` folders to `gh-pages` using the standard `actions/upload-pages-artifact` without waiting for `doc-gen4` output.

## 4. Renaming to `leanblueprint-light`
If you wish to rename the project to `leanblueprint-light` in your fork, you must adjust both the Python package name and the importable module name. Python modules cannot contain hyphens, so the folder must use an underscore (`leanblueprint_light`).

1. **Rename the Python Module Directory:**
   In order to ensure that the file history and original author attributions for the files are 100% kept intact within Git, use `git mv` instead of a regular bash `mv`. Git is designed to preserve file history even across directory renames.
   ```bash
   git mv leanblueprint leanblueprint_light
   ```

2. **Update `setup.cfg`:**
   Change the package configuration to use the new names:
   ```ini
   [metadata]
   name = leanblueprint-light
   # ...
   [options]
   packages = 
     leanblueprint_light
     leanblueprint_light.Packages
   # ...
   [options.entry_points]
   console_scripts = 
     leanblueprint-light = leanblueprint_light.client:safe_cli
   ```

3. **Update `MANIFEST.in`:**
   Replace all instances of `leanblueprint/` with `leanblueprint_light/`.

4. **Update PlasTeX Plugin References:**
   - In `leanblueprint_light/templates/plastex.cfg`, change `plugins=... leanblueprint` to `leanblueprint_light`.
   - In `leanblueprint_light/templates/macros/common.tex`, update any comments regarding the plugin name.

5. **Update CLI and Build Scripts:**
   - In `leanblueprint_light/templates/blueprint.yml`, replace `pip install leanblueprint` with `pip install leanblueprint-light` and `leanblueprint pdf/web/checkdecls` with `leanblueprint-light`.
   - In `README.md` and `plan.md`, replace usages of `leanblueprint` with `leanblueprint-light` (for CLI and PIP) or `leanblueprint_light` (for Python imports).

## Summary of Benefits
By implementing these changes, the forked variant will:
1. Compile significantly faster (no need to compute and render HTML for all Mathlib lemmas).
2. Avoid Lake configuration conflicts with `doc-gen4`.
3. Save extensive CI minutes on GitHub Actions.
4. Provide a lighter, standalone blueprint experience.
