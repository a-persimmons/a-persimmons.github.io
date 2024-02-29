---
author: ["柿子"]
title: '如何找到容器中准确的CPU利用率？'
date: "2024-03-01T01:22:55+08:00"
summary: "TOP 指令怎么准确定位容器的真实情况."
tags: ["linux","top","docker","摘录"]
---

“top”是一个强大的、轻量级的命令行工具，可以提供有关系统范围资源利用率的实时报告。它在各种 Linux 发行版中普遍可用。然而，我们发现它在 Docker 容器中执行时可能无法准确报告信息。这篇文章旨在引起您对这个问题的关注。

## docker容器中的CPU压力测试

让我们进行一个简单的实验。我们将使用 Ubuntu 映像部署容器并有意增加 CPU 消耗。执行以下命令：

```bash
docker run -ti --rm --name tmp-limit --cpus="1" -m="1G" ubuntu bash -c 'apt update; apt install -y stress; stress --cpu 4'
```

提供的命令执行以下操作：

1. 使用 Ubuntu 镜像启动容器
2. 将 CPU 限制设置为 1
3. 设置内存限制为1G
4. 执行命令‘apt update; apt install -y 压力； stress –cpu 4’ 进行 CPU 压力测试

## 主机顶部报告的 CPU 利用率

现在，让我们在运行该 Docker 容器的主机上启动 top 工具。 top工具的输出如下：

![top command from the host](https://i0.wp.com/blog.ycrash.io/wp-content/uploads/2024/02/1-top-from-host.png?w=1467&ssl=1)

<p align="center">图1：来自主机的top命令</p>

请注意图 1 中的橙色矩形。该指标显示为 25% CPU 利用率，并且它是正确的值。主机有 4 个核心，我们为容器分配了 1 个核心的限制。由于该单个核心已得到充分利用，报告的主机级别 CPU 利用率为 25%（即总核心的 1/4）。

## 容器顶部报告的 CPU 利用率

现在，让我们在容器内执行 top 命令。以下是top命令报告的输出：

![top command from the container](https://i0.wp.com/blog.ycrash.io/wp-content/uploads/2024/02/2-top-from-container.png?w=1467&ssl=1)

<p align="center">图 2：来自容器的 top 命令</p>

请观察图 2 中的橙色矩形。CPU 利用率为 25%，反映了主机的值。然而，从容器的角度来看，这是不准确的，因为它已充分利用其分配的 100% CPU 限制。

尽管如此，值得注意的是图 2 中列出的过程是准确的。该工具仅正确报告在此容器内运行的进程，并排除整个主机中的进程。

## 如何找到容器中准确的CPU利用率？

在这样的场景下，要获得容器内准确的CPU利用率，有以下几种解决方案：

1. Docker 容器统计信息（docker stats）

   `docker stats` 命令提供容器级别的基本资源利用率指标。以下是先前启动的容器的“docker stats”的输出：

   ![docker stats output](https://i0.wp.com/blog.ycrash.io/wp-content/uploads/2024/02/3-docker-stats.png?w=1472&ssl=1)

   <p align="center">图 3：docker stats 输出</p>

   请注意图 3 中的橙色矩形。CPU 利用率显示为 100.64%。然而，挑战在于“docker stats”无法在容器内执行（除非将docker套接字传递到容器中，这种情况不常见并且会带来安全风险）。它必须从主机运行。

2. 容器顾问（cAdvisor）

   您可以利用本质上支持 Docker 容器的 [cAdvisor](https://github.com/google/cadvisor)（容器顾问）工具来提供容器级资源利用率指标。

3. yCrash

   ![yCrash – root cause analysis report](https://i0.wp.com/blog.ycrash.io/wp-content/uploads/2024/02/4-ycrash.png?w=1472&ssl=1)

   <p align="center">图 4：yCrash – 根本原因分析报告</p>

   此外，您还可以选择使用 yCrash 工具，该工具不仅提供容器级指标，还分析应用程序级转储（例如垃圾收集日志、应用程序日志、线程、内存转储等）并提供全面的根原因分析报告。

   ## 结论

   虽然“top”是监控系统范围资源利用率的可靠工具，但它在 Docker 容器中的准确性可能会受到影响。这种差异可能会导致对容器性能的误解，尤其是在 CPU 利用率方面。正如我们的实验所示，“top”报告了容器内 25% 的 CPU 使用率，尽管已充分利用了分配的 CPU 限制。

   为了获得 Docker 容器内的精确指标，Docker Container Stats、cAdvisor 和 yCrash 等替代工具可以提供有关资源利用率的宝贵见解。通过利用这些工具，用户可以确保准确监控和优化容器化环境，最终提高性能和运营效率。

<hr style="background:#aaaaaa"/>
<center><font size="2.3" color="#aaaaaa">感谢阅读</font></center>

原文：[‘top’ reporting accurate metrics within containers?](https://blog.ycrash.io/2024/02/12/top-reporting-accurate-metrics-within-containers/)
