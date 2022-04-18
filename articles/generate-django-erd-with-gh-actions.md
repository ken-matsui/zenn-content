---
title: "Django の ER 図を GitHub Actions で自動生成する"
emoji: "📝"
type: "tech"
topics: ["django", "er図", "githubactions", "github"]
published: true
---

会社でプロダクト開発をしていると、ER 図が要求されることがあります。
そういった時に、最新のコミットを元に Django のソースコードから ER 図を CI 上で生成していればそれを渡すだけで済むため、非常に楽です。

度重なる仕様変更で、draw.io 等に作っておいた ER 図は古くなってしまっているでしょうから、ソースコードが最も正しいものになります。
そこから生成した ER 図であれば、渡すものとしては完璧です。

[`django-extensions`](https://github.com/django-extensions/django-extensions) というライブラリには [`graph_models`](https://github.com/django-extensions/django-extensions#using-it) というコマンドがあり、それを使うと、モデルの定義を元に `graphviz` を使用して、いい感じに ER 図を生成してくれます。

# 必要な依存関係を追加する

`django-extensions` と `pygraphviz` を追加します。

```toml: pyproject.toml
[tool.poetry.dependencies]
...
django-extensions = "^3.1.5"
pygraphviz = { version = "^1.7", optional = true }  # groups = ["erd"]

[tool.poetry.extras]
erd = ["pygraphviz"]
```

上記のように、optional dependencies として追加しておくと、不要な時にインストールしなくて済みます。

`settings.py` に `django-extensions` を追加します。

```python: settings.py
INSTALLED_APPS = (
    ...
    'django_extensions',
    ...
)
```

# ER 図を自動生成する GitHub Actions を作成する

以下の GitHub Actions をコピペすると、自動生成できます。

```yaml: .github/workflows/erd.yml
name: ERD

on:
  push:
    branches: [ main ]

jobs:
  generate:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3.0.0

    - name: Set up Graphviz
      uses: ts-graphviz/setup-graphviz@v1

    - name: Set up Python 3.10
      uses: actions/setup-python@v3.1.1
      with:
        python-version: "3.10"

    - name: Set up Poetry 1.1.11
      uses: abatilo/actions-poetry@v2.1.4
      with:
        poetry-version: 1.1.11

    - name: Install dependencies
      run: poetry install --no-interaction -E erd --no-dev

    - name: Generate ERD
      run: python manage.py graph_models -a -g -o erd.png

    - name: Upload the ERD image
      uses: actions/upload-artifact@v3
      with:
        name: erd
        path: erd.png
```

上記を実行すると、ER 図が artifact としてアップロードされます。
そのため、以下のように GitHub Actions の Summary ページの Artifacts 欄にて、アップロードした内容が表示されます。

![Artifacts](/images/generate-django-erd-with-gh-actions/artifacts.png)

`erd` と表示されている部分をクリックすると、`zip` 形式としてファイルがダウンロードできるため、それを解凍すると、`erd.png` が出てきます。

# 最後に

こちらの記事で紹介している方法は、Artifacts にアップロードするため、Storage の使用量を消費します。

![Storage for Actions](/images/generate-django-erd-with-gh-actions/storage.png)

上記の GitHub Actions は main ブランチに push がある度に走るため、ストレージを逼迫させやすいです。
ストレージを逼迫させたくない場合は、GitHub Actions を cron で動かすのも良いかもしれません。

https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#schedule

もしくは、変更したファイルをトリガーにするのも良いかと思います。

https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#running-your-workflow-only-when-a-push-affects-specific-files

この記事が参考になれば、幸いです。
