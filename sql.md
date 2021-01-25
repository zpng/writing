# Golang database/sql深入解析
## 引言
本文主要是深入源码级别讲解一些golang中sql包中的一些困惑。大家在使用sql连接的时候经常会说一些术语比如连接池，连接。可是估计有很多人不懂到底什么是连接，然后为什么有连接池的存在，还有不同请求过来后是怎么处理的，跟goroutine是怎么结合的。如果读者能很清楚的讲明白这些问题，那么可以跳过该文章，本文大概需要阅读二十分钟左右，相信我阅读完这篇文章后，你会对golang sql连接背后的原理上一个台阶。

### 什么是连接
连接是个很抽象的概念，连接在计算机中是socket，其实就是个4元组。(源ip，源端口，目标ip，目标端口)。注意该四元组中的任何一个元素不同都代表不同的连接。
### 为什么有连接池
在我们连接数据库的场景，一般是应用服务器去连接数据库服务器，数据库服务器的ip和端口一般是不变的，这里我们略过一些集群主从等切换导致的ip和端口改变。而应用服务器在当前一般都是部署在docker容器里面，不重启ip一般不会发生变化，那么唯一变化的就是源端口。

那么我们试想一个场景，假如说应用服务器接收请求每秒几千个请求，每一个请求都新建一个连接，因为端口从释放到重新复用一般是有间隔时间的, 在没有配置SO_REUSEADDR的情况下一个端口释放后会等待两分钟之后才能再被使用，而我们知道端口是有范围限制的0到65535, 而且普通应用可以用的端口范围比这个还小。那我们简单计算下2min内需要的不同的端口是120000个，按照每秒1000qps算的话，很明显端口不满足这个要求，而且实际中qps可能远不止这个数，还有可能运行其他的client占用端口。

那么该怎么办呢，聪明的程序员们想到了一个办法，使用连接池，使用连接池的本质就是将一些源端口建立的连接复用，放到池子里，而不是每次释放端口，这样每次请求过来的时候直接从池子里面取建立好的连接就可以了，这样既可以解决刚才的端口问题，也可以提高性能。
### 连接池在golang中是怎么实现的
俗话说talk is cheap，show me the code。那么代码中是怎么实现的呢，我们下面就来分析下，先说下核心思想，其实就是维护一个连接的切片，然后维护这个切片来获取或者新建连接。

本文的代码基于go 1.14
```
type DB struct {

	connector driver.Connector
	// numClosed is an atomic counter which represents a total number of
	// closed connections. Stmt.openStmt checks it before cleaning closed
	// connections in Stmt.css.
	numClosed uint64

	mu           sync.Mutex // protects following fields
	freeConn     []*driverConn
	connRequests map[uint64]chan connRequest
	nextRequest  uint64 // Next key to use in connRequests.
	numOpen      int    // number of opened and pending open connections
	// Used to signal the need for new connections
	// a goroutine running connectionOpener() reads on this chan and
	// maybeOpenNewConnections sends on the chan (one send per needed connection)
	// It is closed during db.Close(). The close tells the connectionOpener
	// goroutine to exit.
	openerCh          chan struct{}
	resetterCh        chan *driverConn
	closed            bool
}
```
上文的DB struct是sql包非常重要的一个结构体, 其中删除了一些跟本文无关的变量, 我们重点关注下上面的freeConn元素, 它是一个driverConn指针的切片，该结构体如下:
```
/ driverConn wraps a driver.Conn with a mutex, to
// be held during all calls into the Conn. (including any calls onto
// interfaces returned via that Conn, such as calls on Tx, Stmt,
// Result, Rows)
type driverConn struct {
	db        *DB
	createdAt time.Time

	sync.Mutex  // guards following
	ci          driver.Conn
	closed      bool
	finalClosed bool // ci.Close has been called
	openStmt    map[*driverStmt]bool
	lastErr     error // lastError captures the result of the session resetter.

	// guarded by db.mu
	inUse      bool
	onPut      []func() // code (with db.mu held) run when conn is next returned
	dbmuClosed bool     // same as closed, but guarded by db.mu, for removeClosedStmtLocked
}
```
上面结构体中包含指向DB的指针，还有特定的数据库驱动需要实现的接口driver.Conn, 这个接口的实现是特定于不同的数据库的，例如mysql的实现在包 [github.com/go-sql-driver/mysql](https://github.com/go-sql-driver/mysql) 中。

请求来时，从缓存池中找一个连接或者新建一个连接, 代码如下:
```
// conn returns a newly-opened or cached *driverConn.
func (db *DB) conn(ctx context.Context, strategy connReuseStrategy) (*driverConn, error) {

	// Prefer a free connection, if possible.
	numFree := len(db.freeConn)
	if strategy == cachedOrNewConn && numFree > 0 {
        //取可用连接的第一个连接
		conn := db.freeConn[0]
		copy(db.freeConn, db.freeConn[1:])
		db.freeConn = db.freeConn[:numFree-1]
		conn.inUse = true
	}

	// Out of free connections or we were asked not to use one. If we're not
	// allowed to open any more connections, make a request and wait.
    // 如果已经到了最大连接池的个数, 那么先将请求放到等待列表中
	if db.maxOpen > 0 && db.numOpen >= db.maxOpen {
		// Make the connRequest channel. It's buffered so that the
		// connectionOpener doesn't block while waiting for the req to be read.
		req := make(chan connRequest, 1)
		reqKey := db.nextRequestKeyLocked()
		db.connRequests[reqKey] = req
		db.waitCount++


		// Timeout the connection request with the context.
		select {
		case <-ctx.Done():
			// Remove the connection request and ensure no value has been sent
			// on it after removing.
			db.mu.Lock()
			delete(db.connRequests, reqKey)
			db.mu.Unlock()

			atomic.AddInt64(&db.waitDuration, int64(time.Since(waitStart)))

			select {
			default:
			case ret, ok := <-req:
				if ok && ret.conn != nil {
					db.putConn(ret.conn, ret.err, false)
				}
			}
			return nil, ctx.Err()
		case ret, ok := <-req:
			atomic.AddInt64(&db.waitDuration, int64(time.Since(waitStart)))

			if !ok {
				return nil, errDBClosed
			}
			if ret.err == nil && ret.conn.expired(lifetime) {
				ret.conn.Close()
				return nil, driver.ErrBadConn
			}
			if ret.conn == nil {
				return nil, ret.err
			}
			// Lock around reading lastErr to ensure the session resetter finished.
			ret.conn.Lock()
			err := ret.conn.lastErr
			ret.conn.Unlock()
			if err == driver.ErrBadConn {
				ret.conn.Close()
				return nil, driver.ErrBadConn
			}
			return ret.conn, ret.err
		}
	}

	db.numOpen++ // optimistically
	db.mu.Unlock()
    //新建一个连接
	ci, err := db.connector.Connect(ctx)
	if err != nil {
		db.mu.Lock()
		db.numOpen-- // correct for earlier optimism
		db.maybeOpenNewConnections()
		db.mu.Unlock()
		return nil, err
	}
	dc := &driverConn{
		db:        db,
		createdAt: nowFunc(),
		ci:        ci,
		inUse:     true,
	}
	return dc, nil
}
```
注意其中新建完请求之后select，其中等待ctx.Done或者 <-req, 也就是说ctx超时或者block等待可用的连接。
那么连接在什么时候释放会缓存池的呢?
```
func (db *DB) execDC(ctx context.Context, dc *driverConn, release func(error), query string, args []interface{}) (res Result, err error) {
	defer func() {
		release(err)
	}()
```

如上所述在exec的defer语句，还有其它query完之后也都会释放连接，释放时会调用下面的函数归还连接到连接池中。


```
func (db *DB) putConnDBLocked(dc *driverConn, err error) bool {
	if c := len(db.connRequests); c > 0 {
		var req chan connRequest
		var reqKey uint64
		for reqKey, req = range db.connRequests {
            //找到了一些正在等待的请求
			break
		}
		delete(db.connRequests, reqKey) // Remove from pending requests.
		if err == nil {
			dc.inUse = true
		}
        //将这个请求绑定上这个要放回连接池的连接
		req <- connRequest{
			conn: dc,
			err:  err,
		}
		return true
	} else if err == nil && !db.closed {
		if db.maxIdleConnsLocked() > len(db.freeConn) {
            //没有正在等待处理的请求，则将该连接放回到连接池切片中
			db.freeConn = append(db.freeConn, dc)
			db.startCleanerLocked()
			return true
		}
		db.maxIdleClosed++
	}
	return false
}
```
如上所示在归还的过程中，如果发现了一些待处理的请求，则直接将该连接绑定到这个请求中，去处理。如果没有待处理的请求，则放回到连接池中。

### mysql连接背后
上文代码中涉及到连接可以看到是通过db.connector.Connect(ctx)这个接口去新建的，那么这个在具体的驱动里面是怎么实现的呢，我们看下[github.com/go-sql-driver/mysql](https://github.com/go-sql-driver/mysql) 这个包中相关代码。

Connect函数实现如下:
```
func (c *connector) Connect(ctx context.Context) (driver.Conn, error) {
	var err error

	// New mysqlConn
	mc := &mysqlConn{
		maxAllowedPacket: maxPacketSize,
		maxWriteSize:     maxPacketSize - 1,
		closech:          make(chan struct{}),
		cfg:              c.cfg,
	}
	mc.parseTime = mc.cfg.ParseTime

	// Connect to Server
	dialsLock.RLock()
	dial, ok := dials[mc.cfg.Net]
	dialsLock.RUnlock()
	if ok {
		dctx := ctx
		if mc.cfg.Timeout > 0 {
			var cancel context.CancelFunc
			dctx, cancel = context.WithTimeout(ctx, c.cfg.Timeout)
			defer cancel()
		}
		mc.netConn, err = dial(dctx, mc.cfg.Addr)
	} else {
		nd := net.Dialer{Timeout: mc.cfg.Timeout}
        //连接新建的代码
		mc.netConn, err = nd.DialContext(ctx, mc.cfg.Net, mc.cfg.Addr)
	}

	if err != nil {
		return nil, err
	}

	return mc, nil
}

```
如上代码中注释所示，最终是使用的net包的DialContext去跟数据库地址建立连接。至此我们分析完了整个流程。
### 总结
golang标准库中database/sql封装了连接池的实现，并将特定于不同驱动如mysql, postgreSQL的实现等抽象成接口，既方便了sql的使用，也提高了性能。
本文梳理了什么是连接，以及连接池的作用，抽丝剥茧，深入浅出的介绍了整个过程，希望大家阅读完之后能切实的连接池背后的原理有了具象的理解。