---
description: Pull the latest ECC repo changes and reinstall the current managed targets.
disable-model-invocation: true
---

# Auto Update

Update ECC from its upstream repo and regenerate the current context's managed install using the original install-state request.

## Usage

```bash
# Preview the update without mutating anything
ECC_ROOT="${CLAUDE_PLUGIN_ROOT:-$(node -e "var r=(function(){var p=require('path'),f=require('fs'),o=require('os');var e=process.env.CLAUDE_PLUGIN_ROOT;if(e&&e.trim())return e.trim();var d=p.join(o.homedir(),'.claude');function L(x){try{return require(p.join(x,'scripts','lib','resolve-ecc-root')).resolveEccRoot({probe:p.join('scripts','auto-update.js')})}catch(_){return null}}var r=L(d);if(r)return r;var s=['ecc','ecc@ecc','marketplaces/ecc','everything-claude-code','everything-claude-code@everything-claude-code','marketplaces/everything-claude-code'];for(var i=0;i<s.length;i++){r=L(p.join(d,'plugins',s[i]));if(r)return r}try{var g=['ecc','everything-claude-code'];for(var j=0;j<g.length;j++){var c=p.join(d,'plugins','cache',g[j]);var O=f.readdirSync(c);for(var k=0;k<O.length;k++){var q=p.join(c,O[k]);var V=f.readdirSync(q);for(var m=0;m<V.length;m++){r=L(p.join(q,V[m]));if(r)return r}}}}catch(_){}return d})();console.log(r)")}"
node "$ECC_ROOT/scripts/auto-update.js" --dry-run

# Update only Cursor-managed files in the current project
node "$ECC_ROOT/scripts/auto-update.js" --target cursor

# Override the ECC repo root explicitly
node "$ECC_ROOT/scripts/auto-update.js" --repo-root /path/to/everything-claude-code
```

## Notes

- This command uses the recorded install-state request and reruns `install-apply.js` after pulling the latest repo changes.
- Reinstall is intentional: it handles upstream renames and deletions that `repair.js` cannot safely reconstruct from stale operations alone.
- Use `--dry-run` first if you want to see the reconstructed reinstall plan before mutating anything.

<!-- ecc-fork:auto-update -->
## Fork override (villa313 fork with C# track)

When `~/.local/bin/ecc-sync` exists, run this instead of the Usage above: `auto-update.js` only refreshes the legacy `~/.claude` file-copy install, while `claude-per` (`~/.claude-per`) and `claude-acc` (`~/.claude-accrueme`) load the `ecc@everything-claude-code` plugin from the fork. Details in the fork's `FORK-MAINTENANCE.md`.

1. Run `ecc-sync --auto` in the background (~7 min). It merges `upstream/main` keeping the C# track, runs tests, ignores failures that also occur on pristine upstream, then pushes `origin/main` and updates the marketplace + plugin in both environments.
2. If it stops on new failures, show `ecc-sync --status` and ask before `ecc-sync --push`. If it stops on a real conflict, follow section 5 of `FORK-MAINTENANCE.md`.
3. Verify: `for d in ~/.claude-per ~/.claude-accrueme; do CLAUDE_CONFIG_DIR=$d claude plugin list | grep -A1 'ecc@'; done`, then tell the user to restart running sessions.
