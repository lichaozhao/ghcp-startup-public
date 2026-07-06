# Copilot Cli作为服务运行的特殊情况

现在越来越多的企业在使用 Copilot Cli 作为服务运行（比如 Code Review，自动化脚本等，作为Copilot Cli SDK应用的服务端），这些服务的账号不应该绑定到个人，并且会承载比较大的调用量，企业需要对这些类型的应用配置独立的 Copilot Cli服务账号。

目前只有 GH Action 有类似于服务账号的概念：https://github.blog/changelog/2026-07-02-copilot-cli-no-longer-needs-a-personal-access-token-in-github-actions/