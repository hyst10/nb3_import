1. github action简介
官方介绍

GitHub Actions
在 GitHub Actions 的仓库中自动化、自定义和执行软件开发工作流程。 您可以发现、创建和共享操作以执行您喜欢的任何作业（包括 CI/CD），并将操作合并到完全自定义的工作流程中。
我的理解：github action 可以运行一个带各种开发环境的docker ，通过yaml配置文件定义 任何你想执行的命令，执行命令的软件环境还支持配置环境变量。基于此，能做很多事情。具体的文档可以到github官网查看

2. github release发布版本
我最先想到的场景就是github release ,首先每次手动发布有点重复劳动，其次我的使用场景最重要的一点是：我本地是windows没有linux开发环境，也懒得搭建交叉编译的环境。那利用github action 就可以很方便的 生成windows linux mac 三个环境的可执行程序还是很方便的。下面介绍下我在项目中使用的自动发布版本的配置,这个配置是在推送tag的时候触发的，想要发布新的release 要做三件事

修改代码push到github
修改CHANGELOG.md 增加新版本描述，版本号和下一步创建的tag要一致
创建tag, push到github 触发action
以下是我用的一个配置，只生成linux程序。拷贝之后修改以这个 bin: nb3_import 的参数为自己构建程序的名称比如 bin: hello_world，不带扩展名。

.github\workflows\release.yaml

name: Release

on:
  push:
    tags:
      - v[0-9]+.*

jobs:
  create-release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: taiki-e/create-gh-release-action@v1
        with:
          # (optional) Path to changelog.
          changelog: CHANGELOG.md
        env:
          # (required) GitHub token for creating GitHub Releases.
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  upload-assets:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: taiki-e/upload-rust-binary-action@v1
        with:
          # (required) Binary name (non-extension portion of filename) to build and upload.
          bin: nb3_import
        env:
          # (required) GitHub token for uploading assets to GitHub Releases.
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
注意：bin: nb3_import 一定要改。CHANGELOG.md内容如下

# Changelog

All notable changes to this project will be documented in this file.

This project adheres to [Semantic Versioning](https://semver.org).

<!--
Note: In this file, do not use the hard wrap in the middle of a sentence for compatibility with GitHub comment style markdown rendering.
-->

## [Unreleased]

## [1.1.1] - 2022-06-09

- 初步版本 简单实现 还未优化

## [1.1.2] - 2022-06-09

- fix warning 


3. crates publish
想要代码推到github 就自动发布版本到rust crates ，github action就能实现。如果本地配置了crates index 的镜像 执行cargo publish时会报错，所以我想到能不能利用github action来自动publish 这样我就不用每次切换config中的配置了。同样的在.github\workflows\ 目录下新建个配置文件publish.yaml

name: Publish to Cargo

on:
 push:
 branches: [ master ]

jobs:
 publish:
 runs-on: ubuntu-latest

 name: 'publish'

 # Reference your environment variables
 environment: cargo

 steps:
      - uses: actions/checkout@master

 # Use caching to speed up your build
      - name: Cache publish-action bin
 id: cache-publish-action
 uses: actions/cache@v3
 env:
 cache-name: cache-publish-action
 with:
 path: ~/.cargo
 key: ${{ runner.os }}-build-${{ env.cache-name }}

 # install publish-action by cargo in github action
      - name: Install publish-action
 if: steps.cache-publish-action.outputs.cache-hit != 'true'
 run:
 cargo install publish-action
 
      - name: Run publish-action
 run:
 publish-action
 env:
 # This can help you tagging the github repository
 GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
 # This can help you publish to crates.io
 CARGO_REGISTRY_TOKEN: ${{ secrets.CARGO_REGISTRY_TOKEN }}
这次不用修改复制就能用。但是需要在github 项目的settings页面配置两个东西

环境变量配置


进入项目settings页面找到环境变量environments然后新增一个名称叫cargo

2. 增加配置 CARGO_REGISTRY_TOKEN， 值是crates的token token获取地址


这样通过修改cargo.yaml 的version 值之后 push到github 触发action 就会自动把新版本发布到crates上


截图里v1.1.1就是自动发布上来的。



用到的action脚本 来源说明:

release:taiki-e/upload-rust-binary-action

publish:tu6ge/publish-action
