# EXT4

## EXT4文件系统三种日志模式

### journal

`data=journal` 模式提供了完全的数据块和元数据快的日志，所有的数据都会被先写入到日志里，然后再写入磁盘（掉电非易失存储介质）上。在文件系统崩溃的时候，日志就可以进行重放，把数据和元数据带回到一个一致性的状态，journal模式性能是三种模式中最低的，因为所有的数据都需要日志来记录。这个模式不支持 allocation 以及 O_DIRECT。

&nbsp;

### ordered(*)

在 `data=ordered` 模式下，ext4文件系统只提供元数据的日志，但它逻辑上将与数据更改相关的元数据信息与数据块分组到一个称为事务的单元中。当需要把元数据写入到磁盘上的时候，与元数据关联的数据块会首先写入。也就是数据先落盘，再做元数据的日志。一般情况下，这种模式的性能会略逊色于`writeback`但是比`journal`模式要快的多。

&nbsp;

### writeback

在 `data=writeback` 模式下，当元数据提交到日志后，data可以直接被提交到磁盘。即会做元数据日志，数据不做日志，并且不保证数据比元数据先落盘。`writeback` 是ext4提供的性能最好的模式。
