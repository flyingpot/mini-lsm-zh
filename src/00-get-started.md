
# 环境设置

入门代码和参考解决方案可在 [https://github.com/skyzh/mini-lsm](https://github.com/skyzh/mini-lsm) 获取。

## 安装 Rust

更多信息请参见 [https://rustup.rs](https://rustup.rs)。

## 克隆仓库

```
git clone https://github.com/skyzh/mini-lsm
```

## 入门代码

```
cd mini-lsm/mini-lsm-starter
code .
```

## 安装工具

你需要最新稳定版的 Rust 来编译此项目。最低要求是 `1.74`。

```
cargo x install-tools
```

## 运行测试

```
cargo x copy-test --week 1 --day 1
cargo x scheck
```

现在，你可以继续开始 [第一周：Mini-LSM](./week1-overview.md)。

{{#include copyright.md}}
