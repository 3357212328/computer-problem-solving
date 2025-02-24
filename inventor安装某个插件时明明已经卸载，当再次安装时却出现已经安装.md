# inventor安装某个插件时明明已经卸载，当再次安装时却出现已经安装

<span style="font-size: 18px;">这是注册表没有删除导致的，打开注册表，导向计算机\HKEY_LOCAL_MACHINE\SOFTWARE\Classes\Installer\Products\点击其中的文件夹，点开之后看ProductName是不是你要安装的插件，如果是删除这个文件夹，如果不是再点击下一个文件夹，直到找到你需要安装的插件</span>
