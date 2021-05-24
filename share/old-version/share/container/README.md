# Environment variables and automatic startup scripts  

关于tmoe container 环境变量选项与自启动脚本的说明:  
注：适用于v1.4539（2021-04-23）及更新的版本安装的proot/chroot容器。  
注2：部分新用法仅支持v1.4767及更新的版本。  
注3：除了 **global env** 外，其他功能仅当login shell为zsh/bash时，才会生效。  

## Global environment variables  

(全局环境变量)  

容器在启动前，会加载环境变量文件，并将其自动生成为容器的启动参数，因此修改环境变量文件理论上对大部分login shell都能生效。  
选择该选项，本质上是修改容器内部的 **/usr/local/etc/tmoe-linux/environment/container.env**  

如何定义容器内部环境的变量并赋值？  

是这个样子吗？  

```shell
NAME="vaule"
```

虽然那样子也可以，但是考虑到种种原因，请写成这种形式。  

```shell
export NAME="vaule"
```

例如：  

```shell
export RUSTUP_HOME=/usr/local/rustup
export GOPATH=/go
```

有一个特殊变量需要注意，那就是 _PATH_  
如果需要自定义PATH为"/go/bin:/usr/local/go/bin",那么请这样子写:  

```shell
export PATH="/go/bin:/usr/local/go/bin${PATH:+:${PATH}}"
```

修改完env文件后，大部分shell都能生效，就算login shell是 **fish** 也可以。  

注：若login shell为bash，则它可能会在全局配置 /etc/profile 中覆盖掉 _PATH_ , 为了防止env文件定义的全局PATH变量失效，tmoe在 /etc/profile.d 中创建了一个 000_env.sh的软链接。  

若 login shell为zsh, 则无需担心，默认情况下 **/etc/zsh/zshenv** 不会重新定义容器初始化的环境变量。  
注意是默认情况下，若为手动定义，则还是会覆盖掉初始变量的值的。  

从优先级来考虑：  

用户目录下的 .zshenv > 全局 /etc/zsh/zshenv > 容器初始化设置的环境变量  

若为单一用户，则建议您优先考虑用户目录下配置文件。  

## Temporary scripts  

注：临时脚本的文件夹位于容器内部的 **/tmp/.tmoe_container_temporary**  

### create a script  

如果需要创建临时自启动脚本，那么建议您使用 `tmoe` 命令。  

这里假设您选择了ubuntu focal, 接着选了create a new container(新建容器)，并自定义了文件名称为“下北澤紅茶”，最后成功新建了一个ubuntu容器。  

这时候启动命令就不再是 `tmoe p u 20.04` 了，而是 `tmoe p u 下北澤紅茶`  

`tmoe`命令最后的两个参数可以使用命令、脚本、二进制文件或文件夹。  

这里假设您在当前路径下新建了一个ruby脚本  

```shell
cat >hello.rb <<-'RUBY'
#!/usr/bin/env ruby
50.times{p "hello"}
RUBY
```

文件名称为hello.rb  

那么我要怎么样才能让 **ubuntu-下北澤紅茶** 容器启动后自动执行 `hello.rb`呢？  

是这样子吗？  

```shell
tmoe p u 下北澤紅茶 hello.rb
```

并不是，如果您这样做了，那么tmoe会认为您想要在容器内执行 `hello.rb` 命令，而不是本地的 `hello.rb` 脚本。  

当前目录下的文件请使用 "./"  

```shell
tmoe p u 下北澤紅茶 "./hello.rb" 
```

若容器内部不存在ruby,那么您可以这样写。

```shell
tmoe p u 下北澤紅茶 "./hello.rb" "sudo apt update;\
sudo apt install -y ruby;\
./hello.rb
"
```

### multi files  

关于使用多参数来执行多个脚本或多命令的说明。  

目前仅支持最后两个参数。  
tmoe对于本地文件的优先级要高于容器内部文件。  
若本地存在 **~/hello.rb**，而容器内部存在同名文件，则优先检测本地文件，仅当本地不存在相应文件时才会执行容器内部的文件。  

___

假设您想要在本地编写程序，在容器内编译并执行。

来看一个简单的示例吧！  

这里假设您没有安装cargo, 因此不能通过 `cargo new hello` 来快速新建项目。

```shell
mkdir -pv hello/src

cat >hello/Cargo.toml<<-'TOML'
[package]
name = "hello"
version = "0.1.0"
authors = ["test <test@xxx.com>"]
edition = "2018"

[dependencies]
TOML

cat >hello/src/main.rs<<-'RUST_MAIN_RS'
fn main() {
    print!("{}", "Hello\n".repeat(50));
}
RUST_MAIN_RS

cat >cargo_run<<-'CARGO_RUN'
#!/usr/bin/env bash
if [[ ! $(command -v cargo) ]];then
    sudo apt update
    sudo apt install -y rustc cargo
fi
cd hello
cargo run
CARGO_RUN
```

接着，我们让容器在启动时调用当前目录下的这些文件，最后看一下运行的结果吧！

```shell
tmoe p u 下北澤紅茶 ./hello ./cargo_run
```

如果程序成功运行的话，那么终端会输出50行 **Hello**

## Permanent scripts & ln argument

注：永久性脚本的文件夹位于容器内部的 **/etc/profile.d/permanent**  

建议将该功能配合ln参数使用。

当ln位于最后一个参数时，后面没有接任何路径，此时将创建容器内部的 **/etc/profile.d/permanent** 的软链接到当前路径的 "container_link_permanent_随机数值"

```shell
tmoe p u 下北澤紅茶 ln
```

将本地脚本copy到软链接里去,这时候启动容器时会自动调用 **/etc/profile.d/permanent** 里的脚本或二进制文件。  

您如果需要将容器目录下的 **/tmp** 软链接到当前目录, 那么可以这样做。  

```shell
tmoe p u 下北澤紅茶 ln /tmp
```

假设随机数值为**114514**，那么当前目录下就会生成一个软链接文件：**container_link_tmp_114514**  
该链接指向容器内部的/tmp  
  
如需删除该文件，那么请输入`unlink container_link_tmp_114514`,或者是`rm container_link_tmp_114514`,而不要输 `rm container_link_tmp_114514/*`  

## Entrypoint

选择该选项，本质上是修改容器内部的 **/usr/local/etc/tmoe-linux/environment/entrypoint**  
（入口点）  

本选项与Permanent scripts存在相似且重合的功能。  
若您使用的是服务型容器，则建议使用Entrypoint。  

您可以输入以下命令快速创建entrypoint文件的软链接。

```shell
tmoe p u 下北澤紅茶 ln en
```

或者是  

```shell
tmoe p u 下北澤紅茶 ln entrypoint
```

您如果需要在执行完指定操作后就退出容器，那么可以将 `exit` 写进entrypoint。  

```shell
echo "exit 0" >> container_link_entrypoint_*
```

如需清空entrypoint，那就输  

```shell
echo "" > container_link_entrypoint_*
```

## summary

在大致了解完以上功能的用法后，让我们用几个例子来做个总结吧！  

示例1：
进入容器后执行特定命令，并于完成后自动退出：

```shell
rm -fv container_link_entrypoint_*
tmoe p u 下北澤紅茶 ln en
echo "exit 0" > container_link_entrypoint_*
tmoe p u 下北澤紅茶 '
for i in toilet lolcat; do
    if [[ ! $(command -v ${i}) ]]; then
        sudo apt update
        sudo apt install -y ${i}
    fi
done
echo hello world | toilet | lolcat
'
```  

注意：上述代码块中包含单引号。  

示例2:  
在容器内编译本地的hello项目，完成后自动退出。  

> 注：使用容器来编译本地项目时，默认会将本地项目复制进容器内部的临时文件夹，因此您无需担心本地项目文件夹的所有权被破坏。  
> 在v1.4539版本中，默认会立即清空临时文件夹。  
> 而在v1.4767版本中，每次启动容器且未检测到锁文件(.container.lock)时，才会自动清空。  

为了简化操作，此处假设您的宿主机和容器都安装了 **rust** 和 **cargo** 。  

```shell
cargo new hello
rm -fv container_link_entrypoint_*
tmoe p u 下北澤紅茶 ln en
echo 'exit 0' > container_link_entrypoint_*
cat>build<<-'EOF'
#!/usr/bin/env bash
cd hello
cargo build --release
mv -v target/release/ /tmp
EOF
tmoe p u 下北澤紅茶 ./hello ./build
tmoe p u 下北澤紅茶 ln /tmp/release
echo '' > container_link_entrypoint_*
```

最后执行一下 `ls -lah container_link_release_*` 或者是 `exa -lTabgh --icons container_link_release_*`  
看看里面有什么好东西吧！  

## 碎碎念  

那么问题来了，自启动容器相关进程有什么用？  
这里引用我在内测期间写的几句话。  

> If it is a temporary script, it will only be executed once and will not take effect next time.If it is a permanent script, it will be executed every time you enter the container.  
> For example, if you need a container to automatically execute ruby -run -e httpd /sd/Download -p 8080, after you create the script, the container will automatically start the ruby webrick http server.  

您如果拉了一个ruby容器，只想要让它开webrick http服务器, 那么只需要编辑 entrypoint 文件，然后输入 `ruby -run -e httpd /media/sd/Download -p 48080`， 最后保存并重进容器。
注1：**/media/sd/Download** 为容器内部的目录，可自定义更改。  
注2：48080为端口，浏览器输入 **localhost:48080** 即可访问服务。  若启动失败，则执行 `gem install webrick`  
之后每次启动这个容器，无需执行任何额外操作，都会自动自动执行这条命令。  
这个容器有一个最主要的作用，那就是提供服务。  
也就是说，一个服务就是一个容器，一个配置好的环境也是一个容器。  