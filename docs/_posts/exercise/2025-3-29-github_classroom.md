---
layout: post
title: 'github classroom'
subtitle: '关于如何利用github搭建一个自己的OJ'
date: 2025-3-29 21:37 +0800
categories: exercise
author: wingrew
cover: 'https://images.unsplash.com/photo-1529322365446-6efd62aed02e?w=1600&q=900'
cover_author: 'inma santiago'
cover_author_link: 'https://unsplash.com/@inmasantiago'
tags: 
- guthub classroom
- grading
---

最近，有机会接触到了利用github的classroom简单设计一个小的操作系统竞赛的OJ，并可以生成排行榜的项目。当然，这其中用到了很多开源项目。

实质就技术而言并不复杂，但是无奈于github classroom的文档太缺了，配备的autograding机制和成绩收集机制有点太简单了，无法支撑复杂功能，并在踩坑的路上花了很多时间。当然，如果你不是基于classroom存储成绩可能反而会没这么麻烦。你只需要有一个组织，建一个classroom，在classroom里利用模板仓库（主要是编写workflow）生成一个assignment，然后基于一个开源项目获取classroom的数据并生成网页。

你需要organization，classroom，assignment，模板仓库（template CI），测例脚本（各种语言编写的），部署网站仓库。

现在简单讲一下流程。

- 首先，你需要建立一个组织，或者加入一个组织成为这个组织的owner。
![图片描述](https://github.com/wingrew/wingrew.github.io/blob/main/docs/_posts/exercise/image1.png)

- 然后，你就可以建立该组织下的classroom了。
![图片描述](https://github.com/wingrew/wingrew.github.io/blob/main/docs/_posts/exercise/image2.png)

- 之后，你就可以建立一个assignment了，然后你就可以邀请其他人加入教室，接受任务了，classroom会自动为其他人生成仓库。
![图片描述](https://github.com/wingrew/wingrew.github.io/blob/main/docs/_posts/exercise/image3.png)

这几步都很简单自然，最重要的就是github模板仓库的编写。

模板仓库最大的作用就是编写CI流，也即是.github/workflows中文件的内容。如果之前没接触过github的CI，可以先去看看基本用法，很好上手。它就是在你的各种设置触发要求下，github服务器会自行运行你编写的CI。具体编写方法就不细讲了，可以自行去看。

然后详细讲一下关于autograding的部分。它实际分为两部分autograding-test和autograding-reporter。前者负责测试吗，后者负责把测试结果报告给classroom。

现在classroom它自己提供了很简单的写法和一些仓库包括python和js之类的，大家在搭建assignment时就可以看到。
比如：
![图片描述](https://github.com/wingrew/wingrew.github.io/blob/main/docs/_posts/exercise/image4.png)
你可以根据add_test的选项和你自己测例脚本的配置来自行选择，它会自动帮你生成相应的CI。优点是简单，缺点是无法支撑复杂配置。

如果你的测试脚本是js，那么你可以对其进行一些魔改，或者改用24年之前的仓库版本。

在这里无论你是用自动生成的还是自己编写的脚本，你都需要注意以下几点：
1. CI名必须为Autograding Tests
2. 整个CI只能有一个job
3. check点的名字必须为run-autograding-tests
4. 测试名必须为testname-test的形式（存疑）
5. autograding-grading-reporter的环境变量输入必须为之前test的id大写字母_RESULTS
6. 最多只支持16个test

下面给出一个示例：
```
name: Autograding Tests
on: 
  push:
    branches:
      - main
    # paths-ignore:
    #   - '.github/**'
env:
  IMG_URL_riscv64: https://github.com/Azure-stars/testsuits-for-oskernel/releases/download/v0.1/sdcard-riscv64.img.gz # 镜像url
  IMG_URL_x86_64: https://github.com/Azure-stars/testsuits-for-oskernel/releases/download/v0.1/sdcard-x86_64.img.gz # 镜像url
  TIMEOUT: 300 # 超时时间
  SCRIPT_REPO: https://github.com/oscontent25/EvaluationScript # 脚本仓库
 
jobs:  
  run-autograding-tests:
    runs-on: ubuntu-latest
    env:
      qemu-version: 9.2.1
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@stable
        with:
          toolchain: nightly-2025-01-18
          components: rust-src, llvm-tools
          targets: x86_64-unknown-none, riscv64gc-unknown-none-elf, aarch64-unknown-none, aarch64-unknown-none-softfloat, loongarch64-unknown-none
      - uses: Swatinem/rust-cache@v2
      - run: cargo install cargo-binutils
      - uses: ./.github/workflows/setup-musl
        with:
          arch: riscv64
      - uses: ./.github/workflows/setup-musl
        with:
          arch: x86_64
      - uses: ./.github/workflows/setup-qemu
        with:
          qemu-version: ${{ env.qemu-version }}
      - name: Build python environment
        run: sudo apt-get install -y python3 python3-pip
      - name: build os.bin
        run: |
          touch riscv64_output.txt
          touch x86_64_output.txt
          make oscomp_run ARCH=riscv64 BLK=y NET=y FEATURES=fp_simd,lwext4_rs > riscv64_output.txt || true
          make clean
          make oscomp_run ARCH=x86_64 BLK=y NET=y FEATURES=fp_simd,lwext4_rs ACCEL=n > x86_64_output.txt || true

      - name: Download EvaluationScript
        run: |
            git clone ${{ env.SCRIPT_REPO }} .github/classroom

      - name: riscvglibc-test
        uses: oscontent25/os-autograding@master
        id: autogradingriscvglibc
        with:
          outputfile: riscv64_output.txt
          scriptPath: .github/classroom/glibc
      - name: x86musl-test
        uses: oscontent25/os-autograding@master
        id: autogradingx86musl
        with:
          outputfile: x86_64_output.txt
          scriptPath: .github/classroom/musl
      - name: Autograding Reporter
        uses: classroom-resources/autograding-grading-reporter@v1
        env:
          AUTOGRADINGRISCVGLIBC_RESULTS: "${{steps.autogradingriscvglibc.outputs.result}}"
          AUTOGRADINGX86MUSL_RESULTS: "${{steps.autogradingx86musl.outputs.result}}"
        with:
          runners: autogradingriscvglibc,autogradingx86musl
          token: ${{ secrets.GITHUB_TOKEN }}
```
你可以看见这个CI的基本流程，主要是环境搭建、测试和收集成绩报告给classroom，有一些小错误，完整版你可以看最后的仓库。
这里用到的os-autograding是经过魔改后的，我最后用的autograding-grading-reporter也是自己魔改的，当然在这里展示的不是。你可以通过[访问魔改仓库](https://github.com/oscontent25/os-autograding)，因为这实际上是老仓库fork过来的，为了和新的autograding相适配我不得不进行一些魔改。理所当然，为了丰富功能，我对新的[autograding-reporter](https://github.com/oscontent25/autograding-grading-reporter)也进行了适配。

这里给出[测试脚本仓库](https://github.com/oscontent25/EvaluationScript)

比较关键的逻辑是每个测试脚本都会返回一个points的数据，具体你可以参考脚本内容。

当然这里处理的都是针对于你运行一个东西，得到输出，再将输出输入autograing进行评判。这种逻辑很简单，可以你可以自行加入一些反作弊机制。

这里给出一个[完整的CI](https://github.com/oscontent25/os25_test)，这也是上文所说的类似操作系统竞赛的OJ。你可以通过这个template来搭建自己的OJ。

关于网站部署，你可以参考[原版](https://github.com/LearningOS/rust-rustlings-ranking)，也可以参考魔改之后的[仓库](https://github.com/LearningOS/oscomptest-grading)，魔改之后相较前面多了各测例得分，你可以自己阅读代码，查看逻辑。本身是由于classroom在同一个assignment下只记录总分，不记录各类测例得分，不然只能用多个模板仓库建立多个assignment，这样很不方便。所以我就魔改os-autograding和autograding-grading-reporter采取特殊的编码方式把各类测例得分编码进总分，再在部署网页的仓库进行相应的解码。

希望能对你有用！







