**0x00** 写在前面|Something you should know before reading|読み前：
======================

在阅读该教程前，该教程假设读者具有以下技能：    
Before reading this tutorial, I assumes you has the following skills:    
読み前、読者は以下のスキルを備えべきである：

    1.  能够(不熟悉地)操作OllyDbg(或x64dbg等调试器)，(阅读者可以速览附件『使用OllyDbg从零开始』的前10章获取基础操作知识)
        Can use OllyDbg or other Debugger like x64dbg(Even Unfamiliarly).(Readers can get basic operating knowledge in first 10 chapters at 『使用OllyDbg从零开始』).
        OllyDbg或いは他のDebugger(例：x64dbg)を使用できる(駆け出しても大丈夫)。添付『使用OllyDbg从零开始』の前10章節読めば基礎操作を覚えできる。
    2.  能使用C\C++。
        Can program through C\C++.
        C\C++プログラムできる。
    3.  知晓(部分)系统函数调用方式以及常用结构。
        Know the (partial) system function call method and struct.
        常用システム関数の使い方と関する結構を知る
    4.  能阅读(部分)汇编代码。
        Can read (partial) assembly code.
        (部分)アセンブリコードを読めできる。
   

并假设读者已经部署好如下环境：    
And assumes you has the following tools:    
そして以下のツールを準備べきである：    

    1.  OllyDbg(或x64dbg etc)
    2.  ida
    3.  Hex Reader(本教程中使用010 Editor)
    4.  C\C++编程环境
    5.  PE Tool(本教程中使用CFF Explorer)
    
        (本教程中1|2|5使用的是吾爱破解工具包中的版本)


在阅读本教程前，请阅读：    
Before reading this tutorial you should read:　    
本チュートリアルを読み前、以下の文章を読みべきである：


[DC2PC汉化实战](https://bbs.sumisora.net/read.php?tid=10916692)

[Galgame 汉化破解初级教程：以 BGI 为例，从解包到测试](https://bbs.sumisora.net/read.php?tid=11042351)

这2篇文章涵盖了早期Gal汉化的方法，同时也介绍了一些系统函数、文字编码等技术细节，当然也是我的启蒙读物，推荐读者阅读。    
These two tutorials contain methods of early Galgame translation and patch, and also introduce some technical details such as windows functions and text encoding. They are my enlightenment books so I recommend readers to read them.    
二つの文章は初期Galgame中国語化方法を含め、若干システム関数と文字コーディングを紹介する。無論これらは私の啓発書であり、読者達に推薦する。

**0x01** 目的|Purpose
======================

该教程以『絆きらめく恋いろは』为例子带领读者了解汉化游戏的过程，旨在让想要着手于破解汉化的读者了解汉化破解的流程。同时介绍逆向分析与数据分析的方法以及介绍程序(以及提供部分实现代码)。    
This tutorial guids readers how to translate game to Chinese and use『絆きらめく恋いろは』game's translate process for example. It's designed to let readers who want to understand the process of translate process and want to start cracking galgame. It also introduces some methods of reverse analysis and data analysis and introduces the program patching method (and provides partial implementation method).    
本チュートリアルは『絆きらめく恋いろは』が例をとして、読者達にゲームの中国語化プロセスを紹介し、技術細部を展示する。同時に、リバース分析とデータ分析の方法及びパッチ方法を紹介する(部分実現コード提供する)。

**0x02** 解包|Unpack
======================

思考一下我们能怎么解包？    
So the question is: How can we unpack it?    
考え見ましょう、どうやってパークを解析するか。    

    1.获取解包函数位置，分析解包算法并自己写个解包器。
    2.获取解包函数回传数据的内存地址，通过Dump内存到文件获取明文数据。
    3.获取解包函数返回与入口位置，通过调用函数获取明文。

    1.Locate unpack function and analysis the algorithm then write your own unpacker.
    2.Find unpack function's memory data then dump decrypted data from memory.
    3.Use unpack function isself, make a programe to call unpack function to get decrypted data.

    1.アンパック関数の位置を探し、アルゴリズムを学んで、自分のアンパックツールを作る。
    2.アンパック関数のメモリの中に明文をダンプする。
    3.そのままアンパック関数を使って、明文データを取る。
    etc...

听起来就很头疼是吧，那让我们从简单的开始:-)    
It looks hard, let's start from simple:-)    
ハード高いでしょう、簡単の部分から手をつける:-) 

Step1 定位读包函数|Locate read pack function|パック読み込み関数を探す
----------------------
(先让我们从错误的胡乱分析开始)  
使用逆向工具挂载主程序，一般来说OD会自动停在程序入口点上，**Ctrl+N**分析导入函数表，可以看到有一个窗口弹出，该窗口例举主程序所有的对外函数调用，诸如**fopen**,**fwrite**,**fclose** etc。  
(Let's start with the wrong analysis method, be patient)    
Use OllyDbg to start the main program.In general, OD will pause at main function's entry point automatically. Press **Ctrl+N** to get import functions. You can see a window contains all import functions from environment such as **fopen**,**fwrite**,**fclose** etc.    
(では、間違いのやり方から手をつけましょう)
先ずはOllyDbgを通じメインプログラムを起動し、普通ODはメイン関数入口で休止し、**Ctrl+N**を押すすると外部から全ての関数を展示され、例えば**Kernel32**の**fopen**,**fwrite**,**fclose** etc。  

找到**CreateFileA**，**右键**->**在每个参考上设置断点**。能看到**BreakPoints**窗口上除了原先的`cmp esi,edx`之外多了很多的call，对，这些call就是对**CreateFileA**的调用了。

也许你会问：既然程序要读取文件，为什么不断**ReadFile**而要断**CreateFileA**呢？

因为*不论任何读取文件的方式*都要获取**Handle**，而在**CreateFileA**上有文件名参数，能更直观的知道读取的是哪个文件名。

**Hint**：**CreateFileA**与**CreateFileW**区别在于**A**后缀是x32位程序调用的而**W**后缀由64位程序调用，可以在PE看到该程序是x86的，所以我们断**CreateFileA**。

<img src="https://img.ztzl.moe/images/2019/03/10/TIM20190310195525.png" width="50%" height="50%">    

图1  从引用函数表中寻找**CreateFileA**函数

**Hint**：双击断点可以跳到对应的汇编地址处  
**Hint**：**ctrl+N**只能显示当前模块的引用函数。在汇编窗口处**右键**->**查看**可以切换模块。

Step2 执行分析
----------------------
让我们随意查看一个**CreateFileA**吧，OD会自动把函数参数分析出来

    0029DC43  |> \6A 00         push 0x0    ; /hTemplateFile = NULL
    0029DC45  |.  68 80000000   push 0x80   ; |Attributes = NORMAL
    0029DC4A  |.  6A 03         push 0x3    ; |Mode = OPEN_EXISTING
    0029DC4C  |.  6A 00         push 0x0    ; |pSecurity = NULL
    0029DC4E  |.  6A 01         push 0x1    ; |ShareMode = FILE_SHARE_READ
    0029DC50  |.  68 00000080   push 0x80000000 ; |Access = GENERIC_READ
    0029DC55  |.  8D4424 68     lea eax,dword ptr ss:[esp+0x68] ; |
    0029DC59  |.  50            push eax    ; |FileName = "赠?
    0029DC5A  |.  FF15 20714300 call dword ptr ds:[<&KERNEL32.CreateFile>;  \CreateFileA

**FileName**便是我们关心的字符串了，我们的目的是断到.**pac**包的读取行为。让我们**F9**开始执行程序，当断点被激活时，我们可以看到当前的**FileName**变量是什么。

但不幸的是，读者会发现所有的**CreateFileA**函数都没有指向目标.**pac**包，那么问题来了，这程序到底是在哪里读取文件的呢？

Step3 DLL模块分析
----------------------
敏锐的读者可能已经察觉到这个问题：为什么主窗口显示了，图像资源都加载了，而却没有一个**CreateFileA**函数进行.**pac**包读取？

答案很简单，读取不是放在主程序来读取的。

让我们取消所有的断点，从整体窥视主程序的运行过程。  
**Ctrl+F2**重载程序，**取消所有断点**并**F9**运行，直到主窗口创建为止。  
OD的**Log Data**窗口忠实地体现了程序执行的过程：

    Log data
    地址       消息
    ...
    002E3E8D   程序入口点
    ...
    10000000   Module F:\kiturakirameku\dll\Pal.dll
    01130000   Module F:\kiturakirameku\dll\vorbisfile.dll
    ...
    ...
    01150000   Module F:\kiturakirameku\dll\ogg.dll
    042E0000   Module F:\kiturakirameku\dll\vorbis.dll
    011C0000   Modules F:\kiturakirameku\dll\RESOURCE.dll
    ...
    011C0000   卸载 F:\kiturakirameku\dll\RESOURCE.dll
    ...

有几个可疑的Dll被加载了，按照顺序来看，第一个加载的是**Pal.dll**，之后加载了一些看似解码器之类的DLL，最后有个**RESOURCE.dll**被加载了又被卸载了。

首先来分析DLL调用层次，**CFF Explorer**挂载第一个被引用的**Pal.dll**，获取引入树。    

<img src="https://img.ztzl.moe/images/2019/03/11/TIM20190311113643.png" width="50%" height="50%">

显然**Pal.dll**是刚刚**Log Data**中外部DLL的根，由**Pal.dll**加载其余DLL。    
第一个引用的DLL永远是最可疑的，让我们打开ida看看**Pal.dll**提供了什么并引用了什么。  
打开ida，**Alt+2**呼出**Exports**窗口

<img src="https://img.ztzl.moe/images/2019/03/11/TIM20190311110655.png" width="50%" height="50%">

图2  从**Exports**窗口查看**Pal.dll**函数  

可以看到**Pal.dll**有大量的绘制，数据交互等函数。  
**Alt+7**呼出**Imports**窗口，我们看到我们关心的**CreateFileA**以及**CreateFontA**(在后面的修改编码章节中会有介绍)。

显然，**Pal.dll**就是该游戏的绘图引擎，而一般2DGame公司是不会单独花钱使用商业绘图引擎，所有这引擎十有八九是开源的，让我们Noogle一下。    
啊哈，**PAL=[Physics_Abstraction_Layer](https://en.wikipedia.org/wiki/Physics_Abstraction_Layer)**，既然找到本体了，我们可以下载它的源码查看头文件，就能知道所有基本函数的功能与该游戏厂商自定义的接口。

让我们回到OD。    
x86程序可以调用**KERNEL32.LoadLibraryA**或是**KERNEL32.LoadLibraryExA**来加载DLL，如果读者想研究DLL在加载前后程序行为可以断这两个函数进行研究。    
在这里我们只关心**CreateFileA**以及**CreateFontA**，先让程序开跑，在**Log Data**窗口显示**Pal.dll**被加载后在主窗口切换模块到PAL(**右键**->**查看**->**模块'pal'**)，**Ctrl+N**在导入表断**CreateFileA**以及**CreateFontA**。  

在窗体创建后**CreateFileA**就被命中，位于：    

    100A7357  |.  6A 00         push 0x0    ; /hTemplateFile
    100A7359  |.  56            push esi    ; |Attributes
    100A735A  |.  57            push edi    ; |Mode
    100A735B  |.  8B3D 04910E10 mov edi,dword ptr ds:[<&KER>; |kernel32.CreateFileA
    100A7361  |.  6A 00         push 0x0    ; |pSecurity
    100A7363  |.  53            push ebx    ; |ShareMode
    100A7364  |.  FFB5 F0F6FFFF push [local.580]    ; |Access = GENERIC_READ
    100A736A  |.  50            push eax    ; |FileName
    100A736B  |.  FFD7          call edi    ; \CreateFileA

经过调用不难发现，Pal尝试读取PAC包的读取策略是：

    ../遍历包名表/包名.pac
    ../包名.pac

<img src="https://img.ztzl.moe/images/2019/03/11/TIM20190312105806.png" width="100%" height="100%">
<img src="https://img.ztzl.moe/images/2019/03/11/TIM20190312105814.png" width="100%" height="100%">
<img src="https://img.ztzl.moe/images/2019/03/11/TIM20190312105821.png" width="100%" height="100%">
...      
<img src="https://img.ztzl.moe/images/2019/03/12/TIM20190312105814.png" width="100%" height="100%">

**Hint**：简短的文件名有利于调试，字符串超出OD窗口很令人头疼    

通过OD我们检验了**Pal.dll**的**100A736B**上的**CreateFileA**是PAC读取入口，让我们记住这个函数地址并打开ida。

ida直接加载**Pal.dll**，打开窗口**Imports**(Alt+8)查找**CreateFileA**，双击后可以跳转到汇编。光标移到**CreateFileA**并按**X**呼出调用地址表，查找和**100A736B**最接近的地址，双击跳转到函数，F5分析伪代码。    
恭喜你，你已经找到了**Pal.dll**的解包函数了！为了方便下次查看，让我们重命名该函数(右键函数名->Rename global item)，我把它换成了**PalReadPac**，下文中也会称该函数为**PalReadPac**。

接下来有3种做法：    

    1.  分析PalReadPac并写一个解包算法
    2.  分析调用方法并写一个程序调用PalReadPac
    3.  分析PalReadPac保持数据的内存位置，直接Dump内存

我们先从方法1开始，让我们看看部分函数**PalReadPac**：

    HANDLE __usercall PalReadPac@<eax>//...   
    {
    	dwDesiredAccess = 0;
    	SetFilePointer(v12, 12, (PLONG)&dwDesiredAccess, 0);
    	ReadFile(v12, &Buffer, 0x7F8u, &NumberOfBytesRead, 0);
    	v11 = (char)v11;
    	v13 = v34[2 * (char)v11];
    	//...
    	v14 = v13 >> 1;
    	v15 = 40 * (v14 + *(&Buffer + 2 * v11));
    	while ( 1 )
    	{
    		dwDesiredAccess = v15 >> 31;
    		SetFilePointer(v12, v15, (PLONG)&dwDesiredAccess, 1u);
    		ReadFile(v12, &FileName, 0x20u, &NumberOfBytesRead, 0);
    		v16 = v30;
    		v17 = &FileName;
    		v18 = 28;
    		//...
    		//某种字符串比较
    		//...
    	}
    	v26 = hFile;
    	ReadFile(hFile, (LPVOID)(a7 + 8), 4u, &NumberOfBytesRead, 0);
    	ReadFile(v26, (LPVOID)a7, 4u, &NumberOfBytesRead, 0);
    	v27 = *(_DWORD *)a7;
    	v28 = *(_DWORD *)a7 + *(_DWORD *)(a7 + 8);
    	dwDesiredAccess = 0;
    	*(_DWORD *)(a7 + 4) = v28;
    	SetFilePointer(v26, v27, (PLONG)&dwDesiredAccess, 0);
    	return v26;
    }

**Hint**：函数返回的HANDLE表示在函数外将会有其他文件操作

从代码中可以看到，文件头部的12个Byte被跳过(0x0C)，接下来读取0x7F8到BUFFER计算某种入口地址，之后是根据这个地址读一个表，表中第一个成员是明码的成员名字，32Byte。    
既然是明码的，那让我们打开一个PAC分析一下！ 

**Hint**：选择文件大小最小的PAC分析以减轻头痛症状。

拿**csv.pac**分析，Hex打开跳转到0x7F8+0x0C=0x804：

<img src="https://img.ztzl.moe/images/2019/03/13/TIM20190313210407.png" width="80%" height="80%"> 

看起来这是由char[32]+DWORD+DWORD构成一个表成员，一般表述成员需要有起始地址与偏移量，那这个十有八九就是起始地址和偏移了，怎么验证呢？    
假设第一个是起始地址，文件最高地址是0xAE37，所以两个DWORD值一定是小于0xAE37。所以第一个DWORD=0x035E，而前0x0804是存放引导地址的，所以数据一定在0x0804后，因此第一个DWORD是数据长度。那第二个DWORD=0x0A0C就是偏移地址了。跳转到0x0A0C发现正好是成员表末尾，偏移地址猜测得证，跳转到0x0A0C+0x035E是0x00(结尾标志)，文件长度猜测得证。

**Hint**：存储分为大端存储和小端存储，一般来说表结束后紧接着就是数据区，可以查看数据区起始地址推测这是大端还是小端存储

现在我们已经解开了文件成员表的秘密，结构如下：

    {
      char memberName[32];
      DWORD size;
      DWORD offset;
    }

让我们继续分析代码，中间部分的伪代码比较复杂，没关系，换回OD让我们继续跟踪变量，跟随到

    100A73D0  |.  6A 00         push 0x0
    100A73D2  |.  68 00000010   push Pal.10000000
    100A73D7  |.  6A 03         push 0x3
    100A73D9  |.  6A 00         push 0x0
    100A73DB  |.  6A 01         push 0x1
    100A73DD  |.  68 00000080   push 0x80000000
    100A73E2  |.  50            push eax
    100A73E3  |.  FFD7          call edi    ;  kernel32.CreateFileA

注意这个`call edi`是函数指针引用的写法，对应**PalReadPac**的

    hFileHandle = CreateFileA(&FileName, 0x80000000, 1u, 0, 3u, 0x10000000u, 0);

**Hint**：`push`指令是入栈指令，调用标准约定最后push的(`push eax`)是函数的第一个参数，最先push的(`push 0x0`)是最后参数，这也符合栈的先入先出特性。

在OD里可以看到eax的地址指向了**游戏目录/xxx.pac**，继续往后看：

    100A745D  |.  6A 01         |push 0x1   ; /Origin = FILE_CURRENT
    100A745F  |.  8D8D F0F6FFFF |lea ecx,[local.580]    ; |
    100A7465  |.  51            |push ecx   ; |pOffsetHi
    100A7466  |.  50            |push eax   ; |OffsetLo
    100A7467  |.  57            |push edi   ; |hFile
    100A7468  |.  FF15 C8900E10 |call dword ptr ds:[<&KERNEL32.SetFilePoin>; \SetFilePointer
    100A746E  |.  6A 00         |push 0x0   ; /pOverlapped
    100A7470  |.  8D85 E4F6FFFF |lea eax,[local.583]    ; |
    100A7476  |.  50            |push eax   ; |pBytesRead = 00000001
    100A7477  |.  6A 20         |push 0x20  ; |BytesToRead = 20 (32.)
    100A7479  |.  8D85 ECFEFFFF |lea eax,[local.69] ; |
    100A747F  |.  50            |push eax   ; |Buffer = 00000001
    100A7480  |.  57            |push edi   ; |hFile = 000002E8
    100A7481  |.  FF15 74910E10 |call dword ptr ds:[<&KERNEL32.ReadFile>]   ; \ReadFile

对应的ida伪代码：

    SetFilePointer( hFileHandle, 
                    offsetLo, 
                    (PLONG)&pRead, 
                    1u);
    ReadFile(   hFileHandle, 
                &FileName, 
                0x20u, 
                &NumberOfBytesRead, 
                0);

这里在读PAC成员的名字到**FileName**([ebp-0x114])

<img src="https://img.ztzl.moe/images/2019/03/12/TIM20190312152559.png" width="50%" height="50%"> 

接下来会在OD中看到`EAX`与`ECX`指向当前读取到的成员文件名和目标文件名，实际是在查找PAC包中的目标文件。    

**Hint**：可以根据OD调试时的真实传参重命名ida的伪代码，方便解读。    
**Hint**：如果有想查询的函数请使用巨硬Doc，如[SetFilePointer](https://docs.microsoft.com/en-us/windows/desktop/api/fileapi/nf-fileapi-setfilepointer)。    
**Hint**：函数返回值(如果有的话)存在EAX。所以如果EAX被作为指针传参，**在函数调用之前**要先把内存地址调整到EAX(内存地址框内**Ctrl+G**输入EAX即可跳转)。
**Hint**：如果OD中看见大量的连续的     

    MOV xxx     //取值范式
    cmp xxx     //比较范式
    j** xxx     //跳转范式    

很大可能是在做**串比较**或是**switch case**，一般情况下**switch case**是可以自动识别的并且**跳转范式**后往往会跟随其他指令。

所以**PalReadPac**的职责是打开一个文件handle，按文件名查找指定成员并把指针定位在成员的offset上，返回handle。    
所以解密操作一定是调用**PalReadPac**的其中一个父函数的行为，返回ida查调用树。       
<img src="https://img.ztzl.moe/images/2019/03/12/TIM20190312143958.png" width="50%" height="50%">     

**Hint**：在IDA View窗口空格键切换到汇编试图，光标选择**目标函数**后**菜单栏**->**View**->**Graphs**->**Xrefs to**可获取调用图。     
**Hint**：还记得之前说的ida按**X**调函数引用表吗？看来你很好的掌握了**Hint**提示！

显然，**PalReadPac**的直接调用有**PalFileCreateEx**与**PalFileGetFullPath**，都是DLL开出的接口，**PalFileCreateEx**被**PalFileCreate**调用，欸，是不是很眼熟？

对，这个函数在主程序被调用过！

ida挂载主程序，查看import，查找所有**PalFileCreate**调用，发现函数sub_441810：

	signed int sub_441810()
	{

	//...
	v0 = (void *)PalFileCreate("graphic.dat", &v9);
	if ( v0 == (void *)-1 )
		return 0;
	lpBuffer = (LPVOID)PalMemoryAlloc(nNumberOfBytesToRead + 1, 0);
	*((_BYTE *)lpBuffer + nNumberOfBytesToRead) = 0;
	ReadFile(v0, lpBuffer, nNumberOfBytesToRead, &NumberOfBytesRead, 0);
	CloseHandle(v0);
	v2 = (char *)lpBuffer;
	if ( *(_BYTE *)lpBuffer == 36 )
	{
		v7 = (char *)lpBuffer + 16;
		v8 = nNumberOfBytesToRead - 16;
		if ( ((_BYTE)nNumberOfBytesToRead - 16) & 3 )
			v8 = nNumberOfBytesToRead - 16 - (((_BYTE)nNumberOfBytesToRead - 16)&3);
		v3 = v7;
		v4 = 4;
		v5 = v8;
		do
		{
		  *v3 = __ROL1__(*v3, v4++);
		  *(_DWORD *)v3 ^= 0x84DF873u;
		  *(_DWORD *)v3 ^= 0xFF987DEE;
		  v3 += 4;
		  v5 -= 4;
		}
		while ( v5 );
		v2 = (char *)lpBuffer;
	}
	memmove_0(&word_7915EC, v2 + 16, 0x7F8u);
	dword_7915E8 = (int)lpBuffer + 2056;
	dword_7915E0 = 1;
	return 1;
	}

大量的数据计算以及返回的是`bool`都在暗示这是数据处理函数，也就是我们要找的解密函数了。    
让我们看看其他调用有没有什么共通的数据处理方式，以**0x84DF873**作为索引查询，同样的结构在**sub_445040**函数中发现，二者都有拼凑字符串**xxx.dat**以及同样的数据操作方式。**sub_4478D0**的函数结构则更是简单明了：    

	signed int __fastcall sub_4478D0(_BYTE *a1, int a2)
	{
		_BYTE *v2; // edi
		char v3; // cl
		int v4; // edx
		signed int result; // eax
		int v6; // [esp+Ch] [ebp-8h]		
		v6 = a2;
		if ( a2 & 3 )
			v6 = a2 - (a2 & 3);
		v2 = a1;
		v3 = 4;
		v4 = v6;
		do
		{
			*v2 = __ROL1__(*v2, v3++);
			*(_DWORD *)v2 ^= 0x84DF873u;
			*(_DWORD *)v2 ^= 0xFF987DEE;
			v2 += 4;
			v4 -= 4;
		}
		while ( v4 );
		return result;
	}

这样我们就获得了解密函数。逆函数记得把`ROL`改成`ROR`。

**Hint**：`ROL`=循环左移，`ROR`=循环右移    
**Hint**：**Alt+B**查询Hex

方法2可以从**sub_4478D0**的父函数开始入手，方法3的内存就是**sub_4478D0**函数的第一参数`_BYTE *a1`指向的地址。

题外话
----------------------

其实这个程序的大致结构是这样的：  

    主程序
    --加载-->
    Pal[初始化--独立线程--读PAC--回送handle]
    --handle-->
    主程序[解密--RUBY脚本解析]
    --显示与绘制指令-->
    Pal[绘制]    

可能是考虑到读取的视频图像等大文件效率把读取放在Pal读取，的确效率是高的，但为什么不把RUBY解析器也包进去呢？反正都是开源的也不会出现开源污染。    

"PAL的代码被暴力地修改，犹如合成兽一样强行添加了一个丑陋的文件加密算法"    
"为了保持主程序与PAL通信正常，在PAL这个傀儡中强行加了个Message收发器(甚至连管线都没用)"    
"华丽的UI外表下是诡异的程序结构"    

![图文无关](https://img.ztzl.moe/images/2019/03/11/images.jpg)

图**  一位加班加的很严重的代码人


