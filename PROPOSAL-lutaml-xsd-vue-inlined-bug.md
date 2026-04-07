# Proposal: Fix vue_inlined mode when frontend/dist is not built

## Problem

When using `lutaml-xsd doc spa <package.lxr> --mode vue_inlined` after cloning the repository (without building the frontend), the generated HTML has empty JavaScript:

```html
<!-- Embedded JavaScript -->
<script>

</script>
```

This is because:

1. `frontend/dist/` is gitignored (in `.gitignore`)
2. The `vue_inlined_strategy.rb` reads from `frontend/dist/` to embed the Vue app
3. When cloning, the dist files don't exist, so the strategy silently produces empty output

## Root Cause

In `lib/lutaml/xsd/spa/strategies/vue_inlined_strategy.rb`:

```ruby
def embed_app
  return "" unless File.exist?(app_js_path)

  # ... embeds the JS
end
```

When the file doesn't exist, it returns empty string instead of raising an error or building the frontend.

## Proposed Solutions

### Option 1: Auto-build frontend when dist is missing (Recommended)

Modify `vue_inlined_strategy.rb` to automatically build the frontend if `frontend/dist/` is missing:

```ruby
def embed_app
  unless File.exist?(app_js_path)
    warn "frontend/dist/ not found. Building frontend..."
    system("npm install && npm run build", chdir: frontend_dir)
  end
  return "" unless File.exist?(app_js_path)

  # ... embeds the JS
end
```

### Option 2: Raise an explicit error

If `frontend/dist/` is missing and mode is `vue_inlined`, raise a clear error explaining the user needs to build the frontend first.

### Option 3: Check in dist files

Un-ignore `frontend/dist/` and commit the built files. However, this creates maintenance overhead and can lead to stale artifacts.

## Recommendation

**Option 1** is recommended because it provides the best UX - users can clone and immediately run `lutaml-xsd doc spa` without manual frontend builds.

## Files to Modify

- `lib/lutaml/xsd/spa/strategies/vue_inlined_strategy.rb`
- Possibly `lib/lutaml/xsd/spa/generator.rb` (if shared initialization logic is needed)

## Verification

After the fix:
1. Fresh clone of lutaml-xsd
2. `bundle install`
3. `lutaml-xsd doc spa <package> --mode vue_inlined --output output.html`
4. `output.html` should contain embedded Vue app JavaScript (not empty)
