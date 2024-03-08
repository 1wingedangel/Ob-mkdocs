---
title: 给 oh my posh 添加 python 版本和 conda 环境标识
class: activity
status: working
created-date: 2024-03-08T10:16:46.059+08:00
tags:
  - date/2024/03/08
aliases: 
share: true
category: EXP
---

%%
> [!example]+ **Comments:**     
>  | Date___ | Comments |
> | ------- | -------- |
> 
>  **Comment**:: 
%%

# 给 oh my posh 添加 python 版本和 conda 环境标识

先导出自己的 oh my posh 的主题：

```powershell
oh-my-posh config export --output ~/.mytheme.omp.json
```

然后编辑这个json文件。标识是在这个 json 文件的 `segments` 模块中，只要在里面插入 python 的模块就可以。

官方有一个详细的参考文档[^1]，不过如果想要加入 conda 的虚拟环境标识，插入指定的 template 就可以。

```json
{
    "type": "python",
    "style": "powerline",
    "powerline_symbol": "\uE0B0",
    "foreground": "#100e23",
    "background": "#906cff",
    "template": " \uE235 {{ .Full }} {{ if .Venv }}{{ .Venv }}{{ end }}",
    "properties": {
        "fetch_virtual_env": true
          }
        },
```

然后再重新加载自己的主题即可。

```powershell
oh-my-posh init pwsh --config ~/.mytheme.omp.json | Invoke-Expression
```

[^1]: [Python | Oh My Posh](https://ohmyposh.dev/docs/segments/python)