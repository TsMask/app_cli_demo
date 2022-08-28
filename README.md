# 教你用 Rust 构建 CLI 命令行菜单

## 简单介绍

`getopts` [https://github.com/rust-lang/getopts] 官方下的 `crate` , 这是一个简单的 getopt 替代方案。通过文档上提供的两个 `Structs` 查看内含的函数方法。
`Options` 用来添加菜单列表的参数和说明，`Matches` 用来获取菜单参数是否有被使用并得到参数后的内容。

## 创建项目

首先创建个项目进行你的项目开发，要是已经有项目可以不用创建了。

```bash
> cargo new app_cli_demo
     Created binary (application) `app_cli_demo` package
```

进入项目文件夹内，创建用来编写命令参数菜单的 `crate` lib。之所以要创建也是为了目录清晰不与主程序内代码进行耦合。

```bash
> cd app_cli_demo
> cargo new --lib argrments
     Created library `argrments` package
```

## 开始编写

CLI 命令行菜单的代码都放到`argrments` crate 里面通过 lib 暴露给主程序调用，需要先进行 crate 关联，给主程序 `Cargo.toml` 文件添加 `argrments` 的 crate 关联。

**Cargo.toml**

```toml
[dependencies]
# 程序获取命令参数
argrments = { path = "./argrments" }
```

`argrments` 里的 `Cargo.toml` 文件内直接加入依赖，0.2.xx 版本都行的哦。

**argrments/Cargo.toml**

```toml
[dependencies]
# 简单配置命令行CLI菜单
getopts = "0.2"
# 配置文件反序列化
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
```

先创建参数数据存放的 `struct` 结构体 `argrments.rs` 同宏`#[derive(Debug, Deserialize)]` 可通过在命令行输出和用 json 文件内容进行反序列化进行赋值

**argrments/src/argument.rs**

```rust
use serde::Deserialize;

/// 参数属性值
#[derive(Debug, Deserialize)]
pub struct Argument {
    // 环境
    pub env: String,
    // 端口
    pub port: u32,
}

/// 声明参数属性默认值
impl Default for Argument {
    fn default() -> Self {
        Self {
            env: String::from("dev"),
            port: 8080,
        }
    }
}
```

开始关键的 CLI 菜单编写，`argrments` crate 中 `lib` 是主要的模块暴露入口，只要加上`pub`前缀的，在引入 crated 都可以读取到。

先将`argument.rs`这个 `struct` 参数数据暴露出去，再是命令行参数解析函数暴露。命令行参数的获取可以通过标准库 `use std::env` 获取，但想要指定有什么参数指令还是要生成 CLI 命令行参数可选菜单，通过菜单上声明的参数值一一获取数据。

使用 `getopts::Options::new()` 这个函数来声明参数项，常用来添加参事项的方法 `optflag` 定义标志属性，不接收参数值，仅判断是否开启标记布尔值，`optopt` 定义接收参数属性，可接收属性后输入的值，一般用在变更属性值，`usage` 给定使用说明返回字符串进行命令行输出。

简单进行`json`格式内容文件反序列到`struct`结构体，需要注意的是`json`文件内容格式与`struct`结构体保持一致。通过 `fs::read_to_string` 读取文件内容字符串给到 `serde_json::from_str` 进行 json 反序列化，`serde` 这个库支持多种格式文件进行内容序列与反序列操作更多使用方法[传送门](https://serde.rs/)

将命令参数获取后对保持的参数数据结构体进行修改并完成命令行参数菜单，将解析参数后的数据结构体返回出去，就要到主程序里获取返回的数据进行后续的使用了。

````rust
/// 参数属性值
pub mod argument;

use crate::argument::Argument;
use getopts::Options;
use serde_json;
use std::fs;

/// 程序版本号
const VERSION: &str = "0.1.0";

/// 定义命令行参数菜单
///
/// `optflag` 定义标志属性，不接收参数值，仅判断是否开启标记布尔值
/// `optopt` 定义接收参数属性，可接收属性后输入的值，一般用在变更属性值
/// `usage` 给定使用说明返回字符串进行命令行输出
fn meun() -> Options {
    let mut opts = Options::new();

    // 可获取参数数据的参数项
    opts.optopt(
        "e",
        "env",
        "服务启动环境 根据程序内置配置文件后缀选择",
        "默认值为 dev",
    );
    opts.optopt(
        "p",
        "port",
        "服务端口号 数值范围：3000-9999",
        "默认值为 8080",
    );

    // 只进行标记不接收参数值的参数项
    opts.optflag("h", "help", "显示使用帮助信息");
    opts.optflag("v", "version", "输出程序版本");

    opts
}

/// 通过文件内容字符串反序列化
///
/// `fs::read_to_string` 读取文件内容给到 `serde_json::from_str` 进行反序列化
///
/// 需要文件内容格式与结构体保持一致
fn file_to_argument(file_path: &str) -> Option<Argument> {
    if let Ok(contents) = fs::read_to_string(file_path) {
        if let Ok(res) = serde_json::from_str::<Argument>(&contents) {
            Some(res)
        } else {
            eprintln!("参数属性配置文件内容有误： \n{}", contents);
            None
        }
    } else {
        eprintln!("找不到参数属性配置文件路径： \n{}", file_path);
        None
    }
}

/// 将命令行参数进行解析返回可选参数
///
/// `Argument` 为定义可选参数属性，可通过文件进行反序列化得到
///
/// **json文件内容**
/// ```json
/// {
///   "env": "prod",
///   "port": 9910,
/// }
/// ```
pub fn parse_args(args: Vec<String>) -> Option<Argument> {
    // 先获取参数默认配置
    let mut argument = Argument::default();

    // 第一个参数是程序
    let program = &args[0];

    // 进行解析程序后参数
    let matches = match meun().parse(&args[1..]) {
        Ok(m) => m,
        Err(f) => {
            let brief = format!(
                "Error:\n {} \nUsage:\n{} [Options] files",
                f.to_string(),
                program
            );
            println!("{}", meun().usage(&brief));
            return None;
        }
    };

    // 使用帮助 -h
    if matches.opt_present("h") {
        let brief = format!("Usage:\n{} [Options] files", program);
        println!("{}", meun().usage(&brief));
        return None;
    }

    // 程序版本号 -v
    if matches.opt_present("v") {
        println!("{} v{}", program, VERSION);
        return None;
    }

    // 环境
    if matches.opt_present("env") {
        if let Some(opt) = matches.opt_str("env") {
            argument.env = opt;
        }
    }

    // 端口号
    if matches.opt_present("port") {
        if let Ok(res) = matches.opt_get::<u32>("port") {
            if let Some(opt) = res {
                argument.port = if opt <= 9999 && opt > 3000 {
                    opt as u32
                } else {
                    8080
                };
            }
        }
    }

    // 无指明参数默认后跟配置文件路径
    if !matches.free.is_empty() {
        if let Some(file_argument) = file_to_argument(&matches.free[0]) {
            argument = file_argument;
        } else {
            return None;
        }
    }

    Some(argument)
}
````

库 `argrments` crate 都编写完，回到主程序入口 `main` 中进行调用解析命令行菜单参数得到解析参数数据。

**\*src/main.rs**

```rust
use std::env;

fn main() {
    let env_args = env::args().collect();
    let args = argrments::parse_args(env_args);
    if let Some(opt) = args {
        println!("我的参数 {:#?}", opt)
    }
}
```

## 运行项目

用直接 `cargo run -v` 是行不通的，这样只会获得默认值。你需要执行 `cargo build` 得到可执行的二进制文件，之后就可以用二进制文件运行使用 CLI 命令菜单了。

```bash
> cargo build
   Compiling unicode-width v0.1.9
   Compiling itoa v1.0.3
   Compiling ryu v1.0.11
   Compiling serde v1.0.144
   Compiling getopts v0.2.21
   Compiling serde_json v1.0.85
   Compiling argrments v0.1.0 (D:\app_cli_demo\argrments)
   Compiling app_cli_demo v0.1.0 (D:\app_cli_demo)
    Finished dev [unoptimized + debuginfo] target(s) in 9.55s
>
> ./target/debug/app_cli_demo.exe -h
Usage:
D:\app_cli_demo\target\debug\app_cli_demo.exe [Options] file

Options:
    -e, --env 默认值为 dev
                        服务启动环境 根据程序内置配置文件后缀选择
    -p, --port 默认值为 8080
                        服务端口号 数值范围：3000-9999
    -h, --help          显示使用帮助信息
    -v, --version       输出程序版本

>
> ./target/debug/app_cli_demo.exe ./argrment.json
```

不带参数执行默认使用配置文件，文件内容如下：

```json
{
	"env": "prod",
	"port": 9910
}
```

## 最后

感谢能看到这里，我接触学习 `Rust` 也就一星期，尝试编写了基础语法和线程基本示例，作为一名`Java` 开发人员不得不感叹这门语言的强大。所有权系统、无 GC 多余的内存占用、杜绝空指针、很棒的 `rust-analyzer` 老师，要是有工作机会我会选择 `Rust` 先进且不失优雅的开发工作。

相信未来的几年 `Rust` 生态会更加繁荣，中国开发者从 `npmjs.com` 卷到 `Crates.io`，在全球排行榜中相信现代开发语言 `Rust` 和 `Go` 将会是后端开发语言新的宠儿，`Go` 简单的语法和学习曲线将为成为众多公司的选择，`Rust` 在系统内核、程序基础服务架构和 `WebAssembly` 进行基础设施的搭建。

以上教程项目通过 [Github 仓库源代码](https://github.com/TsMask/app_cli_demo) 查看
