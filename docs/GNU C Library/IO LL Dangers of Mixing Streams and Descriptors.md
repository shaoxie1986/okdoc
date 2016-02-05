# [混合使用流和文件描述符有风险，请谨慎](https://www.gnu.org/software/libc/manual/html_node/Stream_002fDescriptor-Precautions.html#Stream_002fDescriptor-Precautions)

你可以在同一时刻有多个文件描述符和流连接到同一个文件，(让我们在这里暂且把流和文件描述符称呼为“通道”)，但是必须小心，避免在这些通道间产生混乱。有两种情况需要考虑：“链接通道”共享文件位置，“独立通道”有自己的文件位置。

在你的程序中，对任何文件传输数据，最好只使用一个通道（除了只做输入的情况外）。例如，如果你打开一个管道(只能通过文件描述符)，要么通过这个文件描述符执行所有 IO，要么通过 `fdopen()` 创建一个流，然后通过流执行所有的 IO。

## 链接通道

共享文件位置的通道，称为“链接通道”。链接通道是这样产生的：

* 当你通过 `fdopen()` 从一个文件描述符创建一个流时
* 当你通过 `fileno()` 从一个流获取底层文件描述符时
* 当你通过 `dup`、`dup2()` 复制一个文件描述符时
* 当你通过 `fork()` 派生子进程继承文件描述符时

对于那些不支持随机访问的文件（比如终端、管道），它们所有的通道都是链接的。对于支持随机访问的文件，所有追加型的输出流都是链接的。

如果你一直在用流执行 IO（或者只是刚打开流），现在想用另一个和它链接的通道（可以是流，也可以是文件描述符）执行 IO，那么你必须先清空这个流。

终止进程，或者在进程中执行一个新的程序，会销毁进程所有的流。如果链接这些流的文件描述符被保留在其他进程，它们的文件位置变为未定义的。为了防止这种情况，你必须在销毁流前先清空流。

## 独立通道

当你对支持随机访问的文件独立地打开通道时（流或文件描述符），每个通道有自己的文件位置，这被称为“独立通道”。

内核独立地处理每个通道。大多数时候，IO 操作都是可预知的：每个通道按照自己的顺序读或写文件。不过，如果通道是流，你必须注意到：

* 当用流写数据后，先清空它，然后再执行后续的读或写。
* 当用流读数据前，先清空它。否则，你可能会读到已过时的数据。

当你通过一个独立通道在文件尾部写入数据时，在执行写入前，你无法把文件位置可靠地设置到尾部---设置文件位置和写入数据，它们不是一个原子操作。假如一个进程把文件位置设置到尾部，这时另一个进程向这个文件写入一些数据，就改变了文件的长度，对于第一个进程来讲，它之前设定的位置已经不是文件尾部（因为它们的文件位置都是独立的）。想要可靠地写入到尾部，使用追加型的文件描述符或流，它们总是写入到尾部，而不管文件位置。

## 清空流

在许多情况下，你可以调用 `fflush()` 清空流。

如果你确信流已经清空了，可以忽略 `fflush()`。只要缓冲区是空的，流就是清空的。比如：

* 一个无缓冲流总是清空的
* 一个输入流位于 end-of-file 是清空的
* 一个行缓冲流，当最后一个字符输出是一个新行时，是清空的

然而，一个刚打开的输入流可能不是清空的，因为它的缓冲区可能不是空的。

有一种情况，在大多数系统中清空流是不可能的：当流读取一个不支持随机访问的文件时。这种流通常会预读，并且由于文件不能随机访问，没有办法可以返还已经读到的多余数据。当流读取一个支持随机访问的文件时，调用 `fflush()` 会清空流，但同时把文件指针留在一个不可知的位置；在继续任何 IO 前，你必须设置文件指针。

关闭一个输出流也会执行 `fflush()`，因此，这是一个有效关闭输出流的方法。

当使用流的底层文件描述符执行控制操作时，比如设置终端模式，你不需要先清空流。这些操作不影响文件位置，也不受文件位置的影响。然而，对于流已经放入缓冲区的数据，在后续被清空时，将受到新终端模式的影响。想避免这种情况，在设置终端模式前，先清空流[→ 终端模式]()。