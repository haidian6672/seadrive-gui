# 桌面客户端虚拟磁盘 (SeaDrive) 设计规范

## 功能规范

为用户提供一个 PC 机上的虚拟磁盘，能像访问本地文件系统一样访问 Seafile 的所有资料库。功能上说，可以理解为为文件同步提供了一个虚拟磁盘的界面。

- 在虚拟磁盘里面列出用户可以访问的所有资料库。客户端自动同步所有资料库的元数据（只需要同步目录对象）。在初次同步完成之前，点击进入该资料库会显示空目录。
- 内部实现依然基于 Seafile 的文件同步核心算法，但是只同步元数据，不同步 block。
- 用户写入数据，先写入本地的缓存，后台自动 commit 然后上传服务器。
- 用户读取数据：元数据直接访问本地的缓存，访问文件内容需要实时从服务器下载。由于本地的元数据缓存30秒自动同步一次，所以不一定能立即看到服务器上最新的更改（这个可以进一步优化）。
- 在网络不可用的时候还可以访问，但是只能读写本地的缓存。由于无法下载未缓存的文件，所以不能修改一个未缓存的文件。创建新文件是可以的。本地的更改下次连上服务器的时候自动同步。
- 用户在 windows explorer 里面能通过图标来知道哪些文件已经缓存在本地。explorer 扩展的右键菜单功能在虚拟磁盘中也能使用。
- 如果用户配置多个帐号，则虚拟盘中只显示当前帐号的资料库。

跟 windows 共享相比：

- 离线也能访问
- 能使用 Seafile 的版本管理，共享等功能
- 元数据的访问有一点延迟，不一定能立即看到最新内容
- 大文件的随机读写可能网络延迟较大

后两点可以后续慢慢改进算法。

跟 webdav 挂载相比：

- 离线也能访问
- 列出目录更快
- 读取大文件更快
- 写入大量文件时的上传速度快
- 元数据的访问有一点延迟，不一定能立即看到最新内容

## 架构设计

分为三个组件：

- seadrive daemon
    - 实现虚拟文件系统接口（FUSE, DOKAN）
	- 实现内部的数据管理和同步
	- 基于 Windows pipe/Unix socket 提供 RPC 接口供其他组件调用
- seadrive gui
    - 提供托盘小图标，菜单，以及一些简单的窗口
	- 通过 RPC 与 seadrive daemon 交互
- explorer 扩展
    - 提供文件和目录的状态图标，右键菜单扩展
	- 同步另一个 pipe 与 seadrive gui 通信

## 数据结构

在客户端本地的 storage 目录下，包括以下主要的目录：

- commits：存放 commit 对象
- fs: 存放 fs 对象
- blocks: block 缓存
- journal: 每个资料库有一个 journal，记录从 head commit 到最新状态之间的元数据修改操作。Journal 在磁盘上保存为 SQLite 数据库。
- file-cache: 文件缓存，文件被读写之前都要先下载到这个目录下面

### Journal

Journal 是一个只追加的（append-only）元数据操作数据库。它记录了历史上本地所有的文件系统元数据操作（如创建目录，删除文件，修改文件等）。每一个条目是一个 key/value 对，key 为操作 ID （opid），value 为一个 JSON 格式的操作描述。opid 是一个递增的整数。从历史上某个 commit 对应的 opid 开始，把以后的所有文件操作 replay 一次，就能得到文件系统树的最新状态。

在生成新的 head commit 时，只需要 replay 一次从上一个 head commit 到当前最新 opid 之间的所有操作，就能得到一个 change set，再由 change set 生成一个 commit。这样的好处是在 index 过程中（可能比较耗时），新的写入操作不需要暂停。而且，即使 index 过程中程序终止，下次重启的时候，只需要再 replay 一次 journal 就可以生成 commit 了。

由于 SQLite 数据库单次 insert 操作的延迟较大，所以采用定期批量 insert 的策略。
- 如果积累了超过100个操作，则立即执行 insert
- 如果过去一秒内没有新的操作，则执行 insert

### 内存中的数据结构

SeaDrive daemon 在内存中为每一个资料库保存一个目录树结构（RepoTree）。这个结构与磁盘中最新 fs tree 状态是保持一致的。虚拟文件接口的代码可以直接查询这个树，得到目录里的文件列表，文件元信息，同步状态等。每次重启之后，daemon 先把 head commit 的树导入内存，然后把 head commit 到最新的 opid 之间的操作在内存中 replay 一次，就得到了最新的目录树状态。

### 文件同步和缓存状态

在操作系统的文件管理器中需要通过小图标的方式显示文件的同步和缓存状态。文件管理器通过扩展来获取文件/目录的同步状态，扩展调用 seadrive-gui 的接口，然后 seadrive-gui 调用 seadrive-daemon 的 RPC （get_path_sync_status）来获取状态信息。

目录有5种状态：

- 在云端（"cloud"）：目录下没有文件被缓存，显示一个“云”图标
- 部分缓存（"partial_synced"）：目录下有文件被缓存，显示一个“部分同步”图标
- 用户手工设置缓存 ("synced")：下面所有文件都被缓存，显示“完全同步”图标
- 正在同步 ("syncing")：下面有文件上传或者下载
- 只读：目录为只读权限

文件有7种状态：

- 在云端 ("cloud")：没有缓存下来，或者云端版本比本地新，显示一个“云”图标
- 已缓存 ("synced")：本地版本和云端版本一致，显示“已同步”图标
- 正在同步 ("syncing")：文件正在上传或者下载
- 待同步（"none"）：本地有修改未上传，且还没有调度上传，（一般出现在大量文件同步分批上传的情况），不显示图标
- 只读：文件只读，继承自父目录
- 锁定：被别人锁定
- 自己锁定

其中，只读、锁定、自己锁定三个状态只在目录或者文件处在“已同步”状态下才显示。

在内存中对每个 repo 维护两个 SyncStatusTree 数据结构（源代码在 seafile 的 daemon/sync-status-tree.c）。一个是 `syncing_tree` 一个是 `synced_tree`。

具体的算法细节参考 seafile 的 sync-mgr.c 中关于 sync status 的代码。

同步状态的维护：
- 程序启动之后，启动一个线程在后台扫描 repo 的缓存文件，把已缓存的文件加入`synced_tree`
- 文件下载的时候进入 syncing 状态，下载完毕进入 synced 状态
- 调度文件上传之前，把文件状态设置为 syncing；当上传完成后，把已经上传的文件状态设置为 synced。
- 文件/目录被删除/重命名后，清除状态
- 缓存文件被清理后，清除状态

### File Cache

在 file-cache 目录下，每一个 repo 有一个子目录。在 repo 子目录里面，是一个跟 repo 本身目录结构对应的目录层次结构。其中每一个缓存下来的文件都保存在一个完整的路径下面。比如文件 "a/b/c/d.pdf" 保存在 `file-cache/<repo-id>/a/b/c/d.pdf` 中。

在磁盘上，每个缓存文件关联三个扩展属性（extended attribute，多数文件系统都支持）：
- `seafile-mtime`: 缓存文件在 seafile 内部对应的文件版本的 mtime。这个属性在文件完全缓存到本地之后，设置为对应的 seafile 对象的 mtime。通过对比缓存文件本身的 mtime 和 seafile-mtime 属性，就可以知道缓存文件内容是否被修改。（缓存文件本身的 mtime 需要在一开始被设置成跟 seafile-mtime 一致。）同时，如果 seafile-mtime 属性被设置，则标志着这个文件的缓存已经完成。
- `seafile-size`: 缓存文件在 seafile 内部对应的文件版本的大小。作用类似 seafile-mtime，都是用来检查缓存的文件是否被修改了。
- `file-id`: 缓存的文件对应 seafile 对象 id。

三个属性在每次文件的更新 commit 之后都需要更新。这个由同步算法来控制。

在内存中，需要维护如下的数据结构：
- CacheFile 结构：对应一个缓存文件，记录缓存的文件的元信息。这个结构里面包含一个引用计数字段，每打开一个 file handle 就要对 cachedfile 结构的引用加一。
- FileHandle 结构：对应一个打开的文件描述符。
- 一个由文件路径到 CachedFile 结构的哈希表

FileCache 对外返回提供读写文件的接口，通过一个 FileHandle 对象作为文件的描述符。FileHandle 对象的定义如下：

```
struct FileHandle {
    CachedFile *cached_file;
	int fd;
};
```

可见，FileHandle 对象保持一个到 CachedFile 的引用，每返回一个新的 FileHandle 对象，都需要增加 CachedFile 里面的 `n_open` 变量。

当一个文件被打开的时候，启用一个后台线程从服务器上下载文件到缓存目录中。
- 文件的读操作轮询等到自己的数据区域缓存完成之后，把结果返回给调用进程。
- 文件的写操作轮询等到整个文件缓存完成之后（`seafile-mtime`属性已设置）才能进行。

打开一个文件之前，需要通过一些检查防止意外覆盖本地的未提交文件修改。
- 如果 RepoTree 中的文件的 `file-id` 与缓存文件的 `file-id` 属性不同，而且缓存文件的 mtime 与 `seafile-mtime` 属性不同，则表明 seafile 的内部文件状态已经更新，但是缓存文件的状态（由于某些原因）未更新，或者还有修改未提交。此时需要继续打开已缓存的文件，而不是从服务器下载更新的版本。如果同步算法正常工作，这种情况是很少出现的。
- 如果 RepoTree 中文件的 `file-id` 与缓存文件的 `file-id` 相同，此时继续返回缓存中的内容。
- 如果 RepoTree 中文件的 `file-id` 与缓存文件的 `file-id` 不同，而且文件的 mtime 和 `sefile-mtime` 相同，则需要从服务器重新下载新的文件版本。

自动的缓存文件的清理使用的策略如下：基于空间的缓存清理策略：用户指定本地缓存最多占用多少空间（比如 1GB），当缓存空间占用超过阈值，把缓存空间清理到阈值的 70%。

清理缓存的算法如下：
1. 遍历 file cache 中已缓存的文件，按照文件的创建时间从小到大排序，
    - 如果文件对应的 cached file 结构引用计数不为0，表示目前还有应用在使用该文件，跳过
	- 如果缓存文件已被修改，跳过
	- 如果还没有上传到服务器，跳过
	- 否则删除该缓存文件
2. 重复上述过程直到清理的标准完成
3. 删除待清理链表中的对象

### 数据库

本地 sqlite 数据库的表格与同步客户端基本是一样的，除了以下几点：

- 增加一个 AccountRepos 表格，缓存每个帐号在服务器上的资料库列表。
- Branch 表增加一个 opid 列，其含义如下：
    - 对 local branch，记录生成该 commit 所对应的 opid。生成一下个 commit 时，只需要从 opid + 1 开始将所有的操作执行一次。
	- 对 master branch，记录的是第一个还没有上传到服务器上的操作的 opid。这个用于修复本地损坏的对象。详细见后面的“数据完整性”一节。

## 文件系统操作的实现

### 列出目录以及文件元信息访问

都通过查询内存中的目录树结构返回结果。

### 元数据修改操作

包括创建/删除文件，创建/删除目录，移动（重命名）。修改内存中的目录树，然后写入 journal。

### 读取文件

文件被以读或者写模式打开的时候（open 被调用）缓存管理器在后台启动一个缓存线程，从服务器下载文件。缓存的流程：

1. 从服务器上下载文件的 file 对象
2. 如果文件大于2MB，则从服务器上获取文件的块大小列表
3. 利用块大小列表，计算出上次缓存到那个块，然后从这个块开始继续缓存。

在后台缓存文件的同时，系统调用处理代码返回应用请求的数据。流程如下：

1. 如果应用请求的数据起始位置比当前已缓存的大小大超过 100KB，则判定为随机读操作，尝试直接从服务器获取该段数据返回。（这里可以利用服务器现有的 byte-range GET 支持来实现）。
2. 否则通过轮询的方式等待后台缓存线程缓存到应用所请求的数据位置。
3. 如果在5秒内，后台还没有完全缓存完应用请求的所有数据，在 windows 上，可以只返回用户请求的一部分数据，在 linux/mac 上由于 fuse 要求一定要返回所请求的全部数据，所以只能返回 IO 错误。

### 写入文件

写入文件前，必须要先等待文件被完全缓存。

写入文件的流程大概是：

1. 把修改写入缓存的文件。
2. 把修改后的文件元信息（修改文件路径，最后修改时间，大小）写入 journal。

## 同步算法

clone 过程只需要下载当前状态下所有的目录对象即可。这个过程在客户端完成安装之后自动执行。

同步在后台线程中执行，算法与同步客户端的基本一样：

- 在累积了100MB的文件修改，或者2秒内没有任何的文件活动，则发起 index 操作。index 操作把 journal 中从 head commit 到最新 opid 的所有操作在内存中 replay 一次，得到 change set。根据 change set 即可确定需要 index 的文件。此时，正常的文件写入操作还能继续执行。
- index 完成之后上传数据，上传完成之后检查服务器端是否有冲突，如果有则发起元数据下载操作。
- 下载了最新的元数据之后，需要与 head commit 进行 diff 得到一个修改集合，然后把修改集应用到内存的目录树中。如果发现有并发修改的情况，生成冲突文件，并把生成冲突文件操作添加到 journal。注意没有冲突的情况下，是不需要向 journal 中添加记录的。最后把 head commit 更新为服务器上的版本。此后，如果把 journal 中的操作在最新的 head commit 上 replay，得到的应该也是当前内存中的目录树。当然这也是比较少见的情况，一般下载之后本地应该是没有未 commit 的改动的。
- 每30秒自动检查一次服务器上是否有新的改动需要下载。如果当前还有 commit 没有上传或者还有文件写入活动，则不检查下载，先完成上传操作。

## 数据完整性

由于数据损坏导致不能同步有两种可能：
- 服务器端的数据损坏了，通过 fsck 来修复了
- 客户端的元数据（fs对象等）损坏

SeaDrive 修复这两种情况的方法是类似的，用户需要退出登录当前的账号，然后重新登录。退出登录时会把账号所有 repo 的元数据清除，重新登录之后会重新下载，这样如果是本地元数据损坏的话，就自动修复了。

有时本地会有尚未上传的文件改动，因此在重新登录之后第一次加载 fs tree 的时候，会扫描 file cache 目录中已有的缓存文件，如果发现时间戳、文件内容与服务器上版本不同，会生成冲突文件。这个跟同步客户端重新同步一个资料库的处理方法是类似的。

## RPC

```
seafile_switch_account

设置当前的帐号信息。

server: 服务器地址，需要包括 http/https 前缀
username: 用户名
token: 用户访问服务器的 token
is_pro: 服务器是否专业版
```

```
seafile_delete_account

从底层删除该帐号对应的缓存信息，包括 repo 列表缓存，元数据缓存和文件缓存。

GUI 退出登录的时候，以及删除一个帐号的时候，应该调用这个函数来清理底层的缓存信息。退出登录的时候可以让用户选择是否清理文件缓存。

server: 服务器地址，需要包括 http/https 前缀
username: 用户名
remove_cache: 是否删除文件缓存
```

```
seafile_unmount

unmount 虚拟文件系统。GUI 在退出程序之前应该先调用这个函数。该函数在 Linux/MacOS 和 Windows 上行为有所不同。在 Linux/Mac 上，并不 unmount 挂载点；在 Windows 上会 unmount 虚拟盘。所以，在 Linux/Mac 上，GUI 在下次挂载之前，需要先 umount 或者用 fusermount 命令清除原有的挂载点。
```

```
seafile_get_clean_cache_interval

返回底层清理文件缓存的周期，以秒为单位
```

```
seafile_set_clean_cache_interval

设置底层清理文件缓存的周期，以秒为单位

seconds: 每多少秒执行一次清理文件缓存操作，默认10分钟
```

```
seafile_get_cache_size_limit

返回底层的文件缓存大小上限，以字节为单位。
```

```
seafile_set_cache_size_limit

设置底层文件缓存大小上线，以字节为单位。

limit: 当文件缓存大小超过这个上限时执行清理缓存操作。默认为 10GB（10,000,000,000字节）。
```

```
seafile_get_global_sync_status

desc: 获取全局同步状态
params: void
return val: json obj {"is_syncing":"是否正在同步"， “sent_bytes”:"已发送字节数"， “recv_bytes”:“已接收字节数”}
```

```
seafile_get_path_sync_status

desc: 获取某个路径的同步状态
params: repo_uname(string: repo unique name), path(string)
return val: string(同步状态: "none", "syncing", "error", "synced", "readonly", "locked","locked_by_me")
```

```
seafile_get_sync_notification

desc: 获取同步消息通知
params: void
return val: 有通知: json obj  ; 无通知: NULL
```

JSON object 类型和格式：

* 同步完成

`{"type": "sync.done",  "repo_id": "repo_id", "repo_name": "repo_name", "commit_id": "commit_id", "parent_commit_id": "parent_commit_id", "commit_desc": "commit_desc"}`

* 部分上传完成

`{"type": "sync.multipart_upload",  "repo_id": "repo_id", "repo_name": "repo_name", "commit_id": "commit_id", "parent_commit_id": "parent_commit_id", "commit_desc": "commit_desc"}`

对这类提醒，应显示 “xxx 有文件已上传”。

* 同步错误

`{"type": "sync.error", "repo_id": "repo_id", "repo_name": "repo_name", "path": "path", "err_id": id}` 其中 `path` 域可能不存在。`err_id` 可以是以下的数字：

```
#define SYNC_ERROR_ID_FILE_LOCKED_BY_APP        0
#define SYNC_ERROR_ID_FOLDER_LOCKED_BY_APP      1
/* #define SYNC_ERROR_ID_FILE_LOCKED 2 */
/* #define SYNC_ERROR_ID_INVALID_PATH 3 */
#define SYNC_ERROR_ID_INDEX_ERROR               4
#define SYNC_ERROR_ID_ACCESS_DENIED             5
#define SYNC_ERROR_ID_QUOTA_FULL                6
#define SYNC_ERROR_ID_NETWORK                   7
#define SYNC_ERROR_ID_RESOLVE_PROXY             8
#define SYNC_ERROR_ID_RESOLVE_HOST              9
#define SYNC_ERROR_ID_CONNECT                   10
#define SYNC_ERROR_ID_SSL                       11
#define SYNC_ERROR_ID_TX                        12
#define SYNC_ERROR_ID_TX_TIMEOUT                13
#define SYNC_ERROR_ID_UNHANDLED_REDIRECT        14
#define SYNC_ERROR_ID_SERVER                    15
#define SYNC_ERROR_ID_LOCAL_DATA_CORRUPT        16
#define SYNC_ERROR_ID_WRITE_LOCAL_DATA          17

#define SYNC_ERROR_ID_GENERAL_ERROR             100
```

* 所有资料库加载完成

```
{"type": "fs-loaded"}
```

* 跨资料库移动文件/目录

由于跨资料库移动文件/目录处理需要直接在服务器上进行，所以底层把这个操作做成异步。用户移动文件的时候，客户端在后台向服务器发送一个移动文件的请求，服务器完成后回复请求成功。在这个过程中，用户会在本地目录中看不到任何的变化，需要等服务器上的改动同步到本地才能看到。因此，界面需要弹出气泡提示用户移动操作正在后台进行，完成后再提示用户已经完成。

```
{"type": cross-repo-move-evnet-type,
 "srcpath": "path1", "dstpath": "path2"}
```

其中，cross-repo-move-event-type 可以是 `cross-repo-move.start`, `cross-repo-move.done`, `cross-repo-move.error` 三种。

* 文件操作错误

目前针对两种容易引起用户混淆的文件操作错误发送提醒。**这些提醒发送到 SEADRIVE_EVENT_CHAN**

不能在根目录创建文件

```
{ "type": "fs_op_error.create_root_file", "path": path}
```

不能删除根目录下的 repo。

```
{"type": "fs_op_error.remove_repo", "path": repo_name}
```

* 文件下载操作提醒

**这些提醒发送到 SEADRIVE_EVENT_CHAN**

```
{ "type": "file-download.start", "path": path}
{ "type": "file-download.stop", "path": path}
{ "type": "file-download.done", "path": path}
```

```
seafile_get_upload_progress

desc: 返回正在上传的文件进度以及最近10个上传的文件
param: void
return: json object 
{
"uploaded_files": ["file_path1", "file_path2", ...],
"uploading_files": [
{"file_path": "path1", "uploaded": 10, "total_upload": 100},
......
]
}
```

```
seafile_get_download_progress

desc: 返回正在下载的文件进度以及最近10个下载的文件
param: void
return: json object
{
"downloaded_files": ["file_path1", "file_path2", ...],
"downloading_files": [
{"file_path": "path1", "downloaded": 10, "total_download": 100},
......
]
}
```

```
seafile_cancel_download

desc: 取消下载一个文件
param: full_file_path: 文件的路径，形如 repo_name/path，与 get_download_progress 返回的文件路径格式一致
return: 0 on success, -1 on failure
```

```
seafile_list_sync_errors

desc: 返回当前的同步错误
param: void
return: json object
{
[
{"repo_id": "id", "repo_name": "name", "path": "path", "err_id": 6, "timestamp": 1323243439},
....
]
}
```

界面上显示两种错误：

- 全局网络错误：底层每30秒获取一次 repo 列表，因此能检测网络错误。这种错误，对象里面的 `repo_id`, `repo_name`, `path` 字端都是没有设置的。
- 单个 repo 的同步错误：比如网络错误、本地元数据损坏、quota 满、没有权限等。需要显示 repo 名字和具体的错误字符串。这种错误，对象里面的 `path` 字端没有设置。

界面定期从底层获取错误列表，如果错误列表不为空，则在托盘图标显示一个感叹号。界面提供一个同步错误菜单项，点击打开一个列表，从底层获取当前的错误。除此之外，底层还会发送错误弹出提示。