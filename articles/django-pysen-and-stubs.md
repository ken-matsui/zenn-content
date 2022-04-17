---
title: "Django × Pysen × Stubs"
emoji: "🧩"
type: "tech"
topics: ["django", "pysen", "python", "poetry"]
published: true
---

以下に挙げている、様々なものを組み合わせようとするとかなり大変だったので、備忘録としてこちらの記事にさせていただきます。

* [`Django`](https://www.djangoproject.com/)
* [`Pysen`](https://github.com/pfnet/pysen)
* [`django-configurations`](https://django-configurations.readthedocs.io/en/stable/)
* [`django-stubs`](https://github.com/typeddjango/django-stubs)
* [`djangorestframework-stubs`](https://github.com/typeddjango/djangorestframework-stubs)

前提として、Django プロジェクトのセットアップ済みで、Poetry は導入済みであることとして、進めさせていただきます。

# 1. [`Django`](https://www.djangoproject.com/) × [`Pysen`](https://github.com/pfnet/pysen)

まずは、Django プロジェクトに Pysen を導入します。

```bash
$ poetry add -D pysen[lint]
```

`pyproject.toml`

```toml
[tool.poetry.dev-dependencies]
pysen = {version = "0.9.1", extras = ["lint"]}

[tool.pysen]
version = "0.9"

[tool.pysen.lint]
enable_black = true
enable_flake8 = true
enable_isort = true
enable_mypy = true
mypy_preset = "strict"
line_length = 88
py_version = "py38"
mypy_ignore_packages = ["*.migrations.*"]

[[tool.pysen.lint.mypy_targets]]
  paths = ["."]

[tool.pysen.lint.source]
  includes = ["."]
  exclude_globs = ["**/migrations/*.py"]
```

# 2. `1` × [`django-configurations`](https://django-configurations.readthedocs.io/en/stable/)

Pysen に `django-configurations` のデフォルト値を設定することで、エラーになることを防ぎます。
以下のファイルを作成してください。

`pysen_setup/base_mypy_setup.py`

```python
# ref: https://github.com/typeddjango/django-stubs/pull/180#issuecomment-820062352

import os
from typing import Any

from configurations.importer import install


def plugin(main: Any, version: str) -> Any:
    os.environ.setdefault("DJANGO_SETTINGS_MODULE", "myapi.settings")
    os.environ.setdefault("DJANGO_CONFIGURATION", "Local")
    install()
    return main.plugin(version)
```

Pysen に作成したファイルを認識させるために、`pyproject.toml` を編集します。

`pyproject.toml`

```diff toml
+[tool.pysen-cli]
+settings_dir = "pysen_setup"
+
[tool.pysen]
version = "0.9"

[tool.pysen.lint]
enable_black = true
enable_flake8 = true
enable_isort = true
enable_mypy = true
mypy_preset = "strict"
line_length = 88
py_version = "py38"
mypy_ignore_packages = ["*.migrations.*"]

+[[tool.pysen.lint.mypy_plugins]]
+  script = "./base_mypy_setup.py"  # relative path from tool.pysen-cli.settings_dir
+
[[tool.pysen.lint.mypy_targets]]
  paths = ["."]

[tool.pysen.lint.source]
  includes = ["."]
  exclude_globs = ["**/migrations/*.py"]
```

# 3. `2` × [`django-stubs`](https://github.com/typeddjango/django-stubs)

`django-stubs` を Pysen に読み込ませるには、mypy の設定ファイルを編集する必要があります。

https://github.com/typeddjango/django-stubs#installation

> To make mypy aware of the plugin, you need to add
>
> ```ini
> [mypy]
> plugins =
>     mypy_django_plugin.main
>
> [mypy.plugins.django-stubs]
> django_settings_module = "myproject.settings"
> ```

Pysen は `tool.pysen-cli.settings_dir` で指定されたディレクトリ内に、mypy 等の必要な設定ファイルを自動で生成し、それを読み込ませることで複数 Linter の統合を実現しています。
その生成されるファイルを編集する必要があるため、例えば `mypy.ini` を作成したとしてもそれを読みに行ってはくれません。
`django-stubs` の設定は、`pyproject.toml` に書くこともできるみたいですが、恐らく Pysen はそちらに対応していないのでは無いかと思います。

そこで、Pysen のプラグインを作成して、`django-stubs` 用の `mupy.ini` を生成するように設定します。

以下を参考に作ってみました。

https://github.com/pfnet/pysen/blob/66fb2c1dd6854c149224c53c4c01fbbc8f473a8f/examples/plugin_example/plugin.py

`pysen_setup/plugin.py`

```python
import dataclasses
import pathlib
from typing import Any, DefaultDict, Dict, Sequence, Tuple

import dacite
from pysen.command import CommandBase
from pysen.component import ComponentBase, RunOptions
from pysen.mypy import _SettingFileName as MypySettingFileName
from pysen.plugin import PluginBase
from pysen.pyproject_model import Config, PluginConfig
from pysen.reporter import Reporter
from pysen.runner_options import PathContext
from pysen.setting import SettingFile

_SettingFileName = MypySettingFileName


class DjangoStubsCommand(CommandBase):
    def __init__(self, name: str) -> None:
        self._name = name

    @property
    def name(self) -> str:
        return self._name

    def __call__(self, reporter: Reporter) -> int:
        return 0  # do nothing, just normal end


class DjangoStubsComponent(ComponentBase):
    def __init__(self, django_settings_module: str) -> None:
        self._name = "configure django-stubs"
        self._django_settings_module = django_settings_module
        self._targets = ["lint"]

    @property
    def name(self) -> str:
        return self._name

    def export(self) -> Tuple[Sequence[str], Dict[str, Any]]:
        section_name = "mypy.plugins.django-stubs"
        entries = {"django_settings_module": self._django_settings_module}
        return [section_name], entries

    def export_settings(
        self,
        paths: PathContext,
        files: DefaultDict[str, SettingFile],
    ) -> None:
        setting_file = files[_SettingFileName]
        section, setting = self.export()
        setting_file.set_section(section, setting)

    @property
    def targets(self) -> Sequence[str]:
        return self._targets

    def create_command(
        self, target: str, paths: PathContext, options: RunOptions
    ) -> CommandBase:
        assert target in self._targets
        return DjangoStubsCommand(self._name)


@dataclasses.dataclass
class DjangoStubsPluginConfig:
    django_settings_module: str


class DjangoStubsPlugin(PluginBase):
    def load(
        self, file_path: pathlib.Path, config_data: PluginConfig, root: Config
    ) -> Sequence[ComponentBase]:
        assert (
            config_data.config is not None
        ), f"{config_data.location}.config must be not None"
        config = dacite.from_dict(
            DjangoStubsPluginConfig, config_data.config, dacite.Config(strict=True)
        )
        return [DjangoStubsComponent(config.django_settings_module)]


# NOTE: This is the entry point of a plugin method
def plugin() -> PluginBase:
    return DjangoStubsPlugin()
```

`django-configurations` と共存させるためと、`mypy.ini` の以下の部分を実現するために、setup スクリプトも実装します。

```ini
[mypy]
plugins =
    mypy_django_plugin.main
```

> 参考

https://github.com/typeddjango/django-stubs/pull/180#issuecomment-700686370

`pysen_setup/mypy_django_setup.py`

```python
from mypy_django_plugin import main

from base_mypy_setup import plugin as base_plugin


def plugin(version):  # type: ignore
    return base_plugin(main, version)
```

`django-stubs` をインストールします。

```bash
$ poetry add -D django-stubs
```

また、今回作成したプラグインと、`django-stubs` と `django-configurations` 共存用の setup スクリプトも読み込ませます。

`pyproject.toml`

```diff toml
[tool.poetry.dev-dependencies]
pysen = {version = "0.9.1", extras = ["lint"]}
+django-stubs = "^1.8.0"

[tool.pysen-cli]
settings_dir = "pysen_setup"

[tool.pysen]
version = "0.9"

[tool.pysen.lint]
enable_black = true
enable_flake8 = true
enable_isort = true
enable_mypy = true
mypy_preset = "strict"
line_length = 88
py_version = "py38"
mypy_ignore_packages = ["*.migrations.*"]

[[tool.pysen.lint.mypy_plugins]]
-  script = "./base_mypy_setup.py"  # relative path from tool.pysen-cli.settings_dir
+  script = "./mypy_django_setup.py"  # relative path from tool.pysen-cli.settings_dir

[[tool.pysen.lint.mypy_targets]]
  paths = ["."]

[tool.pysen.lint.source]
  includes = ["."]
  exclude_globs = ["**/migrations/*.py"]
+
+[tool.pysen.plugin."django-stubs"]
+script = "./pysen_setup/plugin.py"
+
+[tool.pysen.plugin."django-stubs".config]
+  django_settings_module = "myapi.settings"
```

`django-stubs` のインストールに必要なそれぞれの設定は、以下のように対応しています。

https://github.com/typeddjango/django-stubs#installation

```ini
[mypy]
plugins =
    mypy_django_plugin.main
```

=

```toml
[[tool.pysen.lint.mypy_plugins]]
  script = "./mypy_django_setup.py"
```

---

```ini
[mypy.plugins.django-stubs]
django_settings_module = "myapi.settings"
```

=

```toml
[tool.pysen.plugin."django-stubs"]
script = "./pysen_setup/plugin.py"

[tool.pysen.plugin."django-stubs".config]
  django_settings_module = "myapi.settings"
```

Pysen のプラグインを作成すると、途中生成のファイルが残ってしまうみたいで、それを `.gitignore` します。

`pysen_setup/.gitignore`

```
pyproject.toml
setup.cfg
```

# 4. `3` × [`djangorestframework-stubs`](https://github.com/typeddjango/djangorestframework-stubs)

さらに、`djangorestframework-stubs` も導入してみます。
DRF の場合は、`mypy_drf_plugin.main` を plugins として指定してあげるだけで良いみたいなので、先程作成した Pysen プラグインは使用しません。

https://github.com/typeddjango/djangorestframework-stubs#installation

> To make mypy aware of the plugin, you need to add
>
> ```ini
> [mypy]
> plugins =
>     mypy_drf_plugin.main
> ```

setup スクリプトだけ実装します。

`pysen_setup/mypy_drf_setup.py`

```python
from mypy_drf_plugin import main

from base_mypy_setup import plugin as base_plugin


def plugin(version):  # type: ignore
    return base_plugin(main, version)
```

`djangorestframework-stubs` をインストールします。

```bash
$ poetry add -D djangorestframework-stubs
```

後は、3 でやったように設定を編集します。

`pyproject.toml`

```diff toml
[tool.poetry.dev-dependencies]
pysen = {version = "0.9.1", extras = ["lint"]}
django-stubs = "^1.8.0"
+djangorestframework-stubs = "^1.4.0"

[tool.pysen-cli]
settings_dir = "pysen_setup"

[tool.pysen]
version = "0.9"

[tool.pysen.lint]
enable_black = true
enable_flake8 = true
enable_isort = true
enable_mypy = true
mypy_preset = "strict"
line_length = 88
py_version = "py38"
mypy_ignore_packages = ["*.migrations.*"]

[[tool.pysen.lint.mypy_plugins]]
  script = "./mypy_django_setup.py"  # relative path from tool.pysen-cli.settings_dir
+[[tool.pysen.lint.mypy_plugins]]
+  script = "./mypy_drf_setup.py"  # relative path from tool.pysen-cli.settings_dir

[[tool.pysen.lint.mypy_targets]]
  paths = ["."]

[tool.pysen.lint.source]
  includes = ["."]
  exclude_globs = ["**/migrations/*.py"]

[tool.pysen.plugin."django-stubs"]
script = "./pysen_setup/plugin.py"

[tool.pysen.plugin."django-stubs".config]
  django_settings_module = "myapi.settings"
```

# 最後に

以上で全てが動作するかと思います。

Pysen を経由している以上、どうしても詰まりやすいところが出てきてしまいました。
こういった対応をするために、Pysen のソースコードを読みに行ったりして、どうにか解決しました。

そういったことを行える環境ではない場合は、導入自体見送る方が安心です。
特に、会社として導入している場合は、開発自体を鈍化させる恐れもあるので、気をつけたほうが良いかと思います。

ただ Pysen は、非常に使いやすく洗練されたツールなため、そういった状況下にいない場合は是非導入しましょう。
Python + Poetry + Pysen は非常に良い開発体験をもたらしてくれます。

この記事が参考になれば、幸いです。
