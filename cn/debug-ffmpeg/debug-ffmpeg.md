# FFmpeg调试环境搭建

<div id="meta-description---">本章节主要讲解 FFmpeg 在各个平台的调试环境搭建，在 clion2021，WinDbg ，vs2019，qt creator5.14，xcode 等环境上 调试 FFmpeg 的源代码。</div>

本章节主要讲解 FFmpeg 在各个平台的调试环境搭建，在 clion2021，WinDbg ，vs2019，qt creator5.14，xcode 等环境上 调试 FFmpeg 的源代码。

因为这些集成的开发环境，可以很方便查看各个变量的内容，所以调试环境的搭建，可以对初学者研究源代码有事半功倍的效果。

虽然 GDB 也能调试，但是我只有在不能用 clion 的情况才会用 GDB，GDB 其实不适合 初学时研究项目代码的逻辑，一个项目有那么多变量，每个你都得 print 一下才能看，多麻烦。

初学者还没开始了解项目的整体逻辑，就会被 GDB 的 print 打消掉积极性。
