## 命令行指令
``` 
curl https://sh.rustup.rs -sSf | sh   //用于下载rustup工具并进一步安装Rust稳定版本
//注意可能需要一个链接器来实现正常的编译，请自行安装gcc

cargo new [project]    //利用这条指令创建一个新项目

cargo build    //编译指令

cargo run    //编译并运行指令

cargo check    //快速检查代码是否可以通过编译

cargo build --release    //以release方式进行项目构建，优化代码，可用于代码运行效率的基准测试
```