# git-hooks
hooksを試すリポジトリ

## vscodeの設定
Code > Preferences > Settings > User(およびWorkspace) > Files: Exclude

.git を削除するとvscodeのExplorerで.git配下のファイルを扱うことができる

## スクリプトの配置
`.git/hooks/`配下に`pre-commit`ファイルを作成し、コミット前に実行したいスクリプトを記述

`pre-commit`
```sh
#!/bin/sh

# --no-pre-commit が指定されている場合はスキップ
if [ "$1" == "--no-pre-commit" ]; then
    echo "Skipping pre-commit hook"
    exit 0
fi

echo -e "pre-commit"

# ステージングされたフロントエンドの変更を探知し、prettierとfrontend-testを実行
changed_files=$(git diff --staged --name-only main | grep -E '\.ts$|\.tsx$')

if [ -n "$changed_files" ]; then
    if echo "$changed_files" | xargs npx prettier --check; then
        echo "Prettier check complete."
        yarn update-storyshot
    else
        echo "Prettier check failed. Aborting commit."
        if echo "$changed_files" | xargs npx prettier --write; then
            echo "git add and commit again."
        fi
        exit 1
    fi
else
    echo "No changes in TypeScript files found"
fi

# ステージングされたバックエンドの変更を探知し、rspecを実行
changed_files=$(git diff --staged --name-only | grep '\.rb$' | xargs -n 1 basename)

if [ -n "$changed_files" ]; then
    rspec_files=""
    for file in $changed_files; do
        spec_file="${file%.rb}_spec.rb"
        spec_paths=$(find spec -type f -name "$spec_file")
        if [ -n "$spec_paths" ]; then
            rspec_files="$rspec_files $spec_paths"
        else
            echo "RSpec file not found for $file"
        fi
    done
    if [ -n "$rspec_files" ]; then
        if ! bundle exec rspec $rspec_files; then
            echo "RSpec tests failed. Aborting commit."
            exit 1
        fi
    fi
else
    echo "No changed Ruby files found"
fi
```

## 実行権限を付与

```sh
chmod +x .git/hooks/pre-commit
```