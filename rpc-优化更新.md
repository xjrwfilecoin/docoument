#### Miner 和 Daemon rpc 通信优化

**注意** : 这个版本的rpc 暂时只解决了 miner和damone 的通信问题，daemon和miner跑这个版本， woerker请继续使用原先的版本 


下载自己的rpc库

    // 进入 lotus/extern 目录
    git clone https://github.com/CoolRao/go-jsonrpc.git

    // 切换 dev1分支
    git checkout -b dev1 origin/dev1 


修改 lotus/go.mod 文件,指定本地依赖,暂时go.mod引入 对代码侵入性较小

    replace github.com/filecoin-project/go-jsonrpc => ./extern/go-jsonrpc


