# `cwd` — Current Working Directory

A fast, single-line Bash utility that shows your current folder as a clean breadcrumb with subtle context tags.

## Install

```bash
mkdir -p ~/.local/bin

cat > ~/.local/bin/cwd <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

RED=$'\033[1;31m'
DIM=$'\033[2;37m'
GRAY=$'\033[90m'
BOLD=$'\033[1;37m'
CYAN=$'\033[0;36m'
YELLOW=$'\033[0;33m'
RESET=$'\033[0m'

dir="${PWD:-$(pwd)}"
home="${HOME:-}"

printf '\n'

if [[ -n "$home" && "$dir" == "$home" ]]; then
    printf '  %s~%s' "$RED" "$RESET"
else
    if [[ -n "$home" && "$dir" == "$home"/* ]]; then
        printf '  %s~%s' "$RED" "$RESET"
        rel="${dir#"$home"/}"
    else
        printf '  %s/%s' "$RED" "$RESET"
        rel="${dir#/}"
    fi

    IFS='/' read -r -a parts <<< "$rel"
    clean_parts=()
    for part in "${parts[@]}"; do
        [[ -n "$part" ]] && clean_parts+=("$part")
    done

    last=$(( ${#clean_parts[@]} - 1 ))
    for i in "${!clean_parts[@]}"; do
        part="${clean_parts[$i]}"
        if (( i == last )); then
            printf ' %s›%s %s%s%s' "$GRAY" "$RESET" "$BOLD" "$part" "$RESET"
        else
            printf ' %s›%s %s%s%s' "$GRAY" "$RESET" "$DIM" "$part" "$RESET"
        fi
    done
fi

tags=()

if [[ -n "$home" && "$dir" == "$home" ]]; then
    tags+=("home")
elif [[ -n "$home" && "$dir" == "$home"/Downloads* ]]; then
    tags+=("downloads")
elif [[ -n "$home" && "$dir" == "$home"/Desktop* ]]; then
    tags+=("desktop")
elif [[ -n "$home" && "$dir" == "$home"/Documents* ]]; then
    tags+=("documents")
elif [[ "$dir" == /etc* ]]; then
    tags+=("system config")
elif [[ "$dir" == /var/log* ]]; then
    tags+=("logs")
elif [[ "$dir" == /tmp* || "$dir" == /var/tmp* ]]; then
    tags+=("temp")
fi

[[ -f "go.mod" ]]                                                   && tags+=("go")
[[ -f "Cargo.toml" ]]                                               && tags+=("rust")
[[ -f "package.json" ]]                                             && tags+=("node")
[[ -f "pyproject.toml" || -f "requirements.txt" || -f "setup.py" ]] && tags+=("python")
[[ -f "Dockerfile" || -f "docker-compose.yml" ]]                    && tags+=("docker")

if git rev-parse --is-inside-work-tree >/dev/null 2>&1; then
    branch="$(git symbolic-ref --short HEAD 2>/dev/null || git rev-parse --short HEAD 2>/dev/null || echo "HEAD")"
    dirty=""
    [[ -n "$(git status --porcelain 2>/dev/null)" ]] && dirty="*"
    tags+=("${CYAN}git:${branch}${dirty}${GRAY}")
fi

files=0
dirs=0
shopt -s nullglob dotglob
for entry in ./*; do
    base="${entry#./}"
    [[ "$base" == "." || "$base" == ".." ]] && continue
    if [[ -d "$entry" && ! -L "$entry" ]]; then
        ((dirs++)) || true
    elif [[ -e "$entry" ]]; then
        ((files++)) || true
    fi
done
shopt -u nullglob dotglob
tags+=("${files}f ${dirs}d")

[[ ! -w "$dir" ]] && tags+=("${YELLOW}ro${GRAY}")

if (( ${#tags[@]} > 0 )); then
    joined="${tags[0]}"
    for (( i=1; i<${#tags[@]}; i++ )); do
        joined+=" · ${tags[$i]}"
    done
    printf '  %s(%b)%s' "$GRAY" "$joined" "$RESET"
fi

printf '\n\n'
EOF

chmod +x ~/.local/bin/cwd
```

## Output

```text
  ~ › Downloads › ChessHv3-main › analyze › assets  (downloads · 4f 2d)
  ~ › tools › corsight  (python · git:main* · 9f 3d)
  / › etc › nginx  (system config · 6f 3d · ro)
```

## Legend

* **Path**: Red `~` or `/`, dimmed parent folders, **bold white** active folder.
* **Location / Stack**: `home`, `downloads`, `desktop`, `documents`, `system config`, `logs`, `temp`, `go`, `rust`, `node`, `python`, `docker`.
* **Git**: `git:<branch>` (`*` = uncommitted changes).
* **Counts & Access**: `f` = file, `d` = dir · `ro` = read-only.
