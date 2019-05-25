#### Citus文档的Sphinx源文件

要生成HTML版本：

1. 从sphinx网站安装[sphinx website](http://www.sphinx-doc.org/en/master/usage/installation.html)
2. 克隆此存储库
3. 生成HTML
    ```bash
    cd citus_docs
    sphinx-build -b html -a -n . _build

    ＃在浏览器中打开 _build/index.html
    ```

---

**Sphinx安装注意：在OS X上，安装sphinx用最好通过[pip](https://pip.pypa.io/en/stable/installing/) 而不是Homebrew.
(brew公式只适用于 keg-only, 主要用于其他工具.)
使用 `pip install sphinx`.


Sphinx安装注意：在OS X上，最好通过pip而不是Homebrew 安装sphinx 。（酿造配方仅限桶装，主要用于其他工具。）使用pip install sphinx。
