# Rqt Plugin Tutorial Example

rqtプラグインを作ってみようと思ってチュートリアルを参考にやったことまとめ。

## 作り方

### 空のpkgを作る
ここでは`rqt_mypkg`というpluginを作る

``` bash
$ cd <catkin_ws>
$ catkin_create_pkg rqt_mypkg rospy rqt_gui rqt_gui_py
```

### package.xmlを変更する

``` bash
$ cd <catkin_ws>/rqt_mypkg
$ emacs package.xml
```
`package.xml`の`export`タグに以下のように追記する。

``` xml
<package>
	:
	<!-- all the existing tags -->
	<export>
		<rqt_gui plugin="${prefix}/plugin.xml" />
	</export>
</package>
```

ちなみに、`build_depend`タグはPythonを使っているなら消してもいいよ。

### plugin.xmlを作成する

``` bash
$ cd <catkin_ws>/rqt_mypkg
$ emacs plugin.xml
```

`plugin.xml`に以下をコピペする。

``` xml
<library path="src">
  <class name="My Plugin" type="rqt_mypkg.my_module.MyPlugin" base_class_type="rqt_gui_py::Plugin" >
    <description>
      An example Python GUI plugin to create a great user interface.
    </description>
    <qtgui>
      <group>
	<label>Group</label>
      </group>
      <group>
	<label>Subgroup</label>
      </group>
      <label>My first Python Plugin</label>
      <icon type="theme">system-help</icon>
      <statustip>Great user interface to provide real value.</statustip>
    </qtgui>
  </class>
</library>
```

大体のタグは見たら分かると思う。（詳しい説明はref.[1]の4.3.1節を参照）  
ちなみに`Group`タグと`Subgroup`タグは名前は何でも良くて、これはpluginをrqtのプルダウンメニューから選択して起動するときのフォルダ分け。

### Pluginのコードを書く

PluginはC++でも書けるみたいだけど、特別な理由が無い限り、Pythonで書くことがおすすめされている。特別な理由というのはC++でしかlibraryにアクセスする方法が無いだとか、Pythonよりも快適（たぶん性能）なとき。ちなみにこれはPythonを想定している。

``` bash
$ cd <catkin_ws>/rqt_mypkg
$ mkdir -p src/rqt_mypkg
$ cd src/rqt_mypkg
$ touch __init__.py
```

`__init__.py`の中身は空で問題無い。何で必要かとかは調べる。  
次にスクリプト本体を書く。  
さっきの`__init__.py`と同じディレクトリで  

``` bash
$ emacs my_module.py
```

`my_module.py`に以下をコピペする。

``` python
import os
import rospy
import rospkg

from qt_gui.plugin import Plugin
from python_qt_binding import loadUi
from python_qt_binding.QtGui import QWidget

class MyPlugin(Plugin):

    def __init__(self, context):
        super(MyPlugin, self).__init__(context)
        # Give QObjects reasonable names
        self.setObjectName('MyPlugin')

        # Process standalone plugin command-line arguments
        from argparse import ArgumentParser
        parser = ArgumentParser()
        # Add argument(s) to the parser
        parser.add_argument("-q", "--quiet", action="store_true",
                            dest="quiet",
                            help="Put plugin in silent mode")
        args, unknowns = parser.parse_known_args(context.argv())
        if not args.quiet:
            print 'arguments: ', args
            print 'unknowns: ', unknowns

        # Create QWidget
        self._widget = QWidget()
        # Get path to UI file which should be in the "resource" folder of this package
        ui_file = os.path.join(rospkg.RosPack().get_path('rqt_mypkg'), 'resource', 'MyPlugin.ui')
        # Extend the widget with all atrributes and children from UI File
        loadUi(ui_file, self._widget)
        # Give QObjects reasonable names
        self._widget.setObjectName('MyPluginUi')
        # Show _widget.windowTitle on left-top of each plugin(when it's set in _widget).
        # This is useful when you open multiple plugins aat once. Also if you open multiple
        # instances of your plugin at once, these lines add number to make it easy to
        # tell from pane to pane.
        if context.serial_number() > 1:
            self._widget.setWindowTitle(self._widget.windowTitle() + (' %d' % context.serial_number()))
        # Add widget to the user interface
        context.add_widget(self._widget)

    def shutdown_plugin(self):
        # TODO unregister all publishers here
        pass

    def save_settings(self, plugin_settings, instance_settings):
        # TODO save intrinsic configuration, usually using:
        # instance_settings.get_value(k, v)
        pass

    def restore_settings(self, pluign_settings, instance_settings):
        # TODO restore intrinsic configuration, usually using:
        # v = instance_settings.value(k)
        pass
```

特に実用的も何も無くて、ただUIを表示するだけでコールバック関数も何も定義されていない。  
余裕があればもう少し実用的な何かを作りたい。  
ちなみに上記のコードはref.[2]にある。  

### UIを用意する

``` bash
$ cd <catkin_ws>/rqt_mypkg
$ mkdir resource
```

TODO  
Qtデザイナーを使って用意するらしいけど、まだやっていない。
`resource`ディレクトリの中に`MyPlugin.ui`を用意する。適当なpluginから持ってきてやればとりあえずは起動まで出来る。
例えば  
https://github.com/ros-visualization/rqt_common_plugins/blob/efc7c7e30533c598869943fb400a5744c7d30881/rqt_service_caller/resource/ServiceCaller.ui  
とか。

### setup.pyを用意する

rqt_pluginを実行する方法はいくつかある。例えば、`rosrun hogehoge hugahuga`、`hogehoge`、rqtのメニューから起動するなど。しかし、どのような方法で実行したいにしてもパスが通ってないと実行できない。

pkg直下にsetup.pyを用意する。

``` bash
$ cd <catkin_ws>/rqt_mypkg
$ emacs setup.py
```

以下の内容をコピペする。

``` python
# -*- coding: utf-8 -*-
from distutils.core import setup
from catkin_pkg.python_setup import generate_distutils_setup

d = generate_distutils_setup(
    packages=['rqt_mypkg'],
    package_dir={'': 'src'},
)

setup(**d)
```

`setup.py`についてはref.[3]を参照すること。

### CMakeLists.txtを修正する

`catkin_create_pkg`を実行した際に作成された`CMakeLists.txt`の中身を少し、修正する。

- まず、20行目の`catkin_python_setup()`のコメントアウトを外す。

- 次に、151行目あたりの`install(hogehoge)`のコメントアウトを外す。

以下のようになる。

``` CMakeList.txt
## Mark executable scripts (Python etc.) for installation
## in contrast to setup.py, you can choose the destination
install(PROGRAMS
  scripts/rqt_mypkg
  DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION}
)
```

上記で`scripts/rqt_mypkg`となっているのは問題ないのかな…

### catkin_makeする

``` bash
$ cd <catkin_ws>
$ catkin_make
$ source devel/setup.bash[zsh]
```

### 実行してみる

```bash
$ rqt --standalone rqt_mypkg
```
もし、次のようなエラーが出れば
```bash
> qt_gui_main() found no plugin matching "rqt_mypkg"
```
```bash
$ rqt --force-discover
```
を実行して、メニューから呼び出す。
ref[4]によれば、`rqt`は起動を速くするために前回の起動から24時間以内ならキャッシュしたpluginリストを見に行くらしい。なので、見つからないということがおこる（要出典）。


ビルドエラーがなければ、rqtでプラグインを実行できる。  
多分、`MyPlugin.ui`が無いと言ってエクセプションするけど、どこかのpluginから持ってくればよい。

## Reference
[1] http://wiki.ros.org/rqt/Tutorials/Create%20your%20new%20rqt%20plugin  
[2] http://wiki.ros.org/rqt/Tutorials/Writing%20a%20Python%20Plugin  
[3] http://docs.ros.org/groovy/api/catkin/html/user_guide/setup_dot_py.html  
[4] https://github.com/ros-visualization/rqt/issues/90
