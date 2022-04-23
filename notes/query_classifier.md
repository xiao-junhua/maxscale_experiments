# Query Classifier分析
Query classfier负责将Maxscale收到的packet解析，确定packet的类型和操作。
Maxscale的query classfier共有两种实现方式，分别是qc_mysqlembedded,qc_sqlite。在使用上，可以通过maxscale的global configuration来实现。  
```
[maxscale]
query_classifier=qc_sqlite
```
## Query Classifier源码分析

Query Classifier的头文件是`include/maxscale/query_classifier.h`,实现文件是`server/core/query_classifier.cc`。  
两种实现方式分别处于`query_classifier/qc_mysqlembedded`和`query_classifier/qc_sqlite`。

在Maxscale之中，将数据存储到一个称为`GWBUF`的链表之中，query classfier使用一个`GWBUF *buffer`作为输入，得到输出内容页作为外部数据添加到`GWBUF`链表之中.GWBUF的定义如下:

```c++
struct buffer_object_st
{
    bufobj_id_t      bo_id;
    void*            bo_data;
    void            (*bo_donefun_fp)(void *);
    buffer_object_t* bo_next;
};
```

而实际进行query命令解析的函数是:  

```c++
qc_parse_result_t qc_parse(GWBUF* buffer);

// 具体实现
static bool parse_query(GWBUF* query, uint32_t collect)
{
    bool parsed = false;
    ss_dassert(!query_is_parsed(query, collect));

    if (GWBUF_IS_CONTIGUOUS(query))
    {
        uint8_t* data = (uint8_t*) GWBUF_DATA(query);

        if ((GWBUF_LENGTH(query) >= MYSQL_HEADER_LEN + 1) &&
            (GWBUF_LENGTH(query) == MYSQL_HEADER_LEN + MYSQL_GET_PAYLOAD_LEN(data)))
        {
            uint8_t command = MYSQL_GET_COMMAND(data);

            if ((command == MXS_COM_QUERY) || (command == MXS_COM_STMT_PREPARE))
            {
                bool suppress_logging = false;

                QcSqliteInfo* pInfo =
                    (QcSqliteInfo*) gwbuf_get_buffer_object_data(query, GWBUF_PARSING_INFO);

                if (pInfo)
                {
                    ss_dassert((~pInfo->m_collect & collect) != 0);
                    ss_dassert((~pInfo->m_collected & collect) != 0);

                    // If we get here, then the statement has been parsed once, but
                    // not all needed was collected. Now we turn on all blinkenlichts to
                    // ensure that a statement is parsed at most twice.
                    pInfo->m_collect = QC_COLLECT_ALL;

                    // We also reset the collected keywords, so that code that behaves
                    // differently depending on whether keywords have been seem or not
                    // acts the same way on this second round.
                    pInfo->m_keyword_1 = 0;
                    pInfo->m_keyword_2 = 0;

                    // And turn off logging. Any parsing issues were logged on the first round.
                    suppress_logging = true;
                }
                else
                {
                    pInfo = QcSqliteInfo::create(collect);

                    if (pInfo)
                    {
                        // TODO: Add return value to gwbuf_add_buffer_object.
                        gwbuf_add_buffer_object(query, GWBUF_PARSING_INFO, pInfo, buffer_object_free);
                    }
                }

                if (pInfo)
                {
                    this_thread.pInfo = pInfo;

                    size_t len = MYSQL_GET_PAYLOAD_LEN(data) - 1; // Subtract 1 for packet type byte.

                    const char* s = (const char*) &data[MYSQL_HEADER_LEN + 1];

                    this_thread.pInfo->m_pQuery = s;
                    this_thread.pInfo->m_nQuery = len;
                    // 关键
                    parse_query_string(s, len, suppress_logging);
                    this_thread.pInfo->m_pQuery = NULL;
                    this_thread.pInfo->m_nQuery = 0;

                    if (command == MXS_COM_STMT_PREPARE)
                    {
                        pInfo->m_type_mask |= QUERY_TYPE_PREPARE_STMT;
                    }

                    pInfo->m_collected = pInfo->m_collect;

                    parsed = true;

                    this_thread.pInfo = NULL;
                }
                else
                {
                    MXS_ERROR("Could not allocate structure for containing parse data.");
                }
            }
            else
            {
                MXS_ERROR("The provided buffer does not contain a COM_QUERY, but a %s.",
                          STRPACKETTYPE(MYSQL_GET_COMMAND(data)));
                ss_dassert(!true);
            }
        }
        else
        {
            MXS_ERROR("Packet size %u, provided buffer is %ld.",
                      MYSQL_HEADER_LEN + MYSQL_GET_PAYLOAD_LEN(data),
                      GWBUF_LENGTH(query));
        }
    }
    else
    {
        MXS_ERROR("Provided buffer is not contiguous.");
    }

    return parsed;
}

// parse_query_string实现,借助sqlite的sql parser，理解query的含义
static void parse_query_string(const char* query, int len, bool suppress_logging)
{
    sqlite3_stmt* stmt = NULL;
    const char* tail = NULL;

    ss_dassert(this_thread.pDb);
    int rc = sqlite3_prepare(this_thread.pDb, query, len, &stmt, &tail);

    const int max_len = 512; // Maximum length of logged statement.
    const int l = (len > max_len ? max_len : len);
    const char* suffix = (len > max_len ? "..." : "");
    const char* format;

    if (this_thread.pInfo->m_operation == QUERY_OP_EXPLAIN)
    {
        this_thread.pInfo->m_status = QC_QUERY_PARSED;
    }

    if (rc != SQLITE_OK)
    {
        if (qc_info_was_tokenized(this_thread.pInfo->m_status))
        {
            format =
                "Statement was classified only based on keywords "
                "(Sqlite3 error: %s, %s): \"%.*s%s\"";
        }
        else
        {
            if (qc_info_was_parsed(this_thread.pInfo->m_status))
            {
                format =
                    "Statement was only partially parsed "
                    "(Sqlite3 error: %s, %s): \"%.*s%s\"";

                // The status was set to QC_QUERY_PARSED, but sqlite3 returned an
                // error. Most likely, query contains some excess unrecognized stuff.
                this_thread.pInfo->m_status = QC_QUERY_PARTIALLY_PARSED;
            }
            else
            {
                format =
                    "Statement was neither parsed nor recognized from keywords "
                    "(Sqlite3 error: %s, %s): \"%.*s%s\"";
            }
        }

        if (!suppress_logging)
        {
            if (this_unit.log_level > QC_LOG_NOTHING)
            {
                bool log_warning = false;

                switch (this_unit.log_level)
                {
                case QC_LOG_NON_PARSED:
                    log_warning = this_thread.pInfo->m_status < QC_QUERY_PARSED;
                    break;

                case QC_LOG_NON_PARTIALLY_PARSED:
                    log_warning = this_thread.pInfo->m_status < QC_QUERY_PARTIALLY_PARSED;
                    break;

                case QC_LOG_NON_TOKENIZED:
                    log_warning = this_thread.pInfo->m_status < QC_QUERY_TOKENIZED;
                    break;

                default:
                    ss_dassert(!true);
                    break;
                }

                if (log_warning)
                {
                    MXS_WARNING(format, sqlite3_errstr(rc),
                                sqlite3_errmsg(this_thread.pDb), l, query, suffix);
                }
            }
        }
    }
    else if (this_thread.initialized) // If we are initializing, the query will not be classified.
    {
        if (!suppress_logging && (this_unit.log_level > QC_LOG_NOTHING))
        {
            if (qc_info_was_tokenized(this_thread.pInfo->m_status))
            {
                // This suggests a callback from the parser into this module is not made.
                format =
                    "Statement was classified only based on keywords, "
                    "even though the statement was parsed: \"%.*s%s\"";

                MXS_WARNING(format, l, query, suffix);
            }
            else if (!qc_info_was_parsed(this_thread.pInfo->m_status))
            {
                // This suggests there are keywords that should be recognized but are not,
                // a tentative classification cannot be (or is not) made using the keywords
                // seen and/or a callback from the parser into this module is not made.
                format = "Statement was parsed, but not classified: \"%.*s%s\"";

                MXS_WARNING(format, l, query, suffix);
            }
        }
    }

    if (stmt)
    {
        sqlite3_finalize(stmt);
    }
}
```

实际上，parse_query的过程是被忽略的，只有到maxscale想要获取query的信息时(即得到query的类型和操作),会调用一次parse_query，之后就直接从相应的GWBUF节点中获取相应信息。

Maxscale最想得到的有关query的信息是query的类型和qery的操作，并根据query的类型和操作将相应的query转发到对应节点，相关定义如下:
```c++

// query类型定义
typedef enum qc_query_type
{
    QUERY_TYPE_UNKNOWN            = 0x000000, /*< Initial value, can't be tested bitwisely */
    QUERY_TYPE_LOCAL_READ         = 0x000001, /*< Read non-database data, execute in MaxScale:any */
    QUERY_TYPE_READ               = 0x000002, /*< Read database data:any */
    QUERY_TYPE_WRITE              = 0x000004, /*< Master data will be  modified:master */
    QUERY_TYPE_MASTER_READ        = 0x000008, /*< Read from the master:master */
    QUERY_TYPE_SESSION_WRITE      = 0x000010, /*< Session data will be modified:master or all */
    QUERY_TYPE_USERVAR_WRITE      = 0x000020, /*< Write a user variable:master or all */
    QUERY_TYPE_USERVAR_READ       = 0x000040, /*< Read a user variable:master or any */
    QUERY_TYPE_SYSVAR_READ        = 0x000080, /*< Read a system variable:master or any */
    /** Not implemented yet */
    //QUERY_TYPE_SYSVAR_WRITE       = 0x000100, /*< Write a system variable:master or all */
    QUERY_TYPE_GSYSVAR_READ       = 0x000200, /*< Read global system variable:master or any */
    QUERY_TYPE_GSYSVAR_WRITE      = 0x000400, /*< Write global system variable:master or all */
    QUERY_TYPE_BEGIN_TRX          = 0x000800, /*< BEGIN or START TRANSACTION */
    QUERY_TYPE_ENABLE_AUTOCOMMIT  = 0x001000, /*< SET autocommit=1 */
    QUERY_TYPE_DISABLE_AUTOCOMMIT = 0x002000, /*< SET autocommit=0 */
    QUERY_TYPE_ROLLBACK           = 0x004000, /*< ROLLBACK */
    QUERY_TYPE_COMMIT             = 0x008000, /*< COMMIT */
    QUERY_TYPE_PREPARE_NAMED_STMT = 0x010000, /*< Prepared stmt with name from user:all */
    QUERY_TYPE_PREPARE_STMT       = 0x020000, /*< Prepared stmt with id provided by server:all */
    QUERY_TYPE_EXEC_STMT          = 0x040000, /*< Execute prepared statement:master or any */
    QUERY_TYPE_CREATE_TMP_TABLE   = 0x080000, /*< Create temporary table:master (could be all) */
    QUERY_TYPE_READ_TMP_TABLE     = 0x100000, /*< Read temporary table:master (could be any) */
    QUERY_TYPE_SHOW_DATABASES     = 0x200000, /*< Show list of databases */
    QUERY_TYPE_SHOW_TABLES        = 0x400000, /*< Show list of tables */
    QUERY_TYPE_DEALLOC_PREPARE    = 0x1000000 /*< Dealloc named prepare stmt:all */
} qc_query_type_t;

// query 操作定义
typedef enum qc_query_op
{
    QUERY_OP_UNDEFINED = 0,

    QUERY_OP_ALTER,
    QUERY_OP_CALL,
    QUERY_OP_CHANGE_DB,
    QUERY_OP_CREATE,
    QUERY_OP_DELETE,
    QUERY_OP_DROP,
    QUERY_OP_EXECUTE,
    QUERY_OP_EXPLAIN,
    QUERY_OP_GRANT,
    QUERY_OP_INSERT,
    QUERY_OP_LOAD_LOCAL,
    QUERY_OP_LOAD,
    QUERY_OP_REVOKE,
    QUERY_OP_SELECT,
    QUERY_OP_SHOW,
    QUERY_OP_TRUNCATE,
    QUERY_OP_UPDATE,
} qc_query_op_t;
```

而query类型的获取通过`qc_query_type_t qc_get_type(GWBUF *buffer)`,它的qc_sqlite实现如下所示:
```c++
// 通过获取当前线程的QcSqliteInfo的内容，这个隐式包括了一个parse_query的过程
// 进而通过get函数获取操作类型
static int32_t qc_sqlite_get_type_mask(GWBUF* pStmt, uint32_t* pType_mask)
{
    QC_TRACE();
    int32_t rv = QC_RESULT_ERROR;
    ss_dassert(this_unit.initialized);
    ss_dassert(this_thread.initialized);

    *pType_mask = QUERY_TYPE_UNKNOWN;
    QcSqliteInfo* pInfo = QcSqliteInfo::get(pStmt, QC_COLLECT_ESSENTIALS);

    if (pInfo)
    {
        if (pInfo->get_type_mask(pType_mask))
        {
            rv = QC_RESULT_OK;
        }
        else if (MXS_LOG_PRIORITY_IS_ENABLED(LOG_INFO))
        {
            log_invalid_data(pStmt, "cannot report query type");
        }
    }
    else
    {
        MXS_ERROR("The query could not be parsed. Response not valid.");
    }

    return rv;
}
```

query操作的获取通过`qc_query_type_t qc_get_type(GWBUF *buffer)`,它的qc_sqlite实现如下所示:
```c++
static int32_t qc_sqlite_get_operation(GWBUF* pStmt, int32_t* pOp)
{
    QC_TRACE();
    int32_t rv = QC_RESULT_ERROR;
    ss_dassert(this_unit.initialized);
    ss_dassert(this_thread.initialized);

    *pOp = QUERY_OP_UNDEFINED;
    QcSqliteInfo* pInfo = QcSqliteInfo::get(pStmt, QC_COLLECT_ESSENTIALS);

    if (pInfo)
    {
        if (pInfo->get_operation(pOp))
        {
            rv = QC_RESULT_OK;
        }
        else if (MXS_LOG_PRIORITY_IS_ENABLED(LOG_INFO))
        {
            log_invalid_data(pStmt, "cannot report query operation");
        }
    }
    else
    {
        MXS_ERROR("The query could not be parsed. Response not valid.");
    }

    return rv;
}
```

## Query Classifier新添加语法

**TODO**