---
title: "Django Ã Pysen Ã Stubs"
emoji: "ð§©"
type: "tech"
topics: ["django", "pysen", "python", "poetry"]
published: true
---

ä»¥ä¸ã«æãã¦ãããæ§ããªãã®ãçµã¿åããããã¨ããã¨ããªãå¤§å¤ã ã£ãã®ã§ãåå¿é²ã¨ãã¦ãã¡ãã®è¨äºã«ããã¦ããã ãã¾ãã

* [`Django`](https://www.djangoproject.com/)
* [`Pysen`](https://github.com/pfnet/pysen)
* [`django-configurations`](https://django-configurations.readthedocs.io/en/stable/)
* [`django-stubs`](https://github.com/typeddjango/django-stubs)
* [`djangorestframework-stubs`](https://github.com/typeddjango/djangorestframework-stubs)

åæã¨ãã¦ãDjango ãã­ã¸ã§ã¯ãã®ã»ããã¢ããæ¸ã¿ã§ãPoetry ã¯å°å¥æ¸ã¿ã§ãããã¨ã¨ãã¦ãé²ãããã¦ããã ãã¾ãã

# 1. [`Django`](https://www.djangoproject.com/) Ã [`Pysen`](https://github.com/pfnet/pysen)

ã¾ãã¯ãDjango ãã­ã¸ã§ã¯ãã« Pysen ãå°å¥ãã¾ãã

```bash
$ poetry add -D pysen[lint]
```

```toml: pyproject.toml
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

# 2. `1` Ã [`django-configurations`](https://django-configurations.readthedocs.io/en/stable/)

Pysen ã« `django-configurations` ã®ããã©ã«ãå¤ãè¨­å®ãããã¨ã§ãã¨ã©ã¼ã«ãªããã¨ãé²ãã¾ãã
ä»¥ä¸ã®ãã¡ã¤ã«ãä½æãã¦ãã ããã

```python: pysen_setup/base_mypy_setup.py
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

Pysen ã«ä½æãããã¡ã¤ã«ãèªè­ãããããã«ã`pyproject.toml` ãç·¨éãã¾ãã

```diff toml: pyproject.toml
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

# 3. `2` Ã [`django-stubs`](https://github.com/typeddjango/django-stubs)

`django-stubs` ã Pysen ã«èª­ã¿è¾¼ã¾ããã«ã¯ãmypy ã®è¨­å®ãã¡ã¤ã«ãç·¨éããå¿è¦ãããã¾ãã

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

Pysen ã¯ `tool.pysen-cli.settings_dir` ã§æå®ããããã£ã¬ã¯ããªåã«ãmypy ç­ã®å¿è¦ãªè¨­å®ãã¡ã¤ã«ãèªåã§çæãããããèª­ã¿è¾¼ã¾ãããã¨ã§è¤æ° Linter ã®çµ±åãå®ç¾ãã¦ãã¾ãã
ãã®çæããããã¡ã¤ã«ãç·¨éããå¿è¦ããããããä¾ãã° `mypy.ini` ãä½æããã¨ãã¦ããããèª­ã¿ã«è¡ã£ã¦ã¯ããã¾ããã
`django-stubs` ã®è¨­å®ã¯ã`pyproject.toml` ã«æ¸ããã¨ãã§ããã¿ããã§ãããæãã Pysen ã¯ãã¡ãã«å¯¾å¿ãã¦ããªãã®ã§ã¯ç¡ããã¨æãã¾ãã

ããã§ãPysen ã®ãã©ã°ã¤ã³ãä½æãã¦ã`django-stubs` ç¨ã® `mupy.ini` ãçæããããã«è¨­å®ãã¾ãã

ä»¥ä¸ãåèã«ä½ã£ã¦ã¿ã¾ããã

https://github.com/pfnet/pysen/blob/66fb2c1dd6854c149224c53c4c01fbbc8f473a8f/examples/plugin_example/plugin.py

```python: pysen_setup/plugin.py
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

`django-configurations` ã¨å±å­ãããããã¨ã`mypy.ini` ã®ä»¥ä¸ã®é¨åãå®ç¾ããããã«ãsetup ã¹ã¯ãªãããå®è£ãã¾ãã

```ini: mypy.ini
[mypy]
plugins =
    mypy_django_plugin.main
```

> åè

https://github.com/typeddjango/django-stubs/pull/180#issuecomment-700686370

```python: pysen_setup/mypy_django_setup.py
from mypy_django_plugin import main

from base_mypy_setup import plugin as base_plugin


def plugin(version):  # type: ignore
    return base_plugin(main, version)
```

`django-stubs` ãã¤ã³ã¹ãã¼ã«ãã¾ãã

```bash
$ poetry add -D django-stubs
```

ã¾ããä»åä½æãããã©ã°ã¤ã³ã¨ã`django-stubs` ã¨ `django-configurations` å±å­ç¨ã® setup ã¹ã¯ãªãããèª­ã¿è¾¼ã¾ãã¾ãã

```diff toml: pyproject.toml
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

`django-stubs` ã®ã¤ã³ã¹ãã¼ã«ã«å¿è¦ãªããããã®è¨­å®ã¯ãä»¥ä¸ã®ããã«å¯¾å¿ãã¦ãã¾ãã

https://github.com/typeddjango/django-stubs#installation

```ini: mypy.ini
[mypy]
plugins =
    mypy_django_plugin.main
```

=

```toml: pyproject.toml
[[tool.pysen.lint.mypy_plugins]]
  script = "./mypy_django_setup.py"
```

---

```ini: mypy.ini
[mypy.plugins.django-stubs]
django_settings_module = "myapi.settings"
```

=

```toml: pyproject.toml
[tool.pysen.plugin."django-stubs"]
script = "./pysen_setup/plugin.py"

[tool.pysen.plugin."django-stubs".config]
  django_settings_module = "myapi.settings"
```

Pysen ã®ãã©ã°ã¤ã³ãä½æããã¨ãéä¸­çæã®ãã¡ã¤ã«ãæ®ã£ã¦ãã¾ãã¿ããã§ãããã `.gitignore` ãã¾ãã

```: pysen_setup/.gitignore
pyproject.toml
setup.cfg
```

# 4. `3` Ã [`djangorestframework-stubs`](https://github.com/typeddjango/djangorestframework-stubs)

ããã«ã`djangorestframework-stubs` ãå°å¥ãã¦ã¿ã¾ãã
DRF ã®å ´åã¯ã`mypy_drf_plugin.main` ã plugins ã¨ãã¦æå®ãã¦ãããã ãã§è¯ãã¿ãããªã®ã§ãåç¨ä½æãã Pysen ãã©ã°ã¤ã³ã¯ä½¿ç¨ãã¾ããã

https://github.com/typeddjango/djangorestframework-stubs#installation

> To make mypy aware of the plugin, you need to add
>
> ```ini
> [mypy]
> plugins =
>     mypy_drf_plugin.main
> ```

setup ã¹ã¯ãªããã ãå®è£ãã¾ãã

```python: pysen_setup/mypy_drf_setup.py
from mypy_drf_plugin import main

from base_mypy_setup import plugin as base_plugin


def plugin(version):  # type: ignore
    return base_plugin(main, version)
```

`djangorestframework-stubs` ãã¤ã³ã¹ãã¼ã«ãã¾ãã

```bash
$ poetry add -D djangorestframework-stubs
```

å¾ã¯ã3 ã§ãã£ãããã«è¨­å®ãç·¨éãã¾ãã

```diff toml: pyproject.toml
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

# æå¾ã«

ä»¥ä¸ã§å¨ã¦ãåä½ãããã¨æãã¾ãã

Pysen ãçµç±ãã¦ããä»¥ä¸ãã©ããã¦ãè©°ã¾ããããã¨ãããåºã¦ãã¦ãã¾ãã¾ããã
ãããã£ãå¯¾å¿ãããããã«ãPysen ã®ã½ã¼ã¹ã³ã¼ããèª­ã¿ã«è¡ã£ãããã¦ãã©ãã«ãè§£æ±ºãã¾ããã

ãããã£ããã¨ãè¡ããç°å¢ã§ã¯ãªãå ´åã¯ãå°å¥èªä½è¦éãæ¹ãå®å¿ã§ãã
ç¹ã«ãä¼ç¤¾ã¨ãã¦å°å¥ãã¦ããå ´åã¯ãéçºèªä½ãéåãããæããããã®ã§ãæ°ãã¤ããã»ããè¯ããã¨æãã¾ãã

ãã  Pysen ã¯ãéå¸¸ã«ä½¿ããããæ´ç·´ããããã¼ã«ãªããããããã£ãç¶æ³ä¸ã«ããªãå ´åã¯æ¯éå°å¥ãã¾ãããã
Python + Poetry + Pysen ã¯éå¸¸ã«è¯ãéçºä½é¨ãããããã¦ããã¾ãã

ãã®è¨äºãåèã«ãªãã°ãå¹¸ãã§ãã
