title: pyenv神器
date: 2016-06-04 17:23:24
tags: [pyenv, python, 碎片]
categories: python
---

python版本管理神器： pyenv

### 安装
```
brew install pyenv
```

### 配置
将一下shell加入.bash_profile或.zshrc
```
# set up pyenv
export PYENV_ROOT=/usr/local/var/pyenv
if which pyenv > /dev/null; then eval "$(pyenv init -)"; fi
```

<!--more-->

### 设置国内镜像

```
# mirrors
export PYTHON_BUILD_MIRROR_URL="http://pyenv.qiniudn.com/pythons/"
```

### 使用方法
```
pyenv versions

pyenv version // 正在使用的版本

pyenv install --list

pyenv install 3.5.0

pyenv uninstall 3.5.0

pyenv local 3.5.0  // 全局设置

pyenv global 3.5.0  // 本地目录设置

pyenv local system  // 直接使用系统自带版本

```


### PS
1. 如何删除已经安装的python版本: [http://stackoverflow.com/questions/22774529/what-is-the-safest-way-to-removing-python-framework-files-that-are-located-in-di](http://stackoverflow.com/questions/22774529/what-is-the-safest-way-to-removing-python-framework-files-that-are-located-in-di)
2. 一般而言，系统库放/System/Library，而应用程序依赖的放/Library，所以，苹果自带的python放在前者，而用户自己装的python（比如官方网站下载的）会自动装在后者。（homebrew安装的就在后者）



