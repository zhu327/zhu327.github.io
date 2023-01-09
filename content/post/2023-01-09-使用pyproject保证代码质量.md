---
title: "使用pyproject.toml保证代码质量"
date: 2023-01-09T15:53:52+08:00
draft: false
---

### 1. pyproject.toml是什么

> https://python.freelycode.com/contribution/detail/1910

在使用`pyproject.toml`前, 我们的Python项目根目录下会存在很多项目相关的配置文件, 比如:

- requirements.txt
- requirements_dev.txt
- .flake8
- mypy.ini
- .isort.cfg
- .bandit

我们的项目代码中充斥这这些与代码无关的配置, `pyproject.toml`就是用来统一纳管Python项目的所有这些配置的东西, 得到了以上大部分工具的支持.

<!--more-->

### 2. poetry

pip还不支持`pyproject.toml`, 所有我们需要使用poetry这个工具来实现项目的依赖管理.

> https://python-poetry.org/docs/basic-usage/

如文档所示, 我们只需要执行:

```bash
poetry init
```

就会在项目的根目录下自动生成一个`pyproject.toml`, 通过poetry添加的依赖项会自动写入`pyproject.toml`, 一般情况下, 项目下的`requirements.txt`, `requirements_dev.txt`就可以删除了.

### 3. Format

写完代码后, 我们希望不同项目成员提交的代码都能有统一个代码格式规范, 所用我们使用以下工具来保证代码风格的一致:

- black 用于格式化代码
- isort 用于对Python import代码行自动排序

安装:

```bash
poetry add black --dev
poetry add isort --dev
```

配置:

```
# FILE: pyproject.toml

[tool.black]
line-length = 119
target-version = ['py36']
exclude = '''
(
  /(
      \.mypy_cache
    | \.git
    | migrations
  )/
)
'''

[tool.isort]
multi_line_output = 3
include_trailing_comma = 'true'
force_grid_wrap = 0
use_parentheses = 'true'
line_length = 119
skip = [".mypy_cache", ".git", "*/migrations"]
```

使用:

```bash
isort --settings-path=./pyproject.toml .
black --config=./pyproject.toml .
```

### 4. Lint

代码静态检查是保证项目代码质量必不可的步骤, 一些现代的静态检查工具, 能够在我们的代码被提交之前就能检查出代码缺陷, 以下是一些建议:

#### 4.1 flake8

除了flake8本身支持的检查规则以外, flake8还支持插件来扩展规则, 这里推荐安装`flake8-bugbear`, 以下是使用参考:

安装:

```bash
poetry add flake8 --dev
poetry add flake8-bugbear --dev
poetry add pyproject-flake8 --dev
```

配置:

```
# FILE: pyproject.toml

[tool.flake8]
ignore = "C901,E203,W503,B010,B009"
max-line-length=119
max-complexity=12
format = "pylint"
show_source = "true"
statistics = "true"
count = "true"
exclude = "*migrations*,*.pyc,.git,__pycache__,node_modules/*,*/templates_module*,*/bin/*,*/settings/*,config,tests/unittest_settings.py"
```

使用:

```bash
pflake8 --config=./pyproject.toml .
```

#### 4.2 mypy

随着python3支持type hitting, Python代码的可读性与编辑器支持都得到了大幅提升, 所以建议在项目中强制使用type hitting, mypy用于对代码中类型标注做检查, 以下为参考配置:

安装:

```bash
poetry add mypy --dev
```

配置:

```
# FILE: pyproject.toml

[tool.mypy]
files=["."]
python_version = 3.6
ignore_missing_imports=true
follow_imports="skip"
strict_optional=true
pretty=true
show_error_codes=true
```

使用:

```bash
mypy --config-file=./pyproject.toml .
```

#### 4.3 bandit

bandit用于检查Python代码中可能出现的安全问题, 但是检查耗时比较长, 推荐在CI工具中使用

安装:

```bash
poetry add bandit --dev
```

配置:

```
# FILE: pyproject.toml

[tool.bandit]
exclude_dirs = ["tests"]
tests = []
skips = ["B101", "B110", "B311", "B303"]
```

使用:

```bash
bandit -c ./pyproject.toml -r .
```

### 5. unittest

单元测试是保证代码质量的重要步骤, 在每次提交代码前运行单元测试是一个好的习惯, 现代的Python项目推荐使用pytest框架实现单元测试

安装:

```bash
poetry add pytest --dev
poetry add pytest-cov --dev
```

配置:

```
# FILE: pyproject.toml

[tool.pytest.ini_options]
DJANGO_SETTINGS_MODULE = "tests.unittest_settings"
addopts = "--disable-pytest-warnings --reuse-db --nomigrations -s"
python_files = "*_tests.py"
testpaths = [
    "tests"
]
```

使用:

```bash
pytest -c ./pyproject.toml .

# 生成代码测试覆盖率报告
pytest --cov-report html --cov=backend -c ./pyproject.toml .
```

### 6. 自动化

我们希望以上`Format`, `Lint`, `Test`在每次代码提交时都能自动执行, 而不是每次手动去跑, 所以我们在git提交前使用pre-commit触发以上环节, 在代码在Github上被合并前使用github actions触发以上环节

#### 6.1 pre-commit

> https://pre-commit.com/

推荐配置`.pre-commit-config.yaml`

```yaml
# See https://pre-pre-commit --versioncommit.com for more information
# See https://pre-commit.com/hooks.html for more hooks
repos:
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v2.3.0
    hooks:
      - id: check-added-large-files
      - id: check-ast
      - id: check-byte-order-marker
      - id: check-case-conflict
      - id: check-executables-have-shebangs
      - id: check-merge-conflict
      - id: debug-statements
      - id: detect-private-key
      - id: end-of-file-fixer
      - id: trailing-whitespace
  - repo: local
    hooks:
      - id: isort
        name: isort
        language: python
        pass_filenames: false
        entry: isort --settings-path=saas/pyproject.toml .
      - id: black
        name: black
        language: python
        pass_filenames: false
        entry: black --config=saas/pyproject.toml .
      - id: flake8
        name: flak8
        language: python
        pass_filenames: false
        entry: pflake8 --config=saas/pyproject.toml .
      - id: mypy
        name: mypy
        language: python
        pass_filenames: false
        entry: mypy --config-file=saas/pyproject.toml .
      - id: pytest
        name: pytest
        language: python
        pass_filenames: false
        entry: pytest -c saas/pyproject.toml .
```

#### 6.2 github actions

`.github/workflows/python.yml`

```yaml
name: Python CI Check

on:
  push:
    branches: [ master, develop ]
  pull_request:
    branches: [ master, develop ]

jobs:
  build:

    strategy:
      fail-fast: false
      matrix:
        python-version: [3.6]
        poetry-version: [1.1.7]
        os: [ubuntu-18.04]
    runs-on: ${{ matrix.os }}

    steps:
    - uses: actions/checkout@v2
    - name: Set up Python
      uses: actions/setup-python@v2
      with:
        python-version: ${{ matrix.python-version }}
    - uses: snok/install-poetry@v1
      with:
        version: 1.1.12
    - name: Install dependencies
      run: poetry install --no-interaction
    - name: Lint with flake8
      run: pflake8 --config=saas/pyproject.toml .
    - name: Lint with bandit
      run: bandit -c saas/pyproject.toml -r .
    - name: Lint with mypy
      run:  mypy --config-file=saas/pyproject.toml .
    - name: Test with pytest
      run: pytest -c saas/pyproject.toml .
```

### 7. Makefile

有了以上这些配置后, 对于一个项目新人, 可能上手成本有点高, 这个时候用`make`命令就能帮助新同学快速熟悉项目

`Makefile`

```
i18n_all: i18n_po i18n_mo

# make messages of python file and django template file to django.po
i18n_po:
	python manage.py makemessages -d django -l en -e html,part -e py
	python manage.py makemessages -d django -l zh_Hans -e html,part -e py

# compile django.po and djangojs.po to django.mo and djangojs.mo
i18n_mo:
	python manage.py compilemessages

init:
	pip install -U pip setuptools
	pip install poetry
	poetry install
	pip install pre-commit
	pre-commit install

lint:
	pflake8 --config=./pyproject.toml .
	bandit -c ./pyproject.toml -r .
	mypy --config-file=./pyproject.toml .

fmt:
	isort --settings-path=./pyproject.toml .
	black --config=./pyproject.toml .

test:
	pytest -c ./pyproject.toml .

cov:
	pytest --cov-report html --cov=backend -c ./pyproject.toml .

serve:
	python manage.py runserver 8000
```

### 8. 参考

以上项目配置来源于[蓝鲸权限中心](https://github.com/TencentBlueKing/bk-iam-saas), 参考了以下文章:

> https://www.b-list.org/weblog/2022/dec/19/boring-python-code-quality/
