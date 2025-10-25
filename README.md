 git add -A && git commit -m "feat(payments): add stripe webhooks" && git push
#!/usr/bin/env bash
# git-commit.sh - simple conventional commits helper

set -e

types=("feat" "fix" "perf" "docs" "chore" "refactor" "test" "build")
echo "select commit type:"
select t in "${types[@]}"; do
  if [[ -n "$t" ]]; then
    type="$t"
    break
  fi
done

read -p "scope (optional, e.g. auth, api): " scope
read -p "short description (imperative, <=72 chars): " subject

echo
read -p "detailed body? (y/N): " want_body
body=""
if [[ "$want_body" =~ ^([yY])$ ]]; then
  echo "Enter body (end with a single '.' on a line):"
  body_lines=()
  while IFS= read -r line; do
    [[ "$line" == "." ]] && break
    body_lines+=("$line")
  done
  body="$(printf "%s\n" "${body_lines[@]}")"
fi

read -p "is this a breaking change? (y/N): " breaking
breaking_text=""
if [[ "$breaking" =~ ^([yY])$ ]]; then
  read -p "breaking change description: " bc_desc
  breaking_text="BREAKING CHANGE: $bc_desc"
fi

# build subject with optional scope
if [[ -n "$scope" ]]; then
  subj="$type($scope): $subject"
else
  subj="$type: $subject"
fi

# show preview
echo
echo "----- commit preview -----"
echo "$subj"
echo
if [[ -n "$body" ]]; then
  echo "$body"
  echo
fi
if [[ -n "$breaking_text" ]]; then
  echo "$breaking_text"
  echo
fi
echo "--------------------------"
read -p "proceed to git add/commit/push? (Y/n): " confirm
confirm=${confirm:-Y}
if [[ ! "$confirm" =~ ^([yY])$ ]]; then
  echo "aborted."
  exit 1
fi

# stage all changes
git add -A

# create commit message file to handle newlines
commit_msg_file=$(mktemp)
{
  echo "$subj"
  echo
  if [[ -n "$body" ]]; then
    echo "$body"
    echo
  fi
  if [[ -n "$breaking_text" ]]; then
    echo "$breaking_text"
  fi
} > "$commit_msg_file"

git commit -F "$commit_msg_file" || { echo "git commit failed"; rm -f "$commit_msg_file"; exit 1; }
rm -f "$commit_msg_file"

# push current branch
branch=$(git rev-parse --abbrev-ref HEAD)
git push origin "$branch"

echo "✅ committed & pushed: $subj"
