---
title: 前端使用 WebAssembly 运行 Kociemba 算法
---

为了完成魔方自动复原的操作，我们需要一个算法来帮助我们以较少的步骤，完成复原过程，这里选择了 [Kociemba 算法](https://github.com/muodov/kociemba)，也叫二阶段算法。

遗憾的是，这个算法并没有比较权威的 JavaScript 实现，它的实现在 Github 上能找到的只有 `Python`、`C` 和 `C++` 的版本。

因此摆在我们面前的，有两条路：

1. 使用 `nodejs` 搭建一个简单的后端服务器，接受前端的请求，在 `node` 里运行 `C` 或 `Python` 代码，再将结果返回给前端；
2. 使用 `Emscripten` 将 `C` 语言编译成 `JavaScript` 代码并使用 `WebAssembly` 技术在浏览器中运行；

本着研究的精神，我决定这两种办法都要试一试，暂且称他们为 `在线模式` 和 `离线模式`。这一篇，首先讲解 `离线模式`， 也就是我们直接在浏览器中使用 `WebAssembly` 技术来运行 [Kociemba 算法](https://github.com/muodov/kociemba)。

## 安装 Emscripten (Mac)

### 1. 克隆官方仓库

```shell
git clone https://github.com/emscripten-core/emsdk.git 
```

![image-20211209211227130](https://zhuye-1308301598.file.myqcloud.com/markdown/image-20211209211227130.png)

### 2. 安装

```shell
cd emsdk
./emsdk install latest # 这里会下载nodejs、python3等一系列依赖
./emsdk activate latest # 激活
source ./emsdk_env.sh # 激活环境变量，可以写到zshrc里
```

![image-20211209213706347](https://zhuye-1308301598.file.myqcloud.com/markdown/image-20211209213706347.png)

### 3. 查看是否安装成功

```shell
emcc -v
```

![image-20211209213840481](https://zhuye-1308301598.file.myqcloud.com/markdown/image-20211209213840481.png)

##  安装 cmake

```shell
brew install cmake
cmake --version
```

![image-20211209214132800](https://zhuye-1308301598.file.myqcloud.com/markdown/image-20211209214132800.png)



## 测试源码算法

### 下载并编译C语言源码

```shell
git clone git@github.com:muodov/kociemba.git
cd kociemba
ls
make
```

![image-20211209214438328](https://zhuye-1308301598.file.myqcloud.com/markdown/image-20211209214438328.png)

### 测试

```shell
cd bin
ls
./kociemba DRLUUBFBRBLURRLRUBLRDDFDLFUFUFFDBRDUBRUFLLFDDBFLUBLRBD
# Output: D2 R' D' F2 B D R2 D2 R' F2 D' F2 U' B2 L2 U2 D R2 U
```

![image-20211209214822923](https://zhuye-1308301598.file.myqcloud.com/markdown/image-20211209214822923.png)



## 开始转译

### 对源文件进行一些微调

我们需要按照我们的需要对源文件进行一些微调

1. 将头文件全部移出到和 `.c` 文件相同的文件夹内。

![image-20211209215129422](https://zhuye-1308301598.file.myqcloud.com/markdown/image-20211209215129422.png)

2. 增加 `solve` 方法，以供 JS 调用

### 使用 Emscripten 将 C 转为 JS

```shell
emcc coordcube.c cubiecube.c facecube.c prunetable_helpers.c search.c solve.c -s WASM=1 -s EXTRA_EXPORTED_RUNTIME_METHODS='["UTF8ToString"]' -s ALLOW_MEMORY_GROWTH=1 -s EXPORTED_FUNCTIONS="['_solve', '_malloc']" -o kociemba.js
```

![image-20211209215759494](https://zhuye-1308301598.file.myqcloud.com/markdown/image-20211209215759494.png)



## 测试

新建一个项目文件夹；

将生成的 `kociemba.js` 和 `kociemba.wasm` 复制到项目文件夹中；

在项目文件夹中添加一个 `index.html`，并填入以下内容

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<body>
<button class="mybutton">运行 Kociemba 算法测试</button>
<script>
    Module = {}
    Module.onRuntimeInitialized = function () {
        document.querySelector('.mybutton').addEventListener('click', function(){
            console.log('--- START ---');
            const rubik = "DRLUUBFBRBLURRLRUBLRDDFDLFUFUFFDBRDUBRUFLLFDDBFLUBLRBD"
            console.log(`Input: ${rubik}`)
            const ptr = allocate(intArrayFromString(rubik), ALLOC_NORMAL)
            const result = Module._solve(ptr); // arguments
            console.log(Module.UTF8ToString(result))
        });
    }
</script>
<script src="./kociemba.js">
</script>
</body>
</html>
```

在浏览器中测试，当点击按钮时，可以看到结果：

![image-20211210013807391](https://zhuye-1308301598.file.myqcloud.com/markdown/image-20211210013807391.png)

在验证了这种方法的可行性后，我们就可以把它运用到我们的项目之中了。